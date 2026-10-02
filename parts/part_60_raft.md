# Part 60: Raft Consensus Algorithm

## สารบัญ
1. [Consensus Problem](#consensus-problem)
2. [Raft Overview](#raft-overview)
3. [Leader Election](#leader-election)
4. [Log Replication](#log-replication)
5. [Safety Properties](#safety)
6. [ตัวอย่างจริง: Raft ใน Erlang](#raft-implementation)

---

## Consensus Problem

```
ปัญหา: distributed nodes ต้องตกลงกันว่า value ใด "ถูกต้อง"
       แม้มีบาง nodes ล้มเหลว

Use cases:
- ไฟล์ database commit log
- Distributed lock
- Configuration management
- Leader election

Raft guarantees:
1. Safety: ถ้า committed ต้องตกลงกัน
2. Liveness: ระบบ progress ได้เสมอ
3. Majority quorum: N/2 + 1 nodes ต้อง alive

Raft roles:
- Follower: default state, listens
- Candidate: requesting votes
- Leader: accepts writes, replicates log
```

---

## Raft Overview

```erlang
%% Raft state machine

-record(raft_state, {
    %% Persistent state (saved to disk)
    current_term = 0,     %% latest term seen
    voted_for = undefined, %% candidateId we voted for in current term
    log = [],             %% log entries [{term, index, command}]
    
    %% Volatile state
    role = follower,      %% follower | candidate | leader
    commit_index = 0,     %% highest log entry known to be committed
    last_applied = 0,     %% highest log entry applied to state machine
    
    %% Leader state (reinitialized after election)
    next_index = #{},     %% for each server: next log index to send
    match_index = #{},    %% for each server: highest replicated index
    
    %% Config
    node_id,
    peers = [],
    election_timeout,
    heartbeat_interval = 150
}).
```

---

## Leader Election

```erlang
-module(raft_election).
-export([start_election/1, request_vote/4, handle_vote_response/3]).

%% Convert to candidate and start election
start_election(State = #raft_state{current_term = Term, node_id = NodeId, peers = Peers}) ->
    NewTerm = Term + 1,
    %% Vote for self
    Votes = 1,
    Timeout = rand:uniform(150) + 150,  %% 150-300ms randomized
    
    %% Persist new term and voted_for
    persist_state(State#raft_state{current_term = NewTerm, voted_for = NodeId}),
    
    %% Request votes from all peers
    LastLog = last_log_info(State#raft_state.log),
    [request_vote_from(Peer, NewTerm, NodeId, LastLog) || Peer <- Peers],
    
    State#raft_state{
        role = candidate,
        current_term = NewTerm,
        voted_for = NodeId
    }.

request_vote_from(Peer, Term, CandidateId, {LastLogIndex, LastLogTerm}) ->
    Peer ! {request_vote, Term, CandidateId, LastLogIndex, LastLogTerm}.

%% Handle incoming vote request
request_vote(State, Term, CandidateId, {LastLogIndex, LastLogTerm}) ->
    #raft_state{current_term = CurrentTerm, voted_for = VotedFor, log = Log} = State,
    
    %% Reject if candidate is out of date
    if Term < CurrentTerm ->
        {reject, State};
    true ->
        NewState = maybe_update_term(State, Term),
        
        %% Grant vote if:
        %% 1. Haven't voted yet (or voted for same candidate)
        %% 2. Candidate log is at least as up-to-date
        CanVote = (VotedFor == undefined orelse VotedFor == CandidateId),
        LogOk = candidate_log_ok(LastLogIndex, LastLogTerm, Log),
        
        case CanVote andalso LogOk of
            true ->
                {grant, NewState#raft_state{voted_for = CandidateId}};
            false ->
                {reject, NewState}
        end
    end.

handle_vote_response(State, _VoterId, {vote_granted, Term}) ->
    #raft_state{current_term = CurrentTerm, role = Role, peers = Peers} = State,
    
    if Term /= CurrentTerm -> State;  %% stale response
    Role /= candidate -> State;       %% no longer candidate
    true ->
        NewVotes = maps:get(votes, State, 1) + 1,
        Majority = length(Peers) div 2 + 1,
        
        if NewVotes >= Majority ->
            %% Won election! Become leader
            become_leader(State);
        true ->
            State
    end
    end;

handle_vote_response(State, _VoterId, {vote_denied, Term}) ->
    maybe_update_term(State, Term).

become_leader(State = #raft_state{peers = Peers, log = Log}) ->
    LastIndex = last_log_index(Log),
    NextIndex = maps:from_list([{P, LastIndex + 1} || P <- Peers]),
    MatchIndex = maps:from_list([{P, 0} || P <- Peers]),
    
    logger:info("Became leader", #{term => State#raft_state.current_term}),
    
    State#raft_state{
        role = leader,
        next_index = NextIndex,
        match_index = MatchIndex
    }.

last_log_info([]) -> {0, 0};
last_log_info(Log) ->
    {_, Index, Term, _} = lists:last(Log),
    {Index, Term}.

candidate_log_ok(LastLogIndex, LastLogTerm, Log) ->
    {MyLastIndex, MyLastTerm} = last_log_info(Log),
    LastLogTerm > MyLastTerm orelse
    (LastLogTerm =:= MyLastTerm andalso LastLogIndex >= MyLastIndex).
```

---

## Log Replication

```erlang
-module(raft_log).
-export([append_entry/3, replicate/2, commit_entries/2]).

%% Leader appends entry and replicates
append_entry(State = #raft_state{role = leader, log = Log, current_term = Term}, Command, From) ->
    %% Create new log entry
    NewIndex = last_log_index(Log) + 1,
    Entry = {entry, NewIndex, Term, Command},
    NewLog = Log ++ [Entry],
    
    %% Store pending reply
    Pending = maps:get(pending, State, #{}),
    NewPending = Pending#{NewIndex => From},
    
    %% Immediately replicate to followers
    NewState = State#raft_state{log = NewLog},
    replicate_to_all(NewState),
    
    NewState#raft_state{pending = NewPending};

append_entry(State, _Command, From) ->
    %% Not leader: redirect to leader
    From ! {error, not_leader, State#raft_state.current_leader},
    State.

%% Send AppendEntries to a follower
replicate_to_peer(State, Peer) ->
    #raft_state{
        current_term = Term,
        node_id = NodeId,
        log = Log,
        commit_index = CommitIndex,
        next_index = NextIndex
    } = State,
    
    PeerNextIndex = maps:get(Peer, NextIndex, 1),
    PrevIndex = PeerNextIndex - 1,
    PrevTerm = get_term_at(Log, PrevIndex),
    Entries = entries_from(Log, PeerNextIndex),
    
    Peer ! {append_entries, Term, NodeId, PrevIndex, PrevTerm, Entries, CommitIndex}.

replicate_to_all(State = #raft_state{peers = Peers}) ->
    [replicate_to_peer(State, Peer) || Peer <- Peers].

%% Handle AppendEntries (follower side)
handle_append_entries(State, LeaderTerm, LeaderId, PrevIndex, PrevTerm, Entries, LeaderCommit) ->
    #raft_state{current_term = CurrentTerm, log = Log} = State,
    
    if LeaderTerm < CurrentTerm ->
        %% Reject: stale leader
        {false, State};
    true ->
        NewState = maybe_update_term(State, LeaderTerm),
        %% Reset election timeout (we heard from leader)
        NewState2 = NewState#raft_state{
            role = follower,
            current_leader = LeaderId
        },
        
        %% Check log consistency
        case log_consistent(Log, PrevIndex, PrevTerm) of
            false ->
                {false, NewState2};
            true ->
                %% Append new entries, truncating conflicting ones
                NewLog = append_new_entries(Log, PrevIndex, Entries),
                
                %% Update commit index
                NewCommit = min(LeaderCommit, last_log_index(NewLog)),
                
                FinalState = NewState2#raft_state{
                    log = NewLog,
                    commit_index = NewCommit
                },
                
                %% Apply committed entries to state machine
                apply_committed(FinalState),
                
                {true, FinalState}
        end
    end.

%% Check if majority replicated → commit
check_commit(State = #raft_state{match_index = MatchIndex, peers = Peers, current_term = Term, log = Log}) ->
    N = last_log_index(Log),
    Majority = length(Peers) div 2 + 1,
    
    %% Find highest N that majority has replicated
    CommittableN = find_committable(N, MatchIndex, Majority, Term, Log),
    
    case CommittableN > State#raft_state.commit_index of
        true ->
            notify_pending_commands(State, CommittableN),
            State#raft_state{commit_index = CommittableN};
        false ->
            State
    end.

find_committable(N, _, _, _, _) when N =< 0 -> 0;
find_committable(N, MatchIndex, Majority, Term, Log) ->
    Replicated = length([M || {_, M} <- maps:to_list(MatchIndex), M >= N]),
    case Replicated >= Majority andalso get_term_at(Log, N) =:= Term of
        true -> N;
        false -> find_committable(N - 1, MatchIndex, Majority, Term, Log)
    end.
```

---

## ตัวอย่างจริง: Raft ใน Erlang

```erlang
%% raft_server.erl: complete Raft gen_server
-module(raft_server).
-behaviour(gen_server).
-export([start_link/2, submit/2, read/1]).
-export([init/1, handle_call/3, handle_cast/2, handle_info/2]).

start_link(NodeId, Peers) ->
    gen_server:start_link({local, raft}, ?MODULE, {NodeId, Peers}, []).

submit(Command, Timeout) ->
    gen_server:call(raft, {submit, Command}, Timeout).

read(Key) ->
    gen_server:call(raft, {read, Key}).

init({NodeId, Peers}) ->
    %% Randomize election timeout to prevent split votes
    Timeout = rand:uniform(150) + 150,
    ElectionRef = schedule_election(Timeout),
    
    State = #raft_state{
        node_id = NodeId,
        peers = Peers,
        election_timeout = Timeout,
        election_ref = ElectionRef,
        state_machine = #{}  %% key-value store
    },
    {ok, State}.

handle_call({submit, Command}, From, State = #raft_state{role = leader}) ->
    NewState = raft_log:append_entry(State, Command, From),
    {noreply, NewState};

handle_call({submit, _}, _From, State = #raft_state{current_leader = Leader}) ->
    {reply, {redirect, Leader}, State};

handle_call({read, Key}, _From, State = #raft_state{state_machine = SM}) ->
    {reply, maps:get(Key, SM, undefined), State};

handle_cast({request_vote, Term, CandidateId, LastLogIndex, LastLogTerm}, State) ->
    {Grant, NewState} = raft_election:request_vote(
        State, Term, CandidateId, {LastLogIndex, LastLogTerm}
    ),
    CandidateId ! {vote_response, Grant, State#raft_state.current_term},
    {noreply, NewState};

handle_cast({vote_response, Granted, Term}, State) ->
    NewState = raft_election:handle_vote_response(State, voted_by, {Granted, Term}),
    {noreply, NewState};

handle_cast({append_entries, Term, LeaderId, PrevIndex, PrevTerm, Entries, Commit}, State) ->
    {Success, NewState} = raft_log:handle_append_entries(
        State, Term, LeaderId, PrevIndex, PrevTerm, Entries, Commit
    ),
    LeaderId ! {append_response, Success, NewState#raft_state.current_term,
                last_log_index(NewState#raft_state.log)},
    reset_election_timer(NewState),
    {noreply, NewState};

handle_info(election_timeout, State = #raft_state{role = Role}) when Role /= leader ->
    NewState = raft_election:start_election(State),
    {noreply, NewState};

handle_info(heartbeat, State = #raft_state{role = leader}) ->
    raft_log:replicate_to_all(State),
    schedule_heartbeat(State#raft_state.heartbeat_interval),
    {noreply, State}.

schedule_election(Timeout) ->
    erlang:send_after(Timeout, self(), election_timeout).

schedule_heartbeat(Interval) ->
    erlang:send_after(Interval, self(), heartbeat).

reset_election_timer(State = #raft_state{election_ref = Ref, election_timeout = Timeout}) ->
    erlang:cancel_timer(Ref),
    NewRef = schedule_election(Timeout + rand:uniform(50)),
    State#raft_state{election_ref = NewRef}.

apply_committed(State = #raft_state{log = Log, commit_index = CI, last_applied = LA, state_machine = SM}) ->
    Entries = [E || {entry, I, _, _} = E <- Log, I > LA, I =< CI],
    NewSM = lists:foldl(fun({entry, _, _, {set, K, V}}, Acc) ->
        Acc#{K => V};
    ({entry, _, _, {delete, K}}, Acc) ->
        maps:remove(K, Acc)
    end, SM, Entries),
    State#raft_state{state_machine = NewSM, last_applied = CI}.
```

---

## สรุป Part 60

| Raft Concept | Implementation |
|--------------|----------------|
| Leader election | Randomized timeout + majority votes |
| Log replication | AppendEntries RPC + majority confirm |
| Safety | Term check + log consistency |
| Commit | N/2+1 majority replicated |
| Failover | Election on timeout |
| Persistence | Log to disk before reply |

**Production alternatives**: Ra (RabbitMQ's Raft), Riak's Paxos, Mnesia quorum

---

*[← Part 59: Large-Scale Design](part_59_large_scale.md) | [Part 61: CRDT Deep Dive →](part_61_crdts.md)*
