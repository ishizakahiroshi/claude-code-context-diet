---
name: context-diet
description: Claude Code の常駐 context を棚卸しし、削る価値を判定してから減らす手順。内容の監査（矛盾・重複・陳腐化・曖昧）を含む。「コンテキスト棚卸して」「context 確認」「常駐減らしたい」「/context 見て無駄ないか」「memory ファイル分割したい」「CLAUDE.md を skill に移したい」「CLAUDE.md の矛盾を洗って」「重複ルールを整理して」「disabledTools」「settings.json の context」等で起動。
---

# context-diet スキル

Claude Code セッションの常駐 context を実測し、削る価値を判定してから、指示の遵守率と prompt cache が外れたときの再処理量を減らす。トークン量は判定材料でありゴールではない。`/context` で内訳と分母を見て、Memory files / Skills / System tools を必要なときだけスリム化する。減らす前に、常駐している指示の内容を 4 観点（矛盾・重複・陳腐化・曖昧）で監査する。

## 起動条件

ユーザーが下記いずれかに該当する発話をしたら起動:

- 「コンテキスト棚卸して」「context 確認」「context 減らしたい」
- 「`/context` の結果これでいい？」「無駄に常駐してる？」
- 「`disabledTools` 効く？」「settings.json で context 削減できる？」
- 「memory ファイル分割したい」「CLAUDE.md 大きすぎる」「CLAUDE.md を skill に移したい」
- 「CLAUDE.md の矛盾を洗って」「重複ルールを整理して」「memory に古い指示が残ってないか見て」

## 手順

### Step 1: 現状把握と「削る価値」の判定

ユーザーに新セッションで次を実行してもらい、結果を貼ってもらう: `/context`（内訳と Total 行）、`/context all`（展開表示）、`/skill-doctor`（skill ごとの listing cost・直近 7 日間の使用回数・一度も呼ばれていない skill の一覧）、`/usage`（最初の応答の後に出る "Prompt cache (main)" 行）。

`/context` の読み方:

- **Total 行の分母**（`45.7k / 1m` のように出る）を最初に見る。Anthropic API 直結では現行モデルが 1M 窓を既定で使う（公式 model-config、2026-09-21 fetch: "On the Anthropic API, Fable 5.1, Fable 5, Sonnet 5, and Opus 4.7 and later run with the 1M window by default."）。200k 窓が残るのは Pro プランで usage credits を使わない Opus と、Bedrock / Vertex / Foundry で `[1m]` を付けずにピン留めしたモデル
- **公式が出す提案行**を読む。`/context` は "optimization suggestions for context-heavy tools, memory bloat, and capacity warnings" を表示する（公式 commands）。自分で分類する前に、そこに挙がった項目を候補に入れる
- **"System tools (deferred)" 行は Total に含まれない。** Tool Search により MCP ツールに加えて組み込みツール（WebFetch / WebSearch / NotebookEdit 等）もスキーマが遅延ロードされ、名前だけが常駐する。削減対象にしない。MCP は名前と server instructions だけが常駐する
- **`/context` の数字は推定。** 同条件の API 実測と 14.6k tokens ずれた（2.1.278・検証環境: 推定 45.7k / API 60,284）。Before / After を正確に取るときは API 実測（[playbook.md](references/playbook.md) Step 1 の 1 行コマンド）を使う

削る価値の判定（3 分岐）:

1. **200k 窓**: 常駐 ÷ 上限 を出す。元記事の環境は 80.7k / 200k = 40% で、削る価値があった
2. **1M 窓**: トークン目的では削らない。`settings.json` のキーで削れる最大は約 3.6k tokens（2.1.278 実測。Total の 0.4%）で、失う機能に見合わない。CLAUDE.md が 1 ファイル 200 行を超えていれば、遵守率の目的で Step 2 へ（公式 memory: "Longer files consume more context and reduce adherence."。2026-08-18 実測: Memory files を -22.7% 減らしても Total 比では 1% 未満）
3. **cache miss が多い**: `/usage` の "Prompt cache (main)" 行で misses が多い、または "likely cause: tool definitions changed" が出るなら、常駐は miss のたびに丸ごと再処理される（cache lifetime はサブスクで 1 時間、usage credits 消費中と API キーでは 5 分）。分母が 1M でも常駐を減らす価値がある

### Step 2: Memory files の棚卸し（skills 優先）

まず `~/.claude/CLAUDE.md` と @import 各ファイルの行数を測る（bash: `wc -l` / PowerShell: `(Get-Content <file>).Count`。`Measure-Object -Line` は空行を数えないので使わない）。空行を含む物理行で数え、連続した空行も削減候補に入れる。200 行超かどうかは測ってから言い、読み込まれた本文の印象で推測しない。`~/.claude` 配下はプロジェクト外なので Read には許可プロンプトが出る（非対話モードでは拒否される）。

**分類の前に、内容を 4 観点で監査する。** 行数とトークンを減らしても、矛盾した指示や陳腐化した根拠はそのまま残る（作者環境の 2026-09-21: 390 行 → 206 行に減らした後の監査で、矛盾 4・重複 8 組・陳腐化 7・曖昧 5 が残っていた）。対象は CLAUDE.md 本体・`@import` 先・auto-memory の索引と topic ファイル・ツールが注入するファイル（hub / plugin / hook が書き込むもの）。

| 観点 | 見つけ方 | 直し方 |
|---|---|---|
| 矛盾 | 同じ場面（出力書式・シェル選択・命名）への指示をファイル間で突き合わせる | 正典を 1 つ決め、他は「正典は X」の 1 行にする。ツールが生成するファイルは生成元のテンプレートを直す（手で直した分は次の再生成で消える） |
| 重複 | 同じ規則が CLAUDE.md の節・memory の索引行・guide の本文に二重三重にある | 本文は 1 か所（skill / guide / rules）。CLAUDE.md は 3〜5 行の要約、memory の索引行は消す |
| 陳腐化 | harness の挙動を根拠にした理由（「一覧は per-turn で絞られる」）・移行済みの告知・動いていない「最終更新」 | 版を併記して書き直すか、根拠ごと消す |
| 曖昧 | 同じ語が別物を指す（2 種類の `MEMORY.md`）・節ごとに言語が違う・例示が本文と違う書式 | 指す先を 1 語で限定し、例示を本文の書式に揃える |

詳細と原則 4 つは [playbook.md](references/playbook.md) Step 2。監査の成果は「矛盾 N・重複 N 組・陳腐化 N・曖昧 N」で数え、行数より先に報告する。

`~/.claude/CLAUDE.md` と参照されている @import 群を Read し、章ごとに 4 分類する。公式の第一選択は skills と `.claude/rules/` で、`@import` に分けても context は減らない（公式 memory、2026-09-21 fetch: "Splitting into `@path` imports helps organization but doesn't reduce context, since imported files load at launch."）。

| 分類 | 対応 |
|---|---|
| 残す | 毎ターンの判定材料。略記の起動条件、ツール選択ルール、出力書式など。3〜5 行で書けるもの |
| skill 化 | 作業種別ごとの手順・規範・チェックリスト（コーディング規範、リリース手順、レビュー観点など）。`~/.claude/skills/<name>/SKILL.md` に本文を移し、frontmatter の `description` に起動語を書く。listing には `description` だけが常駐し、本文は起動時にだけ読まれる（公式 memory: "use skills instead, which only load when you invoke them or when Claude determines they're relevant to your prompt."） |
| rules 化 | 特定のパスに紐づく指示（API 実装規約、特定ディレクトリの書き方など）。`.claude/rules/*.md` に置き、frontmatter `paths:` で対象を絞る。ユーザー階層は `~/.claude/rules/` |
| 削除 | 重複・陳腐化・もう発火しない「念のため」 |

旧方式（`~/.claude/guides/<name>.md` へ切り出して CLAUDE.md に「<条件> のとき必ず Read」のトリガー文を残す）は、skill にできない 3〜5 行の判定材料に限る。理由: トリガー文は AI が Read しに行かない事故が起きる。skill なら起動判定を Claude Code が `description` で行う。skill の `description` も listing に常駐するので、滅多に使わない skill は `skillOverrides` で `name-only` に落とす（Step 3）。具体例は [claude-md-split-pattern.md](references/claude-md-split-pattern.md)。

Memory files には CLAUDE.md 系だけでなく auto-memory の `MEMORY.md`（`~/.claude/projects/<project>/memory/MEMORY.md`。先頭 200 行または 25KB が毎セッション常駐。公式 docs memory.md による）も含まれる。topic ファイルは起動時に全部は載らず、関連するものが recall で system-reminder に入る。棚卸しは 2 段階:

1. `MEMORY.md` は索引だけ残し、各エントリの本文は同じディレクトリの topic ファイルへ移す
2. auto-memory 自体が不要なら `autoMemoryEnabled: false`（settings.json）で読み書きを止める（2.1.278 実測 -716 tokens。減るのは主に memory の指示文）

### Step 3: settings.json の `system tools` 系チューニング

**先に使用実績で切る。** `/skill-doctor` の「never」は記録の窓が短く、作者環境では never 19 本のうち 8 本が 60 日以内に使われていた。判定は transcript の tool_use を数えて行う（[playbook.md](references/playbook.md) Step 4「使用実績の数え方」）。60 日で 0 回のものだけを候補にする:

- 使っていない skill → `skillOverrides` で `off`（自分の skill は `name-only` にすると名前では呼べる）。claude.ai から同期される skill（`anthropic-skills:*`）と plugin の skill も同じ表に出るので忘れない（plugin は `claude plugin disable <name>`）
- 使っていないツール群 → `enableArtifact: false`、`disableClaudeAiConnectors: true`（claude.ai の MCP コネクタ）
- auto-memory のディレクトリにファイルが 1 つも無い → `autoMemoryEnabled: false`
- **設定はどのファイルが読まれているかを先に確かめる。** `$CLAUDE_CONFIG_DIR`（PowerShell は `$env:CLAUDE_CONFIG_DIR`）が別のプロファイルを指していると `~/.claude/settings.json` は読まれない。作者環境では hub のプロファイル側 settings.json に既定のスイッチが無く、`disableArtifact: true` が効いていなかった

作者環境の 2026-09-21 実測（2.1.278・非対話）: skillOverrides 31 本（off 20・name-only 11）+ コネクタ off + autoMemory off の合計で API usage 55,011 → 48,761（−6,250。autoMemory 単独は −716 の既測、残りは未分離）。`/context` の推定では Skills 8.6k → 3.4k、System prompt 2.3k → 1.5k。Artifact off の分は非対話では測れない。

⚠️ **`disabledTools` / `enabledTools` は公式に存在しない（Claude Code 2.1.278 で確認）**。`permissions.deny` は呼び出しブロックのみで context 削減効果なし。**`skillOverrides` は実在する**（下記）。

実在する公式キーと 2.1.278 の API 実測（非対話・検証環境。測定条件は [settings-keys-reference.md](references/settings-keys-reference.md)）。削減量の大きい順:

- `disableBundledSkills: true` + `enableWorkflows: false` — -3,563 tokens（settings キーの最大）。ただし `/loop` `/schedule` `/code-review` 等の bundled skill と Workflow / ultracode が全部消える
- `skillListingMaxDescChars: 256`（既定 1536） — -2,277 tokens。skill listing の各 description が切り詰められ、skill matching の精度が落ちる。1M 窓では listing 予算（`skillListingBudgetFraction` 既定 0.01 = context window の 1%）が 40,000 文字に広がるので、skill が多い環境ほど下げた方が絶対量は減る
- `enableWorkflows: false` — -2,057 tokens。Workflow ツール定義を削除。多 agent / ultracode 等が使えなくなる。1 行削除で即復活（旧キー `disableWorkflows: true` も引き続き動作）
- `autoMemoryEnabled: false` — -716 tokens（検証環境ではプロジェクトの auto-memory が空。減ったのは memory の指示文）
- `includeGitInstructions: false` — -649 tokens。git status snapshot と commit / PR 手順を除去
- `disableBundledSkills: true` 単独 — **+4,679 tokens（増える）。入れない。** 併用時だけ上記のとおり効く
- `enableArtifact: false` — 対話の `/context` 推定で、`disableClaudeAiConnectors: true` と同時に入れて System tools 19.4k → 14.6k（-4.8k・未分離。2026-09-21 作者環境）。非対話では測れない（Artifact ツールは対話セッション限定。説明文は 6,512 文字）。Artifact を使っていなければ安全（旧キー `disableArtifact: true` も引き続き動作）
- `skillOverrides` — skill ごとに listing 表示を `name-only`（description を隠して名前だけ残す）/ `user-invocable-only`（AI からは隠すが `/name` は使える）/ `off`（完全に隠す）へ落とせる。滅多に使わない skill を完全には消したくないとき、`disableBundledSkills` より先にこちら。削減量は skill の選び方次第（未実測）
- `disableClaudeAiConnectors: true` — claude.ai の MCP コネクタを取りに行かない（名前と server instructions の常駐が消える。未実測）
- `claudeMdExcludes` — 特定の CLAUDE.md を読み込み対象から外す（環境次第）

**ユーザーの利用パターンを確認してから提案**:
- Workflow / ultracode 使う → `enableWorkflows: false` は入れない
- bundled skill 多用 → `disableBundledSkills` は入れない（単独では逆効果でもある）
- git 操作が中心 → `includeGitInstructions: false` は入れない
- 既に `enableWorkflows: true` 等で能動利用が明示されている設定があれば、そもそも変更候補から外す

**提案前に Step 1 の判定を踏まえる。** 1M 窓では最大の -3,563 tokens でも Total の 0.4% で、失う機能に見合わないことが多い。その場合は「今は変更しない」を既定の案内にし、強い設定を推奨にしない。

最小リスクの開始点は `enableWorkflows: false` の 1 行（Workflow を使わない人）。新セッションで再測してから次を足す。滅多に使わない skill が個別にあれば `skillOverrides` で `name-only` / `off` に落とす手も併せて案内する。

### Step 4: バックアップ必須

CLAUDE.md / settings.json を編集する **前に必ずバックアップ**（`*.bak.YYYY-MM-DD`）。事故時に即戻せる状態を作ってから着手する。

### Step 5: 効果実測と報告

新セッションで `/context` を再実行し、正確に取るなら [playbook.md](references/playbook.md) Step 1 の API 実測（`claude -p … --output-format json` の `usage` 合計）で Before / After 表を作って報告。期待値（2.1.278・非対話・検証環境の API 実測）:

| 介入レベル | 実測削減量 |
|---|---|
| `enableWorkflows: false` のみ | -2,057 tokens |
| `disableBundledSkills: true` + `enableWorkflows: false` | -3,563 tokens（settings キーの最大） |
| Memory 分割（skill 化 / rules 化） | 環境次第（未実測）。CLAUDE.md の行数と Memory files の減少で評価し、1M 窓では遵守率の改善を目的にする |

環境依存が大きい。すでにスリムな環境ではほぼ効果なし。1M 窓では上記の最大でも Total の 0.4%。

## やらないこと

- 公式に存在しないキー（`disabledTools` / `enabledTools` 等）を勝手に追加しない
- バックアップなしで CLAUDE.md / settings.json を編集しない
- ユーザーの利用パターンを聞かずに「強い」設定（`disableBundledSkills` 等）を勝手に当てない
- 削減幅を大げさに言わない（環境依存）
- このスキル自身の frontmatter に `when_to_use` を足して起動語を分離しない。listing では description と合算で 1536 文字に切り詰められるため効果が無い（`name` / `description` のみで運用する）
- `disableBundledSkills: true` を削減目的で勧めない（2.1.278 の API 実測で単独 +4,679 tokens）
- `/context` の推定値だけで Before / After を語らない（同条件の API 実測と 14.6k tokens ずれた）
- 全部を skill にして listing を肥大させない（`description` は常駐する。listing 全体は `skillListingBudgetFraction` 既定 1% の予算内で自動短縮される）
- 行数とトークンだけ見て内容の監査を飛ばさない（矛盾・重複・陳腐化・曖昧は行数を減らしても残る。ツールが注入するファイルは生成元で直す）

## 関連

- [references/playbook.md](references/playbook.md): 元記事の詳細手順
- [references/settings-keys-reference.md](references/settings-keys-reference.md): settings.json 公式キーの完全リファレンス
- [references/claude-md-split-pattern.md](references/claude-md-split-pattern.md): CLAUDE.md 分割の実例
