<!-- このファイルはプロジェクト固有ルールのみを書く。個人/グローバル AI ルール
（言語・確認スタイル・出力フォーマット等）は各 AI ツールのグローバル設定へ。
fresh public clone でも有効な内容に保つこと。 -->

# claude-code-context-diet 開発ガイド

## プロジェクト概要

Claude Code の常駐 context を実測ベースで減らすための**手順集と配布用スキル**。
「`/context` を開いたら、何もしていないのに 80k tokens 食っていた」という状態から
削る作業を再現可能にしたもの。元になった検証では総量を 80.7k → 69.5k tokens
（-11.2k / -13.9%）まで減らした。

コードを持たないドキュメント／スキル配布リポジトリ。利用者は `SKILL.md` を
`~/.claude/skills/context-diet/SKILL.md` として配置して使う。

## このリポジトリの `SKILL.md` について（重要）

**ルートの `SKILL.md` は配布物であって、このリポジトリの開発ルールではない。**
利用者の環境へコピーされて動く Claude Code スキルの実体なので、次を守る:

- 作者環境固有の絶対パス・個人情報・kb 由来の固有名詞を書かない。**第三者の環境で
  そのまま動く**ことが成立条件
- frontmatter の `name` / `description` は起動語の正本。変更すると利用者の発火条件が
  変わるので、README の説明と必ず同時に更新する
- 本ファイル（`CLAUDE.md`）と混同しない。`CLAUDE.md` は**このリポジトリを開発する
  AI 向け**、`SKILL.md` は**利用者の環境で動く配布物**

## やらないこと（スコープ外）

- **Claude Code 以外の AI CLI 向けの手順追加。** 対象を広げると実測値の根拠が薄まる。
  他ツールは別リポの領分
- 自動化ツール・スクリプトの実装。ここは「手順と判断基準」を配るリポで、実行系は持たない
- 実測していない削減テクニックの追記。**数値の裏付けが無いものは載せない**のが
  このリポの価値の源泉
- 検証環境に依存する設定値の断定。Claude Code のバージョンで変わるものは、確認した
  版を併記する

## 技術スタック

| レイヤ | 採用 |
|---|---|
| 実体 | Markdown のみ（ビルド・実行系なし） |
| 配布 | GitHub（clone または raw URL で `SKILL.md` を取得） |
| ライセンス | MIT |

## ディレクトリ構成

- `SKILL.md` — 配布用スキル本体（利用者が `~/.claude/skills/context-diet/` へ置く）
- `docs/playbook.md` — 詳細手順
- `docs/settings-keys-reference.md` — `settings.json` で実在する context 削減キーの一覧
- `examples/claude-md-split-pattern.md` — CLAUDE.md を分割する具体パターン
- `scripts/` — secrets-scan と hook インストーラ
- `.githooks/` — layer 2 pre-commit
- `.github/workflows/` — layer 3 CI

## 主要コマンド

ビルドもテストもない。編集して push するだけ。

- secrets-scan 手動実行: `node scripts/secrets-scan.mjs --staged --block`

## AI 作業共通ルール

ビルド・コミット禁止、secrets-scan 責務、plan/bugfix/pending md の作成ルール等の AI 作業共通ルールは、各利用者のグローバル AI 設定に従う（作者環境の例: `~/.claude/CLAUDE.md` および `~/.claude/guides/`）。

このリポジトリ固有:

- **数値を書き換えるときは出典を確認する。** README と記事に出ている
  「80.7k → 69.5k / -13.9%」は実測値。別の環境の数字で上書きしない
- **「`disabledTools` は存在しない」のような否定形の事実は、確認した Claude Code の版と
  セットで書く。** 版が上がって実在するようになったら、その時点で訂正が要る
- `SKILL.md` を変更したら README の説明表も同時に直す（起動語と用途が 2 箇所にある）

## secrets-scan（このリポジトリの配線）

書く瞬間の責務（固有名詞の一般化・fixture は合成データ等）は上記「AI 作業共通ルール」の参照先に従う。このリポジトリ固有の配線は以下:

- scanner: `scripts/secrets-scan.mjs`（手動実行: `node scripts/secrets-scan.mjs --staged --block`）
- layer 2: `.githooks/pre-commit`（`core.hooksPath = .githooks` で有効化済み。第三者 clone 時は `bash scripts/install-hooks.sh` または `pwsh scripts/install-hooks.ps1`）
- layer 3: `.github/workflows/secrets-scan.yml`
- env（full coverage に必要・未設定なら構造 regex のみで継続）: `KB_ROOT` / `FAMILY_ROOT`

**このリポは context 削減の実例として `settings.json` や `CLAUDE.md` の断片を載せる。**
作者環境の実物をそのまま貼ると個人パスや固有名詞が混ざるので、掲載前に必ず
一般化すること。scanner はその最後の砦であって、一次防御ではない。

## 関連ドキュメント

| 項目 | パス |
|---|---|
| ユーザー向け README | `README.md` |
| 配布用スキル本体 | `SKILL.md` |
| 詳細手順 | `docs/playbook.md` |
| Codex/他 AI 用入口 | `AGENTS.md` |
| 背景記事（Zenn） | https://zenn.dev/ishizakahiroshi/articles/20260622-claude-code-context-diet |
