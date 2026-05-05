# Frontend Parser Wave

この文書は ECMAScript / TypeScript の lexer/parser 対応を、reference test の偶発的な失敗発見ではなく、仕様から小さな実装 slice に分解するための運用契約である。

通常の feature support 表は `docs/language-reference/javascript-features.md` と `docs/language-reference/typescript-features.md` に置く。この文書は、それらの表から parser/frontend 作業を切り出すときの source order、issue 分割、検証基準を定義する。

## 目的

frontend/parser wave は、構文受理と AST 生成を先に安定させる。parser が受理しただけでは runtime semantics の互換性を主張しない。

この wave の成果は次の状態で完了とする。

- lexer/parser が対象構文を診断なしで受理する
- AST または dump unparse が仕様上の構文構造を保つ
- TypeScript-only syntax は、runtime に影響しない範囲では AST 前後で消去される
- invalid syntax は panic ではなく診断として分類される
- semantic parity が必要な構文は、別 issue で Node differential acceptance criteria を持つ

## 参照ソース

| Source | Local path | 用途 |
|---|---|---|
| ECMA-262 | `reference/ecma262/spec.html` | ECMAScript lexical / syntactic grammar の正本 |
| test262 | `reference/test262/test/language/` | ECMAScript parser/early-error/semantic reference cases |
| test262 generated source | `reference/test262/src/` | grammar family ごとの生成元、代表 case 探索 |
| TypeScript compiler | `reference/typescript/src/compiler/scanner.ts`, `reference/typescript/src/compiler/parser.ts` | TypeScript scanner/parser の oracle |
| TypeScript compiler tests | `reference/typescript/tests/cases/compiler/` | TypeScript parser/checker/emit reference cases |
| TypeScript-Go | `reference/typescript-go/testdata/tests/cases/compiler/` | TypeScript implementation comparison and generated failure buckets |

外部 URL は仕様確認に使ってよいが、issue には可能な限り local path と reproduction command を残す。

## Source Order

ECMAScript syntax は ECMA-262 の lexical grammar から構文 grammar へ進める。TypeScript syntax は ECMAScript の該当構文が parser で表現できる状態になってから overlay として扱う。

| Order | Family | Primary source | Wave output |
|---:|---|---|---|
| 1 | Source text, Unicode, line terminators, comments | ECMA-262 Source Text / Lexical Grammar | tokenization and span stability |
| 2 | Identifiers, keywords, private identifiers | ECMA-262 Names and Keywords | reserved-word and context-sensitive keyword diagnostics |
| 3 | Literals | ECMA-262 Literals | numeric, string, template, regexp, null/boolean literal parsing |
| 4 | Automatic semicolon insertion | ECMA-262 ASI rules | line-terminator-sensitive statement parsing |
| 5 | Primary and member expressions | ECMA-262 Expressions | expression precedence and left-hand-side forms |
| 6 | Unary, update, binary, logical, conditional, assignment | ECMA-262 Expressions | precedence/associativity coverage |
| 7 | Binding and assignment patterns | ECMA-262 Destructuring grammar | parser-level pattern AST and invalid target diagnostics |
| 8 | Statements and declarations | ECMA-262 Statements and Declarations | statement AST coverage and label/control-flow syntax |
| 9 | Functions, arrows, generators, async functions | ECMA-262 Functions | parameter grammar, body forms, line-terminator restrictions |
| 10 | Classes and private elements | ECMA-262 Classes | class element syntax, private names, decorators only when selected |
| 11 | Modules | ECMA-262 Modules | static import/export forms and module-only grammar |
| 12 | Annex B syntax | ECMA-262 Annex B | web-compatible syntax diagnostics or supported parser forms |
| 13 | TypeScript erased syntax | TypeScript parser | annotations, type-only declarations, generic syntax, assertions |
| 14 | TypeScript value syntax | TypeScript parser | enums, parameter properties, namespaces, decorators when supported |

## Issue 分割

Parser syntax epic は直接実装しない。対象 family ごとに implementation-ready child issue を作る。

Child issue は次の単位を上限にする。

- 1 grammar family
- 1 diagnostic family
- 1 AST shape
- 1 reference path window

例えば、`destructuring` は binding pattern、assignment pattern、parameter pattern、for-in/of head のように分ける。`class` は declaration/expression、elements、private names、static blocks、decorators を分ける。

## Child Issue Template

Parser wave の child issue は、少なくとも次を含める。

| Field | 内容 |
|---|---|
| Spec refs | `reference/ecma262/spec.html` の clause 名、または TypeScript parser source path |
| Representative valid snippets | parser が受理すべき最小コード |
| Representative invalid snippets | 診断になるべき最小コード |
| AST expectation | AST node kind、span、または dump `--ast --unparse` の期待形 |
| Erasure rule | TypeScript-only syntax の場合、消去する範囲と残す runtime 構文 |
| Reference paths | `reference/test262/...` / `reference/typescript/...` / `reference/typescript-go/...` の代表 path |
| Validation | frontend unit test、CLI dump/build smoke、必要な reference-coverage command |
| Semantic boundary | parser-only か、別 semantic issue が必要か |

## 検証レベル

Parser wave は validation を三段階に分ける。

| Level | Command class | 意味 |
|---|---|---|
| parser unit | `cargo nextest run -p ts2wasm-frontend` | lexer/parser/AST の直接確認 |
| CLI parser smoke | `cargo run -q -p ts2wasm-cli -- dump --ast --unparse <fixture>` | CLI 経由で parse と unparse が安定している確認 |
| reference slice | `mise run reference-coverage -- <suite> --path-filter <path> --detail` | 外部 reference case の分類が改善した確認 |

Build 成功は parser coverage の根拠にはなるが、semantic coverage の根拠にはしない。Node.js と `iwasm` の stdout/stderr/exit code を比較していない変更は semantic-pass として扱わない。

## Fixture 方針

Parser wave fixtures は、意味論確認用 fixture と混ぜない。

- parser-only fixture は小さく保ち、1 file に複数 family を詰め込まない
- TypeScript erasure fixture は dump unparse で消える構文と残る runtime statement を明確に分ける
- invalid syntax は panic regression と diagnostic code を確認する
- runtime が未対応の構文は build smoke を要求しない

既存 runtime semantic fixture に parser-only 構文を混ぜる場合、その fixture は semantic claim を持たない build/parser smoke として扱う。

## Reference Test の使い方

Reference test は探索の入口ではなく、仕様 slice の検証と分類に使う。

Parser wave の通常順序:

1. ECMA-262 または TypeScript parser source から grammar family を選ぶ
2. 最小 valid / invalid snippet を fixtures または frontend unit test に追加する
3. 対応する test262 / TypeScript / TypeScript-Go path を 1 つ以上選ぶ
4. reference-triage で parser failure か semantic failure かを確認する
5. parser-only acceptance を満たしたら、semantic gap は別 issue に分ける

`issues/open/059-implement-parser-syntax-extensions.md` と `issues/done/200-implement-parser-syntax.md` のような epic/generated bucket は、この分割作業の入力として扱う。直接の implementation work order にはしない。

## 完了条件

Frontend/parser wave の slice は、次を満たすと完了する。

- target grammar family の valid snippets が parser unit test で pass する
- invalid snippets が明示的な diagnostic になる
- TypeScript erased syntax は dump unparse で runtime 構文だけを残す
- reference path の diagnostic 分類が parser-syntax から前進する、または parser-only 非該当として記録される
- semantic parity を主張する場合は、別途 Node differential evidence を持つ
