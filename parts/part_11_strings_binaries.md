# Part 11: Strings and Binaries — การจัดการข้อความ

## สารบัญ
1. [String ใน Erlang](#string-ใน-erlang)
2. [List Strings](#list-strings)
3. [Binary Strings](#binary-strings)
4. [string Module](#string-module)
5. [binary Module](#binary-module)
6. [Unicode](#unicode)
7. [การแปลง String/Binary](#การแปลง-stringbinary)
8. [IOLists](#iolists)
9. [Regular Expressions](#regular-expressions)
10. [ตัวอย่างจริง: Text Processing](#ตัวอย่างจริง-text-processing)

---

## String ใน Erlang

Erlang มี string อยู่ 2 รูปแบบ:

```erlang
%% 1. List String (ดั้งเดิม)
Str1 = "Hello, World!".
%% เป็น list of integers: [72,101,108,108,111,44,32,87,111,114,108,100,33]
io:format("~s~n", ["Hello"]).       %% = Hello
io:format("~w~n", ["Hello"]).       %% = [72,101,108,108,111]

%% 2. Binary String (แนะนำสำหรับโค้ดใหม่)
Bin1 = <<"Hello, World!">>.
%% เป็น compact binary sequence
io:format("~s~n", [<<"Hello">>]).   %% = Hello
io:format("~p~n", [<<"Hello">>]).   %% = <<"Hello">>

%% ทำไมต้องใช้ Binary?
%% + ใช้หน่วยความจำน้อยกว่า (8x สำหรับ ASCII)
%% + ประสิทธิภาพดีกว่าสำหรับ I/O, network
%% + รองรับ Unicode ได้ดีกว่า
%% + ใช้ใน OTP, web frameworks (Cowboy, etc.)
```

---

## List Strings

```erlang
%% List string operations
S = "Hello, World!".

%% Length
length(S).           %% = 13
string:length(S).    %% = 13 (unicode-aware)

%% Access characters
hd(S).               %% = 72 (H)
lists:nth(1, S).     %% = 72 (H) - 1-indexed

%% เปรียบเทียบ
"hello" == "hello".  %% = true
"hello" < "world".   %% = true (lexicographic)

%% Pattern matching
[$H | Rest] = "Hello".
Rest.  %% = "ello"

[First, Second | _] = "Hello".
First.   %% = 72 (H)
Second.  %% = 101 (e)

%% Append (ช้า O(n))
"Hello" ++ " World".    %% = "Hello World"
lists:append("Hello", " World").  %% = "Hello World"

%% Concat multiple
string:concat("Hello", " World").  %% = "Hello World"

%% Reverse
lists:reverse("Hello").   %% = "olleH"
string:reverse("Hello").  %% = "olleH"

%% Contains
string:str("Hello World", "World").   %% = 7 (position, 1-based)
string:str("Hello World", "xyz").     %% = 0 (not found)

%% Substring
string:substr("Hello World", 7).         %% = "World"
string:substr("Hello World", 1, 5).      %% = "Hello"
string:sub_string("Hello World", 7, 11). %% = "World"
```

---

## Binary Strings

```erlang
%% Binary literals
B1 = <<"Hello">>.
B2 = <<"World">>.

%% Concatenation (<<>>)
<<B1/binary, " ", B2/binary>>.
%% = <<"Hello World">>

%% Or use binary:list_to_bin
iolist_to_binary([B1, " ", B2]).
%% = <<"Hello World">>

%% Pattern matching
<<"Hello", Rest/binary>> = <<"Hello, World!">>.
Rest.  %% = <<", World!">>

<<First:8, _/binary>> = <<"Hello">>.
First.  %% = 72 (H)

%% Size
byte_size(<<"Hello">>).      %% = 5
bit_size(<<"Hello">>).       %% = 40
string:length(<<"Hello">>).  %% = 5 (unicode-aware)

%% Binary comprehension
<< <<X+32>> || <<X>> <= <<"HELLO">>, X >= $A, X =< $Z >>.
%% = <<"hello">> (lowercase)

%% Splitting with pattern matching
split_at_space(<<Prefix:5/binary, " ", Rest/binary>>) ->
    {Prefix, Rest};
split_at_space(Other) ->
    {Other, <<>>}.

split_at_space(<<"Hello World">>).
%% = {<<"Hello">>, <<"World">>}

%% Checking prefix/suffix
binary:match(<<"Hello World">>, <<"Hello">>).
%% = {0, 5} (position 0, length 5)
binary:match(<<"Hello World">>, <<"xyz">>).
%% = nomatch

%% เข้าถึงแต่ละ byte
B = <<"ABC">>.
[H|_] = binary_to_list(B).
H.  %% = 65 (A)
```

---

## string Module

```erlang
%% string module (OTP 20+ ทำงานกับทั้ง list และ binary)

S = "  Hello, World!  ".

%% Trim
string:trim(S).           %% = "Hello, World!"
string:trim(S, leading).  %% = "Hello, World!  "
string:trim(S, trailing). %% = "  Hello, World!"

%% Uppercase / Lowercase
string:uppercase("hello").  %% = "HELLO"
string:lowercase("HELLO").  %% = "hello"

%% Binary versions work too
string:uppercase(<<"hello">>).  %% = <<"HELLO">>
string:lowercase(<<"HELLO">>).  %% = <<"hello">>

%% Split
string:split("a,b,c,d", ",").         %% = ["a", "b,c,d"] (first only)
string:split("a,b,c,d", ",", all).    %% = ["a", "b", "c", "d"]
string:split("a,b,c,d", ",", leading). %% = ["a", "b,c,d"]

%% Binary split
string:split(<<"a,b,c">>, <<",">>, all).
%% = [<<"a">>, <<"b">>, <<"c">>]

%% Join
string:join(["hello", "world"], " ").   %% = "hello world"
string:join(["a", "b", "c"], ", ").     %% = "a, b, c"

%% Find
string:find("Hello, World", "World").   %% = "World" (rest of string)
string:find("Hello, World", "xyz").     %% = nomatch

%% Replace (OTP 20+)
string:replace("Hello World", "World", "Erlang").
%% = ["Hello ", "Erlang"] (iolist!)
iolist_to_binary(string:replace("Hello World", "World", "Erlang")).
%% = <<"Hello Erlang">>

string:replace("aababab", "ab", "X", all).
%% = ["a", "X", "X", "X", ""] (iolist)

%% Contains check
string:find("Hello World", "World") /= nomatch.   %% = true
string:find("Hello World", "xyz") /= nomatch.     %% = false

%% Slice
string:slice("Hello World", 6).         %% = "World"
string:slice("Hello World", 6, 5).      %% = "World"
string:slice("Hello World", 0, 5).      %% = "Hello"

%% to_integer / to_float
string:to_integer("42").     %% = {42, ""}
string:to_integer("42abc").  %% = {42, "abc"}
string:to_integer("abc").    %% = {error, no_integer}

string:to_float("3.14").     %% = {3.14, ""}
string:to_float("3.14abc").  %% = {3.14, "abc"}

%% Format number
string:pad("42", 5).          %% = "   42"
string:pad("42", 5, leading). %% = "   42"
string:pad("42", 5, trailing). %% = "42   "
string:pad("42", 5, both).    %% = " 42  "
```

---

## binary Module

```erlang
%% binary module สำหรับ binary operations

B = <<"Hello, World!">>.

%% Search
binary:match(B, <<"World">>).
%% = {7, 5} ({Start, Length})

binary:match(B, [<<"Hello">>, <<"World">>]).
%% = {0, 5} (first match)

binary:matches(B, <<"l">>).
%% = [{2,1}, {3,1}, {10,1}] (all positions)

%% Split
binary:split(<<"a,b,c">>, <<",">>).
%% = [<<"a">>, <<"b,c">>]

binary:split(<<"a,b,c">>, <<",">>, [global]).
%% = [<<"a">>, <<"b">>, <<"c">>]

binary:split(<<"a,,b">>, <<",">>, [global, trim_all]).
%% = [<<"a">>, <<"b">>]

%% Replace
binary:replace(<<"Hello World">>, <<"World">>, <<"Erlang">>).
%% = <<"Hello Erlang">>

binary:replace(<<"aababab">>, <<"ab">>, <<"X">>, [global]).
%% = <<"aXXX">>

%% Part (slice)
binary:part(<<"Hello World">>, 6, 5).
%% = <<"World">> (start, length)

binary:part(<<"Hello World">>, {6, 5}).
%% = <<"World">>

%% Copy (creates unique binary)
binary:copy(<<"Hello">>).
%% = <<"Hello">>

binary:copy(<<"ab">>, 3).
%% = <<"ababab">>

%% Encode/Decode integers
binary:encode_unsigned(255).
%% = <<255>>
binary:encode_unsigned(256).
%% = <<1, 0>>

binary:decode_unsigned(<<255>>).   %% = 255
binary:decode_unsigned(<<1, 0>>).  %% = 256

%% Hex
binary:encode_hex(<<"hello">>).
%% = <<"68656C6C6F">>
binary:encode_hex(<<"hello">>, lowercase).  %% OTP 27+
%% = <<"68656c6c6f">>

binary:decode_hex(<<"68656C6C6F">>).
%% = <<"hello">>
```

---

## Unicode

```erlang
%% Unicode support

%% UTF-8 binary
Thai = <<"สวัสดี">>.
byte_size(Thai).     %% = 18 (UTF-8 bytes)
string:length(Thai). %% = 6 (characters)

%% Unicode codepoints
unicode:characters_to_list(<<"สวัสดี">>, utf8).
%% = [3626,3623,3633,3626,3604,3637] (codepoints)

unicode:characters_to_binary("สวัสดี").
%% = <<"สวัสดี">> (UTF-8)

unicode:characters_to_binary("สวัสดี", unicode, utf16).
%% = <<...>> (UTF-16)

%% Character in binary literal
B = <<$A, $B, $C>>.         %% = <<"ABC">>
B2 = <<16#41, 16#42, 16#43>>.  %% = <<"ABC">>

%% Unicode literal
B3 = <<16#0E2A/utf8>>.  %% = <<"ส">> (ส = U+0E2A)
io:format("~ts~n", [B3]).  %% ใช้ ~ts สำหรับ unicode output

%% Normalize (OTP 20+)
string:normalize(<<"café">>, nfc).  %% NFC normalization
string:normalize(<<"café">>, nfd).  %% NFD normalization

%% Case conversion Unicode-aware
string:uppercase(<<"hello café"/utf8>>).
%% = <<"HELLO CAFÉ">>

%% io:format Unicode format
io:format("~ts~n", [<<"สวัสดีชาวโลก">>]).
io:format("~tc~n", [16#0E2A]).  %% ส
```

---

## การแปลง String/Binary

```erlang
%% List <-> Binary
list_to_binary("Hello").        %% = <<"Hello">>
binary_to_list(<<"Hello">>).    %% = "Hello" (list of ints)

%% Integer <-> String
integer_to_list(42).            %% = "42"
list_to_integer("42").          %% = 42
integer_to_binary(42).          %% = <<"42">>
binary_to_integer(<<"42">>).    %% = 42

integer_to_list(255, 16).       %% = "FF" (hex)
list_to_integer("FF", 16).      %% = 255

integer_to_binary(255, 16).     %% = <<"FF">>
binary_to_integer(<<"FF">>, 16). %% = 255

%% Float <-> String  
float_to_list(3.14).            %% = "3.1400000000000001243..." (ugly)
float_to_list(3.14, [{decimals, 2}]).  %% = "3.14"
float_to_list(3.14, [compact]).        %% = "3.14"
float_to_list(3.14, [{decimals, 2}, compact]).  %% = "3.14"

list_to_float("3.14").          %% = 3.14

float_to_binary(3.14, [{decimals, 2}]).   %% = <<"3.14">>
binary_to_float(<<"3.14">>).    %% = 3.14

%% Atom <-> String
atom_to_list(hello).            %% = "hello"
list_to_atom("hello").          %% = hello (creates atom!)
atom_to_binary(hello).          %% = <<"hello">>
binary_to_atom(<<"hello">>).    %% = hello (creates atom!)

%% Safe atom conversion (ไม่สร้าง atom ใหม่)
list_to_existing_atom("hello").  %% = hello (ถ้ามีอยู่แล้ว)
binary_to_existing_atom(<<"hello">>).  %% = hello

%% Number to formatted string
io_lib:format("~.2f", [3.14159]).   %% = "3.14"
iolist_to_binary(io_lib:format("~p", [#{a => 1}])).
%% = <<"#{a => 1}">>

%% erlang:term_to_binary / binary_to_term (serialization)
Data = #{name => <<"Alice">>, age => 30}.
Bin = term_to_binary(Data).
term_to_binary(Data) == term_to_binary(Data).  %% = true

binary_to_term(Bin).  %% = #{age => 30, name => <<"Alice">>}
```

---

## IOLists

```erlang
%% IOList = nested list of: integers (0-255), binaries, or other iolists
%% ใช้สำหรับ efficient string building โดยไม่ต้อง copy

%% iolist_to_binary รวมทุกอย่างเป็น binary
iolist_to_binary(["Hello", ", ", <<"World">>, "!"]).
%% = <<"Hello, World!">>

%% ซ้อนได้ลึก
Deep = ["outer", [" ", ["inner", [" ", <<"deep">>]]]].
iolist_to_binary(Deep).
%% = <<"outer inner deep">>

%% ใช้ใน string building (ไม่ต้อง copy)
build_html(Title, Body) ->
    [<<"<html><head><title>">>,
     Title,
     <<"</title></head><body>">>,
     Body,
     <<"</body></html>">>].

Html = build_html(<<"Hello">>, <<"<p>World</p>">>).
iolist_to_binary(Html).
%% = <<"<html><head><title>Hello</title><head><body><p>World</p></body></html>">>

%% iolist_size
iolist_size(["hello", " ", "world"]).  %% = 11

%% เปรียบเทียบ efficiency
%% BAD: ++ ทำให้ copy ทุกครั้ง O(n^2)
build_bad(Parts) ->
    lists:foldl(fun(P, Acc) -> Acc ++ P end, "", Parts).

%% GOOD: IOList ไม่ copy O(1) ต่อ element
build_good(Parts) ->
    iolist_to_binary(Parts).

%% ตัวอย่าง: build JSON string
json_string(Key, Value) ->
    [<<"\"">>, Key, <<"\": \"">>, Value, <<"\"">>].

json_object(Pairs) ->
    Joined = lists:join(<<", ">>, [json_string(K, V) || {K, V} <- Pairs]),
    [<<"{">>, Joined, <<"}">>].

iolist_to_binary(json_object([{"name", "Alice"}, {"city", "Bangkok"}])).
%% = <<"{\"name\": \"Alice\", \"city\": \"Bangkok\"}">>
```

---

## Regular Expressions

```erlang
%% re module ใช้ PCRE

%% Match
re:run("Hello, World!", "World").
%% = {match, [{7, 5}]}  {start, length}

re:run("Hello!", "World").
%% = nomatch

%% Named captures
re:run("2024-01-15", "(?P<year>\\d{4})-(?P<month>\\d{2})-(?P<day>\\d{2})",
       [{capture, all_names, binary}]).
%% = {match, [<<"2024">>, <<"15">>, <<"01">>]}  (alphabetical: day, month, year)

%% All captures
re:run("2024-01-15", "(\\d{4})-(\\d{2})-(\\d{2})",
       [{capture, all, binary}]).
%% = {match, [<<"2024-01-15">>, <<"2024">>, <<"01">>, <<"15">>]}

%% Replace
re:replace("Hello World", "World", "Erlang").
%% = <<"Hello Erlang">>

re:replace("aababab", "ab", "X", [global]).
%% = <<"aXXX">>

%% Replace with function
re:replace("hello world", "\\b(\\w)",
           fun([Match]) -> string:uppercase(Match) end,
           [global, {return, binary}]).
%% ไม่รองรับ function replacement โดยตรง ต้องทำเอง

%% Split
re:split("a,b,,c", ",").
%% = [<<"a">>, <<"b">>, <<>>, <<"c">>]

re:split("a,b,,c", ",", [{return, list}, trim]).
%% = ["a", "b", "c"]

%% Compile regex for reuse (ประสิทธิภาพดีกว่า)
{ok, MP} = re:compile("(\\d+)-(\\d+)-(\\d+)").
re:run("2024-01-15", MP, [{capture, all, binary}]).
%% = {match, [<<"2024-01-15">>, <<"2024">>, <<"01">>, <<"15">>]}

%% ตัวอย่าง: validate email
valid_email(Email) ->
    case re:run(Email, "^[a-zA-Z0-9._%+\\-]+@[a-zA-Z0-9.\\-]+\\.[a-zA-Z]{2,}$") of
        {match, _} -> true;
        nomatch -> false
    end.

valid_email("alice@example.com").   %% = true
valid_email("not-an-email").        %% = false

%% ตัวอย่าง: extract all numbers from text
extract_numbers(Text) ->
    case re:run(Text, "\\d+", [global, {capture, all, list}]) of
        {match, Matches} -> [list_to_integer(M) || [M] <- Matches];
        nomatch -> []
    end.

extract_numbers("abc 42 def 100 ghi 7").
%% = [42, 100, 7]
```

---

## ตัวอย่างจริง: Text Processing

```erlang
%% ไฟล์: text_utils.erl
-module(text_utils).
-export([
    slugify/1,
    truncate/2, truncate/3,
    word_wrap/2,
    highlight/3,
    parse_key_value/1,
    format_size/1
]).

%% Convert string to URL-friendly slug
-spec slugify(binary()) -> binary().
slugify(Text) ->
    Lower = string:lowercase(Text),
    NoSpecial = re:replace(Lower, "[^a-z0-9\\s-]", <<"">>, [global, {return, binary}]),
    Dashed = re:replace(NoSpecial, "\\s+", <<"-">>, [global, {return, binary}]),
    Trimmed = string:trim(Dashed, both, "-"),
    Trimmed.

%% Truncate text to N characters
-spec truncate(binary(), pos_integer()) -> binary().
truncate(Text, Max) ->
    truncate(Text, Max, <<"...">>).

-spec truncate(binary(), pos_integer(), binary()) -> binary().
truncate(Text, Max, Suffix) ->
    Len = string:length(Text),
    case Len =< Max of
        true -> Text;
        false ->
            SuffixLen = string:length(Suffix),
            Prefix = string:slice(Text, 0, Max - SuffixLen),
            <<Prefix/binary, Suffix/binary>>
    end.

%% Wrap text at word boundaries
-spec word_wrap(binary(), pos_integer()) -> [binary()].
word_wrap(Text, Width) ->
    Words = binary:split(Text, <<" ">>, [global, trim_all]),
    wrap_words(Words, Width, <<>>, []).

wrap_words([], _, Current, Lines) ->
    lists:reverse(case Current of <<>> -> Lines; _ -> [Current | Lines] end);
wrap_words([Word | Rest], Width, <<>>, Lines) ->
    wrap_words(Rest, Width, Word, Lines);
wrap_words([Word | Rest], Width, Current, Lines) ->
    NewLine = <<Current/binary, " ", Word/binary>>,
    case byte_size(NewLine) > Width of
        true ->
            wrap_words(Rest, Width, Word, [Current | Lines]);
        false ->
            wrap_words(Rest, Width, NewLine, Lines)
    end.

%% Highlight occurrences of Pattern in Text
-spec highlight(binary(), binary(), {binary(), binary()}) -> binary().
highlight(Text, Pattern, {Before, After}) ->
    Replaced = re:replace(Text, re:escape(binary_to_list(Pattern)),
                          <<Before/binary, "&", After/binary>>,
                          [global, {return, binary}]),
    Replaced.

%% Parse "key=value\nkey2=value2" format
-spec parse_key_value(binary()) -> #{binary() => binary()}.
parse_key_value(Text) ->
    Lines = binary:split(Text, [<<"\n">>, <<"\r\n">>], [global, trim_all]),
    lists:foldl(
        fun(Line, Map) ->
            case binary:split(Line, <<"=">>) of
                [Key, Value] ->
                    K = string:trim(Key),
                    V = string:trim(Value),
                    Map#{K => V};
                _ ->
                    Map
            end
        end,
        #{},
        Lines
    ).

%% Format byte size to human-readable
-spec format_size(non_neg_integer()) -> binary().
format_size(Bytes) when Bytes < 1024 ->
    iolist_to_binary(io_lib:format("~B B", [Bytes]));
format_size(Bytes) when Bytes < 1024 * 1024 ->
    iolist_to_binary(io_lib:format("~.1f KB", [Bytes / 1024]));
format_size(Bytes) when Bytes < 1024 * 1024 * 1024 ->
    iolist_to_binary(io_lib:format("~.1f MB", [Bytes / (1024 * 1024)]));
format_size(Bytes) ->
    iolist_to_binary(io_lib:format("~.2f GB", [Bytes / (1024 * 1024 * 1024)])).
```

```erlang
%% ทดสอบ text_utils
text_utils:slugify(<<"Hello, World! This is Erlang">>).
%% = <<"hello-world-this-is-erlang">>

text_utils:truncate(<<"Hello, World!">>, 8).
%% = <<"Hello...">>

text_utils:word_wrap(<<"Hello World this is a long line">>, 15).
%% = [<<"Hello World">>, <<"this is a long">>, <<"line">>]

text_utils:highlight(<<"Hello World">>, <<"World">>, {<<"**">>, <<"**">>}).
%% = <<"Hello **World**">>

text_utils:parse_key_value(<<"name=Alice\nage=30\ncity=Bangkok">>).
%% = #{<<"age">> => <<"30">>, <<"city">> => <<"Bangkok">>, <<"name">> => <<"Alice">>}

text_utils:format_size(1536).      %% = <<"1.5 KB">>
text_utils:format_size(1048576).   %% = <<"1.0 MB">>
text_utils:format_size(1073741824). %% = <<"1.00 GB">>
```

---

## สรุป Part 11

| Operation | List String | Binary String |
|-----------|-------------|---------------|
| Literal | `"hello"` | `<<"hello">>` |
| Length | `length(S)` | `byte_size(B)` / `string:length(B)` |
| Concat | `S1 ++ S2` | `<<S1/binary, S2/binary>>` |
| Pattern | `[$H\|Rest]` | `<<$H, Rest/binary>>` |
| Convert to | `binary_to_list(B)` | `list_to_binary(S)` |
| Trim | `string:trim(S)` | `string:trim(B)` |
| Split | `string:split(S, Sep)` | `binary:split(B, Sep)` |
| Upper | `string:uppercase(S)` | `string:uppercase(B)` |
| Regex | `re:run(S, P)` | `re:run(B, P)` |

**กฎทั่วไป**: ใช้ Binary String สำหรับโค้ดใหม่ทุกกรณีที่ทำได้

---

*[← Part 10: Maps](part_10_maps.md) | [Part 12: Arithmetic and Math →](part_12_arithmetic.md)*
