# settings.json 公式キー リファレンス（Context 削減目的）

確認版: Claude Code 2.1.278（2026-09-21、実行ファイルの文字列走査 + 公式 docs 照合 + `claude -p` の API usage 実測）。前回の確認は 2.1.263（2026-09-06）。
2026-10-10 に 2.1.295 で再照合した（CHANGELOG 2.1.280〜2.1.295・公式 docs・実行ファイルの文字列）。変わっていた記述だけを直し、直した箇所には 2.1.295 または該当版を書いた。削減量の API 実測は 2.1.278 のまま取り直していない。

Claude Code の `settings.json` で context 削減に関係する公式キーの完全リファレンス。
公式 docs（<https://code.claude.com/docs/en/settings-reference>。キーごとの定義はこちら。<https://code.claude.com/docs/en/settings> は設定ファイルの仕組みの説明）の記載と、Claude Code 2.1.278 実行ファイルの
文字列走査（zod スキーマの `describe()` 文字列を含む）を突き合わせ、削減量は API usage の実測（下記「測定条件」）で取った。
describe() 引用は実行ファイルで確認できたもの（binary 確認）。binary で確認できず docs のみで
裏取りした事実は個別に「公式 docs による」と注記する。

## ⚠️ まず: 「存在しない」キー

以下は実行ファイルの文字列走査で 0 件だった。書いても無視される（または無効として捨てられる）:

- `disabledTools` — 確認版: Claude Code 2.1.278
- `enabledTools` — 確認版: Claude Code 2.1.278
- `disabledSkills` — 確認版: Claude Code 2.1.278
- `enabledSkills` — 確認版: Claude Code 2.1.278
- `skillFilters` — 確認版: Claude Code 2.1.278
- `maxSkillDescriptionChars` — 確認版: Claude Code 2.1.278（本リファレンスの 2026-06-22 版が「実在キー」として載せていた名前。誤りだった。正しくは `skillListingMaxDescChars`）

「ツール名指定で無効化」する仕組みは settings.json には用意されていない。**ただし skill 単位の
listing 制御は `skillOverrides` として実在する**（下記「実在キー一覧」）。

紛らわしいもの（実行ファイルにはあるが settings.json のキーではない。確認版: 2.1.278）:

- `disallowedTools` — subagent の定義（agent frontmatter）の項目。describe（binary 確認）: "Array of tool names to explicitly disallow for this agent. MCP server-level specs (mcp__server, mcp__server__*, mcp__*) remove every tool from the named server (or all MCP tools)." その agent の中でツールを外す用途で、メインセッションの常駐は変わらない
- `omitClaudeMd` — 同じく agent frontmatter の項目。describe（binary 確認）: "Run this agent without the user, project and local CLAUDE.md instruction files when it runs as a subagent; managed policy files are kept. For agents that take everything they need from the delegation prompt. No effect on the main session agent." subagent の context を軽くする用途

また `permissions.deny` は呼び出しブロックのみで、**ツール定義は context に残る**。context 削減効果はゼロ
（2026-06-22 時点の確認。2.1.278 でも未再確認）。

## 測定条件（2.1.278 の実測）

- Claude Code 2.1.278、2026-09-21、モデル claude-fable-5-1、context window 1M（auto-compact 967k）
- 非対話 `claude -p`、cwd はプロジェクト CLAUDE.md（推定 2.7k）を持つリポジトリ
- 検証環境: ユーザー CLAUDE.md 390 行（空行 216 行を除くと 174 行）と `@import` 5 本のうち読み込まれた 4 本（プロジェクト CLAUDE.md 2.7k を含めた Memory files の推定 26.8k・6 files）、ユーザー skill 57 本、bundled skill あり、MCP なし
- 非対話に載らないもの（モデルにツール名を列挙させて確認）: Artifact、AskUserQuestion、MCP（claude.ai コネクタを含む）。載る常駐ツール: Agent / Bash / Edit / Glob / Grep / ListAgents / Read / ReportFindings / ScheduleWakeup / Skill / ToolSearch / Workflow / Write と OS 依存のシェルツール。deferred（名前だけ常駐）: Cron 系 / DesignSync / EnterWorktree / ExitWorktree / Monitor / NotebookEdit / PushNotification / RemoteTrigger / SendMessage / TaskStop / WebFetch / WebSearch
- 測り方: `claude -p "Reply with exactly: OK" --output-format json --settings <json>` の `usage` にある `input_tokens` + `cache_creation_input_tokens` + `cache_read_input_tokens`（下記「再確認の方法」）
- baseline（何も入れない）: 60,284 tokens。同条件の `/context` の推定は 45.7k で、14.6k ずれる。カテゴリ配分も当てにならない（`skillListingMaxDescChars: 256` で推定は Total 不変だったが API では -2,277）
- 同じ設定を 2 回測った baseline と `disableBundledSkills: true` は、推定値が 2 回とも同値だった

## 実在キー一覧

確認版: Claude Code 2.1.278。「2.1.278 実測」列は上記の測定条件での API usage の差。

| キー | 効果 | 2.1.278 実測（API usage） | リスク |
|---|---|---|---|
| `disableBundledSkills: true` + `enableWorkflows: false` | bundled skill・workflow を完全に削除し、Workflow ツールも消す | **-3,563 tokens（settings キーの最大）** | `/loop` `/schedule` `/run` `/verify` `/code-review` `/simplify` 等と Workflow / ultracode が全部消える |
| `skillListingMaxDescChars`（既定 1536） | skill 一覧で Claude に送る各 skill の description の文字数上限 | 256 で -2,277 tokens | skill 一覧の description 末尾が切れ、skill matching の精度が落ちる |
| `enableWorkflows: false` | Workflow ツール定義を削除 + bundled workflow コマンド無効化 | -2,057 tokens | ローカル Workflow / ultracode モード / workflow 依存 skill（deep-research 等）が使えなくなる |
| `autoMemoryEnabled: false` | auto-memory ディレクトリ（`MEMORY.md`）の読み書きを停止する | -716 tokens（検証環境では auto-memory が空。減ったのは memory の指示文） | 過去セッションでの学び・修正の蓄積を AI が参照しなくなる |
| `includeGitInstructions: false` | 組み込みの commit/PR workflow + git status snapshot を system prompt から除去 | -649 tokens | 「直近のコミット何だっけ」を毎回自分で `git log` する必要が出る |
| `disableBundledSkills: true`（単独） | bundled skill・workflow を context から削除。built-in slash command は typable のまま AI からは隠す。plugins / `.claude/skills/` / `.claude/commands/` は影響を受けない | **+4,679 tokens（増える）** | 単独では逆効果。原因は未確認（下記） |
| `enableArtifact: false` | Artifact ツール定義を context から削除（claude.ai へセッション出力を公開する機能） | 対話の `/context` 推定: `disableClaudeAiConnectors: true` と同時に入れて System tools 19.4k → 14.6k（-4.8k・未分離・2026-09-21 作者環境）。非対話では測れない（説明文は 6,512 文字） | Artifact 未使用なら無リスク |
| `skillListingBudgetFraction`（既定 0.01） | skill listing 全体の文字数予算を context window に対する割合で指定。超過分は各 skill の description が自動的に短縮される | 未実測 | 下げすぎると全 skill の description がほぼ消える |
| `skillOverrides` | skill ごとに listing 表示を制御する（下記「使用例」） | 選び方次第。作者環境で 60 日未使用の 31 本（off 20・name-only 11）に当て、コネクタ off・autoMemory off と合わせて -6,250 tokens（2026-09-21。`/context` 推定では Skills 8.6k → 3.4k） | 誤って `off` にした skill が呼べなくなる |
| `disableClaudeAiConnectors: true` | claude.ai の MCP コネクタを取りに行かない（名前と server instructions の常駐が消える） | 未分離（上記の -6,250 に含む）。対話の `/context` なら 2.1.283 以降は "MCP server instructions" 行の差で推定できる | claude.ai 側で設定したコネクタが使えなくなる |
| `claudeMdExcludes: ["pattern1", "pattern2"]` | 特定の CLAUDE.md を loading 対象から除外（User / Project / Local の CLAUDE.md に適用） | 環境次第（未実測） | 必要な CLAUDE.md を誤って除外する事故。managed / policy の CLAUDE.md は除外できない |
| `disableAllHooks: true` | hooks と statusLine の実行を停止する | 未実測（hook の出力量に依存） | hooks に依存した自動化・ガードが止まる |

### `enableArtifact` / 旧キー `disableArtifact`

`enableArtifact` の describe() 文字列（binary 確認）: `"Turn the Artifact tool on or off. Off in any of managed,
--settings, or user settings wins; project and local settings can only turn it off. Unset defaults to on once the
feature is available."`

`disableArtifact` の describe() 文字列（binary 確認）: `"Deprecated: use enableArtifact: false. Still honored — true disables
the Artifact tool; false is ignored."`

旧キー `disableArtifact: true` は非推奨だが、まだ動作する（`true` は無効化として有効、`false` は無視される）。
新規に書くときは `enableArtifact: false` を使う。

Artifact ツールは対話セッション限定で、非対話（`-p`）には載らないため本リファレンスの API 実測では効果を取れていない。
実行ファイル内の説明文（"The Artifact tool renders an HTML file as an Artifact" から "does not suggest other ways to host or
share the page." まで）は 6,512 文字で、2026-06-22 版の「< 500 tokens」の推定はもう当てはまらない。効果は対話セッションで
`/context` の Before / After（推定値）を取って判断する。

### `enableWorkflows` / `disableWorkflows`

`enableWorkflows` の describe() 文字列（binary 確認・2.1.295 でも同じ）: `"Enable or disable the Workflows feature for this
user. Unset = default by plan once the feature is available."`

`disableWorkflows` の describe() 文字列（binary 確認・2.1.295）: `"Disable the Workflows feature. Code Review on pull
requests and /ultrareview run in Anthropic's cloud and are not stopped by this setting, except an /ultrareview that has
to restart partway through. …"`（以下、レビューを実行するマシン側の managed settings の扱いが続く。2.1.278 では
`"Disable the Workflows feature (also via CLAUDE_CODE_DISABLE_WORKFLOWS)."` だった）

公式 settings-reference（2026-10-10 fetch）の位置づけ: `disableWorkflows` は "Turn dynamic workflows off for everyone;
use `enableWorkflows` for yourself"、`enableWorkflows` は "Turn dynamic workflows on or off against your plan's default"。
`disableWorkflows` は非推奨ではなく、managed settings 等で全員に効かせるためのキー。`/config` の "Dynamic workflows"
トグルが書き込むのは `enableWorkflows` のほうで、自分だけ止めるならこちらを使う。通常の `settings.json` に
`disableWorkflows: true` を書いてもローカルの Workflow は止まる。どちらか一方でよい（両方書く必要はない）。
2.1.278 の API 実測で -2,057 tokens。

2.1.285 から、`disableWorkflows` を入れても Code Review の PR レビューと `/ultrareview` は止まらない（CHANGELOG:
"Changed Code Review's pull request reviews and `/ultrareview` to run when `disableWorkflows` is on, unless the machine
running the review has it set by its own administrator (MDM or the managed-settings file)"）。

### `disableBundledSkills`

describe() 文字列（binary 確認）: `"Disable the skills and workflows that ship with Claude Code: bundled
skills and workflows are removed entirely; built-in slash commands stay typable but are hidden from the
model. Plugins, .claude/skills/, and .claude/commands/ are unaffected. Equivalent to
CLAUDE_CODE_DISABLE_BUNDLED_SKILLS=1."`

環境変数 `CLAUDE_CODE_DISABLE_BUNDLED_SKILLS=1` と等価。自作 plugin・`.claude/skills/`・`.claude/commands/`
配下の skill / command には影響しない。

**2.1.278 の API 実測では、単独で入れると常駐が増える（+4,679 tokens）。** `enableWorkflows: false` と併用したときだけ
-3,563 tokens 減る。原因は未確認（Workflow ツールが残ると、bundled の workflow 系 skill を失った分を別の形で持つ、
という仮説。実行ファイルでは裏取りしていない）。削減目的で単独では入れない。

### `skillOverrides` の使用例

describe() 文字列（binary 確認）: `"Per-skill listing overrides keyed by skill name. "name-only" lists the
skill without its description; "user-invocable-only" hides it from the model but keeps /name; "off" hides
it from both. Absent = on."`

```json
{
  "skillOverrides": {
    "some-rarely-used-skill": "name-only",
    "another-skill": "off"
  }
}
```

値は 4 種類:

- `"on"`（既定・省略時と同じ） — 通常どおり表示
- `"name-only"` — description を隠して名前だけ listing に残す
- `"user-invocable-only"` — AI（listing）からは隠すが `/name` でユーザーが呼ぶことはできる
- `"off"` — listing からも `/name` 呼び出しからも完全に隠す

`disableBundledSkills` が bundled skill を一括で消すのに対し、`skillOverrides` は skill 単位で調整できる。
「ほとんど使わないが完全には消したくない skill」がある場合は、まずこちらを検討する。claude.ai アカウントで
有効にした skill / plugin がターミナルへ同期される版（2.1.264〜2.1.278 の changelog）では、それらも listing に
載るので、不要なら同じ方法で個別に落とす。

### `skillListingMaxDescChars` / `skillListingBudgetFraction`

`skillListingMaxDescChars` の describe() 文字列（binary 確認）: `"Per-skill description character cap in the skill
listing sent to Claude (default: 1536). Descriptions longer than this are truncated. Raise to opt in to higher
per-turn context cost."`

`skillListingBudgetFraction` の describe() 文字列（binary 確認）: `"Fraction of the context window (in characters)
reserved for the skill listing sent to Claude (default: 0.01 = 1%). When the listing exceeds this, descriptions are
shortened to fit. Raise to opt in to higher per-turn context cost."`

listing 予算の算式（binary の定数から確認）: `context window（既定 200000 tokens）× 4 文字/token ×
skillListingBudgetFraction（既定 0.01）`。既定値で計算すると 200000 × 4 × 0.01 = 8000 文字程度になる。

**この予算は context window の大きさに比例してスケールする**（算式からの推定）。1M 窓の環境では、既定の
`skillListingBudgetFraction: 0.01` のままでも listing に使える文字数予算が 40,000 文字に広がる。skill が多い環境ほど
`skillListingMaxDescChars` を下げる効果が大きく、検証環境（ユーザー skill 57 本・1M 窓）では 256 で -2,277 tokens だった。

環境変数 `SLASH_COMMAND_TOOL_CHAR_BUDGET` を設定すると、両方の設定より優先して listing の文字数上限を
直接指定できる（binary 確認。公式 docs の skills.md にも記載）。

### `claudeMdExcludes`

describe() 文字列（binary 確認）: `"Glob patterns or absolute paths of CLAUDE.md files to exclude from
load… (Managed/policy files cannot be excluded). Examples: "/path/to/monorepo/CLAUDE.md",
"**/code/CLAUDE.md", "**/some-dir/.claude/rules/**""`

（1 つ目の例は原文では POSIX のホーム絶対パス表記。ホームパスをそのまま載せない本リポの方針で
`/path/to/…` に一般化した。意味は「絶対パスも書ける」で変わらない。）

User / Project / Local の各階層の CLAUDE.md に適用される。**managed（組織管理）/ policy の CLAUDE.md は
このキーで除外できない。**

### `autoMemoryEnabled`

describe() 文字列（binary 確認）: `"Enable auto-memory for this project. When false, Claude will not read from or
write to the auto-memory directory."`

毎セッション `MEMORY.md` の先頭 200 行または 25KB が常駐する（公式 docs の memory.md、2026-09-21 fetch: "The first 200
lines of `MEMORY.md`, or the first 25KB, whichever comes first, are loaded at the start of every conversation."）。
これを止めるのが `autoMemoryEnabled: false`（または環境変数 `CLAUDE_CODE_DISABLE_AUTO_MEMORY=1`）。

2.1.278 の auto-memory は「`MEMORY.md` を索引として常駐させ、topic ファイルは関連するものだけ recall で
system-reminder に入れる」設計になっている（実行ファイルの文字列: "the MEMORY.md index first when it exists on disk,
then the memories ordered by file modification time, newest first" / "Recalled memories appearing inside
`<system-reminder>` blocks are background context"）。topic ファイルに本文を移しても起動時に全部は載らない。

API 実測は -716 tokens。検証環境ではプロジェクトの auto-memory が空だったので、減ったのは主に memory の指示文で、
`MEMORY.md` が大きい環境では差が広がる。

### `disableClaudeAiConnectors`

describe() 文字列（binary 確認・2.1.295）: `"When true in any settings source, claude.ai MCP cloud connectors are not
auto-fetched or connected, and a claudeai-proxy server passed explicitly (e.g. via --mcp-config or the SDK mcpServers
option) does not connect either. Any-source-true wins: a project can opt out, but a project-level false cannot override a
user-level true."`（2.1.278 では最初の文だけだった）

claude.ai 側で接続したコネクタ（MCP）は、ターミナルでも名前と server instructions が常駐する（MCP のスキーマ自体は
Tool Search で deferred）。使っていないなら `true` で取りに行かなくなる。API での削減量は未実測（非対話では MCP が載らない）。
対話の `/context` では、2.1.283 から server instructions が "MCP server instructions" の独立した行で出て Total に入る
（CHANGELOG: "Fixed `/context` not counting MCP server instructions: they now appear as their own row and count toward the
total"）ので、この行の Before / After で推定できる。
claude.ai から同期される skill / plugin は別物で、`skillOverrides` で個別に落とす。

### `autoDreamEnabled` / `autoMemoryDirectory`（削減用途ではない）

`autoDreamEnabled` の describe() 文字列（binary 確認）: `"Enable background memory consolidation (auto-dream). When set,
overrides the server-side default."`

`autoMemoryDirectory` の describe() 文字列（binary 確認）: `"Custom directory path for auto-memory storage. Supports ~/
prefix for home directory expansion. Ignored if set in projectSettings (checked-in .claude/settings.json) for security.
When unset, defaults to ~/.claude/projects/<sanitized-cwd>/memory/."`

どちらも context を減らすキーではない。memory 系の設定として存在を知っておくために載せる。

## 推奨適用パターン

**適用する前に `/context` の Total 行で分母を確認する。** 表の「2.1.278 実測」は絶対量であって、Total に対する割合ではない。
現行モデルは 1M 窓が既定なので（公式 model-config、2026-10-10 fetch: "Fable 5.1, Fable 5, Sonnet 5 and later, Haiku 5.5,
and Opus 4.7 and later run with the 1M window by default, with no `[1m]` suffix needed. That includes sessions on Amazon
Bedrock, Google Cloud's Agent Platform, and Microsoft Foundry, and Claude apps gateway sessions."）、最大の -3,563 tokens でも
Total の 0.4%。200k 窓（Opus 4.6 / Sonnet 4.6 を `[1m]` なしで使う、1M 非対応のモデル、`CLAUDE_CODE_DISABLE_1M_CONTEXT=1`
等）では同じ量が 1.8% になる。Bedrock / Vertex / Foundry は 2.1.287 から既定で 1M。

### A. まず測る（何も入れない）

1M 窓ならこれが既定。`/context` の Total 行と `/usage` の "Prompt cache (main)" 行を見て、
SKILL.md Step 1 の判定 3 分岐（200k 窓か / CLAUDE.md が 200 行超か / cache miss が多いか）へ進む。
どれにも当てはまらなければ settings キーは入れない。

### B. Workflow / ultracode を使わない人

```json
{
  "enableWorkflows": false
}
```

-2,057 tokens。必要になったら該当行を削除して再起動で即復活。`disableWorkflows: true` でもローカルの Workflow は止まるが、こちらは全員に効かせるためのキー（上記）。

### C. skill が多い人

```json
{
  "skillListingMaxDescChars": 512
}
```

検証環境では 256 で -2,277 tokens。値は skill matching の精度と相談して決める（256 は起動語が切れる skill が出る）。
特定の skill だけ消したいなら `skillOverrides` で `name-only` / `off`。

### D. 入れない: `disableBundledSkills: true` 単独

単独では +4,679 tokens 増える。bundled skill を封印したいなら `enableWorkflows: false` と併用し（-3,563 tokens）、
`/loop` `/schedule` `/run` `/verify` `/code-review` `/simplify` 等が全部消えることを承知で入れる。

### Artifact

`enableArtifact: false` は安全だが、非対話では測れない。通常のセッションで `/context` を貼り、`~/.claude/settings.json` に
`"enableArtifact": false` を足して新セッションで `/context` を貼り、差を「推定」として記録してから決める。

## 環境変数による削減

settings.json の代わりに環境変数で制御できるものもある:

- `CLAUDE_CODE_DISABLE_ARTIFACT=1` — `enableArtifact: false`（旧 `disableArtifact: true`）と同等
- `CLAUDE_CODE_DISABLE_WORKFLOWS` — Claude Code の起動前に環境へ置くと Workflow を止める（`disableWorkflows` の describe():
  "Set in the environment before Claude Code starts, CLAUDE_CODE_DISABLE_WORKFLOWS disables Workflows."。2.1.295）
- `CLAUDE_CODE_DISABLE_BUNDLED_SKILLS` — `disableBundledSkills: true` と同等
- `CLAUDE_CODE_DISABLE_AUTO_MEMORY=1` — `autoMemoryEnabled: false` と同等
- `SLASH_COMMAND_TOOL_CHAR_BUDGET` — skill listing の文字数上限を直接指定する（文字数）。
  `skillListingMaxDescChars` / `skillListingBudgetFraction` より優先される
- `ENABLE_TOOL_SEARCH` — 既定 ON。MCP ツールのスキーマを遅延ロードする機能（Tool Search）のスイッチ。
  **`false` にすると MCP ツールのスキーマが全部常駐化し、context 削減とは逆方向に働く。** 下げる用途では
  ない
- `CLAUDE_CODE_DISABLE_1M_CONTEXT` — 実行ファイルに文字列あり（"1M context is turned off here
  (CLAUDE_CODE_DISABLE_1M_CONTEXT is set)"）。context 窓を 200k に戻す用途で、常駐を減らすキーではない

ただし環境変数経由は変更管理しづらいので、設定できるものは `settings.json` に書く方が安全。

## 注意事項

- 公式 docs は変わる可能性がある。**最新は <https://code.claude.com/docs/en/settings-reference> を参照**
- このリファレンスは 2026-09-21 時点、Claude Code 2.1.278 で確認したもの。バージョンが上がったら再確認が必要
- ベータ機能や非公開のキーは記載していない
- **キー名の再確認**: `settings.json` のキー名は非公開のスキーマ実装に依存するため、公式 docs に載っていない
  項目は最終的に実行ファイル自体で裏取りするのが確実。PowerShell で ripgrep（`rg`）を使い、`-a`（バイナリを
  テキスト扱いするオプション）を付けて `claude.exe` の文字列を直接検索する。キー名だけでなく、zod スキーマの
  `describe()` 文字列（キー名の直後に続く英語の説明文）まで拾えば、効果の説明もそのまま裏取りできる。再現
  コマンドの例（`$rg` には手元の rg.exe のフルパスを入れる）:

```powershell
$rg = '<rg.exe のフルパス>'
$exe = "$env:USERPROFILE\.local\bin\claude.exe"
# キー名の直後の describe() を全文拾う。minify 後の変数名に依存せず、引用符が " でも ' でも当たる
$pat = '(skillOverrides|enableWorkflows|disableWorkflows|enableArtifact|disableArtifact|disableBundledSkills|disableClaudeAiConnectors):.{0,160}?describe\((?:"[^"]{0,1200}"|''[^'']{0,1200}'')\)'
& $rg -a -o --no-filename $pat $exe | Sort-Object -Unique
# 存在の判定はキー名そのもので数える
& $rg -a -o --no-filename 'disabledTools|enabledTools|maxSkillDescriptionChars|skillListingMaxDescChars' $exe | Group-Object | ForEach-Object { "{0} : {1}" -f $_.Name, $_.Count }
```

  最短経路は「探しているキー名を含む describe() パターンを rg で当てて、返ってきた説明文をそのまま引用する」こと。説明文が返れば「実在キー一覧」側に分類できる。**describe() が返らないだけでは「存在しない」側に分類しない。** 2.1.278 時点の再現コマンドは、2.1.295 で minify 後の変数名が変わって `skillOverrides` が 0 件になり、`disableWorkflows` は説明文が 400 文字を超えて取りこぼした（どちらも実在する）。「存在しない」と書くのは、キー名そのもので数えて 0 件だったときだけにする（2 本目のコマンド）。

- **削減量の再確認**: `/context` は推定なので、削減量は API usage で取る。設定断片を JSON ファイルに書き、非対話で
  1 往復させて `usage` の 3 フィールドを合計する（baseline は `--settings` 無し）。応答は 1 つのオブジェクト（`type: result`）で返る:

```bash
echo '{"enableWorkflows":false}' > /tmp/s.json
claude -p "Reply with exactly: OK" --output-format json --settings /tmp/s.json \
  | jq '.usage | .input_tokens + .cache_creation_input_tokens + .cache_read_input_tokens'
```

```powershell
Set-Content -Path "$env:TEMP\s.json" -Value '{"enableWorkflows":false}'
$r = claude -p "Reply with exactly: OK" --output-format json --settings "$env:TEMP\s.json" | ConvertFrom-Json
$r.usage.input_tokens + $r.usage.cache_creation_input_tokens + $r.usage.cache_read_input_tokens
```

  非対話には Artifact / AskUserQuestion / MCP が載らないので、それらの効果は対話セッションで `/context` の Before / After を取る。

## 関連

- 公式 settings docs: <https://code.claude.com/docs/en/settings-reference>（キーごとの定義）/ <https://code.claude.com/docs/en/settings>（設定ファイルの仕組み）
- 公式 environment variables: <https://code.claude.com/docs/en/env-vars>
- 公式 costs（context を減らす公式の案内。"Move instructions from CLAUDE.md to skills" 等）: <https://code.claude.com/docs/en/costs>
- 元記事: [Zenn 記事](https://zenn.dev/ishizakahiroshi/articles/20260622-claude-code-context-diet)（2026-06-22 時点・200k 窓の内容。settings キー名と削減量は本リファレンスの方が新しい）
