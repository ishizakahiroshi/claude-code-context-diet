# claude-code-context-diet

Claude Code の常駐 context を実測ベースで減らすための手順集と簡易スキル。

`/context` を開いたら「何もしていないのに 80k tokens 食ってた」という人向け。元になった検証では総量を **80.7k → 69.5k tokens（-11.2k / -13.9%）** まで減らせました。

## 何が入っているか

| ファイル | 用途 |
|---|---|
| [`skills/context-diet/SKILL.md`](skills/context-diet/SKILL.md) | スキル本体。AI に「コンテキスト棚卸して」と頼むと走る |
| [`skills/context-diet/references/playbook.md`](skills/context-diet/references/playbook.md) | 詳細手順。元記事の内容を再現性のある形に整えた |
| [`skills/context-diet/references/settings-keys-reference.md`](skills/context-diet/references/settings-keys-reference.md) | Claude Code `settings.json` で実在する context 削減キーの一覧。`disabledTools` は存在しない（Claude Code 2.1.263 で確認）という事実つき |
| [`skills/context-diet/references/claude-md-split-pattern.md`](skills/context-diet/references/claude-md-split-pattern.md) | CLAUDE.md を「トリガー文 + サブファイル」に分割する具体パターン |
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

どちらも 2026-06-22 時点（`disableArtifact` / `disableWorkflows` の時代）の記述です。現行のキー名は [`skills/context-diet/references/settings-keys-reference.md`](skills/context-diet/references/settings-keys-reference.md) を参照してください。

- 詳しい背景と Before/After 数値: [Zenn 記事「Claude Code の常駐 context を 14% 削った話。disabledTools は存在しなかった」](https://zenn.dev/ishizakahiroshi/articles/20260622-claude-code-context-diet)
- 図解版・詳細版: [ishizakahiroshi.github.io/articles/2026-06-22-claude-code-context-diet/](https://ishizakahiroshi.github.io/articles/2026-06-22-claude-code-context-diet/)

## 注意

- 環境依存度がそこそこ高いです。すでにスリムな人はほぼ変化なし
- `enableWorkflows: false`（旧キー `disableWorkflows: true` も引き続き有効）を入れるとローカル Workflow ツール（ultracode モードや一部 skill）が使えなくなる。1 行戻せば即復活
- skill 単位で常駐 listing を削る `skillOverrides`（`name-only` / `user-invocable-only` / `off`）が Claude Code 2.1.263 で実在することを確認済み。詳細は [`skills/context-diet/references/settings-keys-reference.md`](skills/context-diet/references/settings-keys-reference.md)
- `~/.claude/CLAUDE.md` を編集する前に **必ずバックアップ**（`*.bak.YYYY-MM-DD` 推奨）
- **1M token 対応モデル（`[1m]` サフィックス付き）を使っている場合、削減効果は相対的に小さくなります。** 2026-08-18 の別環境での実測では、CLAUDE.md 分割で Memory files を -22.7%（-9.9k tokens）削減できましたが、Total（967k〜1M tokens）に対しては 1% 未満の差でした。まず `/context` の Total 行で context window の規模を確認し、削る価値があるか判断してください（詳細: [`skills/context-diet/references/playbook.md`](skills/context-diet/references/playbook.md) Step 1）

## ライセンス

MIT。Copyright (c) 2026 Hiroshi Ishizaka (ishizakahiroshi)
