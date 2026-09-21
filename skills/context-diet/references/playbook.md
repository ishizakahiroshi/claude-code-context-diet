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

## Step 1: `/context` で現状把握と「削る価値」の判定

新しいセッションで:

```
/context
```

返ってくる内訳例（Claude Code 2.1.278・1M 窓の環境。数字は検証環境で取った推定値、パスは例）:

```
Model:  claude-fable-5-1
Tokens: 45.7k / 1m (5%)

System prompt              4.1k
System tools               6.2k
System tools (deferred)   13.2k   ← Total に含まれない
Memory files              26.8k
Skills                     8.6k
Messages                     10
Free space               921.3k
Autocompact buffer          33k

Memory Files
  User     ~/.claude/CLAUDE.md     12.9k
  User     ~/.claude/notes.md        695   ← @import した各ファイルが個別に出る
  Project  ./CLAUDE.md               2.7k
```

展開表示が欲しい場合は `/context all` を使う。skill ごとの listing cost・直近 7 日間の使用回数・一度も呼ばれていない skill の一覧を見るには `/skill-doctor` を使う。`/usage` は最初の応答の後に "Prompt cache (main)" 行（2.1.251 以降）を出す。

`/context` は公式の提案行も出す（公式 commands、2026-09-21 fetch: "Shows optimization suggestions for context-heavy tools, memory bloat, and capacity warnings."）。自分で分類する前に、そこに挙がった項目を候補に入れる。

"System tools (deferred)" は Tool Search でスキーマが遅延ロードされるツールで、名前だけが常駐し Total に含まれない。2.1.278 では MCP ツールに加えて WebFetch / WebSearch / NotebookEdit 等の組み込みツールも deferred になる。削減対象にしない。MCP は名前と server instructions だけが常駐する（公式 costs: "MCP tool definitions are deferred by default, so only tool names and server instructions enter context until Claude uses a specific tool."）。

公式 docs は CLAUDE.md 単体の行数に目安を示している（<https://code.claude.com/docs/en/memory> より、2026-09-21 fetch）: "Size: target under 200 lines per CLAUDE.md file. Longer files consume more context and reduce adherence. If your instructions are growing large, use path-scoped rules so instructions load only when Claude works with matching files. You can also split content into imports for organization, though imported files still load and enter the context window at launch."（1 ファイル 200 行未満を目安にする。長いと context を消費し遵守率も下がる。大きくなったらパス条件付きの rules へ。`@import` に分けても起動時に載るので context は減らない、の意）。

### 削る価値の判定（3 分岐）

1. **200k 窓**: 常駐（Total から Messages を引いた量）÷ 上限 を出す。元記事の環境は 80.7k / 200k = 40% で、削る価値があった
2. **1M 窓**: トークン目的では削らない。`settings.json` のキーで削れる最大は約 3.6k tokens（2.1.278 の API 実測。Step 4 の表）で Total の 0.4%。失う機能に見合わない。CLAUDE.md が 1 ファイル 200 行を超えていれば、遵守率の目的で Step 2 へ（2026-08-18 実測: Memory files を -22.7% 減らしても Total 比では 1% 未満）
3. **cache miss が多い**: `/usage` の "Prompt cache (main)" 行で misses が多い、または "likely cause: tool definitions changed" が出るなら、常駐は miss のたびに丸ごと再処理される。分母が 1M でも常駐を減らす価値がある

### 1M 窓が既定になった

公式 model-config（2026-09-21 fetch）: "On the Anthropic API, Fable 5.1, Fable 5, Sonnet 5, and Opus 4.7 and later run with the 1M window by default." / "On the Anthropic API, Sonnet 5 always runs with the 1M context window. There is no 200K variant, no `[1m]` suffix to select, and no usage credits required on any plan." / "On Max, Team, and Enterprise plans, including both Team Standard and Team Premium seats, Opus is automatically upgraded to 1M context with no additional configuration."

200k 窓が残るのは、Pro プランで usage credits を使わない Opus と、Bedrock / Vertex / Foundry で `[1m]` を付けずにピン留めしたモデル。1M 窓のセッションは "at about 967K tokens by default" で auto-compact する。**上限が大きい環境ほど、削減の絶対量だけで「効果があった」と判断しない。**

### prompt cache を前提に読む

公式 costs（2026-09-21 fetch）: "Claude Code automatically optimizes costs through prompt caching, which reduces costs for repeated content like system prompts" / "With prompt caching, Claude Code re-reads that history at the cached token rate" / "The lifetime is an hour on a subscription and drops to five minutes once you're drawing on usage credits; on an API key or cloud provider, it's five minutes by default."

常駐 context は毎ターン素の単価で課金されるのではなく cache read 単価で読まれる。cache が外れる（1 時間空けた後の最初の 1 通、`/usage` に "likely cause: tool definitions changed" と出る等）と丸ごと再処理されるので、常駐量は「miss 1 回あたりの再処理量」として効く。削る意味を測るなら、`/usage` の misses の頻度と常駐量を掛けて考える。

### 正確な実測の取り方（API usage）

`/context` の数字は推定で、同条件の API 実測と 14.6k tokens ずれた（2.1.278・検証環境: 推定 45.7k / API 60,284）。カテゴリ配分も当てにならない（`skillListingMaxDescChars: 256` で推定は Total 不変だったが API では -2,277）。Before / After を正確に取るときは、設定断片を JSON ファイルに書いて非対話で 1 往復させ、応答の `usage` を合計する。

bash:

```bash
echo '{"enableWorkflows":false}' > /tmp/s.json
claude -p "Reply with exactly: OK" --output-format json --settings /tmp/s.json \
  | jq '.usage | .input_tokens + .cache_creation_input_tokens + .cache_read_input_tokens'
```

PowerShell:

```powershell
Set-Content -Path "$env:TEMP\s.json" -Value '{"enableWorkflows":false}'
$r = claude -p "Reply with exactly: OK" --output-format json --settings "$env:TEMP\s.json" | ConvertFrom-Json
$r.usage.input_tokens + $r.usage.cache_creation_input_tokens + $r.usage.cache_read_input_tokens
```

baseline は `--settings` 無しで同じコマンドを打つ。差が削減量。2.1.278 の応答は 1 つのオブジェクトで返る（配列で返る版では最後の要素の `usage` を見る）。

制約: 非対話（`-p`）には Artifact / AskUserQuestion / MCP ツールが載らない（2.1.278 で確認）。これらの効果は対話セッションで新セッションを起こし、`/context` の Before / After で取る（推定値になるので「推定」と併記する）。

## Step 2: CLAUDE.md の章を 4 分類（skills 優先）

まず各ファイルの行数を測る（bash: `wc -l ~/.claude/CLAUDE.md` / PowerShell: `(Get-Content ~/.claude/CLAUDE.md).Count`。`Measure-Object -Line` は空行を数えないので使わない）。空行を含む物理行で数え、200 行超かどうかは測ってから判定し、推測しない。連続した空行も削減候補に入れる。

### 内容の監査（4 観点・分類の前に）

行数を減らす前に、書いてある内容を 4 観点で監査する。CLAUDE.md 本体だけでなく、`@import` 先・auto-memory の索引と topic ファイル・ツールが注入するファイル（hub / plugin / hook が書き込む承認ルールや guidance）まで同じ表で見る。作者環境では 390 行 → 206 行に減らした後に監査して、矛盾 4・重複 8 組・陳腐化 7・曖昧 5 が残っていた（2026-09-21）。

| 観点 | 例 | 直し方 |
|---|---|---|
| 矛盾 | CLAUDE.md は「行頭の `- ` を使わない」、注入ファイルは「箇条書き（`- `）で書く」 | 正典を 1 つ決めて他を合わせる。注入ファイルは生成元（テンプレート・ソース）を直す |
| 重複 | 同じ規則が CLAUDE.md の節・memory の索引行・guide の本文の 3 か所にある | 本文は 1 か所に置き、CLAUDE.md は要約 3〜5 行、memory の索引行は消す |
| 陳腐化 | 「skill 一覧は per-turn で絞られるので当てにしない」（旧版の挙動を根拠にした理由）・「〜は slash command へ移行済」（告知） | 版を併記して書き直すか消す |
| 曖昧 | 「`MEMORY.md` は索引に保つ」（グローバルの索引かプロジェクト auto-memory かが不明）・節ごとに言語が違う | 指す先を限定する。1 ファイル 1 言語にする |

原則:

1. **1 規則 1 正典。** 同じ規則を 2 か所に書くなら、片方は「正典は X」のポインタにする。要約とポインタは CLAUDE.md、本文は skill / guide / rules
2. **ツールが生成するファイルは生成元で直す。** 手で直した分は次の再生成で消える。生成元を直せないなら、その旨を CLAUDE.md 側に 1 行書いて矛盾を明示する
3. **harness の挙動を根拠にした理由は版で古くなる。** 「〜が通知されない」「〜は存在しない」は確認した Claude Code の版を併記し、版が上がったら再確認する
4. **展開されない import・連続した空行・移行告知は、読まれていないのに常駐している。** import 行は `/context all` の一覧に出るかで確認し、空行は物理行で数える

監査の結果は「矛盾 N・重複 N 組・陳腐化 N・曖昧 N」と数えて報告する。行数の削減量より、この数が減ったことを成果にする。

`~/.claude/CLAUDE.md` を全部読み、章ごとに分類する。公式の第一選択は skills と `.claude/rules/`。`@import` に分けても context は減らない（公式 memory、2026-09-21 fetch: "Splitting into `@path` imports helps organization but doesn't reduce context, since imported files load at launch."）。

### 残す

毎ターン判定材料として必要なルール。3〜5 行で書けるもの:
- 略記ルール（ショートカットの起動条件）
- ツール選択ルール（シェルの使い分け等）
- 出力フォーマット規約（応答末尾の定型等）
- ファイル種別判定（`plan_*.md` / `bugfix_*.md` 等のショートカット）

### skill 化（第一選択）

作業種別ごとにしか要らない手順・規範・チェックリスト。公式 costs（2026-09-21 fetch）: "If it contains detailed instructions for specific workflows (like PR reviews or database migrations), those tokens are present even when you're doing unrelated work. Skills load on-demand only when invoked, so moving specialized instructions into skills keeps your base context smaller."

- コーディング規範（根本原因優先・憶測断定の禁止・UX 最優先 等）
  → `~/.claude/skills/coding-stance/SKILL.md`
- ドメイン参照ルール（社内 KB の引き方等）
  → `~/.claude/skills/<domain>-lookup/SKILL.md`（特定のパスに紐づくなら下の rules 化）
- リリース手順・配布方針
  → `~/.claude/skills/release/SKILL.md`

skill の作り方:

1. `~/.claude/skills/<name>/` を作り、`SKILL.md` に本文をそのまま移す
2. frontmatter は `name: <name>` と `description: <何をする skill か + 起動語>` の 2 行。`description` は listing に常駐するので短く具体的に（起動語は「実装して」「直して」のようにユーザーの言い方で）
3. CLAUDE.md 側の章は削除する（残すなら「<作業> は `<name>` skill を使う」の 1 行）
4. 新セッションで、起動語に該当する依頼をして skill が起動するか確認する。起動しなければ `description` の起動語を直す
5. `/skill-doctor` で listing cost と使用回数を見る。滅多に使わない skill は `skillOverrides` で `name-only` に落とす（Step 4）
6. プロジェクト固有なら `.claude/skills/<name>/` に置く

具体例は [claude-md-split-pattern.md](claude-md-split-pattern.md)。

### rules 化（パスに紐づくもの）

特定のファイル種別・ディレクトリにだけ効く指示は `.claude/rules/*.md` に置き、frontmatter `paths:` で対象を絞る（Step 3 の「公式代替」参照）。

### トリガー文で残す（skill にできない判定材料だけ）

skill にできない 3〜5 行の判定材料は、`~/.claude/guides/<name>.md` に本文を切り出し、CLAUDE.md にトリガー文を残す旧方式でもよい。AI が Read しに行かない事故が起きるので、Step 3 の条件を満たす書き方にする。CLAUDE.md からは **3〜5 行のトリガー文だけ残す**:

```markdown
## コーディング規範（参照ガイド・コードに触る前に必ず Read）

コード・設定ファイル・スクリプトに **変更を加える前** に、
`~/.claude/guides/rule_coding-stance.md` を Read してその規範に従うこと。

トリガー: 既存コードの編集 / 新規実装 / バグ修正 / リファクタ / 「ここを直して」「実装して」等の依頼を受けた時。
```

### 削除候補

上の監査で挙がったもの:

- 重複: 同じ内容を別ファイルで持っているもの（本文を 1 か所に寄せ、他は「正典は X」のポインタにする）
- 陳腐化: 根拠にした挙動の版が古いルール・移行済みの告知・動いていない「最終更新」
- 「念のため」で残していて、もう発火しないもの
- 連続した空行（物理行で数える）
- 展開されない `@import` 行（`/context all` の一覧に出ないもの）。ただし他の AI CLI が同じファイルを読む環境では、その CLI での挙動を確かめてから消すか、読まれていない旨を併記する

## Step 3: トリガー文の品質管理（skill にできない判定材料だけ）

対象は Step 2 で「トリガー文で残す」に分類したものだけ。作業種別ごとの手順は skill にする（起動判定を Claude Code が `description` で行うので、この Step の落とし穴を踏まない）。

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

### 着手前に: Step 1 の判定を見る

`settings.json` のキーは機能を犠牲にしてトークンを削る（`enableWorkflows: false` なら Workflow ツール・ultracode ごと消える）。**削減量が Total に対してどれだけの割合か、Step 1 で確認した分母と照らしてから判断する。** 2.1.278 の API 実測（非対話・検証環境）では、settings キーで削れる最大は `disableBundledSkills: true` + `enableWorkflows: false` の -3,563 tokens で、1M 窓では Total の 0.4%。失う機能とのトレードオフが見合わないことが多い。すでに `enableWorkflows: true` 等で Workflow を能動的に使っている設定が入っている場合は、そもそも変更候補から外れる。

### 使用実績の数え方（キーを当てる前に）

`/skill-doctor` は「never」を出すが、記録の窓は短い（作者環境: never 19 本のうち 8 本が 60 日以内に使われていた）。60 日の transcript から tool_use を数えて判定する。厳密な形で数えないと、transcript に記録されたツール定義文まで一致する（作者環境: `"name":"Artifact"` の緩い一致は 734 件、厳密形では 0 件）。

```bash
# macOS / Linux / Git Bash。CLAUDE_CONFIG_DIR を使う環境は、そのディレクトリの projects も roots に足す
roots="$HOME/.claude/projects"
find $roots -name '*.jsonl' -mtime -60 -print0 \
  | xargs -0 rg -o --no-filename '"type":"tool_use","id":"[^"]+","name":"[^"]+"' \
  | sed -E 's/.*"name":"([^"]+)"/\1/' | sort | uniq -c | sort -rn
find $roots -name '*.jsonl' -mtime -60 -print0 \
  | xargs -0 rg -o --no-filename '"skill":"[^"]+"' | sort | uniq -c | sort -rn
```

読み方と対処:

- 0 回の skill → `skillOverrides` で `off`。自分で書いた skill は `name-only`（説明文だけ落とし、名前では呼べる）。claude.ai から同期される `anthropic-skills:*` と plugin の skill も `/context all` の Skills 表に出るので同じ基準で見る。plugin は skill 単位で切れないので `claude plugin disable <name>`
- 0 回のツール群 → Artifact なら `enableArtifact: false`、claude.ai の MCP コネクタ（`mcp__claude_ai_*`）なら `disableClaudeAiConnectors: true`
- auto-memory のディレクトリ（`<config dir>/projects/*/memory/`）にファイルが無い → `autoMemoryEnabled: false`
- **どの settings.json が読まれているかを先に確かめる。** `echo $CLAUDE_CONFIG_DIR`（PowerShell: `$env:CLAUDE_CONFIG_DIR`）が設定されていれば `~/.claude/settings.json` は読まれない。作者環境では many-ai-cli のプロファイルが別の settings.json を持ち、既定側で入れた `disableArtifact: true` と `disableClaudeAiConnectors: true` が効いていなかった

作者環境の実測（2026-09-21・2.1.278・非対話）: skillOverrides 31 本（off 20・name-only 11）+ コネクタ off + autoMemory off で API usage 55,011 → 48,761（−6,250）。`/context` の推定は Skills 8.6k → 3.4k、System prompt 2.3k → 1.5k だが、System tools が 6.2k → 11.3k に増えて Total は −0.7k にしか見えなかった。推定の内訳は当てにせず、API usage で前後を取る。

### やってはいけないこと

❌ `disabledTools` キーを追加 — **公式に存在しない（Claude Code 2.1.278 で確認）**。書いても無視される

❌ `permissions.deny` を「context 削減のため」追加 — 呼び出しブロックのみで context は減らない

❌ `disableBundledSkills: true` を単独で追加 — **常駐が増える（2.1.278 の API 実測 +4,679 tokens）**。`enableWorkflows: false` と併用したときだけ -3,563 tokens。原因は未確認

### 効くキー（2.1.278 の API 実測・削減量の大きい順）

| 設定 | 削減量 | 失うもの |
|---|---|---|
| `disableBundledSkills: true` + `enableWorkflows: false` | -3,563 tokens | bundled skill（`/loop` `/schedule` `/code-review` 等）と Workflow / ultracode |
| `skillListingMaxDescChars: 256` | -2,277 tokens | skill listing の description が切れ、skill matching の精度が落ちる |
| `enableWorkflows: false` | -2,057 tokens | Workflow / ultracode / workflow 依存の skill |
| `autoMemoryEnabled: false` | -716 tokens | auto-memory の読み書き（検証環境では auto-memory が空。減ったのは指示文） |
| `includeGitInstructions: false` | -649 tokens | git status snapshot と commit / PR 手順 |
| `enableArtifact: false` | 未再測（対話セッション限定のツール。説明文 6,512 文字） | Artifact 公開 |

最初の 1 行は、Workflow を使わない人なら `enableWorkflows: false`:

```json
{
  "enableWorkflows": false
}
```

旧キー `disableArtifact: true` / `disableWorkflows: true` も引き続き動作するが、新規に書くときは `enableArtifact` / `enableWorkflows` を使う。完全な公式キー一覧と測定条件は [settings-keys-reference.md](settings-keys-reference.md) 参照。

`enableArtifact: false` は安全だが非対話では測れないので、対話セッションで `/context` の Before / After（推定値）を取ってから入れる。

`enableWorkflows: false` のリスク:
- ローカル Workflow ツール（多 agent オーケストレーション）が使えなくなる
- ultracode モード（Claude Code の workflow ベース機能）が動かなくなる
- 一部 skill（deep-research 等の workflow 依存）が消える

→ Workflow を能動的に使っていなければ入れて良い。必要になったら該当行を削除して再起動で即復活。

滅多に使わない skill が個別にある場合は、一括封印の `disableBundledSkills` の代わりに `skillOverrides` で対象を絞る（値: `on` / `name-only` / `user-invocable-only` / `off`。詳細は [settings-keys-reference.md](settings-keys-reference.md) 参照）。skill が多い環境（1M 窓では listing 予算が 40,000 文字に広がる）は `skillListingMaxDescChars` を下げる方が効く。

### 環境限定ファイルの扱い

特定の環境でしか意味がないファイル（CI 用の承認書式、あるツールの配下でだけ効く規約など）を「環境変数を見て該当時だけ Read する」条件付き Read にするのは勧めない。AI が判定を飛ばして Read せず、規約違反が繰り返される（同じ違反が 2 回続いて、結局 `@import` に戻した例がある）。選択肢は 2 つ:

- `@import` のまま常駐させる。確実だが常駐コストを払う
- skill 化して `description` に環境名と起動条件を書く。起動判定を Claude Code に任せる

## Step 5: 効果実測

**新しいセッションを起こす**（編集前の context を持つ既存セッションでは正確な値が出ない）。

```
/context
```

Before/After 比較表を作る。`/context` の値は推定なので「推定」と書き、正確に取るなら Step 1 の API 実測を併記する:

| カテゴリ（`/context` 推定） | Before | After | 削減 |
|---|---|---|---|
| Total | XX.Xk | XX.Xk | -X.Xk |
| System tools | XX.Xk | XX.Xk | -X.Xk |
| Memory files | XX.Xk | XX.Xk | -X.Xk |
| Skills | XX.Xk | XX.Xk | -X.Xk |

| 設定（API 実測） | prompt tokens | baseline との差 |
|---|---|---|
| baseline | NN,NNN | — |
| `enableWorkflows: false` | NN,NNN | -N,NNN |

## Step 6: しばらく様子見

- 2 週間ほど運用して挙動に問題なければバックアップ削除
- Workflow / ultracode / deep-research が必要になったら `enableWorkflows: false`（旧 `disableWorkflows: true` も可）の行を削除する
- トリガー判定漏れの事故があれば、該当トリガー文を grep 可能な明示語彙に書き直す

## トラブルシュート

### 「AI がサブファイル化したルールを読まない」

→ トリガー文を見直す。Step 3 の「良いトリガー文の条件」を満たしているか。

### 「context が思ったほど減らない」

→ 1M 窓では減っても Total 比が小さい（settings キーの最大で 0.4%）。目的を遵守率に置き直し、CLAUDE.md の行数と skill の起動で評価する。Memory が元々スリムなら、System tools 側で `enableWorkflows: false`（旧 `disableWorkflows: true` も可）を試す。

### 「`disableBundledSkills: true` を入れたら増えた」

→ 2.1.278 の実測どおり（単独で +4,679 tokens）。`enableWorkflows: false` と併用するか、外して `skillOverrides` で個別に消す。

### 「`/context` の数字が `/usage` や API の usage と合わない」

→ `/context` は推定。同条件で API 実測と 14.6k tokens ずれた。Before / After は Step 1 の API 実測で取る。

### 「`enableWorkflows: false` 入れたら ultracode が動かない」

→ 想定通り。`settings.json` の `"enableWorkflows": false`（または旧キー `"disableWorkflows": true`）の行を削除して再起動。

### 「`/context` の数値が変わらない」

→ 既存セッションを使っている。新セッションを起こして再実行。それでも変わらなければ、編集した settings.json が読まれていない。`echo $CLAUDE_CONFIG_DIR`（PowerShell: `$env:CLAUDE_CONFIG_DIR`）を見て、指しているディレクトリの settings.json を直す（Step 4「使用実績の数え方」の最後の項目）。
