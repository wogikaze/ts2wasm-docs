# WebAssembly Features Reference

この文書は WebAssembly の提案・機能について、本プロジェクトでの対応方針と実装状況をまとめる。WebAssembly 仕様は [WebAssembly Spec](https://github.com/WebAssembly/spec) を正とする。

## 仕様リファレンス

| 仕様 | URL | 用途 |
|---|---|---|
| WebAssembly Spec | <https://github.com/WebAssembly/spec> | 言語仕様の正典 |
| WebAssembly Proposals | <https://github.com/WebAssembly/proposals> | 提案段階の機能 |
| wasm-tools | <https://github.com/bytecodealliance/wasm-tools> | ツールチェーンと提案実装 |

## WebAssembly 仕様詳細

### WebAssembly Core Spec 構成

| セクション | 内容 | 関連機能 |
|---|---|---|
| Introduction | 仕様概要 | 全体像 |
| Structure | モジュール構造 | モジュールフォーマット |
| Values | 値と型 | 値型 |
| Instructions | 命令セット | 命令 |
| Modules | モジュール | モジュール定義 |
| Validation | 検証 | 型検証 |
| Execution | 実行 | 実行セマンティクス |
| Appendix | 付録 | 付録情報 |

### WebAssembly モジュール構造

| セクション | 説明 | 実装関連 |
|---|---|---|
| `type` | 関数型定義 | 関数シグネチャ |
| `import` | インポート | 外部依存 |
| `function` | 関数定義 | 関数インデックス |
| `table` | テーブル定義 | 関数テーブル |
| `memory` | メモリ定義 | 線形メモリ |
| `global` | グローバル定義 | グローバル変数 |
| `export` | エクスポート | 公開インターフェース |
| `start` | 開始関数 | 初期化関数 |
| `elem` | 要素セクション | テーブル初期化 |
| `data` | データセクション | メモリ初期化 |
| `code` | コードセクション | 関数本体 |

### WebAssembly 値型

| 型 | 説明 | 実装関連 |
|---|---|---|
| `i32` | 32-bit 整数 | 整数演算 |
| `i64` | 64-bit 整数 | 整数演算 |
| `f32` | 32-bit 浮動小数点 | 浮動小数点演算 |
| `f64` | 64-bit 浮動小数点 | 浮動小数点演算 |
| `funcref` | 関数参照 | 関数テーブル |
| `externref` | 外部参照 | ホストオブジェクト |

### WebAssembly 命令カテゴリ

| カテゴリ | 命令例 | 実装関連 |
|---|---|---|
| 制御フロー | `block`, `loop`, `if`, `br`, `br_if`, `br_table`, `return` | 制御構造 |
| パラメータ操作 | `local.get`, `local.set`, `local.tee`, `global.get`, `global.set` | 変数アクセス |
| 数値演算 | `i32.add`, `i32.sub`, `i32.mul`, `f32.add`, etc. | 演算 |
| ビット演算 | `i32.and`, `i32.or`, `i32.xor`, `i32.shl`, etc. | ビット操作 |
| 比較 | `i32.eq`, `i32.lt`, `f32.eq`, etc. | 比較 |
| 変換 | `i32.wrap_i64`, `f32.convert_i32`, etc. | 型変換 |
| メモリ操作 | `memory.size`, `memory.grow`, `i32.load`, `i32.store`, etc. | メモリアクセス |
| テーブル操作 | `table.get`, `table.set`, `table.size`, `table.grow` | テーブルアクセス |
| 関数呼び出し | `call`, `call_indirect` | 関数呼び出し |
| 定数 | `i32.const`, `i64.const`, `f32.const`, `f64.const` | 定数 |

### WebAssembly 実行モデル

| コンポーネント | 説明 | 実装関連 |
|---|---|---|
| Store | グローバル状態 | モジュール、メモリ、テーブル |
| Module | モジュール定義 | 静的定義 |
| Instance | モジュールインスタンス | 実行時インスタンス |
| Frame | スタックフレーム | 関数呼び出しスタック |
| Stack | 値スタック | オペランドスタック |
| Memory | 線形メモリ | バイト配列 |
| Table | 関数テーブル | 関数参照配列 |

### WebAssembly 検証

| 検証項目 | 説明 | 実装関連 |
|---|---|---|
| 型整合性 | 命令の型チェック | コンパイル時検証 |
| 制御フロー整合性 | ブロックの型チェック | 構造検証 |
| ローカル整合性 | ローカル変数の型チェック | 変数検証 |
| グローバル整合性 | グローバル変数の型チェック | グローバル検証 |
| メモリ整合性 | メモリアクセスの境界チェック | 実行時検証 |
| テーブル整合性 | テーブルアクセスの境界チェック | 実行時検証 |

## Stage 4+ 提案（wasm-tools 実装済み）

| 提案 | Stage | 対応方針 | 実装状況 | 優先度 | Issue ID |
|---|---|---|---|---|---|
| annotations | Stage 4 | デバッグ情報 | 未実装 | P3 | - |
| branch-hinting | Stage 4 | 分岐ヒント | 未実装 | P3 | - |
| bulk-memory | Stage 4 | メモリ操作 | 未実装 | P2 | - |
| component-model | (例外) | Component Model / WIT | 将来対応 | 将来検討 | - |
| exception-handling | Stage 4 | 例外処理 | 未実装 | P2 | - |
| extended-const | Stage 4 | 定数拡張 | 未実装 | P3 | - |
| extended-name-section | (例外) | 名前セクション拡張 | 未実装 | P3 | - |
| function-references | Stage 4 | 関数参照 | 未実装 | P2 | - |
| gc | Stage 4 | ガベージコレクション | 将来対応 | 将来検討 | - |
| memory64 | Stage 4 | 64-bit メモリ | 未実装 | P3 | - |
| multi-memory | Stage 4 | 複数メモリ | 未実装 | P3 | - |
| multi-value | Stage 4 | 複数戻り値 | 未実装 | P2 | - |
| mutable-global | Stage 4 | 可変グローバル | 未実装 | P2 | - |
| reference-types | Stage 4 | 参照型 | 未実装 | P2 | - |
| relaxed-simd | Stage 4 | SIMD 拡張 | 未実装 | P3 | - |
| saturating-float-to-int | Stage 4 | 浮動小数点→整数変換 | 未実装 | P3 | - |
| sign-extension-ops | Stage 4 | 符号拡張 | 未実装 | P3 | - |
| simd | Stage 4 | SIMD | 未実装 | P3 | - |
| tail-call | Stage 4 | 末尾再帰最適化 | 未実装 | P3 | - |
| threads | Stage 4 | スレッド | WAMR で WASI 経由実行可能 | P2 | - |
| wat-numeric-values | (例外) | WAT 数値表現 | 未実装 | P3 | - |

## Stage 4 未満提案（wasm-tools 実装済み）

| 提案 | Stage | 対応方針 | 実装状況 | 優先度 | Issue ID |
|---|---|---|---|---|---|
| custom-page-sizes | Stage 3 | カスタムページサイズ | 未実装 | P3 | - |
| memory-control | Stage 3 | メモリ制御 | 未実装 | P3 | - |
| shared-everything-threads | Stage 3 | 共有スレッド | 未実装 | P3 | - |
| stack-switching | Stage 2 | スタック切り替え | 未実装 | 将来検討 | - |
| wide-arithmetic | Stage 1 | 広域演算 | 未実装 | 将来検討 | - |

## Core WebAssembly (MVP)

| 機能 | 対応方針 | 実装状況 | 優先度 | Issue ID |
|---|---|---|---|---|
| 値型 (i32, i64, f32, f64) | 基本型 | 実装済み | - | - |
| 制御フロー (if, block, loop, br) | 制御構造 | 実装済み | - | - |
| 関数呼び出し (call, call_indirect) | 関数呼び出し | 実装済み | - | - |
| ローカル変数 (local.get, local.set, local.tee) | 変数 | 実装済み | - | - |
| グローバル変数 (global.get, global.set) | グローバル | 未実装 | P2 | - |
| メモリ操作 (memory.load, memory.store) | メモリアクセス | 実装済み | - | - |
| 線形メモリ (memory) | メモリ管理 | 実装済み | - | - |
| テーブル (table) | 関数テーブル | 未実装 | P2 | - |
| インポート/エクスポート (import, export) | モジュール境界 | 実装済み | - | - |
| 開始関数 (start) | 初期化 | 未実装 | P2 | - |
| データセクション (data) | 静的データ | 未実装 | P2 | - |
| 要素セクション (elem) | 静的テーブル初期化 | 未実装 | P2 | - |

## Component Model / WIT

| 機能 | 対応方針 | 実装状況 | 優先度 | Issue ID |
|---|---|---|---|---|
| Component Model | 型付き host interface | 将来対応 | 将来検討 | - |
| WIT (WebAssembly Interface Types) | インターフェース定義 | 将来対応 | 将来検討 | - |
| jco transpile | JS/TS → Components | 将来対応 | 将来検討 | - |
| WASI Preview 2/3 | 新世代 WASI | 将来対応 | 将来検討 | - |

## 実装方針の原則

1. **段階的対応**: MVP → Stage 4+ 提案 → Stage 4 未満提案の順で対応
2. **iwasm 互換**: 初期は WAMR (iwasm) で動く core wasm を優先
3. **Component Model 準備**: 将来的な Component Model 対応を見据えた IR 設計
4. **wasm-tools 活用**: 既存ツールチェーンの提案実装を参照
5. **WASI 統合**: WASI Preview 1 との統合を優先
