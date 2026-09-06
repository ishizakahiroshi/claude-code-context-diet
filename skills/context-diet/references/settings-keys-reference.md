# settings.json 公式キー リファレンス（Context 削減目的）

確認版: Claude Code 2.1.263（2026-09-06、実行ファイルの文字列走査 + 公式 docs 照合）

Claude Code の `settings.json` で context 削減に関係する公式キーの完全リファレンス。
公式 docs（<https://code.claude.com/docs/en/settings>）の記載と、Claude Code 2.1.263 実行ファイルの
文字列走査（zod スキーマの `describe()` 文字列を含む）を突き合わせ、効果とリスクを整理した。
下記の describe() 引用は実行ファイルで確認できたもの（binary 確認）。binary で確認できず docs のみで
裏取りした事実は個別に「公式 docs による」と注記する。

## ⚠️ まず: 「存在しない」キー

以下は実行ファイルの文字列走査で 0 件だった。書いても無視される（または無効として捨てられる）:

- `disabledTools` — 確認版: Claude Code 2.1.263
- `enabledTools` — 確認版: Claude Code 2.1.263
- `disabledSkills` — 確認版: Claude Code 2.1.263
- `enabledSkills` — 確認版: Claude Code 2.1.263
- `skillFilters` — 確認版: Claude Code 2.1.263
- `maxSkillDescriptionChars` — 確認版: Claude Code 2.1.263（本リファレンスの 2026-06-22 版が「実在キー」として載せていた名前。誤りだった）

「ツール名指定で無効化」する仕組みは現状の Claude Code には用意されていない。**ただし skill 単位の
listing 制御は `skillOverrides` として実在する**（下記「実在キー一覧」）。2026-06-22 版の本リファレンスは
`skillOverrides` を「存在しない」側に、`maxSkillDescriptionChars` を「実在」側に書いていたが、2.1.263 の
実行ファイル走査では逆だった。skill description を切り詰める正しいキー名は `skillListingMaxDescChars` /
`skillListingBudgetFraction`（同じく下記）。

また `permissions.deny` は呼び出しブロックのみで、**ツール定義は context に残る**。context 削減効果はゼロ
（2026-06-22 時点の確認。2.1.263 では未再確認）。

## 実在キー一覧

確認版: Claude Code 2.1.263。

| キー | 効果 | 推定削減（2026-06-22 時点の推定値・2.1.263 未再測） | リスク |
|---|---|---|---|
| `enableArtifact: false` | Artifact ツール定義を context から削除（claude.ai へセッション出力を公開する機能） | < 500 tokens | Artifact 未使用なら無リスク |
| `enableWorkflows: false` | Workflow ツール定義を削除 + bundled workflow コマンド無効化 | **5〜10k tokens（最大効果）** | ローカル Workflow / ultracode モード / workflow 依存 skill（deep-research 等）が使えなくなる |
| `disableBundledSkills: true` | bundled skill・workflow を context から完全に削除。built-in slash command は typable のまま AI からは隠す。plugins / `.claude/skills/` / `.claude/commands/` は影響を受けない | 〜1.7k tokens | `/loop` `/schedule` `/run` `/verify` `/code-review` `/simplify` 等が全部封印される（過激） |
| `skillListingMaxDescChars`（既定 1536） | skill 一覧で Claude に送る各 skill の description の文字数上限 | 未実測（2026-06-22 版の「〜1.5k tokens」は実在しないキー名に付いていた推定なので撤回） | skill 一覧の description 末尾が切れ、skill matching の精度が落ちる |
| `skillListingBudgetFraction`（既定 0.01） | skill listing 全体の文字数予算を context window に対する割合で指定。超過分は各 skill の description が自動的に短縮される | 環境次第（未実測） | 下げすぎると全 skill の description がほぼ消える |
| `skillOverrides` | skill ごとに listing 表示を制御する（下記「使用例」） | 環境次第（未実測） | 誤って `off` にした skill が呼べなくなる |
| `includeGitInstructions: false` | 組み込みの commit/PR workflow + git status snapshot を system prompt から除去 | 〜1k tokens | 「直近のコミット何だっけ」を毎回自分で `git log` する必要が出る |
| `claudeMdExcludes: ["pattern1", "pattern2"]` | 特定の CLAUDE.md を loading 対象から除外（User / Project / Local の CLAUDE.md に適用） | 環境次第 | 必要な CLAUDE.md を誤って除外する事故。managed / policy の CLAUDE.md は除外できない |
| `autoMemoryEnabled: false` | auto-memory ディレクトリ（`MEMORY.md`）の読み書きを停止する | 未実測（数字なし） | 過去セッションでの学び・修正の蓄積を AI が参照しなくなる |
| `disableAllHooks: true` | hooks と statusLine の実行を停止する（describe: "Disable all hooks and statusLine execution"） | 未実測（hook の出力量に依存） | hooks に依存した自動化・ガードが止まる |

### `enableArtifact` / 旧キー `disableArtifact`

describe() 文字列（binary 確認）: `"Deprecated: use enableArtifact: false. Still honored — true disables
the Artifact tool; false is ignored."`

旧キー `disableArtifact: true` は非推奨だが、まだ動作する（`true` は無効化として有効、`false` は無視される）。
新規に書くときは `enableArtifact: false` を使う。

### `enableWorkflows` / 旧キー `disableWorkflows`

`enableWorkflows` の describe() 文字列（binary 確認）: `"Enable or disable the Workflows feature for this
user. Unset = default by plan once the feature is available."`

`disableWorkflows` の describe() 文字列（binary 確認）: `"Disable the Workflows feature (also via
CLAUDE_CODE_DISABLE_WORKFLOWS)"`

`/config` の "Dynamic workflows" トグルが書き込むのは `enableWorkflows` のほう。`disableWorkflows: true` は
managed 向けの制限用途に分類されているが、通常の `settings.json` でも引き続き有効。どちらか一方でよい
（両方書く必要はない）。

### `disableBundledSkills`

describe() 文字列（binary 確認）: `"Disable the skills and workflows that ship with Claude Code: bundled
skills and workflows are removed entirely; built-in slash commands stay typable but are hidden from the
model. Plugins, .claude/skills/, and .claude/commands/ are unaffected. Equivalent to
CLAUDE_CODE_DISABLE_BUNDLED_SKILLS=1."`

環境変数 `CLAUDE_CODE_DISABLE_BUNDLED_SKILLS=1` と等価。自作 plugin・`.claude/skills/`・`.claude/commands/`
配下の skill / command には影響しない。

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
「ほとんど使わないが完全には消したくない skill」がある場合は、まずこちらを検討する。

### `skillListingMaxDescChars` / `skillListingBudgetFraction`

listing 予算の算式（binary の定数から確認）: `context window（既定 200000 tokens）× 4 文字/token ×
skillListingBudgetFraction（既定 0.01）`。既定値で計算すると 200000 × 4 × 0.01 = 8000 文字程度になる。

**この予算は context window の大きさに比例してスケールする**（算式からの推定。実測なし）。`[1m]` サフィックス付きモデルなど 1M
トークン級の環境では、既定の `skillListingBudgetFraction: 0.01` のままでも listing に使える文字数予算が
自動的に大きくなる。1M トークン級で listing を絞りたい場合は、`skillListingMaxDescChars` を下げるか
`skillListingBudgetFraction` の割合自体を下げる必要がある。

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

describe() 文字列（binary 確認）: `"When false, Claude will not read from or write to the auto-memory
directory."`

毎セッション `MEMORY.md` の先頭 200 行または 25KB が常駐する（公式 docs の memory.md による。実行ファイル
にも `MEMORY.md` の直後に定数 200 / 25000 があることを確認）。これを止めるのが `autoMemoryEnabled: false`
（または環境変数 `CLAUDE_CODE_DISABLE_AUTO_MEMORY=1`）。削減量は未実測のため数字は書かない。

## 推奨適用パターン

**適用する前に `/context` の Total 行（例: `343.1k/967k tokens`）で context window の上限を確認する。** 表の「推定削減」は絶対量であって、Total に対する割合ではない。200k トークン級の環境では B パターンの合計 5〜10k tokens は Total の数%〜十数%になり体感できるが、`[1m]` サフィックス付きモデルなど 1M トークン級の環境では同じ削減量が Total の 1% 未満になることがある（2026-08-18 実測）。**上限が大きい環境ほど、機能を犠牲にするキー（`enableWorkflows: false` 等）を入れる前に、その割合を計算してから決める。**

### A. 安全最優先（誰にでも推奨）

```json
{
  "enableArtifact": false
}
```

リスクゼロ。効果も小さい（< 500 tokens、2026-06-22 時点の推定・2.1.263 未再測）。旧キー `disableArtifact: true`
も引き続き動作する（非推奨だが無効化はされていない）。

### B. バランス型（多くのケース推奨）

```json
{
  "enableArtifact": false,
  "enableWorkflows": false
}
```

Workflow / ultracode を能動的に使っていなければ最適。最大効果（合計 5〜10k tokens、2026-06-22 時点の推定・
2.1.263 未再測）が得られる。必要になったら該当行を削除して再起動で即復活。旧キー `disableWorkflows: true`
も同じ効果で引き続き動作する。

### C. 過激（基本非推奨）

```json
{
  "enableArtifact": false,
  "enableWorkflows": false,
  "disableBundledSkills": true,
  "skillListingMaxDescChars": 512,
  "includeGitInstructions": false
}
```

ヘビーに削るが副作用も多い（`disableBundledSkills` 〜1.7k tokens、`includeGitInstructions: false`
〜1k tokens。いずれも 2026-06-22 時点の推定・2.1.263 未再測。`skillListingMaxDescChars: 512` は未実測）。
`/loop` `/schedule` `/run` `/verify` `/code-review` `/simplify` 等の bundled slash command が全部封印
される。**自分で何を失うか把握してから入れる**。

bundled skill を一括封印せず、使っていない skill だけを個別に消したい場合は、`disableBundledSkills` の
代わりに `skillOverrides` で対象を絞る方法もある（上記「`skillOverrides` の使用例」）。こちらは削減量が
skill の選び方次第で変わるため未実測。

## 環境変数による削減

settings.json の代わりに環境変数で制御できるものもある:

- `CLAUDE_CODE_DISABLE_ARTIFACT=1` — `enableArtifact: false`（旧 `disableArtifact: true`）と同等
- `CLAUDE_CODE_DISABLE_WORKFLOWS` — `enableWorkflows: false`（旧 `disableWorkflows: true`）と同等
- `CLAUDE_CODE_DISABLE_BUNDLED_SKILLS` — `disableBundledSkills: true` と同等
- `CLAUDE_CODE_DISABLE_AUTO_MEMORY=1` — `autoMemoryEnabled: false` と同等
- `SLASH_COMMAND_TOOL_CHAR_BUDGET` — skill listing の文字数上限を直接指定する（文字数）。
  `skillListingMaxDescChars` / `skillListingBudgetFraction` より優先される
- `ENABLE_TOOL_SEARCH` — 既定 ON。MCP ツールのスキーマを遅延ロードする機能（Tool Search）のスイッチ。
  **`false` にすると MCP ツールのスキーマが全部常駐化し、context 削減とは逆方向に働く。** 下げる用途では
  ない

ただし環境変数経由は変更管理しづらいので、設定できるものは `settings.json` に書く方が安全。

## 注意事項

- 公式 docs は変わる可能性がある。**最新は <https://code.claude.com/docs/en/settings> を参照**
- このリファレンスは 2026-09-06 時点、Claude Code 2.1.263 で確認したもの。バージョンが上がったら再確認が必要
- ベータ機能や非公開のキーは記載していない
- **再確認の方法**: `settings.json` のキー名は非公開のスキーマ実装に依存するため、公式 docs に載っていない
  項目は最終的に実行ファイル自体で裏取りするのが確実。PowerShell で ripgrep（`rg`）を使い、`-a`（バイナリを
  テキスト扱いするオプション）を付けて `claude.exe` の文字列を直接検索する。キー名だけでなく、zod スキーマの
  `describe()` 文字列（キー名の直後に続く英語の説明文）まで拾えば、効果の説明もそのまま裏取りできる。再現
  コマンドの例（`$rg` には手元の rg.exe のフルパスを入れる）:

```powershell
$rg = '<rg.exe のフルパス>'
$exe = "$env:USERPROFILE\.local\bin\claude.exe"
& $rg -a -o --no-filename 'skillOverrides:pe\(s\(\),K\(\["on".{0,700}' $exe
& $rg -a -o --no-filename '(disableWorkflows|disableArtifact|enableArtifact|disableBundledSkills):D\(\)\.optional\(\)\.describe\("[^"]{0,400}"' $exe
& $rg -a -o --no-filename 'disabledTools|enabledTools|maxSkillDescriptionChars|skillListingMaxDescChars' $exe | Group-Object | ForEach-Object { "{0} : {1}" -f $_.Name, $_.Count }
```

  最短経路は「探しているキー名を含む describe() パターンを rg で当てて、返ってきた説明文をそのまま引用する」こと。0 件なら「存在しない」側、説明文が返れば「実在キー一覧」側に分類できる。

## 関連

- 公式 settings docs: <https://code.claude.com/docs/en/settings>
- 公式 environment variables: <https://code.claude.com/docs/en/environment-variables>
- 元記事: [Zenn 記事](https://zenn.dev/ishizakahiroshi/articles/20260622-claude-code-context-diet)（2026-06-22 時点の内容。settings キー名は本リファレンスの方が新しい）
