# Rule catalog

Every rule and code-health budget in this release, as `hoovr rules` lists them. Ids
are stable and never reused, grouped by what they are about:

| Range | Theme |
|---|---|
| `HR0000` | the file does not parse |
| `HR00xx` | correctness — constructs with a known failure mode |
| `HR01xx` | layout and whitespace |
| `HR02xx` | naming |
| `HR03xx` | opt-in guardrails: team policies, off unless selected |
| `HR9xxx` | code-health budgets |

`hoovr explain <rule>` prints the full reasoning for any of them, which is where the
argument for a rule lives; the summaries below are headlines. Rules marked `opt-in`
run only when named in `select` or `--select`.

The lintr column names the lintr linter a rule corresponds to, which is also a name
`# nolint` accepts for it. A — there means HoovR has no lintr counterpart for that
rule: the two tools made different choices about what is worth reporting.
[Measurements](measurements.md) says how closely HoovR agrees with lintr.

## Rules

| Code | Name | On by default | Severity | Fix | lintr equivalent | Summary |
|---|---|---|---|---|---|---|
| `HR0001` | `assignment` | yes | style | yes | `assignment_linter` | Use `<-` for assignment |
| `HR0002` | `equals-na` | yes | warning | yes | `equals_na_linter` | Use `is.na()` instead of comparing with `NA` |
| `HR0003` | `t-and-f-symbol` | yes | style | yes | `T_and_F_symbol_linter` | Use `TRUE` and `FALSE`, not `T` and `F` |
| `HR0004` | `seq-along` | yes | warning | yes | `seq_linter` | Use `seq_along()` or `seq_len()` instead of `1:length(x)` |
| `HR0005` | `vector-logic` | yes | warning | yes | `vector_logic_linter` | Use `&&` and `\|\|` in conditions |
| `HR0006` | `any-is-na` | yes | style | yes | `any_is_na_linter` | Use `anyNA()` instead of `any(is.na())` |
| `HR0007` | `undesirable-function` | yes | warning | — | `undesirable_function_linter` | Avoid a function with a known failure mode |
| `HR0010` | `commented-code` | yes | style | — | `commented_code_linter` | Remove commented-out code |
| `HR0011` | `unused-local` | yes | warning | — | `object_usage_linter/unused-local` | Remove a local variable nothing reads |
| `HR0012` | `pipe-consistency` | yes | style | — | `pipe_consistency_linter` | Use the native pipe `\|>` |
| `HR0015` | `pipe-continuation` | yes | style | — | `pipe_continuation_linter` | One step per line in a multi-line pipeline |
| `HR0016` | `return` | yes | style | yes | `return_linter` | Let the last expression be the value; drop a final `return()` |
| `HR0013` | `undefined-function` | yes | warning | — | `object_usage_linter/no-visible-global-function` | Call only functions that exist |
| `HR0017` | `undefined-variable` | yes | warning | — | `object_usage_linter/no-visible-binding` | Read only variables that exist |
| `HR0019` | `undefined-superassignment` | yes | warning | — | `object_usage_linter/superassign` | Superassign only to a variable an enclosing function or the file binds |
| `HR0018` | `package-not-installed` | yes | warning | — | `object_usage_linter/unresolved-package` | Load only packages that are installed |
| `HR0300` | `no-comments` | opt-in | style | — | — | Explain code with names, not comments |
| `HR0301` | `magic-number` | opt-in | style | — | — | Give a numeric literal a name |
| `HR0020` | `unreachable-code` | opt-in | warning | — | `unreachable_code_linter` | Remove code that can never run |
| `HR0021` | `unused-import` | opt-in | warning | — | `unused_import_linter` | Attach only packages the file uses |
| `HR0022` | `unused-function` | opt-in | warning | — | — | Remove a package function nothing uses |
| `HR0008` | `semicolon` | yes | style | yes | `semicolon_linter` | Separate statements with a newline, not `;` |
| `HR0009` | `single-quotes` | yes | style | yes | `quotes_linter` | Prefer `"` for string literals |
| `HR0103` | `infix-spaces` | yes | style | yes | `infix_spaces_linter` | Put spaces around infix operators |
| `HR0104` | `commas` | yes | style | yes | `commas_linter` | Put a space after a comma and none before it |
| `HR0105` | `function-left-parentheses` | yes | style | yes | `function_left_parentheses_linter` | No space between a function and its `(` |
| `HR0107` | `paren-body` | yes | style | yes | `paren_body_linter` | Put a space between `)` and the body |
| `HR0108` | `brace-spacing` | yes | style | yes | `brace_linter/space-before-brace` | Put a space before `{` |
| `HR0109` | `if-else-braces` | yes | style | — | `brace_linter/if-else-symmetry` | Brace both branches of an `if`/`else`, or neither |
| `HR0110` | `function-body-braces` | yes | style | — | `brace_linter/function-body-braces` | Brace a function body that spans more than one line |
| `HR0111` | `open-brace-placement` | yes | style | — | `brace_linter/open-brace-placement` | End the line with `{`, and never start one with it |
| `HR0112` | `close-brace-placement` | yes | style | — | `brace_linter/close-brace-placement` | Put `}` on its own line |
| `HR0113` | `else-placement` | yes | style | — | `brace_linter/else-placement` | Put `else` on the same line as the `}` before it |
| `HR0114` | `indentation` | yes | style | — | `indentation_linter` | Indent by two spaces per level |
| `HR0115` | `space-before-paren` | yes | style | yes | `spaces_left_parentheses_linter` | Put a space before `(`, except in a call |
| `HR0116` | `spaces-inside` | yes | style | yes | `spaces_inside_linter` | No space just inside brackets |
| `HR0100` | `line-length` | yes | style | — | `line_length_linter` | Keep lines within the configured width |
| `HR0101` | `trailing-whitespace` | yes | style | yes | `trailing_whitespace_linter` | Remove whitespace at end of line |
| `HR0102` | `trailing-blank-lines` | yes | style | yes | `trailing_blank_lines_linter` | End the file with exactly one newline |
| `HR0106` | `tabs` | yes | style | yes | `whitespace_linter` | Indent with spaces, not tabs |
| `HR0200` | `object-name` | yes | style | — | `object_name_linter` | Name objects in snake_case |
| `HR0201` | `object-length` | yes | style | — | `object_length_linter` | Keep declared names reasonably short |

## Code-health budgets

| Code | Name | Default | Subject |
|---|---|---|---|
| `HR9001` | `cyclomatic` | < 22 | function |
| `HR9002` | `cognitive` | < 22 | function |
| `HR9007` | `halstead` | < 30 | function |
| `HR9003` | `nesting-depth` | < 5 | function |
| `HR9004` | `parameters` | < 6 | function |
| `HR9005` | `function-lines` | < 100 | function |
| `HR9006` | `file-lines` | < 500 | file |
| `HR9008` | `maintainability` | >= 31 | function |
| `HR9009` | `duplicate-bodies` | < 1 | file |
| `HR9010` | `dynamic-evaluation` | < 4 | file |
| `HR9011` | `dead-code` | < 2 | file |
| `HR9012` | `coverage` | >= 80 | file |
| `HR9013` | `crap` | < 25 | function |
| `HR9014` | `surviving-mutants` | < 1 | file |
