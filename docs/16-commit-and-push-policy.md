# Commit and Push Policy

## 基本方針

コミットは「戻しやすい論理単位」で行う。

巨大な一括コミットは禁止する。
実装、テスト、issue更新、docs更新、format差分、generated差分は、可能な限り分ける。

## issues 実装時の commit 方針

issue を実装する場合、以下の単位でコミットする。

1. acceptance checklist の 1 項目を満たした時
2. テストが前進した時
3. 診断・fixture・coverage などの結果が改善した時
4. 実装の論理単位が完了した時
5. issue status / progress note / evidence を更新した時

原則:

- 1 commit = 1 check または 1 logical step
- 複数の acceptance criteria をまとめて満たした場合でも、分けられるなら分ける
- test-only commit と implementation commit は可能なら分ける
- format-only commit は混ぜない
- generated file update は可能なら分ける
- failing state を commit しない
- ただし、調査・監査・issue整理だけの commit は許可する

commit message 例:

```text
issue-023: add manifest validation for wasm imports
issue-023: add fixture for missing fd_write capability
issue-023: mark AC2 complete with evidence
coverage: update reference matrix after M6 stdin cases
issues: add follow-up for unsupported dynamic import
format: run cargo fmt
```

## issue 追加・更新時の commit 方針

issue の追加・更新は、ファイル作成・更新ごとにこまめにコミットする。

対象:

- `issues/open/*.md`
- `issues/done/*.md`
- `issues/index.md`
- `issues/dependency-graph.md`
- audit note
- progress note
- checklist tracking issue
- future work issue
- reopened issue

原則:

- 新規 issue 作成ごとに commit
- issue 移動ごとに commit
- index / dependency graph 同期は同じ commit に含めてよい
- format / generated sync が挟まった場合は定期的に commit
- issue 本文更新と product implementation は混ぜない

commit message 例:

```text
issues: add coverage breakdown issue for test262 builtins
issues: reopen 014 after false-done audit
issues: sync index after reopening 014
issues: add checklist tracking items for release validation
```

## docs / workflow / skill 更新時の commit 方針

docs、workflow、skill は以下の単位で分ける。

- docs の意味単位ごと
- workflow rule 追加ごと
- skill 追加ごと
- skill 改善ごと
- agent prompt 追加ごと
- review checklist 追加ごと

commit message 例:

```text
docs: define commit and push policy
workflow: add false-done audit post-wave rules
skills: add issue-state-sync workflow
skills: tighten gatekeeper review checklist
```

## format / generated files

format 差分は定期的に commit する。

原則:

- format-only commit にする
- generated-only commit にする
- implementation commit に混ぜない
- ただし、対象ファイルが少なく、明らかに同一作業に属する場合のみ同梱可

commit message 例:

```text
format: run cargo fmt
format: normalize markdownlint output
generated: update issue dependency graph
generated: refresh coverage matrix
```

## push 方針

commit がある程度溜まったら自動 push する。

目安:

- 5〜10 commits 溜まったら push
- 1 issue が完了したら push
- 重要な false-done audit / issue reopen が終わったら push
- 大きな refactor の安全な中間点に到達したら push
- 作業 wave が終わったら push
- 長時間作業の前に、既存の clean な成果を push

push 前の必須条件:

```bash
git status --short
mise run discord-report -- reports/runs/<run_id>/cycle_report.md --run-id <run_id>
```

確認事項:

- 意図しない差分がない
- staged / unstaged が整理されている
- 秘密情報が含まれていない
- 明らかな壊れた中間状態ではない
- 必要な軽量 gate が通っている
- webhook 送信が成功している

webhook 送信ルール:

- push 前に必ず `mise run discord-report` で webhook に送信する。
- Discord 送信用レポートは非常に簡潔にする（状態、issue ID、検証、blocker、次アクションのみ）。
- Discord 送信用レポート本文は日本語で書く（コマンド、パス、issue ID の英字は可）。
- `未記入` だらけのレポートは `discord-report` が reject する。
- Discord 制限に近い payload は `discord-report` が自動的に 2 通へ分割して送信する。
- `reports/` は git 追跡しないローカル生成物として扱う。
- `.md` / `.json` ファイルを `discord-report` で送信した場合、送信済み registry に記録し、同じファイルの再送をエラーにする。
- `.githooks/pre-push` は gate 成功後に pre-push report を生成して webhook 送信し、送信失敗時は push を止める。
- `.githooks/pre-push` や push 前 gate が失敗した場合、`git push --no-verify` などで bypass してはならない。既知 baseline に見える失敗でも、修正するか blocker として報告して push を止める。
- `DISCORD_WEBHOOK_URL` は環境変数または `.env` で設定する。
- webhook URL は secret として扱い、commit しない。

push 前に可能なら実行:

```bash
cargo fmt --all --check
cargo nextest run
mise run check issues
```

ただし、重い gate は毎回必須にしない。
重い gate は issue 完了時、push前の節目、または CI 前提の確認時に実行する。

## push 禁止条件

以下の場合は自動 push しない。

- secret / token / credential の混入が疑われる
- webhook 送信に失敗している、または `DISCORD_WEBHOOK_URL` が未設定
- pre-push hook / push 前 gate が失敗している
- merge conflict が残っている
- working tree に分類不能な差分がある
- failing test を既知の失敗として記録していない
- force push が必要
- remote branch の履歴を書き換える必要がある
- user が明示的に push 禁止している

## force push 方針

自動 force pushは禁止。

許可されるのは、ユーザーが明示した場合のみ。

禁止:

```bash
git push --force
```

明示許可がある場合のみ:

```bash
git push --force-with-lease
```

## autonomous agent rule

agent は以下を守る。

1. 実装を進めたら、小さい論理単位で commit する
2. acceptance checklist の 1 項目が完了したら commit する
3. テストが前進したら commit する
4. issue / index / dependency graph を更新したら commit する
5. format / generated 差分を溜めすぎない
6. 5〜10 commits 程度溜まったら push する
7. push 前に `git status --short` を確認する
8. force push はしない
9. dirty tree を理由に停止せず、差分を分類して commit または明示的に保留する
10. product implementation と orchestration-state diff を混ぜない

## 推奨 commit 粒度

| 作業                   | commit 粒度                            |
| -------------------- | ------------------------------------ |
| issue implementation | 1 acceptance check ごと                |
| test improvement     | テスト前進ごと                              |
| fixture追加            | fixture group ごと                     |
| diagnostics改善        | diag code / behavior ごと              |
| docs更新               | 意味単位ごと                               |
| issue追加              | issue file 作成ごと                      |
| issue移動              | reopen / close ごと                    |
| issue close の一括禁止 | close 理由の異なる issue を 1 commit にまとめない。実装が伴わない close（既に実装済み/調査のみ）も close ごとに分ける。 |
| index同期              | issue移動と同じ commit か generated commit |
| format               | format-only                          |
| generated            | generated-only                       |
| refactor             | API境界 / module境界ごと                   |
