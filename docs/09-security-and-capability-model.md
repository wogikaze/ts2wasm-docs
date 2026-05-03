# Security and capability model

この文書は host capability、WASI preopen、Node host shim、manifest 監査を扱う。capability manifest の canonical schema は `docs/11-shared-definitions.md` に置き、この文書では security policy と audit 観点を定義する。

## Security premise

このプロジェクトでは、Node.js を実行本体ではなく host API provider として扱う。外部能力は暗黙に渡さず、compiler が使用 API を解析し、manifest に明示された capability だけを link する。

## Capability categories

| Capability | Example | Default |
|---|---|---|
| `wasi.stdin` | `fs.readFileSync(0, "utf8")` | used when source reads fd 0 |
| `wasi.stdout` | `console.log` | used when source writes stdout |
| `wasi.stderr` | `console.error` | used when source writes stderr |
| `wasi.args` | `process.argv` | used when source reads argv |
| `wasi.env` | `process.env` read/enumeration | used when source reads env |
| `wasi.filesystem.read` | `fs.readFileSync(path)` | requires preopen policy |
| `wasi.filesystem.write` | file write | requires explicit permission |
| `wasi.random` | random bytes via WASI | requires random policy |
| `wasi.network` | socket API (Berkeley/Posix Socket) | WAMR で WASI 経由実行可能 |
| `wasi.threads` | multi-thread / wasi-threads | WAMR で WASI 経由実行可能 |
| `host.timer` | `setTimeout` | Node host required |
| `host.crypto` | `crypto.randomBytes` | Node host or WASI random policy required |
| `host.http` | `fetch` / network | Node host or future WASI capability required |

## Filesystem policy

| Case | Policy |
|---|---|
| fd 0 read | `wasi.stdin` |
| preopened file read | `wasi.filesystem.read` + preopen check |
| arbitrary absolute path | default deny |
| Node host file read | function-level import, not full `fs` object |
| write access | read とは別 capability |

Filesystem capability は read/write/preopen を分ける。`read` が許可されても `write` は許可しない。preopen なしの absolute path は portable semantics と security の両面で default deny とする。

## Environment policy

| Case | Policy |
|---|---|
| read env | WASI environ から runtime object を作る |
| enumerate env | manifest に `wasi.env` を記録 |
| mutate runtime env object | runtime-local mutation |
| mutate host OS env | portable semantics として保証しない |

`process.env` は単純に Node host 必須とはしない。WASI environ から初期化できる facade と、Node host の完全互換 API を分ける。

## Manifest audit

Generated artifact must be auditable. Manifest の schema と例は `docs/11-shared-definitions.md` の capability manifest schema を正とする。

Audit では、少なくとも次を確認する。

| Check | Purpose |
|---|---|
| source API detection | ソース上で使用した API を capability に分類する |
| lowering decision | WASI lowering / runtime builtin / Node host import を決める |
| import verification | 実際の WASM import と manifest の一致を検証する |
| host-deny verification | standalone 対象が Node host import を要求しないことを検証する |
| preopen verification | filesystem access が許可された preopen 内に収まることを検証する |
| security review | filesystem/env/network/crypto/timer capability を一覧化する |

## Manifest CLI output（canonical）

`--emit-manifest` は `docs/11-shared-definitions.md` の canonical capability manifest schema を出力する。`--emit-capabilities` は同一 output を返す deprecated alias である。

```bash
ts2wasm build input.ts -o out.wasm --emit-manifest out.manifest.json
```

例:

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

- `schema_version`: `1` 固定
- `target`: 実行ターゲット識別子
- `standalone`: Node host import が不要なら `true`
- `wasi.*`: WASI capability と filesystem の read/write/preopen 区分
- `node_host`: Node host が必要な場合の関数単位 import
- `capability_reasons`: capability を必要としたソース API の根拠

## Host shim trimming

host shim は固定の巨大 runtime として配布しない。compiler はソースコードと lowering 結果を解析し、実行に必要な host capability だけを manifest として列挙する。その manifest に基づいて、必要な host shim 関数だけを生成または link する。

WASI に lowering できる API は Node.js host shim に含めない。たとえば `console.log`、`fs.readFileSync(0, "utf8")`、`process.argv`、`process.env` は standalone WASI execution の対象であり、Node.js shim を要求しない。

Node.js host shim は、WASI では表現できない API に限って使う。さらに、その場合でも `fs` 全体、`process` 全体、`node:*` 全体をまとめて import するのではなく、必要な関数単位で import する。

```text
bad:
  import node_fs_all
  import node_process_all
  import node_crypto_all

good:
  import host.timer.setTimeout
  import host.crypto.randomBytes
  import host.fs.watch
```

これにより、生成物は次の性質を持つ。

| 性質 | 内容 |
|---|---|
| 最小依存 | 使っていない host API は出力に含めない |
| 監査しやすい | どの外部 API に依存しているか manifest で分かる |
| iwasm 判定が容易 | Node shim が空なら standalone 実行可能 |
| 性能劣化を防ぐ | 不要な JS bridge を通らない |
| セキュリティが強い | 不要な capability を渡さない |

## API capability mapping

| ソース上の API | capability | standalone |
|---|---|---:|
| `console.log` | `wasi.stdout` | yes |
| `console.error` | `wasi.stderr` | yes |
| `fs.readFileSync(0, "utf8")` | `wasi.stdin` | yes |
| `process.argv` | `wasi.args` | yes |
| `process.env` | `wasi.env` | yes |
| `fs.readFileSync(path)` | `wasi.filesystem.read` | 条件付き |
| `setTimeout` | `host.timer` | no |
| `crypto.randomBytes` | `host.crypto` or `wasi.random` | 条件付き |
| `fetch` | `host.http` | no |

Host shim trimming は DCE というより capability-based linking に近い。単に使っていない JS 関数を消すだけではなく、どの外部能力を要求しているかを compiler が把握する。
