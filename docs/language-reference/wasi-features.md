# WASI Features Reference

この文書は WASI (WebAssembly System Interface) の機能について、本プロジェクトでの対応方針と実装状況をまとめる。WASI 仕様は [WASI GitHub](https://github.com/WebAssembly/WASI) を正とする。

## 仕様リファレンス

| 仕様 | URL | 用途 |
|---|---|---|
| WASI Main | <https://github.com/WebAssembly/WASI> | WASI メインリポジトリ |
| WASI Preview 1 | <https://github.com/WebAssembly/WASI/blob/main/docs/Preview1.md> | Preview 1 仕様 |
| WASI Preview 2 | <https://github.com/WebAssembly/WASI/blob/main/docs/Preview2.md> | Preview 2 仕様 |
| WASI libc | <https://github.com/WebAssembly/wasi-libc> | WASI libc 実装 |
| WAMR WASI Support | <https://github.com/bytecodealliance/wasm-micro-runtime> | WAMR の WASI 実装 |

## WASI 仕様詳細

### WASI Preview 1 構成

| カテゴリ | 機能 | 実装関連 |
|---|---|---|
| args | コマンドライン引数 | `args_get`, `args_sizes_get` |
| environ | 環境変数 | `environ_get`, `environ_sizes_get` |
| clock | クロック | `clock_res_get`, `clock_time_get` |
| fd | ファイル記述子操作 | `fd_*` 系関数 |
| path | パス経由ファイル操作 | `path_*` 系関数 |
| poll | イベント待機 | `poll_oneoff` |
| proc | プロセス操作 | `proc_exit`, `proc_raise` |
| random | 乱数 | `random_get` |
| sched | スケジューラ | `sched_yield` |
| sock | ソケット | `sock_*` 系関数 |

### WASI ファイル記述子

| fd | 説明 | 実装関連 |
|---|---|---|
| 0 | 標準入力 (stdin) | `fd_read` |
| 1 | 標準出力 (stdout) | `fd_write` |
| 2 | 標準エラー (stderr) | `fd_write` |
| 3+ | ユーザーファイル記述子 | `path_open` 経由 |

### WASI ファイル権限

| 権限 | 説明 | 実装関連 |
|---|---|---|
| `READ` | 読み取り権限 | `fd_read` |
| `WRITE` | 書き込み権限 | `fd_write` |
| `EXECUTE` | 実行権限 | 未実装 |
| `FDSTAT_SET_FLAGS` | フラグ設定 | `fd_fdstat_set_flags` |

### WASI ファイルタイプ

| タイプ | 説明 | 実装関連 |
|---|---|---|
| `UNKNOWN` | 不明 | 未実装 |
| `BLOCK_DEVICE` | ブロックデバイス | 未実装 |
| `CHARACTER_DEVICE` | キャラクタデバイス | 未実装 |
| `DIRECTORY` | ディレクトリ | 未実装 |
| `REGULAR_FILE` | 通常ファイル | 未実装 |
| `SOCKET_DGRAM` | データグラムソケット | 未実装 |
| `SOCKET_STREAM` | ストリームソケット | 未実装 |
| `SYMLINK` | シンボリックリンク | 未実装 |

### WASI エラーコード

| エラー | 説明 | 実装関連 |
|---|---|---|
| `SUCCESS` | 成功 | 正常終了 |
| `2BIG` | 引数リストが長すぎる | 未実装 |
| `ACCES` | アクセス拒否 | 未実装 |
| `ADDRINUSE` | アドレス使用中 | 未実装 |
| `ADDRNOTAVAIL` | アドレス利用不可 | 未実装 |
| `AFNOSUPPORT` | アドレスファミリ非サポート | 未実装 |
| `AGAIN` | 再試行 | 未実装 |
| `ALREADY` | 接続済み | 未実装 |
| `BADF` | 不正なファイル記述子 | 未実装 |
| `BUSY` | デバイス/リソースビジー | 未実装 |
| `CONNREFUSED` | 接続拒否 | 未実装 |
| `CONNRESET` | 接続リセット | 未実装 |
| `DEADLK` | デッドロック | 未実装 |
| `DESTADDRREQ` | 宛先アドレス必須 | 未実装 |
| `DOM` | ドメインエラー | 未実装 |
| `DQUOT` | ディスククォータ超過 | 未実装 |
| `EXIST` | ファイル存在 | 未実装 |
| `FAULT` | 不正アドレス | 未実装 |
| `FBIG` | ファイルサイズ超過 | 未実装 |
| `HOSTUNREACH` | ホスト到達不能 | 未実装 |
| `IDLE` | 割り込み発生 | 未実装 |
| `ILSEQ` | 不正バイト列 | 未実装 |
| `INPROGRESS` | 操作中 | 未実装 |
| `INTR` | 割り込み | 未実装 |
| `INVAL` | 不正引数 | 未実装 |
| `IO` | I/O エラー | 未実装 |
| `ISCONN` | 接続済み | 未実装 |
| `ISDIR` | ディレクトリ | 未実装 |
| `LOOP` | シンボリックリンクループ | 未実装 |
| `MFILE` | 開きすぎ | 未実装 |
| `MLINK` | リンク多すぎ | 未実装 |
| `MSGSIZE` | メッセージサイズ超過 | 未実装 |
| `MULTIHOP` | マルチホップ | 未実装 |
| `NAMETOOLONG` | 名前長すぎ | 未実装 |
| `NETDOWN` | ネットワークダウン | 未実装 |
| `NETRESET` | ネットワークリセット | 未実装 |
| `NETUNREACH` | ネットワーク到達不能 | 未実装 |
| `NFILE` | 開きすぎ | 未実装 |
| `NOBUFS` | バッファなし | 未実装 |
| `NODEV` | デバイスなし | 未実装 |
| `NOENT` | ファイルなし | 未実装 |
| `NOEXEC` | 実行不可 | 未実装 |
| `NOLCK` | ロックなし | 未実装 |
| `NOMEM` | メモリ不足 | 未実装 |
| `NOMSG` | メッセージなし | 未実装 |
| `NOPROTOOPT` | プロトコルオプションなし | 未実装 |
| `NOSPC` | ディスク容量不足 | 未実装 |
| `NOSYS` | 機能未実装 | 未実装 |
| `NOTCONN` | 未接続 | 未実装 |
| `NOTDIR` | ディレクトリでない | 未実装 |
| `NOTEMPTY` | ディレクトリ空でない | 未実装 |
| `NOTRECOVERABLE` | 回復不能 | 未実装 |
| `NOTSOCK` | ソケットでない | 未実装 |
| `NOTSUP` | 非サポート | 未実装 |
| `NOTTY` | 端末でない | 未実装 |
| `NXIO` | デバイスなし | 未実装 |
| `OVERFLOW` | オーバーフロー | 未実装 |
| `OWNERDEAD` | オーナー終了 | 未実装 |
| `PERM` | アクセス拒否 | 未実装 |
| `PIPE` | パイプ破損 | 未実装 |
| `PROTO` | プロトコルエラー | 未実装 |
| `PROTONOSUPPORT` | プロトコル非サポート | 未実装 |
| `PROTOTYPE` | プロトタイプエラー | 未実装 |
| `RANGE` | 範囲外 | 未実装 |
| `ROFS` | 読み取り専用ファイルシステム | 未実装 |
| `SPIPE` | パイプへ書き込み不可 | 未実装 |
| `SRCH` | 検索なし | 未実装 |
| `STALE` | 古いファイルハンドル | 未実装 |
| `TIMEDOUT` | タイムアウト | 未実装 |
| `TXTBSY` | テキストファイルビジー | 未実装 |
| `XDEV` | デバイス跨り | 未実装 |
| `NOTCAPABLE` | ケーパビリティ不足 | 未実装 |

### WASI クロック ID

| ID | 説明 | 実装関連 |
|---|---|---|
| `REALTIME` | 実時間 | 未実装 |
| `MONOTONIC` | 単調増加時間 | 未実装 |
| `PROCESS_CPUTIME_ID` | プロセス CPU 時間 | 未実装 |
| `THREAD_CPUTIME_ID` | スレッド CPU 時間 | 未実装 |

### WASI Event Type

| タイプ | 説明 | 実装関連 |
|---|---|---|
| `FD_READ` | ファイル読み込み可能 | 未実装 |
| `FD_WRITE` | ファイル書き込み可能 | 未実装 |
| `CLOCK` | クロック | 未実装 |

## WASI Preview 1 (標準)

| 機能 | 対応方針 | 実装状況 | 優先度 | Issue ID |
|---|---|---|---|---|
| `args_get` / `args_sizes_get` | コマンドライン引数 | 実装済み (WAMR) | - | - |
| `environ_get` / `environ_sizes_get` | 環境変数 | 実装済み (WAMR) | - | - |
| `clock_res_get` / `clock_time_get` | クロック | 未実装 | P2 | - |
| `fd_advise` | ファイルアクセスアドバイス | 未実装 | P2 | - |
| `fd_allocate` | ファイル領域確保 | 未実装 | P2 | - |
| `fd_close` | ファイル記述子クローズ | 未実装 | P2 | - |
| `fd_datasync` | データ同期 | 未実装 | P2 | - |
| `fd_fdstat_get` / `fd_fdstat_set_flags` | ファイル記述子状態 | 未実装 | P2 | - |
| `fd_filestat_get` / `fd_filestat_set_size` | ファイル状態 | 未実装 | P2 | - |
| `fd_pread` / `fd_pwrite` | ファイル読み書き | 未実装 | P2 | - |
| `fd_prestat_get` / `fd_prestat_dir_name` | preopen 情報 | 未実装 | P2 | - |
| `fd_read` / `fd_write` | ファイル読み書き (fd 0/1/2) | 実装済み (stdin/stdout/stderr) | - | - |
| `fd_readdir` | ディレクトリ読み込み | 未実装 | P2 | - |
| `fd_renumber` | ファイル記述子番号変更 | 未実装 | P2 | - |
| `fd_seek` / `fd_tell` | ファイルシーク | 未実装 | P2 | - |
| `fd_sync` | ファイル同期 | 未実装 | P2 | - |
| `path_create_directory` | ディレクトリ作成 | 未実装 | P2 | - |
| `path_filestat_get` / `path_filestat_set_size` | パス経由ファイル状態 | 未実装 | P2 | - |
| `path_link` | ハードリンク作成 | 未実装 | P2 | - |
| `path_open` | ファイルオープン | 未実装 | P2 | - |
| `path_readlink` | シンボリックリンク読み込み | 未実装 | P2 | - |
| `path_remove_directory` | ディレクトリ削除 | 未実装 | P2 | - |
| `path_rename` | ファイル/ディレクトリ名前変更 | 未実装 | P2 | - |
| `path_symlink` | シンボリックリンク作成 | 未実装 | P2 | - |
| `path_unlink_file` | ファイル削除 | 未実装 | P2 | - |
| `poll_oneoff` | イベント待機 | 未実装 | P2 | - |
| `proc_exit` | プロセス終了 | 未実装 | P2 | - |
| `proc_raise` | シグナル送信 | 未実装 | P2 | - |
| `random_get` | 乱数 | 未実装 | P2 | - |
| `sched_yield` | スケジューラ譲渡 | 未実装 | P2 | - |
| `sock_accept` | ソケット接続受付 | 実装済み (WAMR socket API) | - | - |
| `sock_recv` / `sock_send` | ソケット送受信 | 実装済み (WAMR socket API) | - | - |
| `sock_shutdown` | ソケットシャットダウン | 実装済み (WAMR socket API) | - | - |

## WASI Preview 2 (新世代)

| 機能 | 対応方針 | 実装状況 | 優先度 | Issue ID |
|---|---|---|---|---|
| Component Model 統合 | 型付き host interface | 将来対応 | 将来検討 | - |
| WIT インターフェース | インターフェース定義 | 将来対応 | 将来検討 | - |
| jco WASI Preview 2/3 shim | JS/TS 統合 | 将来対応 | 将来検討 | - |
| 改良されたエラー処理 | エラー型システム | 将来対応 | 将来検討 | - |
| 非同期 I/O | async I/O モデル | 将来対応 | 将来検討 | - |

## WAMR 固有 WASI 拡張

| 機能 | 対応方針 | 実装状況 | 優先度 | Issue ID |
|---|---|---|---|---|
| libc-wasi library (~21.4K) | WASI libc | 実装済み (WAMR) | - | - |
| wasi-threads | POSIX スレッド | 実装済み (WAMR) | - | - |
| Berkeley/Posix Socket | ソケット API | 実装済み (WAMR) | - | - |
| multi-thread | マルチスレッド | 実装済み (WAMR) | - | - |
| AOT / JIT | コンパイルと実行 | 実装済み (WAMR) | - | - |

## Capability Mapping

| WASI 機能 | Capability | standalone |
|---|---|---:|
| `fd_read` (fd 0) | `wasi.stdin` | yes |
| `fd_write` (fd 1) | `wasi.stdout` | yes |
| `fd_write` (fd 2) | `wasi.stderr` | yes |
| `args_get` / `args_sizes_get` | `wasi.args` | yes |
| `environ_get` / `environ_sizes_get` | `wasi.env` | yes |
| `path_open` (read) | `wasi.filesystem.read` | 条件付き |
| `path_open` (write) | `wasi.filesystem.write` | 条件付き |
| `random_get` | `wasi.random` | yes |
| `sock_*` | `wasi.network` | yes (WAMR) |
| wasi-threads | `wasi.threads` | yes (WAMR) |

## 実装方針の原則

1. **WASI Preview 1 優先**: 初期は WASI Preview 1 に対応
2. **WAMR 活用**: WAMR の WASI 実装を活用 (libc-wasi, socket, threads)
3. **最小依存**: 必要な WASI 機能だけを manifest に記録
4. **Preview 2 準備**: 将来的な Preview 2 対応を見据えた設計
5. **Capability ベース**: WASI 機能を capability として管理
