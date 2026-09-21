# CLAUDE.md 分割パターン（skill 化が主、トリガー文が従）

「CLAUDE.md の章を skill / rules / トリガー文へ移し、CLAUDE.md には毎ターンの判定材料だけ残す」の具体例。

## Before（分割前の CLAUDE.md 断片）

```markdown
## コーディング前の姿勢

### 複数解釈は黙って選ばず提示する

要求に複数の解釈が存在する場合、黙って1つを選んで実装しない。
解釈の候補を提示し、どれで進めるか確認する。

### 変更行の出所テスト

「変更した全行がユーザーの要求に直接トレースできるか？」を自己チェックする。
トレースできない変更（隣接コードの整形・未依頼のリファクタ等）は含めない。

## 根本原因優先

症状を消す対応より根本原因の解決を優先する。
理由: 症状抑制は再発するので最終コストが増えるため。

修正前に自問:

1. 直すのは症状か原因か。症状なら 1 段階遡る
2. 同じ原因から他の症状が出ていないか
3. テストを通すためにハードコード・skip・例外握り潰しをしていないか

## 場当たり対応の再発防止

既存コード・設定に手を入れるとき、「症状が消える方向の対処」だけで方針を確定させない。

1. 起源と契約を辿る ...
2. 仮説と確定事実を分ける ...
3. 場当たりを選ぶときは明示宣言 ...

## 憶測を断定で言わない

原因・状態・ファイル内容・数量などを述べるとき、
確認できる手段（Read / Grep / 実行 / ログ確認）がその場にあるなら、
**述べる前に確認する**。確認していない事柄を断定形（「〜です」「〜が原因」）で言わない。
（以下続く・70 行以上）
```

これだけで CLAUDE.md 内に 70 行・約 1.5〜2k tokens 消費していた（2026-06-22 の推定）。

## After A（主）: skill 化

`~/.claude/skills/coding-stance/SKILL.md`:

```markdown
---
name: coding-stance
description: コード・設定ファイル・スクリプトを編集・新規作成する前の規範（複数解釈の提示・根本原因優先・場当たり抑制・憶測断定の禁止・残置掃除・UX 最優先）。「実装して」「直して」「リファクタして」「ここを変えて」で起動。
---

# コーディング規範

## コーディング前の姿勢

### 複数解釈は黙って選ばず提示する
（元の本文をそのまま）

## 根本原因優先
（元の本文をそのまま）

（以下続く・元の 70 行をそのまま）
```

CLAUDE.md 側には何も残さない。残すなら 1 行:

```markdown
コードに触る作業は `coding-stance` skill の規範に従う。
```

listing に常駐するのは `description` だけ（検証環境の `/context all` では 1 skill あたり ~40〜~90 tokens と表示された）。本文は起動時にだけ読まれる。起動判定は Claude Code が `description` で行うので、「AI が Read しに行かない」型の事故は起きない。滅多に使わないなら `skillOverrides` で `name-only` に落とす。

公式 memory（2026-09-21 fetch）: "For task-specific instructions that don't need to be in context all the time, use skills instead, which only load when you invoke them or when Claude determines they're relevant to your prompt."

## After B（従）: トリガー文 + サブファイル

skill にできない 3〜5 行の判定材料に限る。

### CLAUDE.md 側（トリガー文のみ・3〜5 行）

```markdown
## コーディング規範（参照ガイド・コードに触る前に必ず Read）

コード・設定ファイル・スクリプトに **変更を加える前** に、
`~/.claude/guides/rule_coding-stance.md` を Read してその規範に従うこと。
6 章（コーディング前の姿勢・根本原因優先・場当たり対応・憶測断定・残置掃除・UX 最優先）を統合。

トリガー: 既存コードの編集 / 新規実装 / バグ修正 / リファクタ / 設計判断 /
「ここを直して」「実装して」等の依頼を受けた時。
**初回のみ Read すれば良い**（同一セッション内では記憶に残す）。
```

CLAUDE.md 内は 8 行・約 200 tokens に圧縮。

### サブファイル `~/.claude/guides/rule_coding-stance.md`

```markdown
# コーディング規範（コードに触る前に Read）

> 最終更新: 2026-06-22
> グローバル共通ルール。`~/.claude/CLAUDE.md` から「コードに触る前に Read」ルールで参照される。

## コーディング前の姿勢

### 複数解釈は黙って選ばず提示する
（元の本文をそのまま）

### 変更行の出所テスト
（元の本文をそのまま）

## 根本原因優先
（元の本文をそのまま）

（以下続く・元の 70 行をそのまま）
```

トリガー時にしか Read されないので、常駐 context には載らない。ただし AI がトリガー判定を飛ばすと読まれない。

## 効果

- After A: CLAUDE.md から章が丸ごと消え、listing に `description` 分だけ残る
- After B: CLAUDE.md にトリガー文 8 行（約 200 tokens）が残る
- どちらも Memory files の削減量は章の大きさ次第。旧記述の「-1.5k tokens 程度」は 2026-06-22 の推定
- 1M 窓では削減量より、CLAUDE.md が 200 行以下に収まって遵守率が上がることを目的にする（公式 memory: "Longer files consume more context and reduce adherence."）

## 同じパターンが効く対象

| 元の章 | 移し先 |
|---|---|
| コーディング規範系 | skill（`~/.claude/skills/coding-stance/`） |
| ドメインデータ参照ルール（社内 KB、顧客リスト 等） | skill。特定ディレクトリのファイルに紐づくなら `.claude/rules/` の `paths:` |
| リリース手順・配布方針 | skill（`~/.claude/skills/release/`） |
| 特定プロジェクト向けデプロイ手順 | プロジェクト側の `.claude/skills/deploy/` か `.claude/rules/` |
| `pending_*.md` / `plan_*.md` / `bugfix_*.md` の作成ルール | skill（起動語に「plan 作って」等） |
| 略記の起動条件・ツール選択・出力書式（毎ターン必要） | CLAUDE.md に残す |

## アンチパターン

❌ **トリガー文が曖昧** — 「重要な時に参照」だと AI は読まない（After B のとき）
❌ **サブファイル化しすぎ** — メタトリガー本体（略記の起動条件等）まで切り出すと参照経路が切れる
❌ **トリガー文を書き忘れ** — 本文だけ削除して移し先を作り忘れると知識が失われる
❌ **`description` を長くしすぎる** — listing では 1536 文字（`skillListingMaxDescChars` 既定）で切られ、その分だけ常駐が増える。起動語と要旨だけにする
❌ **全部を skill にして listing を肥大させる** — listing 全体は `skillListingBudgetFraction`（既定 0.01 = context window の 1%）の予算内で自動短縮される。滅多に使わない skill は `skillOverrides` で `name-only` / `off`
❌ **絶対パスで書かない** — 相対パスだと環境差で壊れる。ただし `~/` 表記は可搬。`@import` の `~/` は公式 docs（memory.md）がホームディレクトリへ解決すると明記しており、トリガー文で「`~/.claude/guides/x.md` を Read」と書いた場合も AI 側がホームに読み替えて開くので、ユーザー名の違う環境へ持って行っても壊れない

## 検証方法

1. `/context` で Memory files の合計が減っていることを確認（`/context all` で展開表示すればファイル単位の内訳まで見える）
2. トリガー条件に該当する依頼をして、skill が起動するか（After B ならサブファイルを Read しに行くか）確認する
3. 起動しない / Read しない場合は、`description` の起動語（After B ならトリガー文）をより明示的な語彙に書き直す
4. `/skill-doctor` で listing cost と使用回数を確認する
