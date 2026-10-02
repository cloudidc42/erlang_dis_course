# Part 55: Interoperability — Python, Node.js, Java

## สารบัญ
1. [Erlang Port Protocol](#erlang-port-protocol)
2. [Python Integration](#python-integration)
3. [Node.js Integration](#nodejs-integration)
4. [Java Integration กับ JInterface](#java-integration-กับ-jinterface)
5. [REST API Bridge](#rest-api-bridge)
6. [ตัวอย่างจริง: ML Model Integration](#ตัวอย่างจริง-ml-model-integration)

---

## Erlang Port Protocol

```erlang
%% Port: communicate with external processes
%% Protocol: length-prefixed messages (4-byte big-endian length)

-module(port_protocol).
-export([encode/1, decode/1]).

%% Encode: length + data
encode(Data) when is_binary(Data) ->
    Len = byte_size(Data),
    <<Len:32/big, Data/binary>>.

encode(Term) ->
    encode(term_to_binary(Term)).

%% Decode: parse length-prefixed message
decode(<<Len:32/big, Data:Len/binary>>) ->
    {ok, binary_to_term(Data)};
decode(Data) ->
    {incomplete, Data}.

%% Generic port worker
-module(port_worker).
-behaviour(gen_server).
-export([start_link/1, call/2]).
-export([init/1, handle_call/3, handle_info/2]).

start_link(Command) ->
    gen_server:start_link(?MODULE, Command, []).

call(Pid, Request) ->
    gen_server:call(Pid, {call, Request}, 30000).

init(Command) ->
    Port = open_port({spawn_executable, Command}, [
        binary,
        {packet, 4},
        use_stdio,
        exit_status
    ]),
    {ok, #{port => Port, pending => #{}}}.

handle_call({call, Request}, From, #{port := Port, pending := Pending} = State) ->
    Ref = make_ref(),
    Encoded = jsx:encode(Request#{id => term_to_binary(Ref)}),
    port_command(Port, Encoded),
    {noreply, State#{pending := Pending#{Ref => From}}};

handle_info({Port, {data, Data}}, #{port := Port, pending := Pending} = State) ->
    Response = jsx:decode(Data, [return_maps]),
    Ref = binary_to_term(maps:get(<<"id">>, Response)),
    case maps:get(Ref, Pending, undefined) of
        undefined -> {noreply, State};
        From ->
            gen_server:reply(From, {ok, Response}),
            {noreply, State#{pending := maps:remove(Ref, Pending)}}
    end;

handle_info({Port, {exit_status, Code}}, State) ->
    logger:error("Port exited", #{code => Code}),
    {stop, {port_exited, Code}, State}.
```

---

## Python Integration

```python
# python_worker.py: Python side of Erlang port
import sys
import json
import struct
import numpy as np

def read_message():
    header = sys.stdin.buffer.read(4)
    if not header:
        return None
    length = struct.unpack('>I', header)[0]
    data = sys.stdin.buffer.read(length)
    return json.loads(data)

def write_message(msg):
    data = json.dumps(msg).encode('utf-8')
    sys.stdout.buffer.write(struct.pack('>I', len(data)))
    sys.stdout.buffer.write(data)
    sys.stdout.buffer.flush()

def handle_request(req):
    method = req.get('method')
    id_ = req.get('id')
    params = req.get('params', {})
    
    try:
        if method == 'numpy_stats':
            data = params['data']
            arr = np.array(data)
            result = {
                'mean': float(np.mean(arr)),
                'std': float(np.std(arr)),
                'min': float(np.min(arr)),
                'max': float(np.max(arr)),
                'percentiles': {
                    'p50': float(np.percentile(arr, 50)),
                    'p95': float(np.percentile(arr, 95)),
                    'p99': float(np.percentile(arr, 99))
                }
            }
            write_message({'id': id_, 'result': result})
        
        elif method == 'matrix_multiply':
            a = np.array(params['a'])
            b = np.array(params['b'])
            result = np.matmul(a, b).tolist()
            write_message({'id': id_, 'result': result})
            
        else:
            write_message({'id': id_, 'error': f'Unknown method: {method}'})
            
    except Exception as e:
        write_message({'id': id_, 'error': str(e)})

if __name__ == '__main__':
    while True:
        msg = read_message()
        if msg is None:
            break
        handle_request(msg)
```

```erlang
%% Erlang: call Python worker
-module(python_bridge).
-export([start/0, numpy_stats/1, matrix_multiply/2]).

start() ->
    port_worker:start_link("/usr/bin/python3 python_worker.py").

numpy_stats(Data) ->
    Request = #{method => <<"numpy_stats">>, params => #{data => Data}},
    port_worker:call(python_worker, Request).

matrix_multiply(A, B) ->
    Request = #{method => <<"matrix_multiply">>, params => #{a => A, b => B}},
    port_worker:call(python_worker, Request).

%% Example usage:
example() ->
    {ok, Stats} = numpy_stats([1.0, 2.0, 3.0, 4.0, 5.0, 6.0, 7.0, 8.0, 9.0, 10.0]),
    io:format("Stats: ~p~n", [Stats]).
```

---

## Node.js Integration

```javascript
// node_worker.js: Node.js port worker
const { Readable, Writable } = require('stream');

class ErlangProtocol {
    constructor() {
        this.buffer = Buffer.alloc(0);
        process.stdin.on('data', (chunk) => this.onData(chunk));
        process.stdin.on('end', () => process.exit(0));
    }
    
    onData(chunk) {
        this.buffer = Buffer.concat([this.buffer, chunk]);
        this.processBuffer();
    }
    
    processBuffer() {
        while (this.buffer.length >= 4) {
            const length = this.buffer.readUInt32BE(0);
            if (this.buffer.length < 4 + length) break;
            
            const messageData = this.buffer.slice(4, 4 + length);
            this.buffer = this.buffer.slice(4 + length);
            
            try {
                const message = JSON.parse(messageData.toString());
                this.handleMessage(message);
            } catch(e) {
                this.send({ error: `Parse error: ${e.message}` });
            }
        }
    }
    
    send(message) {
        const data = Buffer.from(JSON.stringify(message));
        const header = Buffer.alloc(4);
        header.writeUInt32BE(data.length, 0);
        process.stdout.write(Buffer.concat([header, data]));
    }
    
    async handleMessage(msg) {
        const { id, method, params } = msg;
        try {
            let result;
            switch(method) {
                case 'fetch_url':
                    result = await this.fetchUrl(params.url);
                    break;
                case 'parse_css':
                    result = this.parseCss(params.css);
                    break;
                default:
                    throw new Error(`Unknown method: ${method}`);
            }
            this.send({ id, result });
        } catch(e) {
            this.send({ id, error: e.message });
        }
    }
    
    async fetchUrl(url) {
        const response = await fetch(url);
        return {
            status: response.status,
            headers: Object.fromEntries(response.headers),
            body: await response.text()
        };
    }
    
    parseCss(css) {
        // Use Node.js CSS parser
        return { parsed: true, rules: [] };  // simplified
    }
}

new ErlangProtocol();
```

```erlang
%% node_bridge.erl
-module(node_bridge).
-export([start/0, fetch_url/1]).

start() ->
    port_worker:start_link("node node_worker.js").

fetch_url(Url) ->
    Request = #{method => <<"fetch_url">>, params => #{url => Url}},
    port_worker:call(node_worker, Request).
```

---

## Java Integration กับ JInterface

```java
// JavaServer.java: Java process acting as Erlang node via JInterface
import com.ericsson.otp.erlang.*;

public class JavaServer {
    public static void main(String[] args) throws Exception {
        // Create an Erlang node
        OtpNode node = new OtpNode("java_server", "my_cookie");
        OtpMbox mbox = node.createMbox("java_server");
        
        System.out.println("Java server started as: " + node.node());
        
        while (true) {
            OtpErlangObject msg = mbox.receive();
            if (msg instanceof OtpErlangTuple) {
                OtpErlangTuple tuple = (OtpErlangTuple) msg;
                OtpErlangPid from = (OtpErlangPid) tuple.elementAt(0);
                OtpErlangObject request = tuple.elementAt(1);
                
                OtpErlangObject response = handleRequest(request);
                mbox.send(from, response);
            }
        }
    }
    
    static OtpErlangObject handleRequest(OtpErlangObject request) {
        if (request instanceof OtpErlangTuple) {
            OtpErlangTuple t = (OtpErlangTuple) request;
            String method = ((OtpErlangAtom) t.elementAt(0)).atomValue();
            
            switch (method) {
                case "process":
                    OtpErlangBinary data = (OtpErlangBinary) t.elementAt(1);
                    byte[] result = processData(data.binaryValue());
                    return new OtpErlangBinary(result);
                default:
                    return new OtpErlangAtom("unknown_method");
            }
        }
        return new OtpErlangAtom("error");
    }
    
    static byte[] processData(byte[] input) {
        // Java processing logic
        return input;
    }
}
```

```erlang
%% java_client.erl: call Java node from Erlang
-module(java_client).
-export([call/1]).

call(Request) ->
    JavaNode = 'java_server@localhost',
    case net_adm:ping(JavaNode) of
        pong ->
            {java_server, JavaNode} ! {self(), Request},
            receive
                Response -> {ok, Response}
            after 5000 ->
                {error, timeout}
            end;
        pang ->
            {error, java_node_down}
    end.
```

---

## ตัวอย่างจริง: ML Model Integration

```python
# ml_worker.py: serve ML model via Erlang port

import sys
import json
import struct
import pickle
import numpy as np

# Load pre-trained model
with open('model.pkl', 'rb') as f:
    model = pickle.load(f)

def read_msg():
    header = sys.stdin.buffer.read(4)
    if not header: return None
    length = struct.unpack('>I', header)[0]
    return json.loads(sys.stdin.buffer.read(length))

def write_msg(data):
    encoded = json.dumps(data).encode()
    sys.stdout.buffer.write(struct.pack('>I', len(encoded)) + encoded)
    sys.stdout.buffer.flush()

while True:
    msg = read_msg()
    if not msg: break
    
    id_ = msg['id']
    try:
        if msg['method'] == 'predict':
            features = np.array(msg['params']['features']).reshape(1, -1)
            prediction = model.predict(features)
            proba = model.predict_proba(features)
            write_msg({
                'id': id_,
                'result': {
                    'prediction': int(prediction[0]),
                    'confidence': float(max(proba[0]))
                }
            })
        elif msg['method'] == 'batch_predict':
            features = np.array(msg['params']['features'])
            predictions = model.predict(features)
            write_msg({'id': id_, 'result': predictions.tolist()})
    except Exception as e:
        write_msg({'id': id_, 'error': str(e)})
```

```erlang
%% ml_service.erl: Erlang service wrapping ML model
-module(ml_service).
-behaviour(gen_server).
-export([start_link/0, predict/1, batch_predict/1]).
-export([init/1, handle_call/3]).

start_link() ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, [], []).

predict(Features) ->
    gen_server:call(?MODULE, {predict, Features}, 10000).

batch_predict(FeaturesList) ->
    gen_server:call(?MODULE, {batch_predict, FeaturesList}, 30000).

init(_) ->
    MlWorkerPath = code:priv_dir(my_app) ++ "/ml_worker.py",
    Port = open_port({spawn, "python3 " ++ MlWorkerPath}, [
        binary, {packet, 4}, use_stdio, exit_status
    ]),
    {ok, #{port => Port, pending => #{}, counter => 0}}.

handle_call({predict, Features}, From, #{port := Port, pending := P, counter := C} = State) ->
    Id = integer_to_binary(C),
    Req = jsx:encode(#{id => Id, method => <<"predict">>,
                       params => #{features => Features}}),
    port_command(Port, Req),
    {noreply, State#{pending := P#{Id => From}, counter := C + 1}};

handle_call({batch_predict, FeaturesList}, From, #{port := Port, pending := P, counter := C} = State) ->
    Id = integer_to_binary(C),
    Req = jsx:encode(#{id => Id, method => <<"batch_predict">>,
                       params => #{features => FeaturesList}}),
    port_command(Port, Req),
    {noreply, State#{pending := P#{Id => From}, counter := C + 1}}.
```

---

## สรุป Part 55

| Integration | Library | Use Case |
|-------------|---------|----------|
| Python | Port + JSON | ML, data science, scripting |
| Node.js | Port + JSON | Web scraping, async I/O |
| Java | JInterface | Enterprise systems |
| C/C++ | NIF or Port Driver | High performance |
| .NET | Port + JSON | Windows integrations |
| Any language | Port + JSON | Universal integration |

**Port > NIF**: safer (crash isolation), but slower due to IPC overhead

---

*[← Part 54: Metaprogramming](part_54_metaprogramming.md) | [Part 56: Streaming Data →](part_56_streaming.md)*
