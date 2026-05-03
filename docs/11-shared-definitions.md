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

本プロジェクトは以下の workstream を並行して進める。gate 判定は各 workstream が独立して進められるが、下流 workstream は上流の gate を前提とする。

| ID | Name | 概要 |
|---|---|---|
| W0 | Runtime substrate | linear memory、value representation (RawValue)、WAT/wasm emission の基盤。iwasm で実行可能な最小 wasm を出せる状態を維持する |
| W1 | Standalone WASI execution | console.log / fd_write / stdin の WASI lowering。Node host 不要で実行できる最小プログラムを通す |
| W2 | JS semantic core | truthiness、`===`、`+`、number/string semantics、operator 優先度、関数呼び出し、`undefined`/`null` の JS 意味論 |
| W3 | Data model | object literal、array、property get/set、string heap object、length、index |
| W4 | Control / module / class | closure、exception、destructuring、module import/export、class / prototype |
| W5 | Differential testing and coverage | test262 / TypeScript corpus の executed ramp、Node differential、status schema 運用、`artifacts/coverage/reference-coverage-matrix.md` の継続更新 |
| W6 | Host capability boundary | capability manifest 出力、standalone vs host-required 分類、host import の条件化 |
| W7 | Optimization | typed fast path、packed array、benchmark gate、correctness-preserving 最適化 |

## Gates

gate は workstream の完了条件ではなく、「次の判断を下すために必要な証拠が揃っているか」を判定する基準である。gate を満たした場合のみ、その先へ進む判断ができる。gate を削除・スキップして進行可能に見せることは禁止する。

| Gate | 前提 workstream | 成立条件 |
|---|---|---|
| A | W0, W1 | 単一ファイル TS/JS が WASI wasm に変換でき、`iwasm` 実行が成功する。`cargo nextest run` がすべて通る |
| B | W0, W1, W2 | curated fixture セット全件で Node.js との stdout 差分がゼロ。differential test が CI で運用されている |
| C | W1, W6 | `--emit-manifest` が capability manifest JSON を出力する。manifest と実際の wasm import が一致することを検証するテストがある |
| D | W5 | test262 の executed count が 100 件以上。`artifacts/coverage/reference-coverage-matrix.md` が最新。`mise run update-coverage-matrix -- --check` が通る |
| E | W2, W3, W5 | test262 の build-pass count が 50 件以上、かつ semantic-pass (Node differential 一致) count が 20 件以上。build-pass と semantic-pass が分離して集計されている |
| F | W1, W3, W6 | standalone 対象プログラムが Node host import なしで動く。`--emit-manifest` で `standalone: true` が出力される。host-deny test が通る |
| G | W7 | benchmark suite が固定されている。`mise run benchmark-tracker` が median / p95 / wasm_size を記録する。前回比で regression が出た場合に gate が落ちる |
