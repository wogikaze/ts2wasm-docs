## Windows 開発環境セットアップ

### 1. 必要ツールのインストール

#### Python 3

```powershell
# python.org から Python 3.10+ をインストール
# https://www.python.org/downloads/

# インストール確認
python --version
```

#### Rust Toolchain

```powershell
# rustup-init.exe をダウンロードして実行
# https://rustup.rs/
# または winget 使用:
winget install Rustlang.Rustup

# cargo-nextest インストール
cargo install cargo-nextest
```

#### Node.js

```powershell
# nodejs.org から LTS 版をインストール
# https://nodejs.org/

# 確認
node --version
npm --version
```

#### Git

```powershell
# git-scm.com からインストール
# https://git-scm.com/

# 確認
git --version
```

#### 追加ツール

```powershell
# jq: https://stedolan.github.io/jq/download/ からダウンロード
# ripgrep: https://github.com/BurntSushi/ripgrep/releases からダウンロード
# wasm-tools:
cargo install wasm-tools
# ast-grep:
cargo install ast-grep
# WAMR (iwasm): https://github.com/bytecodealliance/wasm-micro-runtime/releases から Windows バイナリをダウンロード
```

すべてのツールを PATH に追加すること。

### 2. jsonschema のインストール（重要）

```powershell
python -m pip install jsonschema
```

これは `mise run check agent-state` で必須です。

### 3. リポジトリのクローン

```powershell
git clone <repository-url>
cd ts2wasm
```

### 4. mise task の使用

開発コマンドは `mise` に統一します。

#### 基本コマンド

```powershell
# コードフォーマット
mise run fmt

# Clippy 実行
mise run clippy

# テスト実行
mise run nextest

# 標準ゲート（fmt + issue health + architecture + coverage matrix + tests）
mise run gate

# nextest をスキップした高速ゲート
mise run gate-fast

# ヘルプ表示
mise tasks
```

#### 利用可能なコマンド

- `fmt` - cargo fmt --all --check
- `clippy` - cargo clippy --all-targets -- -D warnings
- `nextest` - cargo nextest run
- `check` - repo smoke。引数付きで個別 check（例: `mise run check issues`）
- `gate` - fmt + issue health + architecture + coverage matrix + nextest
- `gate-fast` - `gate` から nextest を除いた高速ゲート
- `gate-all` - harness/toolchain を含むフルゲート
- `check issues` - issues/ ディレクトリの検証
- `update-issue-index` - issues/index.md の再生成
- `check agent-state` - エージェント状態の検証
- その他多数（`mise tasks` で確認）

### 5. 開発ワークフロー

#### 典型的な開発サイクル

```powershell
# 1. コードフォーマット
mise run fmt

# 2. リンター実行
mise run clippy

# 3. テスト実行
mise run nextest

# 4. フルゲート実行（コミット前）
mise run gate
```

#### Issue 管理

```powershell
# Issue 検証
mise run check issues

# Issue index 更新
mise run update-issue-index

# カバレッジから Issue 生成
mise run gen-issues-from-coverage -- --suite test262
mise run reference-triage -- test262 reference/test262/test/path/to/case.js
```

### 6. 開始前の最終確認

```powershell
# Issue index 検証
mise run update-issue-index -- --check

# Issue health 検証
mise run check issues

# Agent state 検証
mise run check agent-state

# Repo smoke 検証
mise run check

# ファストゲート（テストスキップ）
mise run gate-fast
```

これら5つがすべて通れば開発を開始できます。

#### "bash not found" でスクリプト失敗

実行しようとしている下位スクリプトが bash 依存。Windows では `mise` 経由の task を使用し、完全機能が必要な場合は WSL2 を使用。

### 8. 制限事項

すべての高優先度スクリプトは Python に移行済みで Windows 互換です。

**Python 移行済み（Windows互換）:**

- `check`, `gate`, `gate-fast`, `gate-all`
- `fmt`, `clippy`, `nextest`
- `update-issue-index`, `gen-issues-from-coverage`
- `update-coverage-matrix`, `coverage-report`, `reference-coverage`, `reference-triage`
- `test262`, `test-differential-reporter`
- `benchmark-tracker`, `install-hooks`

旧 `check-*` task は互換 alias として残っています。通常は `mise run check <part>` を使ってください。

test262 とカバレッジレポートの完全機能には WSL2 の使用を検討してください。WSL セットアップはメイン README を参照。

### 9. 代替案: Mise 使用（実験的）

```powershell
# mise インストール
winget install jdx.mise

# mise tasks 使用
mise tasks
mise run fmt
mise run nextest
```

注意: 一部の mise タスクはまだ bash スクリプトを呼び出し、Windows で動作しない可能性があります。
