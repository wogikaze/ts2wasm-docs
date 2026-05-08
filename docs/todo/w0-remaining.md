# W0 (Runtime substrate) — remaining work

Last updated: 2026-05-08

## Gate A precondition

Gate A: `cargo nextest run` がすべて通ること。

- [ ] **Frontend `ParsedBindingPattern` span field** — `crates/frontend/src/parser.rs:55` の struct に `span: Span` フィールドがなく、`statements_class.rs` / `statements_ts.rs` で構築時にエラー。他 agent の変更との整合で 18 errors 残存。
- [ ] **残存 dirty files の整理** — `git status --short` に他 agent 由来の未コミット変更多数。`cargo nextest run` を通すにはこれらも解決が必要。

## W0 契約: iwasm で実行可能な最小 wasm を出せる状態を維持する

- [ ] **Build gate の常時監視** — 再発防止として merge/revert 時の IR 型変更と backend コードの整合チェックを CI または pre-merge hook に入れる。

## Architecture debt

- [ ] **Binary WASM emitter 拡充** — `crates/backend-wasm/src/binary_mvp.rs` (263行) は console.log hello 程度のプログラムしか扱えない。WAT テキスト形式への過剰依存を減らし、プロダクション品質の wasm binary emitter を整備する。
- [ ] **Runtime core の raw WAT 削減** — `runtime_core_emitter_part1.rs` (1305行) + `part2.rs` (1535行) の巨大 WAT template を typed writer に置換する（`docs/12-coding-standard.md` §1.2 の禁止事項に準じる）。
- [ ] **ABI 二重表現の解消** — 論理 ABI (`jsval` = `i64`, `AbiType::JsVal`) と wire 表現 (`i32` tagged RawValue) の乖離。実際の wasm import/export は `i32` で行われており、`i64` 論理 ABI はコード上で未使用。backend 層で明示的な bridge を定義し、テストで固定する。

## Reference

- Gate definitions: `docs/11-shared-definitions.md`
- W0 definition: `docs/11-shared-definitions.md` §Workstreams
- Commit `60ffe717`: backend span fix (46 insertions, 11 deletions)
