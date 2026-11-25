# GitHub Actions expression evaluator

Evaluate a GitHub Actions `${{ }}` expression against a context JSON and see every type cast the runner applied, including the four casts where the runner disagrees with JavaScript.

**Live demo:** https://0xelitesystem.github.io/github-actions-expression-evaluator/

The evaluator is reimplemented from GitHub's published runner sources, so it reproduces the casts that the documentation cast table leaves out. String against string is `OrdinalIgnoreCase`, not an exact match, which is why a branch guard on a ref name fires on any casing. `null` casts to `0` and the comparison is then retried, which is why `null` equals both `0` and the empty string. A container compared with `==` never reaches a `NaN` cast at all, because the coercion function has no Array or Object branch, so it falls through the kind check and returns `false`. This tool is independent. It is not affiliated with, endorsed by, or produced by GitHub.

The four rows that JavaScript knowledge gets wrong, computed live in your browser when the page loads:

| expression | this evaluator | the same operator in JavaScript |
| --- | --- | --- |
| `'ABC' == 'abc'` | true | false |
| `null == 0` | true | false |
| `null == ''` | true | false |
| `fromJSON('[]') == 0` | false | true |

The non-transitive results people quote are the lead-in, not the headline. `'' == 0` is true, `0 == '0'` is true, `'' == '0'` is false and `'true' == true` is false in this evaluator, and raw JavaScript returns exactly the same four answers, so knowing JavaScript is enough to predict them. The page renders those four in their own table so you can see they agree.

## Use

1. Open the page. Both boxes are prefilled with the worked example: the expression `github.ref == 'refs/heads/main'` against a context with two top level keys, `github.ref` set to `refs/heads/Main` and `inputs.prerelease` set to `false`. It evaluates to **true**, and the same comparison in JavaScript is **false**.
2. Edit the expression box. The `${{ }}` wrapper is optional. Evaluation is debounced by 300ms and the verdict panel is a live region, so a screen reader hears the new value without a button press.
3. Edit the context box. It must be a JSON object at the top level. An empty box is treated as `{}`, and a name that is not in the context resolves to `Null` rather than raising an error, because that is what the runner does and it is the path that makes `null == 0` matter.
4. Read the trace. Every node gets a row naming the kind of each operand, the coercion branch that fired, and the branch that produced the verdict. When a string is cast to a number, the row names which `ParseNumber` branch fired: empty after trim, decimal, lowercase `0x` hex, lowercase `0o` octal, the `Infinity` literal in any casing, or no branch at all.
5. Press **Run self test**. It runs 123 assertions plus one positive control, an assertion written to fail on purpose so that a suite which silently stops running is visible instead of green. Three of those checks also run on page load in an always-on strip.

Functions that cannot be computed in a browser are refused by name rather than guessed. `hashFiles` needs the runner workspace. `success`, `failure`, `always` and `cancelled` need job status from the runner. `toJSON` is refused because the runner writes JSON through its own writer and that writer was not reproduced and verified here. `contains`, `startsWith`, `endsWith`, `join`, `fromJSON` and `format` are implemented.

## Why this exists

The published cast table is incomplete as a description of `==`. It lists Array and Object casting to `NaN`, but those two rows are unreachable from every comparison operator, equality and relational alike. Its String row begins "Parsed from any legal JSON number format, otherwise NaN", and the runner's parser is wider than that: it accepts hexadecimal, octal, a leading plus, surrounding whitespace and the literal `Infinity` in any casing, and not one of those is a legal JSON number. It also rejects binary literals and the uppercase `0X` and `0O` prefixes that JavaScript's `Number()` accepts. So the two obvious ways to answer "will this `if:` run" are both wrong: reading the docs table understates the parser, and reasoning from JavaScript gets the four rows above backwards.

Everything runs in one file with no build step, no network calls and no analytics, and the whole thing is MIT licensed, so you can read the algorithm rather than trust it.

## Privacy

Everything runs in your browser. The expression and the context JSON never leave your machine. The page makes no network calls of any kind: no external scripts, no external stylesheets, no web fonts, no analytics, no telemetry, no beacons. There is nothing to opt out of because nothing is collected. Open the file, read it top to bottom, and confirm it for yourself.

Your input is treated as data and never as code. There is no `eval`, no `new Function`, and no timer given a string. Every result cell is built with `createElement` and `textContent`, so nothing you paste is ever parsed as markup. Nothing is assigned onto an object built from your JSON, and context lookups read own properties only, so keys named `__proto__`, `constructor` or `prototype` are ordinary data with no path to the prototype chain.

## Run locally

    git clone https://github.com/0xelitesystem/github-actions-expression-evaluator.git
    cd github-actions-expression-evaluator

Then open `index.html` directly in a browser. It works from `file://` with no server. If you prefer a server:

    python -m http.server 8000

and open http://localhost:8000/.

## Build

There is no build. One file, `index.html`, with inline CSS and inline JavaScript. No package manager, no bundler, no dependencies, no build artifacts.

## How the answers were checked

The behaviour was reimplemented from GitHub's published runner sources, fetched and read on 2026-09-09. No code is reproduced here. The files, with the byte sizes and line numbers they had that day:

| file | bytes | what was read |
| --- | --- | --- |
| `src/Sdk/DTExpressions2/Expressions2/EvaluationResult.cs` | 17010 | `IsFalsy` lines 50 to 71, `IsTruthy` line 75, String equality line 256, reference equality line 267, `CoerceTypes` lines 360 to 397, `ConvertToNumber` lines 402 to 418 |
| `src/Sdk/DTExpressions2/Expressions2/Sdk/ExpressionUtility.cs` | 8581 | `ParseNumber` lines 193 to 257 |
| `src/Sdk/DTExpressions2/Expressions2/ExpressionConstants.cs` | 2749 | `MaxDepth = 50` line 30, `MaxLength = 21000` line 31, `NumberFormat = "G15"` line 35 |
| `src/Sdk/DTExpressions2/Expressions2/Tokens/LexicalAnalyzer.cs` | 20898 | number literals go through the same `ParseNumber`, line 131 |

The comment above the equality functions in `EvaluationResult.cs`, lines 78 and 79, reads: "Similar to the Javascript abstract equality comparison algorithm http://www.ecma-international.org/ecma-262/5.1/#sec-11.9.3. Except string comparison is OrdinalIgnoreCase, and objects are not coerced to primitives."

Number to string uses the runner's `G15` format, so `1e-7` renders as `1E-07` and `1e21` renders as `1E+21`, where JavaScript would print `1e-7` and `1e+21`. That matters wherever `format`, `join` or `contains` turns a number into a string. The self test ships eighteen `G15` vectors so you can check the formatter without leaving the page.

GitHub also publishes a JavaScript port of this language, the npm package `@actions/expressions`, built in the repository `actions/languageservices` and MIT licensed. In version 0.3.61 of that package the string to number cast is a plain JavaScript `Number()` call, in `dist/data/string.js` at line 12, so it disagrees with the C# runner on `'0b101'`, `'0X10'`, `'0O17'`, `'0xFFFFFFFF'` and `'0x100000000'`. This page follows the C# runner, because that is the implementation which executes on hosted runners. Every one of those five is a self test row.

Sixteen of the results on this page were measured on a real GitHub hosted `ubuntu-latest` runner on 2026-09-21, in a private workflow whose raw job output was read back verbatim. The jobs are `35656695534` and `35656787367`. The measured expressions are `'' == 0`, `0 == '0'`, `'' == '0'`, `'true' == true`, `'ABC' == 'abc'`, `null == 0`, `fromJSON('[]') == 0`, `'infinity' == 0`, `'0x10' == 16` and `'0o17' == 15` in the first job, and `'infinity' > 0`, `'Infinity' > 0`, `'INFINITY' > 0`, `'NaN' > 0`, `'+5' == 5` and `' 5 ' == 5` in the second. This evaluator returns the same verdict on all sixteen, and each one ships as a self test row.

The second job exists because equality alone cannot separate `Infinity` from `NaN`. Both `'infinity' == 0` and `'NaN' == 0` are false whichever one the string parsed to, so the first job could not answer the open question. A relational comparison answers it. Lowercase `'infinity'` does parse to a number on the runner, and the literal is matched case insensitively, so `'infinity'`, `'Infinity'` and `'INFINITY'` all parse to `Infinity` here. `'NaN'` is not a parse branch and stays `NaN`. That claim was deliberately withheld from the build until this test ran.

Everything else comes from this evaluator, from raw JavaScript running in your browser, or from the source files named above, and was not read off a hosted runner. That is the boundary, and it is a wide one: the `G15` number format table, the Int32 hexadecimal and octal edge cases other than the two measured rows, the `OrdinalIgnoreCase` folding rows including the dotless i, the `if:` truthiness rows, the context lookup rows, the refusals and the short circuit rows are all source-derived, not measured. A hosted runner measurement is also narrower than it sounds. It says what one runner image did on one date under the runner version that was live that day. It is not a guarantee about a future runner, and it is not a substitute for running the row that matters to you in your own workflow.

The colour palette was verified by arithmetic rather than by looking at it, every text token against every background token in both themes, 56 pairs. The lowest ratio is 5.46:1 in the light theme and 7.47:1 in the dark theme, both above the WCAG AA threshold of 4.5:1. The tokens are at the top of `index.html` if you want to recompute them.

## Third-party notices

No source code from any third party is reproduced in this repository. The behaviour was reimplemented from the published specifications and sources listed below.

- **Project:** actions/runner. **Copyright holder, copied verbatim from that project's LICENSE file:** `Copyright (c) 2019 GitHub`. **Licence:** The MIT License (MIT). **URL:** https://github.com/actions/runner. **What was taken:** nothing but behaviour. The coercion, comparison, truthiness, number parsing and number formatting rules were read from the C# sources named above and reimplemented in JavaScript. Two sentences of a source comment are quoted verbatim above, with attribution.
- **Project:** `@actions/expressions`, version 0.3.61. **Copyright holder, copied verbatim from that package's LICENSE file:** `Copyright GitHub`. **Licence:** MIT License. **URL:** https://github.com/actions/languageservices. **What was taken:** nothing but behaviour. It was used as a cross-check, and one rule was confirmed against it: that the invariant uppercase mapping used for `OrdinalIgnoreCase` leaves U+0131 LATIN SMALL LETTER DOTLESS I alone.

GitHub is a trademark of its owner. This project is independent and unaffiliated, and nothing here implies endorsement.

## License

MIT. Copyright (c) 2026 RBL Industries LLC.

## Related

- [yaml-type-coercion-auditor](https://github.com/0xelitesystem/yaml-type-coercion-auditor) that tool decides what the YAML parser turned your token into, this one decides what the expression evaluator does with the value the parser produced.
- [cron-plain-english-reader](https://github.com/0xelitesystem/cron-plain-english-reader) for the other half of a workflow trigger, the schedule.
- [electron-release-matrix-auditor](https://github.com/0xelitesystem/electron-release-matrix-auditor) reads a whole workflow file, where this one evaluates a single expression.
