# Shared definitions

この文書は、複数の設計文書から参照する横断的な定義を集約する。個別文書はここにある表や schema を再定義せず、必要な文脈だけを説明する。

## Test status schema

すべてのテスト結果は、次の状態のいずれかに分類する。単なる skip は禁止する。

| 状態 | 意味 | 必須情報 |
|---|---|---|
| `pass` | 仕様通り成功 | suite、case、target |
| `fail` | 実装バグまたは仕様差分 | expected、actual、再現 target |
| `unsupported` | 未実装機能 | reason、feature label、issue ID または tracking label |
| `blocked` | runtime / host / toolchain の外部制約 | blocking condition、owner または upstream |
| `skip-with-reason` | テスト環境上の除外 | reason、除外条件、再確認条件 |

機械可読な test record は、少なくとも `suite`、`case`、`target`、`status`、`reason`、`tracking` を持つ。`reason` と `tracking` は `pass` では省略できるが、`unsupported`、`blocked`、`skip-with-reason` では必須とする。

## Capability manifest schema

生成物は、使用する外部能力を manifest として監査できる必要がある。filesystem は read/write/preopen を分離し、Node host import は関数単位で列挙する。

```json
{
  "schema_version": 1,
  "target": "wasm32-wasi",
  "standalone": true,
  "wasi": {
    "stdin": true,
    "stdout": true,
    "stderr": false,
    "args": false,
    "env": false,
    "clock": {
      "realtime": false
    },
    "filesystem": {
      "read": [],
      "write": [],
      "preopens": []
    },
    "random": false
  },
  "node_host": {
    "required": false,
    "imports": []
  },
  "capability_reasons": {
    "wasi.stdin": [
      "fs.readFileSync(0, \"utf8\")"
    ],
    "wasi.stdout": [
      "console.log"
    ]
  }
}
```

Node host が必要な場合は、`standalone` を `false` にし、`node_host.imports` に `host.<domain>.<function>` 形式で必要な関数だけを列挙する。

```json
{
  "schema_version": 1,
  "target": "wasm32-wasi+node-host",
  "standalone": false,
  "wasi": {
    "stdin": false,
    "stdout": true,
    "stderr": false,
    "args": false,
    "env": false,
    "clock": {
      "realtime": false
    },
    "filesystem": {
      "read": [],
      "write": [],
      "preopens": []
    },
    "random": false
  },
  "node_host": {
    "required": true,
    "imports": [
      "host.timer.setTimeout"
    ]
  },
  "capability_reasons": {
    "wasi.stdout": [
      "console.log"
    ],
    "host.timer.setTimeout": [
      "setTimeout"
    ]
  }
}
```

### Schema versioning and migration policy

The capability manifest has an explicit `schema_version` field, currently `1`. The named constant `ts2wasm_shared::capability::SCHEMA_VERSION` defines the current version.

**Backward compatibility:**
- Consumers MUST accept `schema_version >= 1` and MAY ignore unknown fields.
- New fields MUST be optional or have sensible defaults so existing consumers are not broken.
- Removing or renaming fields is a breaking change and MUST bump `schema_version`.

**Migration procedure:**
1. Bump `SCHEMA_VERSION` in `crates/shared/src/capability.rs`.
2. Update `validate()` to accept both old and new versions during transition.
3. Update `docs/11-shared-definitions.md` examples to show the new schema.
4. Regenerate all reference manifest fixtures.
5. Update any downstream consumers (web-ui / scripts) that parse manifest JSON.
6. Document the breaking change and version delta in this section.

## Optimization and safety modes

CLI の optimization level と semantic safety mode は別概念として扱う。`-O3` でも observable JavaScript semantics は壊さない。意味論差分を許す実験は `unsafe-fast` として明示的に分離する。

| CLI level | default safety mode | 方針 |
|---|---|---|
| `-O0` | `safe` | デバッグ性と差分確認を優先し、ほぼ素直に lowering する |
| `-O1` | `safe` | 明らかに安全な局所最適化だけを適用する |
| `-O2` | `typed` | 型情報と制御フローを使い、runtime check を保ちながら fast path を増やす |
| `-O3` | `typed` / proven `strict-wasm` | 証明できる範囲で特殊化、インライン化、表現最適化を強める |
| explicit `unsafe-fast` | `unsafe-fast` | 意味論差分を許容する実験モード。デフォルトにしない |

`strict-wasm` は、型と runtime guard によって観測可能な意味論差分がないと示せる範囲だけで使う。property store、function call、object escape、host boundary では canonical `JsValue` へ戻す。

## Benchmark policy

性能比較は、測定条件を固定して継続的に記録する。少なくとも benchmark 名、input size、target、runner version、cold/warm 区分、iteration count、median、p95、peak memory、wasm size、host call count を記録する。

## Feature priority guidelines

`docs/language-reference/*.md` の優先度列で使用する優先度の定義。

| 優先度 | 定義 | 例 |
|---|---|---|
| **P0** | 緊急: 意味論バグ修正、基本型・制御フローの未実装、セキュリティ関連 | computed property semantics bug (issue 012)、heap OOM check |
| **P1** | 高: よく使われる構文、基本ビルトイン、WASI 基本機能 | switch/while/break、string methods、Map/Set、class、import/export |
| **P2** | 中: 高度な機能、TypeScript 型システム、Stage 4+ WASM 提案 | Promise/async/await、TypeScript 型解析、multi-value、reference-types |
| **P3** | 低: レガシー機能、将来技術、最適化系 | eval/with、Component Model、WASI Preview 2、SIMD |
| **将来検討** | 将来対応または Stage 4 未満の提案 | Component Model、stack-switching、wide-arithmetic |
| **-** | 実装済み (優先度なし) | すべての実装済み機能 |

### 優先度付けの原則

1. **意味論正確性優先**: 意味論バグは P0
2. **広く使われる機能**: よく使われる構文・ビルトインは P1
3. **段階的実装**: 基本機能 → 高度な機能 → 将来技術の順
4. **WASM 段階**: MVP → Stage 4+ → Stage 4 未満
5. **TypeScript は P2 基本方針**: 型情報は最適化と診断に活用するが、実行時消去されるため P2
6. **セキュリティ関連**: capability boundary、host-deny は P0/P1

## Workstreams

本プロジェクトの主要目標は **test262 semantic_pass 90%**（約 48,100 / 53,445 ファイル）である。
gate 判定は各 workstream が独立して進められるが、下流 workstream は上流の gate を前提とする。

| ID | Name | 概要 | test262 への影響 |
|---|---|---|---|
| W0 | Runtime substrate | linear memory、value representation (RawValue)、WAT/wasm emission の基盤。iwasm で実行可能な最小 wasm を出せる状態を維持する | 必須基盤（直接 pass 増には寄与しない） |
| W1 | Standalone WASI execution | console.log / fd_write / stdin の WASI lowering。Node host 不要で実行できる最小プログラムを通す | 必須基盤（直接 pass 増には寄与しない） |
| W2 | Syntax coverage | 全 ECMAScript / TypeScript 構文の parser 実装。現在の unsupported 最大要因である parser gap を埋める | limit 500 で UnsupportedSyntax=204。全構文対応で unsupported の ~40% を解消 |
| W3 | Name resolution | global builtin / scope chain / hoisting / function 解決の完全実装。UnresolvedName / UnresolvedFunction をゼロにする | limit 500 で名前解決不足が 190/500。解消で unsupported を大幅削減 |
| W4 | Builtin runtime | String / Array / Date / Math / RegExp / JSON / Promise / Map / Set 等の runtime 実装 | 各 builtin カテゴリごとに数百〜数千の test262 ケースが通過可能に |
| W5 | Runtime semantics | object model、prototype chain、this binding、closure、class、exception、module、control flow | semantic_pass の質を決める基盤。builtin が実装されても正しい意味論がなければ pass にならない |
| W6 | test262 coverage ramp | test262 実行数の段階的拡大（500→2,000→10,000→30,000→53,445）、triage 自動化、unsupported → pass の転換 pipeline | 直接的：実行数を拡大し 90% を測定可能にする。間接的：各 W の progress を可視化 |
| W7 | Host capability boundary | capability manifest 出力、standalone vs host-required 分類、host import の条件化 | host 依存 test262 ケースの分類と対応 |
| W8 | Optimization | typed fast path、packed array、benchmark gate、正確性維持の最適化 | test262 90% 達成後が主対象。pass 率に直接寄与しない |

## Gates

gate は workstream の完了条件ではなく、「次の判断を下すために必要な証拠が揃っているか」を判定する基準である。gate を満たした場合のみ、その先へ進む判断ができる。gate を削除・スキップして進行可能に見せることは禁止する。

最終目標 gate は **H（test262 semantic_pass 90%）** であり、G→H の推移が本プロジェクトの primary success metric である。

| Gate | 前提 workstream | 成立条件 |
|---|---|---|
| A | W0, W1 | 単一ファイル TS/JS が WASI wasm に変換でき、`iwasm` 実行が成功する。`cargo nextest run` がすべて通る |
| B | W0, W1, W2 | curated fixture セット全件で Node.js との stdout 差分がゼロ。differential test が CI で運用されている。test262 limit 500 で UnsupportedSyntax がゼロ |
| C | W1, W7 | `--emit-manifest` が capability manifest JSON を出力する。manifest と実際の wasm import が一致することを検証するテストがある |
| D | W6 | test262 の executed count が 2,000 件以上。`artifacts/coverage/reference-coverage-matrix.md` が最新。`mise run update-coverage-matrix -- --check` が通る |
| E | W2, W3 | test262 の build_pass が 5,000 件以上（~10%）。build_pass と semantic_pass が分離して集計されている |
| F | W1, W7, W5 | standalone 対象プログラムが Node host import なしで動く。`--emit-manifest` で `standalone: true` が出力される。host-deny test が通る |
| G | W4, W5, W6 | test262 semantic_pass が 50% 以上（約 26,700 ファイル）。`artifacts/coverage/reference-coverage-matrix.md` に semantic_coverage% 列が記録されている |
| H | W4, W5, W6 | test262 semantic_pass が 90% 以上（約 48,100 ファイル）。90% 達成をもって本プロジェクトの primary success metric を満たしたと判断する |
