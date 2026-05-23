# Riven compiler issues found while building Rondo

A running log of compiler gaps + parser quirks encountered while
building Rondo (a Sinatra-style web framework on Riven). Each entry
has: a minimal reproducer, the fix or current workaround, and
whether the upstream Riven repo has it patched.

Some of these I fixed in-place and committed upstream during this
session; the rest are documented workarounds that Rondo applies
today and need real fixes later.

Date: 2026-05-23. Riven branch: `v1-missing-features`.

---

## Fixed upstream (this session)

### F1. `def var <method> -> T; … self` returned garbage

**Symptom:** `r = r.with_status(201)` on a builder that returns
`self` from a `def var` method read back `r.status == 4` (or
`6788`, depending on heap reuse) instead of `201`. The mutation
itself was correct; the return value was wrong.

**Root cause:** `mir/lower/expr/assign.rs:42-48` unconditionally
inserted `riven_dealloc(R)` before `Assign { dest: R, value: Use(X) }`
when `R` was a heap-owned local being rebound. When `X` came from
a call returning `self`, `X` aliased `R` — the dealloc freed the
very heap object the assign re-read. Use-after-free.

**Fix:** new post-lowering pass
`drops::elide_returns_self_realloc` (riven commit `0a12b18`).
Identifies functions returning their first param (`Return(Some(Use(params[0])))`
on every returning block), walks every block looking for the
buggy triple `Call X = F(R,…) ; riven_dealloc(R) ; Assign R = X`
where `F` is in the returns-self set, and removes the dealloc.

### F2. Match-arm pattern bindings had `Ty::Infer` instead of payload type

**Symptom:** `match opt; Some(t) -> t.describe(); ...` with
`opt: Option[Thing]` link-failed with
"no runtime symbol for `?T518_describe`". Pattern bindings used in
interpolation printed as integers (`hi 54303703232`).

**Root cause:** `typeck/infer/expr.rs::HirExprKind::Match`
inferred the scrutinee then walked arm bodies without touching
pattern bindings. The bindings allocated DefIds at resolve time
got their types updated nowhere.

**Fix:** new `helpers::propagate_pattern_types` walker that
structurally matches the pattern against the scrutinee's resolved
type and writes each `HirPattern::Binding`'s DefId-type back into
the symbol table BEFORE the arm body is inferred. Handles built-in
`Ty::Option`/`Ty::Result` (peels payload directly) and user-
defined enums (looks up the variant by `variant_idx`, reads
payload tys from `DefKind::EnumVariant.kind`, substitutes generic
params using the scrutinee's `generic_args`). Tuple / Struct /
Or / Ref / Wildcard / Rest also handled. Riven commit `9c7028b`.

### F3. Nested multi-statement match arms didn't parse

**Symptom:**

```riven
match a
  Some(x) ->
    match b
      Some(y) ->
        let z = x + y
        puts "z=#{z}"
      None -> puts "no b"
    end
  None -> puts "no a"
end
```

produced "expected expression, found Arrow" at the inner `None ->`.

**Root cause:** `parser/methods.rs::parse_body` only terminated
on `End`/`Else`/`Elsif`/`Eof`. Inside a nested match, the inner
arm body greedily consumed the outer arm's pattern as a
statement.

**Fix:** new `parse_match_arm_body` variant called from
`parse_match_arm` that adds "next tokens look like a sibling arm
header" as a termination signal. `looks_like_sibling_match_arm`
peeks up to 16 tokens for `->` at the current bracket depth,
hard-stopping at any statement-starting keyword
(`let`/`var`/`if`/`while`/`for`/`match`/`return`/`break`/
`continue`), newline at depth 0, or terminator. Riven commit
`0257989`.

### F4. Option's `Some` variant_idx off-by-one in MIR

**Symptom:** Even with F2 in place, `match Some(s); Some(n) -> "hi #{n}"`
printed `n` as a pointer-valued integer instead of the captured
string. Pattern bindings looked correct in typeck but MIR
allocated the binding's local with `Ty::Int`.

**Root cause:** `mir/lower/match_arms.rs:148` matched on
`variant_idx == 0` for `Some(T)`, but the resolver registers
Option as `None=0, Some=1` (`resolve/stdlib/option.rs:43-65`).
So `Some(x)` matched the `variant_idx == 0` arm and got an empty
`variant_field_types`, which fell back to `Ty::Int`.

**Fix:** flip the variant-idx guard so `Some(T)` matches
`variant_idx == 1`. Result's arms were already correct. Riven
commit `80acb14`.

---

## Workarounds in current Rondo (need real upstream fixes)

### W1. Multi-statement match arm bodies need a statement keyword first

**Symptom:** A multi-statement arm body whose first token is an
expression-start (identifier, method call, `if`, assignment)
parses as a SINGLE expression — subsequent lines are not part of
the body.

```riven
Some(params) ->
  matched_idx = i         # ← consumed as the whole arm body
  matched_params = params # ← parser thinks this starts the next arm
None -> 0                 # ← "expected expression, found Arrow"
```

**Workaround in Rondo:** lead the body with a `let _foo = …`
binding to force block parsing:

```riven
Some(params) ->
  let _captured = params
  matched_idx = i
  matched_params = _captured
```

(see `Rondo.dispatch` in `src/lib.rvn` for the live example).

**Proper fix:** `parser::is_expression_start` is consulted in
`parse_match_arm` to choose Expr-vs-Block. The single-vs-block
distinction shouldn't live there — instead, parse arm bodies as
"a block whose terminator is `<next arm header>` OR `end`". The
sibling-arm-header heuristic I added for F3 should drive the
block-mode choice unconditionally; if the heuristic finds a
sibling, that's a block; otherwise we have one or two statements
ending in an expression.

### W2. `if/else` arm body drops subsequent statements

**Symptom:** Same root as W1 — `if` returns true from
`is_expression_start`, so an arm body starting with `if` is parsed
as a single if-expression and any statements after `end` are
mis-attributed.

**Workaround:** Extract the multi-statement body into a free fn,
keep the arm body as a single call.

### W3. `()` doesn't reliably parse as an expression

**Symptom:** Used as a no-op match arm body (`None -> ()`),
the `()` doesn't always parse — Riven seems to lack the unit
literal as a standalone expression in some positions.

**Workaround:** Use `0` (or any other expression of compatible
type) for no-op arms. The match's value is discarded by the
caller anyway.

### W4. Closure bodies are single-expression only

**Symptom:** `{ |req: Request| let x = …; let y = …; … }` — only
the first statement parses as the body; the rest is dropped or
reported as syntax error.

**Workaround:** Extract closure bodies into free fns and call
them from the closure:

```riven
app.get(&"/hello/:name", { |req: Request|
  Response.text(&hello_body(&req))   # helper does the work
})
```

This is actually fine for Sinatra-style ergonomics — Sinatra
encourages thin routes — but it'd be nicer to support
multi-statement closure bodies via a block delimiter.

### W5. Generic-method dispatch fails when receiver type is `?T`

**Symptom:** `for route in self.routes.iter` followed by
`route.matches(...)` link-fails with "no runtime symbol for
`?T100_matches`". The iterator's element type isn't carried into
the method-call lowering.

**Workaround:** Indexed loop via `routes.get(i)` + pattern-match
on the returned `Option[&Route]`. Combined with F2 (pattern type
propagation), the binding's type resolves cleanly.

```riven
let n: Int = self.routes.len() as Int
var i: Int = 0
while i < n
  match self.routes.get(i)
    Some(route) ->
      match route.matches(&req)
        ...
```

**Proper fix:** Monomorphization should propagate the array's
element type into iterator-element method dispatch. Tracked in
project memory as the "generic stdlib types lose T in MIR" gap.

### W6. `Option.unwrap()` / `Option.is_some()` etc. link-fail on generics

**Symptom:** `opt.unwrap()` where `opt: Option[String]` link-fails
with `Option[String]_unwrap` — codegen mangles the parameterised
type name literally and no monomorphized symbol exists.

**Workaround:** Pattern-match exclusively.

```riven
let v = match opt
  Some(x) -> x
  None    -> default
end
```

### W7. `BufReader.Tcp.read_line` dotted-class FFI alias doesn't resolve

**Symptom:** Calling `BufReader.Tcp.new(stream).read_line` link-
fails with undefined symbols `BufReader.Tcp_read_line`,
`BufReader.Tcp_into_inner`, `BufWriter.Tcp_flush`. The FFI alias
clause in stdlib (`def read_line as "riven_bufreader_read_line"`)
should rewrite the mangled callee, but the dotted-module class
name (`BufReader.Tcp_…`) escapes the rewriter.

**Workaround in Rondo:** Don't use `BufReader.Tcp`. The TCP
server in `Rondo.listen` reads NOTHING from incoming connections
(writes a canned response and closes). Request parsing over a
real socket lands once this alias gap closes.

### W8. No bytes→String conversion in stdlib

**Symptom:** `TcpStream.read(&var Array[Int])` reads bytes into
an Array[Int], but there's no `String.from_bytes(&Array[Int])`
or similar to turn those bytes into a String we can feed to
`Request.parse`.

**Workaround in Rondo:** Same as W7 — skip request parsing over
sockets for now.

**Proper fix:** Either expose `riven_string_from_utf8(bytes)`
through stdlib or add a `TcpStream.read_to_string` that delegates
to the existing `riven_stream_read_to_string` runtime helper
(currently only used by `File` and `Stdin`).

### W9. `derive Clone` (and other derive keywords) silently ignored

**Symptom:** `class Request; derive Clone; …` errors with
"expected Colon, found TypeIdentifier" — the parser doesn't
recognise `derive` as a body keyword for user classes. Looking
at `parser/classes.rs`, `derive_traits` is always initialised to
`Vec::new()` and never populated.

**Workaround in Rondo:** Avoid needing Clone — Rondo's
before-hooks take `&Request` not `Request`, so no copy is needed.

### W10. Forward references in helper signatures fail

**Symptom:** A free `def foo(x: &Request)` declared BEFORE
`class Request` errors with "undefined type `Request`". Forward
references work in fn bodies (resolved at the second pass) but
not in parameter / return-type positions.

**Workaround in Rondo:** All helper fns whose signatures
reference user types (`Request`, `Response`, `Route`, `Rondo`,
`TcpStream`) live at the END of `src/lib.rvn`, after the class
definitions.

### W11. `String.split("/")` returns SplitIter on `Str` receivers

**Symptom:** Calling `.split(&"/")` on a string literal directly
gives back `Ty::Class { name: "SplitIter" }` instead of
`Array[String]` — and `SplitIter` doesn't expose `.get(i)` or
`.len()`, so indexed iteration link-fails.

**Workaround:** Make sure the receiver is an owned `String`, not
a `&str`. In Rondo's case, fields are `String`-typed and helper
args bind via `&String`, so this is fine on the framework's hot
paths.

### W12. String-literal-to-String coercion in Option/Result payloads

**Symptom:** `Some("ada")` where the declared return type is
`Option[String]` fails typeck with
"expected Option[String], found Option[&str]". String literals
default to `&str` and aren't auto-coerced into the payload type.

**Workaround:** `Some(String.from(&"ada"))` — explicit
construction. Verbose but correct.

### W13. `pub` keyword not supported

**Symptom:** `pub class Foo` / `pub def bar` errors with
"expected top-level declaration, found Identifier(\"pub\")".
Visibility is implicit in Riven; there's no equivalent today.

**Workaround in Rondo:** Just omit `pub`. All items in
`lib.rvn` are accessible to depending packages by default.

### W14. Path-dep symbols are flat-merged, not module-namespaced

**Symptom:** `use rondo.Rondo` from a dependent package fails
with "unknown module 'rondo'". The dep's source IS being merged
into the user's compile (via the `compile_project` flat-merge
fix Codex committed earlier today), but the symbols end up at
the top level rather than namespaced under `rondo`.

**Workaround in Rondo:** No `use` statements in `rondo-smoke`;
`Rondo`, `Request`, `Response` are available at the top level.

**Proper fix:** Wrap each dep's program in a module DefId during
resolve so `use rondo.X` resolves through `Module(rondo) →
Class(X)`.

---

## Summary

Rondo's v0.4 surface (route DSL with five HTTP verbs, path-param
extraction, query-string parsing, request parser, response
serializer, TCP listener+accept+write, before-hooks for
short-circuit auth/logging) **works today** with these
workarounds. Real production-grade serving (request parsing over
a socket, async I/O, halt sentinel, after-hooks) is gated on the
remaining items above — primarily W5 (generic-method dispatch),
W6 (Option.unwrap on generics), W7 (dotted-class FFI alias),
and W8 (bytes→String).

The four upstream fixes I committed (F1-F4) are infrastructure
that benefits every external Riven project, not just Rondo —
each was already a footgun for any user who hit the right shape.
