# settings.json 公式キー リファレンス（Context 削減目的）

Claude Code の `settings.json` で context 削減に関係する公式キーの完全リファレンス。
公式 docs（<https://code.claude.com/docs/en/settings>）の記載をベースに、効果とリスクを整理した。

## ⚠️ まず: 「存在しない」キー

公式 docs に **記載なし**。書いても無視される（または無効として捨てられる）:

- `disabledTools`
- `enabledTools`
- `skillOverrides`
- `disabledSkills`
- `enabledSkills`
- `skillFilters`

「ツール名指定で無効化」する仕組みは現状の Claude Code には用意されていない。代わりに下記のキーを使う。

また `permissions.deny` は呼び出しブロックのみで、**ツール定義は context に残る**。context 削減効果はゼロ。

## 実在キー一覧

| キー | 効果 | 推定削減 | リスク |
|---|---|---|---|
| `disableArtifact: true` | Artifact ツール定義を context から削除（claude.ai へセッション出力を公開する機能） | < 500 tokens | Artifact 未使用なら無リスク |
| `disableWorkflows: true` | Workflow ツール定義を削除 + bundled workflow コマンド無効化 | **5〜10k tokens（最大効果）** | ローカル Workflow / ultracode モード / workflow 依存 skill（deep-research 等）が使えなくなる |
| `disableBundledSkills: true` | bundled skill 群を context から削除。`/init` 等の slash command は typable のまま隠す | 〜1.7k tokens | `/loop` `/schedule` `/run` `/verify` `/code-review` `/simplify` 等が全部封印される（過激） |
| `maxSkillDescriptionChars: 512`（既定 1536） | 各 skill の description を切り詰める | 〜1.5k tokens | skill 一覧の description 末尾が切れ、skill matching の精度が落ちる |
| `includeGitInstructions: false` | 組み込みの commit/PR workflow + git status snapshot を system prompt から除去 | 〜1k tokens | 「直近のコミット何だっけ」を毎回自分で `git log` する必要が出る |
| `claudeMdExcludes: ["pattern1", "pattern2"]` | 特定 CLAUDE.md を loading 対象から除外 | 環境次第 | 必要な CLAUDE.md を誤って除外する事故 |

## 推奨適用パターン

**適用する前に `/context` の Total 行（例: `343.1k/967k tokens`）で context window の上限を確認する。** 表の「推定削減」は絶対量であって、Total に対する割合ではない。200k トークン級の環境では B パターンの合計 5〜10k tokens は Total の数%〜十数%になり体感できるが、`[1m]` サフィックス付きモデルなど 1M トークン級の環境では同じ削減量が Total の 1% 未満になることがある（2026-08-18 実測）。**上限が大きい環境ほど、機能を犠牲にするキー（`disableWorkflows` 等）を入れる前に、その割合を計算してから決める。**

### A. 安全最優先（誰にでも推奨）

```json
{
  "disableArtifact": true
}
```

リスクゼロ。効果も小さい（数百 tokens）。

### B. バランス型（多くのケース推奨）

```json
{
  "disableArtifact": true,
  "disableWorkflows": true
}
```

Workflow / ultracode を能動的に使っていなければ最適。最大効果（合計 -5〜-10k tokens）が得られる。
必要になったら `disableWorkflows` の 1 行を削除して再起動で即復活。

### C. 過激（基本非推奨）

```json
{
  "disableArtifact": true,
  "disableWorkflows": true,
  "disableBundledSkills": true,
  "maxSkillDescriptionChars": 512,
  "includeGitInstructions": false
}
```

ヘビーに削るが副作用も多い。`/loop` `/schedule` `/run` `/verify` `/code-review` 等の bundled slash command が全部封印される。**自分で何を失うか把握してから入れる**。

## 環境変数による削減

settings.json の代わりに環境変数で制御できるものもある:

- `CLAUDE_CODE_DISABLE_ARTIFACT=1` — `disableArtifact` と同等

ただし環境変数経由は変更管理しづらいので、`settings.json` に書く方が安全。

## 注意事項

- 公式 docs は変わる可能性がある。**最新は <https://code.claude.com/docs/en/settings> を参照**
- このリポジトリの記載は 2026-06-22 時点のもの
- ベータ機能や非公開のキーは記載していない

## 関連

- 公式 settings docs: <https://code.claude.com/docs/en/settings>
- 公式 environment variables: <https://code.claude.com/docs/en/environment-variables>
- 元記事: [Zenn 記事](https://zenn.dev/ishizakahiroshi/articles/claude-code-context-diet)
