<!-- このファイルはプロジェクト固有ルールのみを書く。個人/グローバル AI ルール
（言語・確認スタイル・出力フォーマット等）は各 AI ツールのグローバル設定へ。
fresh public clone でも有効な内容に保つこと。 -->

# claude-code-context-diet 開発ガイド

## プロジェクト概要

Claude Code の常駐 context を実測ベースで減らすための**手順集と配布用スキル**。
「`/context` を開いたら、何もしていないのに 80k tokens 食っていた」という状態から
削る作業を再現可能にしたもの。元になった検証では総量を 80.7k → 69.5k tokens
（-11.2k / -13.9%）まで減らした。

コードを持たないドキュメント／スキル配布リポジトリ。利用者は plugin として
（`/plugin install context-diet@ishizakahiroshi`）、または `skills/context-diet/` を
`~/.claude/skills/context-diet/` へディレクトリごとコピーして使う。

## このリポジトリの `skills/context-diet/SKILL.md` について（重要）

**`skills/context-diet/SKILL.md` と `references/` は配布物であって、このリポジトリの
開発ルールではない。** 利用者の環境へコピーされて動く Claude Code スキルの実体なので、
次を守る:

- 作者環境固有の絶対パス・個人情報・kb 由来の固有名詞を書かない。**第三者の環境で
  そのまま動く**ことが成立条件
- frontmatter の `name` / `description` は起動語の正本。変更すると利用者の発火条件が
  変わるので、README の説明と必ず同時に更新する
- `SKILL.md` からのリンクは `references/…` の相対パスにする（配置先でも切れない形）
- `.claude-plugin/plugin.json` / `.claude-plugin/marketplace.json` も配布物。**スキルを
  変更したら `plugin.json` の `version` を上げる。** 利用者へ更新が届くのは version が
  上がったときだけ
- 本ファイル（`CLAUDE.md`）と混同しない。`CLAUDE.md` は**このリポジトリを開発する
  AI 向け**、`skills/context-diet/SKILL.md` は**利用者の環境で動く配布物**

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
| 配布 | Claude Code plugin（`.claude-plugin/` の manifest 経由）／`skills/context-diet/` のディレクトリコピー |
| ライセンス | MIT |

## ディレクトリ構成

- `skills/context-diet/SKILL.md` — 配布用スキル本体（plugin として自動検出される。手動なら
  ディレクトリごと `~/.claude/skills/context-diet/` へ置く）
- `skills/context-diet/references/playbook.md` — 詳細手順
- `skills/context-diet/references/settings-keys-reference.md` — `settings.json` で実在する context 削減キーの一覧
- `skills/context-diet/references/claude-md-split-pattern.md` — CLAUDE.md を分割する具体パターン
- `.claude-plugin/plugin.json` — plugin manifest（`skills/` は自動検出なので列挙しない）
- `.claude-plugin/marketplace.json` — marketplace manifest（このリポ自身を `"./"` として配る）
- `scripts/` — secrets-scan と hook インストーラ
- `.githooks/` — layer 2 pre-commit
- `.github/workflows/` — layer 3 CI

## 主要コマンド

ビルドもテストもない。編集して push するだけ。

- secrets-scan 手動実行: `node scripts/secrets-scan.mjs --staged --block`
- manifest 検証: `claude plugin validate .`（marketplace 側）/ `claude plugin validate .claude-plugin/plugin.json`（plugin 側）。CI 相当の厳しさで見るときは `--strict`
- ローカル起動テスト: `claude --plugin-dir .`（この状態のリポを plugin として読み込んだセッションが立つ）

## AI 作業共通ルール

ビルド・コミット禁止、secrets-scan 責務、plan/bugfix/pending md の作成ルール等の AI 作業共通ルールは、各利用者のグローバル AI 設定に従う（作者環境の例: `~/.claude/CLAUDE.md` および `~/.claude/guides/`）。

このリポジトリ固有:

- **数値を書き換えるときは出典を確認する。** README と記事に出ている
  「80.7k → 69.5k / -13.9%」は実測値。別の環境の数字で上書きしない
- **「`disabledTools` は存在しない」のような否定形の事実は、確認した Claude Code の版と
  セットで書く。** 版が上がって実在するようになったら、その時点で訂正が要る。2026-06-22 版は
  `skillOverrides` を「存在しない」と書いていたが、2.1.263 の実行ファイル走査で実在が確認され
  訂正した（このリポ自身が踏んだ例）
- `SKILL.md` を変更したら README の説明表も同時に直す（起動語と用途が 2 箇所にある）。
  あわせて `.claude-plugin/plugin.json` の `version` を上げる（上げないと利用者へ更新が届かない）

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
| 配布用スキル本体 | `skills/context-diet/SKILL.md` |
| 詳細手順 | `skills/context-diet/references/playbook.md` |
| plugin / marketplace manifest | `.claude-plugin/plugin.json` / `.claude-plugin/marketplace.json` |
| Codex/他 AI 用入口 | `AGENTS.md` |
| 背景記事（Zenn） | https://zenn.dev/ishizakahiroshi/articles/20260622-claude-code-context-diet |
