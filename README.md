ocamllex
========

Experimental version of ocamllex.

To build, just type ```make``` in an ocaml environment. The new ```ocamllex```
is generated in the current directory.

Purpose
=======

Lexers generated by standard ocamllex grab the control flow. Desireable
interruption points are user actions (e.g. when chaining lexing rules) and
refill points.

However interrupting the latter is not achievable without hacks: the refill
function from lexbuf is called, then flow gets back to lexing code.

How to
======

This version of ```ocamllex``` extends the grammar of lexer rule with an
optional "refill" handler.

    rule rule_name rule_arguments = refill {refill_action} parse parse_actions

The ```refill_action``` should be an expression of type:

    (rule_arguments -> Lexing.lexbuf -> 'b -> result_type) ->
        rule_arguments -> Lexing.lexbuf -> 'b -> result_type

When refilling, the ```refill_action``` gets called, receiving the
continuation, rule_arguments, lexing buffer and a lexer determined value (the
state).

From this point, you can do whatever you want: you have full control of
the flow -- interrupt lexing and store the state for latter use, change
lexbuf, etc.

Examples
========

#### Lexing arithmetic in an arbitrary monad

```ocaml
{
type token = EOL | INT of int | PLUS | MINUS | TIMES | DIV | LPAREN | RPAREN

module Make (M : sig
               type 'a t
               val return: 'a -> 'a t
               val bind: 'a t -> ('a -> 'b t) -> 'b t
               val fail : string -> 'a t

               (* Set up lexbuf *)
               val on_refill : Lexing.lexbuf -> unit t
             end)
= struct

let refill_handler k lexbuf arg =
    M.bind (M.on_refill lexbuf) (fun () -> k lexbuf arg)

}
rule token = refill {refill_handler} parse
| [' ' '\t']
    { token lexbuf }
| '\n'
    { M.return EOL }
| ['0'-'9']+ as i
    { M.return (INT (int_of_string i)) }
| '+'
    { M.return PLUS }
| '-'
    { M.return MINUS }
| '*'
  { M.return TIMES }
| '/'
    { M.return DIV }
| '('
    { M.return LPAREN }
| ')'
    { M.return RPAREN }
| eof
    { M.fail "EOF" }
| _
    { M.fail "unexpected character" }
{
end
}
```

#### Using ```Lwt_io```

```ocaml
{
type token = EOL | INT of int | PLUS | MINUS | TIMES | DIV | LPAREN | RPAREN

type state = {
  ic: Lwt_io.input_channel;
  buf: string;
  mutable pos: int;
  mutable len: int;
}

let refill_handler k st lexbuf arg =
    if st.pos >= st.len
    then
      Lwt.bind (Lwt_io.read_into st.ic st.buf 0 (String.length st.buf))
        (fun n ->
           st.pos <- 0;
           st.len <- n;
           k st lexbuf arg)
    else
      k st lexbuf arg

}

rule token state = refill {refill_handler} parse
| [' ' '\t']
  { token state lexbuf }
| '\n'
  { Lwt.return EOL }
| ['0'-'9']+ as i
  { Lwt.return (INT (int_of_string i)) }
| '+'
  { Lwt.return PLUS }
| '-'
  { Lwt.return MINUS }
| '*'
  { Lwt.return TIMES }
| '/'
  { Lwt.return DIV }
| '('
  { Lwt.return LPAREN }
| ')'
  { Lwt.return RPAREN }
| eof
  { Lwt.fail (Failure "EOF") }
| _
  { Lwt.fail (Failure "unexpected character") }

{
let from_channel ic : state * Lexing.lexbuf =
  let buf = String.create 1024 in
  let state = { ic; buf; pos = 0; len = 0 } in
  state,
  Lexing.from_function (fun buf' n ->
      let available = state.len - state.pos in
      if available <= 0
      then 0
      else let n' = min available n in
        String.blit state.buf state.pos buf' 0 n';
        state.pos <- state.pos + n'; n')

}
```
