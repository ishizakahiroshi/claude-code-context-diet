# Agent Entry Point (claude-code-context-diet)

このリポジトリの運用ガイダンスは `CLAUDE.md` を正本とする。

- プロジェクト概要・ルール: `./CLAUDE.md`
- ユーザー向けドキュメント: `./README.md`
- 配布用スキル本体: `./SKILL.md`（**このリポの開発ルールではなく配布物**）
- ローカル/プライベート追記（存在する場合・コミットしない）: `./CLAUDE.local.md` / `./AGENTS.local.md` / `./docs/local/`

個人/グローバル AI ルールは意図的にこのリポジトリの外に置く。各 AI ツールの
グローバル設定を使うこと。本ファイルは fresh public clone でも有効に保つ。

## Non-negotiables (full detail in CLAUDE.md)

- **ルートの `SKILL.md` は利用者の環境へコピーされて動く配布物。** 作者環境固有の
  絶対パス・個人情報・kb 由来の固有名詞を書かない。`CLAUDE.md`（このリポを開発する
  AI 向け）と混同しない
- **実測していない削減テクニックを載せない。** 数値の裏付けが無いものを足すと、この
  リポの価値そのものが失われる
- **否定形の事実（`disabledTools` は存在しない 等）は、確認した Claude Code の版と
  セットで書く。** 版が上がれば訂正が要る性質のものだから
- 実例として設定ファイルの断片を載せるときは、作者環境の実物をそのまま貼らず一般化する
- ビルド・コミット禁止、secrets-scan 責務、plan/bugfix/pending md の作成ルール等の AI 作業共通ルールは、各利用者のグローバル AI 設定に従う（作者環境の例: `~/.claude/CLAUDE.md` および `~/.claude/guides/`）
- secrets-scan のこのリポジトリの配線（scanner パス・手動実行コマンド等）は `CLAUDE.md` の「secrets-scan（このリポジトリの配線）」節を参照

ガイダンス間で矛盾が出たら `CLAUDE.md` を優先する。

<!-- many-ai-cli の承認マーカーブロックはここに自動注入される。本ファイルでは持たない。 -->
