# API and host capability

この文書は、WASI に lowering できる API、Node host が必要な API、host shim trimming、capability manifest を扱う。capability manifest の canonical schema は `docs/11-shared-definitions.md` を参照する。

WASI 機能の詳細な対応状況は `docs/language-reference/wasi-features.md` を参照。

## API 対応方針

API は「standalone で動く API」と「Node.js host が必要な API」に分ける。

standalone で動く API は、WASI と WASM runtime だけで実行する。標準入出力、基本的なファイル読み書き、引数、環境変数、文字列処理、数値処理、配列、Map / Set の基本操作、JSON、TextEncoder / TextDecoder の基礎部分はこの領域に入る。

WAMR は WASI libc-wasi library (~21.4K) を提供し、WASI Preview 1 の標準機能をサポートしている。また multi-thread、socket API (Berkeley/Posix Socket) もサポートしており、ネットワーク機能も WASI 経由で実行可能である。

Node.js 併用 API は、WASI 単体では扱いにくいものを対象にする。`process`、Node 固有の `fs` 挙動、`Buffer`、`crypto`、`path` の細部、イベントループ、タイマー、非同期 I/O などが該当する。ただし、これらも「Node.js で全部処理する」のではなく、WASM 側から必要な host function を呼ぶ形にする。

将来的に Component Model / WIT を採用する場合、jco は WASI Preview 2/3 shim を提供しており、より型付きの host interface が可能になる。wasm-bindgen は Web IDL bindings 提案を目指しており、JavaScript shims を経由せずに直接 DOM メソッドを呼べる可能性がある。

| API 領域  | standalone WASI | Node host 併用 | 方針                                        |
| ------- | --------------: | -----------: | ----------------------------------------- |
| stdio   |              対応 |           不要 | `console.log` も最終的には WASI `fd_write` に落とす |
| argv    |              対応 |           不要 | WASI args を runtime に渡す                   |
| env     |            部分対応 |      最終手段 | WASI environ から runtime facade を初期化し、不足時のみ fallback |
| fs      |            部分対応 | Node 固有挙動は併用 | 同期 API から優先                               |
| path    |            対応可能 | Node 完全互換は併用 | POSIX / Windows 差分を明示管理                   |
| process |            部分対応 |           併用 | `argv` / `env` は WASI 優先、完全互換は host fallback |
| Buffer  |      runtime 実装 |     必要に応じて併用 | `Uint8Array` との関係を明確化                     |
| crypto  |             難しい |           併用 | WASI random と Node crypto を分離             |
| Date live time | 対応 | 不要 | WASI realtime clock を明示 capability として要求 |
| timer   |             難しい |           併用 | event loop 設計後に対応                         |
| network |       WASI 対応可能 |           併用 | WAMR socket API で WASI 経由実行可能          |

| API / idiom                        | 実行方法                                       | Node.js host 必要性 |
| ---------------------------------- | ------------------------------------------ | ---------------: |
| `fs.readFileSync(0, "utf8")`       | WASI `fd_read` + WASM runtime bytes-backed string（stdin 経路の現状実装） |               不要 |
| `fs.readFileSync("/path", "utf8")` | WASI preopen dir 経由の file read             |          条件付きで不要 |
| `console.log(...)`                 | WASI `fd_write`                            |               不要 |
| `process.argv`                     | WASI args                                  |               不要 |
| `process.env`                      | WASI environ                               |               不要 |
| `process.cwd()`                    | WASI だけでは弱い。preopen / host policy 依存       |            場合による |
| `fs.existsSync`, `statSync`        | WASI filesystem API に対応可能                  |             条件付き |
| `path.join`                        | WASM runtime builtin                       |               不要 |
| `Buffer.from`                      | WASM runtime builtin                       |               不要 |
| `Date.now()` / `new Date()`        | WASI `clock_time_get` realtime clock       |               不要 |
| `setTimeout`                       | WASI だけでは不足                                |  Node host などが必要 |
| network                            | WASI Preview 1 では不足                        |         host が必要 |

## WASI-compatible Node Idioms

本プロジェクトでは、Node.js の API 名で書かれているコードであっても、必ず Node.js host を必要とするとは限らない。`fs.readFileSync(0, "utf8")`、`console.log`、`process.argv`、`process.env` のような idiom は、WASI の標準機能に対応付けられるため、Node.js なしで `.wasm` 単体実行できる。

この場合、`require("fs")` を実行時に Node.js の module system へ渡すのではなく、compiler が builtin module として解決する。`readFileSync(0, "utf8")` は WASI `fd_read` に lowering され、現段階では bytes-backed string として返す（UTF-8 decode / UTF-16 semantics は後続対応）。

したがって、次のコードは Node.js host なしで実行できる対象に含める。

```ts
const input = require("fs").readFileSync(0, "utf8");
const nums = input.trim().split(/\s+/).map(Number);

let sum = 0;
for (let i = 0; i < nums.length; i++) {
    sum += nums[i];
}

console.log(sum);
```

このコードの理想的な実行分担はこうなる。

| 処理                        | 実行場所                            |
| ------------------------- | ------------------------------- |
| `require("fs")` の解決       | compile-time builtin resolution |
| `readFileSync(0, "utf8")` | WASI `fd_read` + WASM runtime   |
| `trim`                    | WASM runtime                    |
| `split(/\s+/)`            | WASM runtime                    |
| `map(Number)`             | WASM runtime / 最適化後 inline      |
| `for` loop                | generated WASM                  |
| `console.log`             | WASI `fd_write`                 |

この場合、JavaScript host shim は不要である。必要なのは WASI imports だけで、iwasm がそれを提供する。

> 本プロジェクトは、Node.js 風の API を使った TypeScript コードであっても、WASI に対応可能な idiom は Node.js host に逃がさず、WASI import と WASM runtime に lowering する。`fs.readFileSync(0, "utf8")`、`console.log`、`process.argv`、`process.env` などは standalone WASI 実行の対象とする。JavaScript host shim は、WASI で表現できない API に限って使用する。

## Host Shim Trimming

本プロジェクトでは、host shim を固定の巨大 runtime として配布しない。compiler はソースコードと lowering 結果を解析し、実行に必要な host capability だけを manifest として列挙する。その manifest に基づいて、必要な host shim 関数だけを生成または link する。

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

| 性質          | 内容                              |
| ----------- | ------------------------------- |
| 最小依存        | 使っていない host API は出力に含めない        |
| 監査しやすい      | どの外部 API に依存しているか manifest で分かる |
| iwasm 判定が容易 | Node shim が空なら standalone 実行可能  |
| 性能劣化を防ぐ     | 不要な JS bridge を通らない             |
| セキュリティが強い   | 不要な capability を渡さない            |

manifest の schema、filesystem の粒度、Node host import の表記は `docs/11-shared-definitions.md` を正とする。compiler の内部では、API ごとに capability を割り当てる。

| ソース上の API                    | capability                          | standalone |
| ---------------------------- | ----------------------------------- | ---------: |
| `console.log`                | `wasi.stdout`                       |        yes |
| `console.error`              | `wasi.stderr`                       |        yes |
| `fs.readFileSync(0, "utf8")` | `wasi.stdin`                        |        yes |
| `process.argv`               | `wasi.args`                         |        yes |
| `process.env`                | `wasi.env`                          |        yes |
| `fs.readFileSync(path)`      | `wasi.filesystem.read`              |       条件付き |
| `Date.now` / `new Date()`    | `wasi.clock.realtime`               |        yes |
| `setTimeout`                 | `host.timer`                        |         no |
| `crypto.randomBytes`         | `host.crypto` or `wasi.random`      |       条件付き |
| `fetch`                      | `host.http`                         |         no |

設計としては、host shim trimming は DCE というより capability-based linking に近い。単に使っていない JS 関数を消すだけではなく、そもそもどの外部能力を要求しているかを compiler が把握する。

この方針を次の文で固定する。

> Host shim は monolithic にしない。compiler は必要な host capability を解析し、WASI で表現できる API は WASI import に lowering し、WASI で表現できない API のみを関数単位で Node.js host shim として生成する。未使用の host shim 関数は出力に含めない。

## 追加設計: `process.env` の扱い

`process.env` は、可能な限り WASI environ で対応する。Node.js host fallback は、WASI から作った runtime facade では実装できない挙動に限る。単に `process.env` を参照しただけで Node.js host 必須にしてはいけない。

| API | standalone WASI | Node 完全互換 | 方針 |
|---|---:|---:|---|
| `process.env.FOO` の読み取り | yes | yes | WASI environ から runtime object を初期化 |
| `Object.keys(process.env)` | yes | mostly | runtime object として列挙可能にする |
| `process.env.X = "y"` | partial | yes | runtime 内の facade mutation として扱う |
| OS 環境への mutation 反映 | no | limited | portable semantics としては保証しない |
| `delete process.env.X` | partial | yes | runtime facade から削除し、OS 反映は保証しない |
| getter / setter / Proxy 経由の `process.env` 監視 | no | partial | dynamic object semantics が必要なら host fallback 候補 |

したがって、`process.env` は単純に Node host 必須とはしない。WASI から初期化可能な `process.env` facade と、Node host の完全互換 API を分ける。

### Env lowering rule

`process.env` の lowering は、次の順序で決める。

| Step | 判定 | 出力 |
|---|---|---|
| 1 | `process.env.NAME` / `process.env["NAME"]` の読み取り | `wasi.env` + runtime facade |
| 2 | `Object.keys` / `in` / property enumeration | `wasi.env` + enumerable runtime object |
| 3 | runtime 内だけで完結する代入・削除 | `wasi.env` + mutable runtime facade |
| 4 | OS 環境への mutation 反映を観測するコード | `host.process.env.native_mutation` |
| 5 | Node の `process` object identity や descriptor 互換を要求するコード | `host.process.env.node_compat` |

standalone WASI では、起動時の environ snapshot から通常の JS object 風 facade を作る。読み取り、列挙、runtime-local mutation はこの facade に対する操作として実行する。OS 側の環境変数を書き換えること、親プロセスへ反映すること、Node.js と同じ descriptor や exotic object 挙動を再現することは standalone WASI の保証対象外である。

manifest は `docs/11-shared-definitions.md` の schema に従い、fallback 理由は `capability_reasons` に記録する。

## 追加設計: Date live time の扱い

`new Date()` と `Date.now()` は観測可能な host wall-clock time を読むため、決定的な `new Date(<epoch-ms integer>)` とは別の外部 capability として扱う。`wasm32-wasi` では WASI Preview 1 の realtime clock を標準方針とし、Node host time import は通常の Date live-time 実装には使わない。

| API | capability | import | standalone |
|---|---|---|---:|
| `Date.now()` | `wasi.clock.realtime` | `wasi_snapshot_preview1.clock_time_get` | yes |
| `new Date()` | `wasi.clock.realtime` | `wasi_snapshot_preview1.clock_time_get` | yes |
| future non-WASI target time | `host.time.now_ms` | `host.time.now_ms` | no |

manifest は `docs/11-shared-definitions.md` の schema に従い、`wasi.clock.realtime: true` と `capability_reasons["wasi.clock.realtime"]` を必ず出力する。reason string は source API 名に固定する。

- `Date.now`
- `new Date()`

`wasi.clock.realtime` は standalone WASI capability なので、これだけを理由に `standalone: false` や `node_host.required: true` にしてはいけない。Node host を使う場合は future non-WASI target や event-loop/timer 互換などの別設計として扱い、`host.time.now_ms` のような関数単位 import と理由を manifest に記録する。

host-deny test は live time を外部 capability として扱う。決定的または no-host を主張する fixture では `wasi.clock.realtime` と `clock_time_get` import が存在したら失敗とする。live-time fixture では、manifest reason と実際の wasm import が一致することを確認する。

テストは host clock の正確な値に依存しない。決定的な Date coverage は `new Date(0).getTime()` のような明示 epoch millisecond 入力を使う。live-time coverage は `Date.now()` と no-argument `new Date()` が epoch millisecond の number を返すこと、値が実行前後の host clock window に入ること、manifest/import が明示されることだけを確認する。Node differential は exact timestamp equality ではなく、型・範囲・manifest の構造を比較する。

## Capability manifest checks

| Check | Purpose |
|---|---|
| source API detection | ソース上で使用した API を capability に分類する |
| lowering decision | WASI lowering / runtime builtin / Node host import を決める |
| import verification | 実際の WASM import と manifest の一致を検証する |
| host-deny verification | standalone 対象が Node host import を要求しないことを検証する |
| security audit | filesystem/env/network/crypto/timer capability を一覧化する |
