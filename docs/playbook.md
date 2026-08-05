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

**Memory が 15k tokens を超えていれば削減候補**。10k 以下ならほぼ何もできない（誤差レベル）。

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

## Step 4: `settings.json` のチューニング

### やってはいけないこと

❌ `disabledTools` キーを追加 — **公式に存在しない**。書いても無視される

❌ `permissions.deny` を「context 削減のため」追加 — 呼び出しブロックのみで context は減らない

### 効くキー（リスク低い順）

```json
{
  "disableArtifact": true,
  "disableWorkflows": true
}
```

完全な公式キー一覧は [settings-keys-reference.md](settings-keys-reference.md) 参照。

`disableWorkflows` のリスク:
- ローカル Workflow ツール（多 agent オーケストレーション）が使えなくなる
- ultracode モード（Claude Code の workflow ベース機能）が動かなくなる
- 一部 skill（deep-research 等の workflow 依存）が消える

→ Workflow を能動的に使っていなければ入れて良い。必要になったら 1 行削除で即復活。

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
- Workflow / ultracode / deep-research が必要になったら `disableWorkflows` を消す
- トリガー判定漏れの事故があれば、該当トリガー文を grep 可能な明示語彙に書き直す

## トラブルシュート

### 「AI がサブファイル化したルールを読まない」

→ トリガー文を見直す。Step 3 の「良いトリガー文の条件」を満たしているか。

### 「context が思ったほど減らない」

→ Memory が元々スリムだった可能性大。System tools 側で `disableWorkflows` を試す。

### 「`disableWorkflows` 入れたら ultracode が動かない」

→ 想定通り。`settings.json` の `"disableWorkflows": true` を削除して再起動。

### 「`/context` の数値が変わらない」

→ 既存セッションを使っている。新セッションを起こして再実行。
