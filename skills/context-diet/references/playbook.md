# Context Diet Playbook

Claude Code の常駐 context をスリム化する完全手順。

## 前提

- Claude Code が動いている環境
- `~/.claude/CLAUDE.md` または同等のグローバル指示ファイルが存在する
- `settings.json` を編集する権限がある

## Step 0: バックアップ

⚠️ **必ず最初にバックアップ**。グローバル設定の編集ミスは全セッションに波及する。

```bash
date_tag=$(date +%Y-%m-%d)
cp ~/.claude/CLAUDE.md ~/.claude/CLAUDE.md.bak.$date_tag
cp ~/.claude/settings.json ~/.claude/settings.json.bak.$date_tag
# プロジェクト側に CLAUDE.local.md があれば
cp CLAUDE.local.md CLAUDE.local.md.bak.$date_tag
```

PowerShell:

```powershell
$date = Get-Date -Format "yyyy-MM-dd"
Copy-Item ~/.claude/CLAUDE.md "~/.claude/CLAUDE.md.bak.$date"
Copy-Item ~/.claude/settings.json "~/.claude/settings.json.bak.$date"
```

## Step 1: `/context` で現状把握

Claude Code セッションで:

```
/context
```

返ってくる内訳例:

```
Memory files · /memory
├ ~/.claude/CLAUDE.md: 13k tokens
├ ~/.claude/local-accounts.md: 698 tokens
├ ~/.many-ai-cli/approval-rules.md: 3.3k tokens
└ ...
```

展開表示が欲しい場合は `/context all` を使う。skill ごとの listing cost・直近 7 日間の使用回数・一度も呼ばれていない skill の一覧を見るには `/skill-doctor` を使う。

`/context` で "deferred" と表示される MCP ツールは、Tool Search によりスキーマが遅延ロードされていて常駐していないので、削減対象にしない。

**Memory が 15k tokens を超えていれば削減候補**。10k 以下ならほぼ何もできない（誤差レベル）。

公式 docs も CLAUDE.md 単体の行数に目安を示している（<https://code.claude.com/docs/en/memory> より、2026-09-06 fetch）: "Size: target under 200 lines per CLAUDE.md file. Longer files consume more context and reduce adherence."（CLAUDE.md 1 ファイルあたり 200 行未満を目安にする。長いファイルは context を消費し遵守率も下がる、の意）。トークン数の目安（本 Step の 15k tokens）とは測る単位が違うので、どちらか一方だけでなく両方で確認する。

**絶対値だけでなく Total 行の context window 上限も見る。** `/context` の 1 行目に出るモデル名・トークン上限（例: `343.1k/967k tokens (35%)`）を確認する。200k トークン級の環境では Memory files 数万トークンの削減が Total の数%〜十数%に効くが、`[1m]` サフィックス付きモデルなど 1M トークン級の環境では、同じ削減幅でも Total に対しては 1% 未満のことがある（2026-08-18 実測: Memory files -9.9k / -22.7% だが Total 967k〜1M に対しては 1% 未満）。**上限が大きい環境ほど、削減の絶対量だけで「効果があった」と判断しない。**

## Step 2: CLAUDE.md の章を 3 分類

`~/.claude/CLAUDE.md` を全部読み、章ごとに分類する。

### 残す（メタトリガー）

毎ターン判定材料として必要なルール:
- 略記ルール（`P` `S` 等のショートカット起動条件）
- ツール選択ルール（PowerShell vs Bash 等）
- 出力フォーマット規約（応答末尾フッター等）
- ファイル種別判定（`plan_*.md` / `bugfix_*.md` 等のショートカット）

### 移す（サブファイル化）

トリガー時にだけ意味がある本文:
- コーディング規範（根本原因優先・憶測断定の禁止・UX 最優先 等）
  → `~/.claude/guides/rule_coding-stance.md`
- ドメイン参照ルール（家族情報 CSV・社内 KB 等）
  → `~/.claude/guides/reference_<domain>.md`
- リリース手順・配布方針
  → `~/.claude/guides/reference_<topic>.md`

CLAUDE.md からは **3〜5 行のトリガー文だけ残す**:

```markdown
## コーディング規範（参照ガイド・コードに触る前に必ず Read）

コード・設定ファイル・スクリプトに **変更を加える前** に、
`~/.claude/guides/rule_coding-stance.md` を Read してその規範に従うこと。

トリガー: 既存コードの編集 / 新規実装 / バグ修正 / リファクタ / 「ここを直して」「実装して」等の依頼を受けた時。
```

### 削除候補

- 同じ内容を別ファイルで持っているもの
- 既に陳腐化したルール
- 「念のため」で残していて、もう発火しないもの

## Step 3: トリガー文の品質管理

サブファイル化で最大の落とし穴は「トリガー判定漏れ」。AI が「読むべき場面なのに参照しに行かない」事故が起きる。

良いトリガー文の条件:

- **grep 可能な明示語彙**（「コードに触る前に」「家族の話題が出たら」など、曖昧な「重要な時」を避ける）
- **発火条件が具体的**（「コード編集 / 新規実装 / バグ修正 / リファクタ」のように列挙）
- **参照先パスが絶対パス**（相対パスだと環境差で壊れる）

悪い例:

```markdown
## 重要なルール
時々参照してください: ~/.claude/guides/rules.md
```

→ いつ発火するか曖昧。スキップされる。

良い例:

```markdown
## コーディング規範（コードに触る前に Read）
ファイルを編集・新規作成する前に必ず `~/.claude/guides/rule_coding-stance.md` を Read。
トリガー: Edit / Write / NotebookEdit ツールを使う前、または「実装して」「直して」依頼を受けた時。
```

### 公式代替: `.claude/rules/`（プロジェクト階層の条件付き読込）

上記はユーザー（AI）に「読め」と指示する手動の遅延読込だが、Claude Code にはプロジェクト階層向けの公式機構 `.claude/rules/*.md` がある。frontmatter に `paths:` を書くと、パターンに一致するファイルを Claude が開いたときだけそのルールが読み込まれる。`paths:` を書かなければ CLAUDE.md と同様に起動時読込になる（公式 docs memory.md による）:

```markdown
---
paths:
  - "src/api/**/*.ts"
---

# API 開発ルール

- すべての API エンドポイントは入力検証を必須にする
```

役割分担:

- ユーザー階層（`~/.claude/CLAUDE.md`）: 本 Step 2〜3 のサブファイル化 + トリガー文方式
- プロジェクト階層で特定ファイル種別にだけ効かせたいルール: `.claude/rules/` の `paths:` 条件付き読込

ユーザー階層向けの `~/.claude/rules/` も公式 docs（memory.md の「User-level rules」節）に記載がある。プロジェクトの rules より先に読み込まれる。

## Step 4: `settings.json` のチューニング

### 着手前に: context window の規模を見る

`settings.json` のキーは機能を犠牲にしてトークンを削る（`enableWorkflows: false` なら Workflow ツール・ultracode ごと消える）。**削減量（数百〜1 万トークン程度）が Total に対してどれだけの割合か、Step 1 で確認した数字と照らしてから判断する。** 1M トークン級の環境では、最大効果とされる `enableWorkflows: false` ですら Total に対して 1% 未満のことがあり、失う機能とのトレードオフが見合わないことが多い。すでに `enableWorkflows: true` 等で Workflow を能動的に使っている設定が入っている場合は、そもそも変更候補から外れる。

### やってはいけないこと

❌ `disabledTools` キーを追加 — **公式に存在しない（Claude Code 2.1.263 で確認）**。書いても無視される

❌ `permissions.deny` を「context 削減のため」追加 — 呼び出しブロックのみで context は減らない

### 効くキー（リスク低い順）

```json
{
  "enableArtifact": false,
  "enableWorkflows": false
}
```

旧キー `disableArtifact: true` / `disableWorkflows: true` も引き続き動作するが、新規に書くときは `enableArtifact` / `enableWorkflows` を使う。完全な公式キー一覧は [settings-keys-reference.md](settings-keys-reference.md) 参照。

`enableWorkflows: false` のリスク:
- ローカル Workflow ツール（多 agent オーケストレーション）が使えなくなる
- ultracode モード（Claude Code の workflow ベース機能）が動かなくなる
- 一部 skill（deep-research 等の workflow 依存）が消える

→ Workflow を能動的に使っていなければ入れて良い。必要になったら該当行を削除して再起動で即復活。

滅多に使わない skill が個別にある場合は、一括封印の `disableBundledSkills` の代わりに `skillOverrides` で対象を絞る方法もある（値: `on` / `name-only` / `user-invocable-only` / `off`。詳細は [settings-keys-reference.md](settings-keys-reference.md) 参照）。

### Hub 系の条件付きロード

`~/.many-ai-cli/approval-rules.md` のような「特定環境でしか意味がないファイル」は @import を撤去し、CLAUDE.md に条件付き Read ルールを書く:

```markdown
## many-ai-cli Hub セッション時の承認マーカー（条件付き Read）

セッション開始時に `MANY_AI_CLI` 環境変数を確認:
- Windows (PowerShell): `$env:MANY_AI_CLI`
- macOS / Linux / Git Bash: `echo "$MANY_AI_CLI"`

`MANY_AI_CLI=1` の場合は `~/.many-ai-cli/approval-rules.md` を Read し、
そのファイルに書かれた承認マーカー出力フォーマットに従う。
```

CLAUDE.md 末尾の `@~/.many-ai-cli/approval-rules.md` 行は削除。

## Step 5: 効果実測

**新しいセッションを起こす**（編集前の context を持つ既存セッションでは正確な値が出ない）。

```
/context
```

Before/After 比較表を作る:

| カテゴリ | Before | After | 削減 |
|---|---|---|---|
| Total | XX.Xk | XX.Xk | -X.Xk |
| System tools | XX.Xk | XX.Xk | -X.Xk |
| Memory files | XX.Xk | XX.Xk | -X.Xk |

## Step 6: しばらく様子見

- 2 週間ほど運用して挙動に問題なければバックアップ削除
- Workflow / ultracode / deep-research が必要になったら `enableWorkflows: false`（旧 `disableWorkflows: true` も可）の行を削除する
- トリガー判定漏れの事故があれば、該当トリガー文を grep 可能な明示語彙に書き直す

## トラブルシュート

### 「AI がサブファイル化したルールを読まない」

→ トリガー文を見直す。Step 3 の「良いトリガー文の条件」を満たしているか。

### 「context が思ったほど減らない」

→ Memory が元々スリムだった可能性大。System tools 側で `enableWorkflows: false`（旧 `disableWorkflows: true` も可）を試す。

### 「`enableWorkflows: false` 入れたら ultracode が動かない」

→ 想定通り。`settings.json` の `"enableWorkflows": false`（または旧キー `"disableWorkflows": true`）の行を削除して再起動。

### 「`/context` の数値が変わらない」

→ 既存セッションを使っている。新セッションを起こして再実行。
