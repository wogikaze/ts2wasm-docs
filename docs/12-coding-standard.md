# Coding Standard

このドキュメントは ts2wasm のコード規約を定める。compiler path のすべての変更はこの規約に従う。

この規約の目的は、Rust の表面上の style を揃えることではない。TypeScript / JavaScript semantics、name resolution、builtin API、runtime ABI、WASM backend、host capability、data layout が混ざって再び emitter に沈殿することを防ぐことである。

## 0. 絶対原則

以下の原則に反する PR はマージしない。

```text
1. source 起因の問題は Diagnostic にする。
2. compiler bug は InvariantViolation または bug! で明示する。
3. Parser は構文だけを読む。
4. Resolver は名前だけを解決する。
5. BuiltinResolver は builtin API だけを解決する。
6. Lowering は解決済み表現を Lowered IR に落とす。
7. Backend は validate 済み Lowered IR と RuntimeLinkPlan だけを受け取る。
8. Runtime / host import / capability / runtime string は catalog と linker で決める。
9. raw WAT を増やさない。
10. 機能追加は docs / validation / tests / differential を同時に更新する。
```

## 1. これまでの過ちと再発防止策

### 1.1 emitter がすべてを決めていた

過去の問題:

```text
JS semantics
runtime ABI
WASM 命令列
host import
memory layout
runtime helper の選択
```

これらが emitter 内で同時に決まっていた。

**理由**: emitterに責務を集中させると、変更の影響範囲が広がり、テストが困難になる。責務分離により、各phaseを独立してテスト可能にする。

禁止:

```text
backend 内で名前解決する
backend 内で builtin 判定する
backend 内で arity check する
backend 内で capability を直接決める
backend 内で runtime dependency を手書きする
```

必須:

```text
NameResolver
BuiltinResolver
Lowering
validate_lowered
RuntimeLinkPlan
Backend
```

の順に通す。

### 1.2 WAT 文字列直書きで backend が壊れやすかった

過去の問題:

```text
wat.push_str / format! で命令を生成
raw WAT template の括弧ミス
stack discipline が人力
runtime_builder が巨大文字列化
```

**理由**: 文字列操作は型安全ではなく、括弧ミスやstack disciplineのエラーが実行時まで発見されない。typed writerによりコンパイル時に検出可能にする。

禁止:

```rust
wat.push_str("(local.get 0)\n");
wat.push_str(&format!("(i32.const {})\n", value));
```

移行期間の例外:

```text
既存 WAT backend の関数を触る場合のみ、当面は raw string 編集を許可する。
ただし新規 runtime helper を巨大 raw string として追加してはいけない。
関数単位に分割し、linker / WAT / differential test を追加する。
```

目標:

```text
typed WAT writer
または wasm-encoder
```

### 1.3 console.log を parser special form にした

過去の問題:

```text
Stmt::ConsoleLog
console / log keyword 化
console.log だけ parser で特別扱い
```

**理由**: parserにAPI特別扱いを入れると、新しいAPIごとにparserを修正する必要があり、拡張性が低下する。BuiltinResolverで統一的に扱うべき。

禁止:

```text
Parser で console.log を特別扱いする
新しい API を parser special form として追加する
Resolver::as_console_log_call のような ad hoc 判定を増やす
```

正しい流れ:

```text
source:
  console.log(x)

Parser:
  Expr::Call {
    callee: Expr::Member { object: Ident("console"), property: "log" },
    args: [x]
  }

BuiltinResolver:
  BuiltinCall(ConsoleLog, [x])

Lowering:
  FunctionCallKind::Builtin(BuiltinId::ConsoleLog)

RuntimeLinkPlan:
  RuntimeFn::Log
  RuntimeFn::Write
  RuntimeFn::ValueToStringInto
  RuntimeFn::Copy
  HostImport::FdWrite
  Capability::StdoutWrite
```

### 1.4 RuntimeFn がただの symbol table だった

過去の問題:

```text
RuntimeFn::symbol() だけがあり、deps/imports/capabilities/runtime_strings/result がなかった
```

**理由**: symbolだけでは依存関係やcapability追跡が不可能。RuntimeSpecにより依存関係を明示し、linkerとmanifest生成を自動化する。

必須:

```rust
pub struct RuntimeSpec {
    pub symbol: &'static str,
    pub deps: &'static [RuntimeFn],
    pub imports: &'static [HostImport],
    pub capabilities: &'static [Capability],
    pub runtime_strings: &'static [&'static str],
    pub result: RuntimeResult,
}
```

runtime helper を追加する PR は必ず次を更新する。

```text
RuntimeFn variant
RuntimeSpec
emission_order
all
runtime_builder emit function
linker structure test
differential test if behavior changes
```

### 1.5 runtime 関数は trim したが runtime strings は常時入っていた

過去の問題:

```text
console.log なしでも undefined/null/true/false/newline が data segment に入る
```

**理由**: 不要なruntime stringを含めると、WASMサイズが無駄に増大し、standalone要件を満たせなくなる。必要なものだけinternする。

禁止:

```text
RuntimeString を WatEmitter::new で無条件 intern する
```

必須:

```text
required RuntimeFn を収集
RuntimeFn::spec().runtime_strings を集約
必要な runtime string だけ intern
user string literal とは区別可能にする
```

### 1.6 fd_write を常時 import していた

過去の問題:

```text
console.log を使わない program でも fd_write import が入る
```

**理由**: 不要なhost importがあると、standalone実行要件を満たせず、security auditも困難になる。必要時のみimportする。

禁止:

```text
backend が fd_write import 文字列を無条件で出す
```

必須:

```text
RuntimeFn::spec().imports
→ RuntimeLinkPlan.required_imports
→ import emission
```

### 1.7 capability を集約しても manifest になっていなかった

過去の問題:

```text
Capability::StdoutWrite は型としてあるが、監査可能な出力がない
```

**理由**: capability型だけでは監査不可能。manifestとしてJSON出力することで、security auditとgate検証を可能にする。

必須:

```text
required_capabilities() を RuntimeLinkPlan から取得できる
capability manifest として JSON 出力できる設計にする
```

manifest 例:

```json
{
  "imports": ["wasi_snapshot_preview1.fd_write"],
  "capabilities": ["stdout.write"],
  "runtime": ["Log", "Write", "ValueToStringInto", "Copy"]
}
```

### 1.8 Span を導入したが AST に保持しなかった

過去の問題:

```text
SpannedToken はあるが Expr / Stmt が span を持たない
lowering / validation diagnostic が span: None になる
```

**理由**: spanがないとエラー箇所の特定が不可能になり、ユーザー体験が著しく低下する。すべてのsource-derived nodeにspanを持たせる。

禁止:

```text
新規 AST / HIR node を span なしで追加する
source 起因 Diagnostic に span: None を返す
```

必須:

```text
Token
Stmt
Expr
HIR node
BuiltinCall
```

は source span を持つ。

### 1.9 small-int subset を JS number のように扱った

過去の問題:

```text
ValueTag は small-int なのに JavaScript number として説明した
number range check がなかった
多桁/負数 stringify が壊れていた
```

**理由**: 表現範囲を超える値を扱うとundefined behaviorになる。range checkとdiagnosticにより、安全境界を明確にする。

必須:

```text
ValueTag::can_encode_number
NumberOutOfRange diagnostic
Node differential test
small-int subset 制約の docs 記載
```

small-int subset の数値表現:

```text
対応:
  tagged small-int
  decimal stringify for integer
  negative integer stringify

非対応:
  f64
  NaN
  Infinity
  -Infinity
  BigInt
  fractional number
```

### 1.10 tests が実行結果に偏り、構造を見ていなかった

過去の問題:

```text
Node vs iwasm の stdout は見るが、runtime linker の中身を見ない
```

**理由**: 出力結果だけでは内部構造の正しさを保証できない。linker structure testにより、依存関係とimport生成を検証する。

必須テスト:

```text
parse snapshot
semantic / builtin resolution snapshot
lowered IR snapshot
linker structure test
WAT/import snapshot or wasm validation
Node vs wasm/iwasm differential
```

runtime linker を変えた PR は必ず linker structure test を追加する。

## 2. Panic / unwrap / expect 禁止

compiler path で `panic!`, `unwrap()`, `expect()` を使わない。

対象:

```text
Lexer
Parser
AST validation
NameResolver
BuiltinResolver
Lowering
IR validation
RuntimeLinkPlan
Backend
```

禁止:

```rust
let x = map.get(k).unwrap();
let x = map.get(k).expect("must exist");
panic!("unreachable");
```

許可:

```rust
bug!("internal invariant violated: {:?}", value);
```

`bug!` は入力起因ではない compiler bug のみ。

## 3. Diagnostic ポリシー

compiler phase は `Result<T, Diagnostic>` または `Result<T, Vec<Diagnostic>>` を返す。

禁止:

```rust
Result<T, String>
anyhow::Result<T> in compiler path
panic as error handling
```

Diagnostic の標準形:

```rust
pub struct Diagnostic {
    pub span: Span,
    pub code: DiagCode,
    pub message: String,
    pub notes: Vec<String>,
}
```

`Option<Span>` は移行期間だけ許可。新規 diagnostic で `None` を増やさない。

必須 DiagCode:

```text
UnsupportedSyntax
UnsupportedBuiltin
UnsupportedDate
UnsupportedRegExp
UnsupportedModule
UnsupportedEval
UnsupportedTypeScriptSyntax
UnsupportedRuntimeSubset
UnresolvedName
UnresolvedFunction
DuplicateLocal
DuplicateParameter
DuplicateFunction
ArityMismatch
InvalidTopLevelReturn
NumberOutOfRange
InvariantViolation
BackendIo
```

### 3.1 TypeScript diagnostic ownership

TypeScript-only syntax must fail at the frontend boundary with a source span and a diagnostic that identifies the owning phase.

Use `UnsupportedTypeScriptSyntax` when the source construct is valid TypeScript but the compiler cannot yet parse, erase, or transform its TypeScript-only shape. Examples include type aliases, interfaces, type annotations, type assertions, ambient declarations, decorators, JSX, enums, parameter properties, and declaration-only forms before their boundary slice exists.

Use `UnsupportedModule` when the parser can represent the form but the missing behavior is module graph, module kind, import/export shape, module resolution, AMD/System/CommonJS transform policy, or declaration emit that affects module shape.

Use `UnsupportedRuntimeSubset` only after TypeScript syntax has been erased or transformed into executable JavaScript and the remaining blocker is ordinary JavaScript runtime semantics. Do not report a TypeScript erasure gap as a runtime subset gap.

Generated reference labels such as `parser-syntax` and `unknown-unsupported` are triage labels. They must be split to a concrete TypeScript boundary category before the issue is marked implementation-ready.

## 4. Span ポリシー

すべての source-derived node は span を持つ。

```rust
pub struct Span {
    pub start: u32,
    pub end: u32,
}
```

synthetic node は generated span を持つ。

```rust
Span::generated("lowered implicit undefined return")
```

必ず span を持つ diagnostic:

```text
UnsupportedSyntax
UnsupportedBuiltin
UnsupportedDate
UnsupportedRegExp
UnsupportedModule
UnsupportedEval
UnsupportedTypeScriptSyntax
UnsupportedRuntimeSubset
UnresolvedName
UnresolvedFunction
DuplicateLocal
DuplicateParameter
DuplicateFunction
ArityMismatch
InvalidTopLevelReturn
NumberOutOfRange
```

## 5. Phase Separation

各 phase の責務を固定する。

```text
Lexer:
  tokenization only

Parser:
  syntax only
  no builtin judgment
  no host/API judgment

AST Validator:
  syntax-level restrictions
  top-level return
  duplicate declarations where syntax/scope-level obvious

NameResolver:
  lexical scope
  function declarations
  local bindings

BuiltinResolver:
  console.log / Math.* / process.* / fs.*
  arity contract
  result contract
  capability contract

Lowering:
  resolved representation → Lowered IR
  no host import strings
  no WAT symbols

validate_lowered:
  structural invariants
  value/effect context
  arity
  local/function id consistency

RuntimeLinkPlan:
  runtime deps/imports/capabilities/runtime strings

Backend:
  encode validated program
  no name resolution
  no builtin discovery
```

## 6. AST / HIR / Lowered IR 更新規約

IR variant 追加時は以下を同時に更新する。

```text
validator
debug/snapshot printer
parse snapshot
semantic snapshot
lowered snapshot
linker structure test if runtime is affected
differential test if behavior is affected
current-state.md
unsupported diagnostic test
```

これが揃わない PR はマージしない。

## 7. validate_lowered 必須検査

`validate_lowered` は backend 前の最後の防壁である。

必須検査:

```text
FuncId in range
function.id == program.functions[index]
params are contiguous LocalId starting from 0
locals are contiguous LocalId starting after params
top_level_locals are contiguous LocalId starting from 0
LocalId in each expression is in scope range
user function call arity
builtin call arity
builtin value/effect context
number literal encodable by ValueTag
Return appears only in function context
```

backend はこれらを再チェックしない。

## 8. RuntimeFn Catalog

runtime helper は必ず catalog に登録する。

```rust
pub enum RuntimeFn { ... }

pub struct RuntimeSpec {
    pub symbol: &'static str,
    pub deps: &'static [RuntimeFn],
    pub imports: &'static [HostImport],
    pub capabilities: &'static [Capability],
    pub runtime_strings: &'static [&'static str],
    pub result: RuntimeResult,
}
```

RuntimeFn 追加時の必須更新:

```text
spec
emission_order
all
emit function
linker test
differential test if semantic behavior changes
```

禁止:

```text
emitter が "$add" などの runtime symbol を直書きする
runtime helper を catalog 外で emit する
RuntimeFn::all() が emission_order() を返す
```

`all()` と `emission_order()` は意図的に独立させる。

```rust
// Intentionally independent from emission_order().
// Tests use this as the enum inventory.
pub const fn all() -> &'static [RuntimeFn] { ... }
```

## 9. RuntimeLinkPlan

`WatEmitter` が直接 linker にならない。RuntimeLinkPlan を独立させる。

```rust
pub struct RuntimeLinkPlan {
    pub required_runtime: BTreeSet<RuntimeFn>,
    pub required_imports: BTreeSet<HostImport>,
    pub required_capabilities: BTreeSet<Capability>,
    pub required_runtime_strings: BTreeSet<&'static str>,
}
```

生成手順:

```text
LoweredProgram scan
→ direct RuntimeFn requirements
→ dependency closure
→ imports aggregation
→ capabilities aggregation
→ runtime strings aggregation
```

WatEmitter は RuntimeLinkPlan を受け取って emit するだけにする。

## 10. Host Import / Capability Manifest

host import は RuntimeLinkPlan から生成する。

禁止:

```rust
wat.push_str("(import \"wasi_snapshot_preview1\" \"fd_write\" ...)");
```

許可:

```rust
for import in plan.required_imports() {
    module.import(import);
}
```

capability は manifest として出力できること。

```json
{
  "imports": ["wasi_snapshot_preview1.fd_write"],
  "capabilities": ["stdout.write"],
  "runtime": ["Log", "Write", "ValueToStringInto", "Copy"]
}
```

## 11. Runtime String / Data Layout

runtime strings は required RuntimeFn によってのみ intern する。

禁止:

```text
UNDEFINED / NULL / TRUE / FALSE / NEWLINE を無条件 intern
```

user string と runtime string は origin を区別可能にする。

実装済み: `StringOrigin` enum (`crates/backend-wasm/src/runtime_fn.rs`)、
`WatEmitter::intern_string_with_origin()`、`WatEmitter::is_runtime_string()`。
`RuntimeLinkPlan::string_origins()` が各 runtime string の由来を `RuntimeFn` 単位で追跡する。

```rust
enum StringOrigin {
    UserLiteral,
    Runtime(RuntimeFn),
}
```

memory layout validation は backend emission 前に必須。

```text
static data end <= SCRATCH_OFFSET
SCRATCH_OFFSET + SCRATCH_SIZE <= HEAP_START
SCRATCH_OFFSET < HEAP_START
heap start alignment
string data alignment
```

## 12. Builtin API 規約

builtin 追加時は以下を同時に定義する。

```text
BuiltinId
source pattern in BuiltinResolver
arity
result: Value or EffectOnly
capability requirement
RuntimeFn mapping if needed
unsupported diagnostic if partial
linker structure test
differential test
```

禁止:

```text
Parser に special form を追加
NameResolver に builtin 判定を追加
Backend に builtin 判定を追加
```

## 13. Value Representation

value representation は `runtime/value.rs` に閉じ込める。

禁止:

```rust
(value << 3) | 4
v & 0b111
v & !0b111
```

推奨:

```rust
ValueTag::encode_number(value)
ValueTag::can_encode_number(value)
ValueTag::is_string(value)
```

backend / runtime builder が layout constants を直接使う場合は、`Layout` / `ValueTag` 経由に限定する。

## 14. Feature Gate / Compatibility Level

機能追加は compatibility level を持つ。

```text
initial subset: single-file JS、small-int、function、let/if/while、console.log
resolver workstream: BuiltinResolver 分離と WASI-safe API 拡張
data-model workstream: array/object/heap-backed string
semantic tests: fixture corpus と differential oracle の拡大
```

unsupported case は silent fallback しない。

```rust
return Err(Diagnostic {
    span,
    code: DiagCode::UnsupportedSyntax,
    message: "object literal is not supported in this subset".to_owned(),
    notes: vec!["tracked as object-literals work".to_owned()],
});
```

## 15. Test Policy

変更種別ごとの必須テスト:

```text
Lexer / Parser:
  parse snapshot
  unsupported syntax diagnostic

Resolver / BuiltinResolver:
  semantic snapshot
  unresolved / duplicate / arity diagnostic

Lowering:
  lowered IR snapshot
  validate_lowered negative test

RuntimeFn / RuntimeLinkPlan:
  required RuntimeFn test
  required imports test
  required capabilities test
  required runtime strings test
  emission_order/all consistency test

Runtime semantics:
  Node vs wasm/iwasm differential

Backend emission:
  wasm validation
  WAT/import snapshot（typed writer 移行までの回帰用）
```

必須 linker tests:

```text
console.log なし → fd_write import なし
console.log あり → fd_write import あり
runtime 不要 → runtime strings なし
Add with string → Add + IsString + Concat + ValueToStringInto + Copy
StrictEqual → StrictEqual + IsString + StringEqual
If / While → TruthyBool
```

## 16. Documentation Update Policy

コード変更と docs 更新を分離しない。

以下を変更した場合は docs 更新必須。

```text
syntax
semantics
runtime ABI
host capability
compatibility level
test policy
unsupported feature set
value representation
memory layout
```

最低限更新対象:

```text
docs/05-compatibility-and-semantics.md
docs/06-testing-and-coverage.md
docs/09-security-and-capability-model.md
docs/11-shared-definitions.md
current-state.md
```

## 17. Commit / Review Policy

commit は論理単位で分ける。

良い例:

```text
backend: add runtime dependency linker
backend: test runtime linker contracts
ir: validate builtin and function layout invariants
runtime: fix integer stringification
```

悪い例:

```text
misc fixes
update compiler
big refactor
```

各 commit 前に必須:

```bash
cargo fmt --all --check
cargo nextest run
```

runtime / backend / differential に関わる変更では、Node vs wasm/iwasm differential も通す。

## 18. Review Checklist

PR reviewer は以下を確認する。

```text
panic/unwrap/expect が compiler path にないか
String error が compiler path にないか
source diagnostic に span があるか
parser が API/builtin を特別扱いしていないか
Resolver に builtin 判定が混ざっていないか
backend が name/builtin/arity を判断していないか
RuntimeFn catalog が更新されているか
RuntimeLinkPlan test があるか
fd_write など host import が条件化されているか
runtime strings が必要時のみ intern されるか
ValueTag の範囲検査があるか
current-state.md が更新されているか
```

## 19. Gatekeeper Checklist

この checklist は、実装者ではなく gatekeeper が使う。
PR / agent output / autopilot result を mainline に入れてよいかを判定するための最低条件である。

### 19.1 即 reject 条件

以下のいずれかに該当する場合、内容が動いていても reject する。

```text
Parser が builtin / API / host capability を判定している
Resolver / Lowering が host import を知っている
Backend が名前解決・builtin 判定・arity check をしている
RuntimeFn を追加したのに RuntimeSpec / emission_order / all / emit function / linker test が更新されていない
HostImport を追加したのに Capability / manifest / conditional import test がない
runtime helper を追加したのに RuntimeLinkPlan test がない
runtime string を無条件 intern している
fd_write / fd_read など host import が常時 import されている
source 起因 Diagnostic に span: None を新規追加している
既存 docs の gate を削除して進行可能に見せている
「次 gate に進んだ」と docs だけ書き換えて、負債を返済していない
```

### 19.2 差分レビューの最初に見るコマンド

```bash
git status --short
git log --oneline -8
git diff --stat HEAD~1..HEAD
git show --name-only --oneline HEAD
cargo fmt --all --check
cargo nextest run
```

runtime / backend / WASI / differential に関係する変更では追加で確認する。

```bash
cargo nextest run --include-ignored
```

または、プロジェクトで定義された iwasm differential suite を実行する。

### 19.3 grep gate

以下の grep で、過去のやらかしの再発を確認する。

```bash
rg "as_console_log_call" crates/cli/src
rg 'property == "length"' crates/ir/src/lowered.rs
rg 'mod backend;|mod parser;|mod compiler;|mod driver;' crates/cli/src
rg 'struct Lexer|struct Parser|ts2wasm_backend_wasm' crates/cli/src
rg 'fd_write|fd_read' crates/backend-wasm/src -g'*.rs'
rg 'RuntimeString::.*intern|intern_required_runtime_strings' crates/backend-wasm/src
rg 'span: None' crates/cli/src
rg 'unwrap\\(|expect\\(|panic!' crates/cli/src
```

判定基準:

```text
as_console_log_call が復活していたら reject
lowered.rs が .length を直接判定していたら reject
crates/cli/src に backend/parser/compiler/driver module 宣言や lexer/parser 実装が復活していたら reject
crates/cli/src/lib.rs が backend crate を直接参照していたら reject
fd_write / fd_read が RuntimeLinkPlan 経由でない場所に直書きされていたら reject
runtime string の無条件 intern が増えていたら reject
source 起因 diagnostic の span: None が増えていたら reject
compiler path の unwrap/expect/panic が増えていたら原則 reject
```

例外は test code のみ。test code 以外の例外はコメントで理由を書く。

### 19.4 RuntimeFn 追加時 checklist

RuntimeFn variant を追加した PR は、以下をすべて満たすこと。

```text
RuntimeFn variant を追加した
RuntimeSpec を追加した
deps を定義した
imports を定義した
capability を定義した
runtime_strings を定義した（StringOrigin で runtime 文字列と user 文字列が区別されること）
result を定義した
emission_order に追加した
all に追加した
runtime_builder emit function を追加した
RuntimeFn::symbol / manifest name を更新した
RuntimeLinkPlan test を追加した
capability manifest test を追加した
behavior が変わる場合は Node vs wasm/iwasm differential test を追加した
current-state.md を更新した
docs/14-runtime-abi.md を更新した
```

1つでも欠けたら reject する。

### 19.5 HostImport / Capability 追加時 checklist

HostImport または Capability を追加した PR は、以下をすべて満たすこと。

```text
HostImport enum を追加した
Capability enum を追加した
RuntimeSpec.imports に接続した
RuntimeSpec.capability に接続した
RuntimeLinkPlan.required_imports / required_capabilities test を追加した
capability manifest JSON に出る test を追加した
host import が必要時だけ WAT に出る test を追加した
host import が不要時に WAT に出ない test を追加した
docs/09-security-and-capability-model.md を更新した
current-state.md を更新した
```

host import が常時 emit される変更は reject する。

### 19.6 Builtin 追加時 checklist

BuiltinId / BuiltinPropertyId を追加した PR は、以下をすべて満たすこと。

```text
ir/builtin.rs に contract を追加した
BuiltinResolver に source pattern を追加した
arity / result を定義した
unsupported argument form の diagnostic を追加した
誤認識しない negative test を追加した
Lowering の扱いを明示した
runtime に接続する場合は RuntimeFn catalog に接続した
runtime に接続しない場合は compile-time gate で止めた
docs/05 または current-state.md に subset semantics を記録した
```

Parser に builtin special form を追加していたら reject する。

### 19.7 RuntimeLinkPlan gate

runtime / capability / manifest に触る PR は、以下を確認する。

```text
RuntimeLinkPlan が required_runtime を直接収集している
WatEmitter が runtime dependency を計算していない
required_imports が RuntimeSpec から導出されている
required_capabilities が RuntimeSpec から導出されている
required_runtime_strings が RuntimeSpec から導出されている
manifest は RuntimeLinkPlan から生成されている
```

次のような実装は reject する。

```text
WatEmitter が add_required_runtime を持つ
backend が fd_write / fd_read を機能名で直接判断する
manifest が LoweredProgram を直接再走査する
manifest が RuntimeLinkPlan と別ロジックで imports/capabilities を決める
```

### 19.8 Memory layout gate

Layout / heap / scratch / stdin buffer / data segment に触る PR は、以下を確認する。

```text
Layout constants が docs/14-runtime-abi.md と一致している
static data end <= SCRATCH_OFFSET
SCRATCH_OFFSET + SCRATCH_SIZE <= next fixed buffer
fixed buffer end <= HEAP_START
HEAP_START alignment が ValueTag::HEAP_MASK と整合する
新しい固定領域を追加した場合、validate_memory_layout が更新されている
large allocation / OOM policy が docs に書かれている
```

固定領域を追加して memory validation を更新していない場合は reject する。

### 19.9 Diagnostic / Span gate

source 起因のエラーを追加する PR は、以下を満たすこと。

```text
Diagnostic code が適切
message が source user 向け
span が Some(span)
test が span.is_some() を確認している
unsupported と invariant violation を混同していない
```

以下は reject する。

```text
source 起因なのに InvariantViolation
source 起因なのに span: None
unsupported feature を silent fallback
未実装 runtime を undefined で返して通す
```

### 19.10 Docs gate

以下を変更した場合は docs 更新必須。

```text
syntax
semantics
runtime ABI
value representation
memory layout
host import
capability
manifest schema
merge gate（`docs/11` の Gate 定義）
unsupported feature set
test policy
```

最低限の更新候補:

```text
docs/05-compatibility-and-semantics.md
docs/09-security-and-capability-model.md
docs/11-shared-definitions.md
current-state.md
docs/14-runtime-abi.md
```

docs の gate を削除して進行可能にする変更は reject する。gate を満たした場合のみ、status を done に更新する。

### 19.11 Handoff packet

gatekeeper に渡すときは、以下を必ず添付する。

```text
目的:
  何を実装したか。何を実装していないか。

変更範囲:
  touched files
  code commits
  docs commits

検証:
  cargo fmt --all --check
  cargo nextest run
  differential / iwasm test if relevant
  grep gate 結果

設計上の禁止事項:
  今回あえてやっていないこと
  次 slice に残したこと

既知の未関連変更:
  git status --short の未追跡/未コミット差分
```

テンプレート:

```text
Gate handoff

Scope:
- Implemented:
- Not implemented:

Commits:
- <hash> <title>
- <hash> <title>

Validation:
- cargo fmt --all --check: pass/fail
- cargo nextest run: pass/fail
- iwasm differential: pass/fail/not applicable
- grep gate:
  - as_console_log_call: 0
  - property == "length" in lowered.rs: 0
  - source diagnostic span None added: no/yes

Risk:
- Runtime behavior changed: yes/no
- Host import changed: yes/no
- Memory layout changed: yes/no
- Manifest schema changed: yes/no

Docs:
- current-state.md updated: yes/no
- docs/15 updated if ABI changed: yes/no
- docs/09 updated if capability changed: yes/no

Known unrelated working tree changes:
- <file>
```

### 19.12 Gate advance gate

次の gate 進行前に、current-state.md と docs/11-shared-definitions.md の gate 定義を確認する。

```text
done と書かれた項目は実装と test があること
deferred と書かれた項目を無視して次 gate に進まないこと
Gate 定義を書き換えるだけの変更をしないこと
```

次 gate に進める条件:

```text
current gate の acceptance tests が通っている
current-state.md が実装状態を正確に反映している
runtime/backend/capability 変更なら manifest と linker tests がある
unsupported / limitation が docs に残っている
未実装を実装済みのように書いていない
```

### 19.13 WAT generation

新しい WAT 生成コードは、生の文字列連結よりも `wat_writer` モジュールの typed API を優先する。

```text
OK:
  WatWriter::new()
      .add_import(&WatImport::new("module", "name", "$symbol"))

NOT OK:
  wat.push_str(&format!("  (import \"{}\" \"{}\" (func {}))\n", module, name, symbol))
```

既存の `runtime_builder.rs` の生の文字列連結は、段階的に typed writer に置換する。

## 20. 現在の優先順位

次の順で負債を潰す。

```text
P0:
  RuntimeLinkPlan を WatEmitter から分離 (done)
  BuiltinResolver pass 分離 (done)
  capability manifest 出力 (done)
  manifest import verification (done)
  stale milestone and transitional docs cleanup (done)
  AST node span

P1:
  typed WAT writer
  raw WAT runtime_builder の段階的置換
  user/runtime string origin 管理
  linker snapshot fixture 化
  reference coverage prerequisites hardening
  host-deny and auditable E2E manifest

P2:
  object / array / module
  frontend module extraction
  strict warning gate debt cleanup (`issues/open/238-*.md`)
```

object / array / module は、少なくとも BuiltinResolver と AST span が入るまで着手しない。
