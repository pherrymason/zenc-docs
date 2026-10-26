# RFC 0001: Explicit string interpolation and printing

- Status: draft
- Target: Zen C language specification and the C backend

## Summary

1. Only `f"..."` interpolates. A plain `"..."` literal is an ordinary string.
2. `print`, `println`, `eprint` and `eprintln` are called like functions:
   `println(f"x = {x}")`. A string literal on its own no longer prints.
3. An `f"..."` used as a value has a defined owner. This RFC lists the options and
   proposes one, but the choice depends on the memory model and can be settled
   separately from parts 1 and 2.

## Motivation

Today every string literal interpolates `{...}`, `println "..."` is a statement with
its own syntax, and a string literal written as a statement is printed. This causes
several problems:

- **Every literal needs `{` escaped**, even when no interpolation is intended.
  `"{"` has to be written `"{{"`.
- **Printing collides with block values.** A string literal as the last expression
  of a block is a print order, not the block's value. Both of these fail in clang today:

  ```zc
  let x = if c { "a" } else { "b" };      // each branch prints its string
  match s { "a" => println "a", _ => {} } // `println` is not an expression
  ```

- **Nobody owns the result of an interpolated value.** `let s = "{x}"` writes into a
  `static char[8192]`, one per f-string in the source rather than one per
  evaluation. Two results of the same f-string share their contents:

  ```zc
  fn greet(name: string) -> string { return "hi {name}"; }

  let a = greet("ann");
  let b = greet("bob");
  println "{a} / {b}";   // prints "hi bob / hi bob"
  ```

  Each interpolated piece also goes through `sprintf` into a `char[2048]` and is
  appended with `strcat`, without checking sizes. A value longer than 2047 characters,
  or a result longer than 8191 characters, overflows the stack buffer.
- **`println` is not a function.** It cannot be passed, searched for as a call, or
  replaced. It accepts only a literal. Its implementation is a single ~830-line
  function (`process_printf_sugar`) that generates C as text, with one branch per value
  type for printing and another for building a value.
- Related reports: interpolation lost with `-O2` (#385), interpolation in `assert`
  messages (#402), a malformed format crashing (#183), escapes and raw or multiline
  strings (#331, #368, #397, #411), a bare string without `;` disappearing (#141, #379),
  and the shared static buffer (#339).

## Current rules

| Form | Today |
|---|---|
| `"text {x}"` | interpolates |
| `f"text {x}"` | interpolates (same as without `f`) |
| `r"text {x}"` | raw, no interpolation |
| `println "text {x}"` | prints, with a newline (statement syntax) |
| `"text {x}";` as a statement | prints, with a newline |
| `"text {x}"` used as a value | builds the text in a static buffer |

## Proposal

### 1. Only `f"..."` interpolates

- `"..."` is an ordinary string literal: `{` and `}` are plain characters.
- `f"..."` interpolates, with the current syntax for expressions and format specifiers
  (`{x}`, `{x:03d}`).
- `r"..."` stays raw.

This removes the `{{` escapes from ordinary strings and makes interpolation visible
where it happens.

### 2. Printing is a call

- `print(...)`, `println(...)`, `eprint(...)` and `eprintln(...)` are written with
  parentheses and take one string argument.
- When the argument is an `f"..."`, the compiler keeps doing what it does today for
  `println "..."`: it writes each piece directly to the stream, with no allocation.
  With any other string expression it prints that string.
- A string literal written as a statement is just an unused expression (a warning),
  not a print order. This frees string literals to be block values.

`std/io.zc` already has `print(fmt, ...)` and `println(fmt, ...)` functions in the
`printf` style. Those could stay under another name or merge with these. That
decision is independent of the rest.

### 3. Who owns an `f"..."` used as a value

Options, roughly matching what `std/io.zc` already offers:

| Option | Precedent | Correct | Leaks | Cost |
|---|---|---|---|---|
| A. A `String` with `Drop` | `std/string.zc` | yes | no | the value is no longer a `string` (`char*`); needs the move/drop rules to be settled |
| B. Into a buffer the caller passes | `format_into` | yes | no | needs syntax for the buffer |
| C. A new heap `char*`, the caller frees | `format_new` | yes | if not freed | silent leaks when nobody frees |
| D. A ring of N thread-local buffers | `format` | while fewer than N results are alive | no | fails silently past N |

Proposed: **A** as the target, because it is the only option that is correct, does
not leak and gives no silent limit. It depends on the memory model (moves, drops and
containers), which is still open, so this part can be decided later without blocking
parts 1 and 2. Until then, the value form should at least size its buffer to the text
(no overflow) and its sharing behaviour should be documented.

## Prior art

- **Zen C itself.** `f"..."` is already part of the language: the README and the
  lexical grammar document it as an "F-String" that forces interpolation
  (`FStringLiteral ::= 'f"' StringChar* '"'`), the lexer gives it its own token
  (`TOK_FSTRING`) and the test suite uses it. Today it means the same as no prefix,
  since every literal interpolates. Part 1 adds no syntax: it makes an existing prefix
  the only way to interpolate.
- **The project's Discord.** Interpolating only with an `f` prefix, "the Python way",
  was suggested there, along with the remark that these should not be called
  f-strings without the prefix. Making `print` a plain call and asking who owns a
  formatted string came up in the same discussions.
- **Python.** `f"..."` interpolates and `"..."` does not (PEP 498, Python 3.6).
  Python 3 also turned the `print` statement into a function (PEP 3105), for reasons
  close to part 2: a statement with its own syntax cannot be passed, replaced or
  extended.
- **C# and Rust.** C# marks interpolation with a prefix too (`$"..."`). Rust
  interpolates only inside explicit macro calls (`format!("{x}")`,
  `println!("{x}")`), so a plain literal never interpolates and printing is always a
  call.
- **Swift and Kotlin** interpolate every literal, but with a two-character marker
  (`"\(x)"`, `"${x}"`) that rarely appears in ordinary text. Zen C interpolates every
  literal with a bare `{`, which ordinary text does contain, hence the `{{` escapes.

## Impact

Counted in the standard library, the test suite and the community examples
(awesome-zenc):

| | std | tests | examples |
|---|---|---|---|
| `print`/`println` without parentheses | 4 | 357 | 1290 |
| `print`/`println` with parentheses | 4 | 5 | 3 |
| string literal alone as a statement (prints) | 0 | ~146 | ~148 |
| lines with an escaped `{{` | 1 | 14 | 5 |
| interpolated strings used as a value | 2 | 45 | 26 |

The last row comes from transpiling every file and counting the static buffers in the
generated C; 9 of those 73 sites already use the `f` prefix. None of them gives a
wrong result today: almost all use the value at once or copy it. The `return` sites
work only because the caller prints the result before the next call.

Almost all of this is mechanical:

- `println "..."` → `println(f"...")` when the literal has `{...}`, `println("...")`
  otherwise.
- A string statement → `println(...)`.
- An interpolating `"...{x}..."` used as a value → `f"...{x}..."`.
- `"{{"` → `"{"` in literals without `f`.

## Migration

1. One release accepts both forms. The old ones give a warning with the replacement
   (`'println "..."' is deprecated; use 'println(f"...")'`), the same way the removed
   `unless`, `repeat` and `do ... while` now give a migration error.
2. A script, or a `zc fix` command, rewrites the forms above.
3. The next release turns the warnings into errors.

## Drawbacks

- More characters for the most common line in small programs: `println(f"x = {x}")`
  instead of `println "x = {x}"`.
- Every existing program changes, even if mechanically.
- Interpolation stays compiler support for `f"..."` (it needs the types of the
  expressions), so printing is not a plain library function either. It becomes an
  explicit call with a visible prefix.

## Alternatives

- **Keep the current syntax and fix the bugs.** Fixes the buffers but keeps the `{`
  escaping, the clash with block values and the 830-line generator.
- **Only part 2 (calls) without part 1.** Removes the statement syntax but keeps
  interpolation in every literal.
- **A keyword for block values** (`give "a"`) instead of making string statements
  plain expressions. Fixes the clash for blocks but not for `match` arms or the
  escaping.

## Unresolved questions

- Part 3: which option, and when (tied to the memory model).
- Whether `print(fmt, ...)` from `std/io.zc` stays, is renamed, or merges with the new
  `print`.
- Whether `f"..."` accepts quotes inside `{...}` (C#-style), which the current lexer
  does not.
- The length of the deprecation period.
