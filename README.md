# claude-code-context-diet

Claude Code の常駐 context を実測ベースで棚卸しし、削る価値を判定してから減らすための手順集と簡易スキル。

`/context` を開いたら「何もしていないのに 80k tokens 食ってた」という人向け。元になった検証（2026-06-22・200k 窓）では総量を **80.7k → 69.5k tokens（-11.2k / -13.9%）** まで減らせました。

2026-09-21 に Claude Code 2.1.278 で再測し、目的を置き直しました。Anthropic API 直結では現行モデルが 1M 窓を既定で使うため、同じ常駐 80k でも Total の 8% にしかならず、`settings.json` のキーで削れるのは最大でも約 3.6k tokens（`disableBundledSkills: true` + `enableWorkflows: false` で -3,563。Total の 0.4%）でした。削る理由は「トークン量」から「指示の遵守率」と「prompt cache が外れたときの再処理量」へ移ります。

## まず判断すること

1. `/context` の Total 行の分母を見る。1M 窓ならトークン目的では削らず、目的を遵守率と cache miss 時の再処理量に置く
2. 行数を減らす前に、内容を 4 観点（矛盾・重複・陳腐化・曖昧）で監査する。CLAUDE.md・`@import`・memory・ツールが注入するファイルの間の矛盾は、トークンを減らしても残る
3. CLAUDE.md が 1 ファイル 200 行を超えていれば、公式の目安どおり skills / `.claude/rules/` へ移す（`@import` に分けても context は減らない）
4. `settings.json` のキーは最後。`disableBundledSkills: true` は単独で常駐が増える（2.1.278 実測 +4,679 tokens）ので入れない
5. Skills と System tools は使用実績で切る。`/skill-doctor` だけで判断せず、transcript の tool_use を 60 日ぶん数える。設定はどの settings.json が読まれているか（`CLAUDE_CONFIG_DIR`）を先に確かめる

## 何が入っているか

| ファイル | 用途 |
|---|---|
| [`skills/context-diet/SKILL.md`](skills/context-diet/SKILL.md) | スキル本体。AI に「コンテキスト棚卸して」と頼むと走る。削る価値の判定（3 分岐）→ 内容の監査（4 観点）→ 4 分類 → settings キーの順 |
| [`skills/context-diet/references/playbook.md`](skills/context-diet/references/playbook.md) | 詳細手順。判定 3 分岐、内容の監査の 4 観点と原則、skill の作り方、API usage で削減量を実測する 1 行コマンド |
| [`skills/context-diet/references/settings-keys-reference.md`](skills/context-diet/references/settings-keys-reference.md) | Claude Code `settings.json` で実在する context 削減キーの一覧と 2.1.278 の API 実測。`disabledTools` は存在しない（Claude Code 2.1.278 で確認）という事実つき |
| [`skills/context-diet/references/claude-md-split-pattern.md`](skills/context-diet/references/claude-md-split-pattern.md) | CLAUDE.md の章を skill / rules / トリガー文へ移す具体パターン |
| [`.claude-plugin/`](.claude-plugin) | plugin / marketplace の manifest。`/plugin` で導入するときに使われる |
| `LICENSE` | MIT |

`references/` の 3 本は SKILL.md から参照されたときだけ読み込まれるので、常駐 context を増やしません。

## 使い方（最短ルート）

### 1. plugin として入れる（推奨）

```
/plugin marketplace add ishizakahiroshi/claude-code-context-diet
/plugin install context-diet@ishizakahiroshi
```

Claude Code を再起動すると使えます。AI に「コンテキスト棚卸して」と言えば発火し、`/context` 取得 → 分類 → 削減案提示の手順で進みます。スラッシュで直接呼ぶ場合は `/context-diet:context-diet`（plugin のスキルは `plugin名:スキル名` の形になります）。

更新するときは次の 2 つ。plugin 側の `version` が上がったときだけ更新が届きます。

```
/plugin marketplace update ishizakahiroshi
/plugin update context-diet@ishizakahiroshi
```

### 2. 手動で置く

`skills/context-diet/` を**ディレクトリごと** `~/.claude/skills/context-diet/` へコピーして再起動。この場合のスラッシュ形式は `/context-diet` です。

旧手順（`SKILL.md` 1 本だけをコピーする方法）でも起動はしますが、`references/` が付いてこないので SKILL.md 内のリンクが切れ、詳細手順と settings キー一覧に辿れません。ディレクトリごとコピーしてください。

スキル形式に乗らない場合は [`skills/context-diet/references/playbook.md`](skills/context-diet/references/playbook.md) を手で読みながらやっても同じ結果になります。

## 関連記事

どちらも 2026-06-22 時点（200k 窓・旧キー名 `disableArtifact` / `disableWorkflows` の時代）の記述です。現行の判定基準は [`skills/context-diet/references/playbook.md`](skills/context-diet/references/playbook.md) Step 1、現行のキー名と 2.1.278 の実測値は [`skills/context-diet/references/settings-keys-reference.md`](skills/context-diet/references/settings-keys-reference.md) を参照してください。

- 詳しい背景と Before/After 数値: [Zenn 記事「Claude Code の常駐 context を 14% 削った話。disabledTools は存在しなかった」](https://zenn.dev/ishizakahiroshi/articles/20260622-claude-code-context-diet)
- 図解版・詳細版: [ishizakahiroshi.github.io/articles/2026-06-22-claude-code-context-diet/](https://ishizakahiroshi.github.io/articles/2026-06-22-claude-code-context-diet/)

## 注意

- 環境依存度がそこそこ高いです。すでにスリムな人はほぼ変化なし
- `enableWorkflows: false`（旧キー `disableWorkflows: true` も引き続き有効）を入れるとローカル Workflow ツール（ultracode モードや一部 skill）が使えなくなる。1 行戻せば即復活。2.1.278 の API 実測（非対話・検証環境）で -2,057 tokens
- `disableBundledSkills: true` は単独では常駐が増える（2.1.278 実測 +4,679 tokens）。`enableWorkflows: false` と併用したときだけ -3,563 で、その場合 `/loop` `/schedule` 等の bundled skill も消える
- skill 単位で常駐 listing を削る `skillOverrides`（`name-only` / `user-invocable-only` / `off`）が Claude Code 2.1.278 で実在することを確認済み。作者環境では 60 日間未使用の 31 本に当てて Skills の推定 8.6k → 3.4k、コネクタ off・autoMemory off と合わせて API 実測 −6,250 tokens、対話の `/context` 推定では Artifact off も含めて Total 56.7k → 45k（2026-09-21）。詳細は [`skills/context-diet/references/settings-keys-reference.md`](skills/context-diet/references/settings-keys-reference.md)
- `~/.claude/CLAUDE.md` を編集する前に **必ずバックアップ**（`*.bak.YYYY-MM-DD` 推奨）
- **1M 窓が既定の環境では、削減効果は Total に対して 1% 未満です。** 公式 model-config（2026-09-21 fetch）: "On the Anthropic API, Fable 5.1, Fable 5, Sonnet 5, and Opus 4.7 and later run with the 1M window by default."。200k 窓が残るのは Pro プランで usage credits を使わない Opus と、Bedrock / Vertex / Foundry で `[1m]` を付けずにピン留めしたモデルです。2026-08-18 の実測では、CLAUDE.md 分割で Memory files を -22.7%（-9.9k tokens）削減できましたが、Total（967k〜1M tokens）に対しては 1% 未満の差でした。まず `/context` の Total 行で分母を確認し、削る価値があるか判断してください（詳細: [`skills/context-diet/references/playbook.md`](skills/context-diet/references/playbook.md) Step 1）

## ライセンス

MIT。Copyright (c) 2026 Hiroshi Ishizaka (ishizakahiroshi)
