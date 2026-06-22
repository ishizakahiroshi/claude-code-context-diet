# claude-code-context-diet

Claude Code の常駐 context を実測ベースで減らすための手順集と簡易スキル。

`/context` を開いたら「何もしていないのに 80k tokens 食ってた」という人向け。元になった検証では総量を **80.7k → 69.5k tokens（-11.2k / -13.9%）** まで減らせました。

## 何が入っているか

| ファイル | 用途 |
|---|---|
| [`SKILL.md`](SKILL.md) | `~/.claude/skills/context-diet/SKILL.md` として配置できる簡易スキル。AI に「コンテキスト棚卸して」と頼むと走る |
| [`docs/playbook.md`](docs/playbook.md) | 詳細手順。元記事の内容を再現性のある形に整えた |
| [`docs/settings-keys-reference.md`](docs/settings-keys-reference.md) | Claude Code `settings.json` で実在する context 削減キーの一覧。`disabledTools` は存在しないという事実つき |
| [`examples/claude-md-split-pattern.md`](examples/claude-md-split-pattern.md) | CLAUDE.md を「トリガー文 + サブファイル」に分割する具体パターン |
| `LICENSE` | MIT |

## 使い方（最短ルート）

1. このリポジトリを clone（または raw URL から `SKILL.md` だけ取ってきて）
2. `~/.claude/skills/context-diet/SKILL.md` として配置
3. Claude Code を再起動
4. AI に「コンテキスト棚卸して」と言うとスキルが発火し、`/context` 取得 → 分類 → 削減案提示の手順で進む

スキル形式に乗らない場合は [`docs/playbook.md`](docs/playbook.md) を手で読みながらやっても同じ結果になります。

## 関連記事

- 詳しい背景と Before/After 数値: [Zenn 記事「Claude Code の常駐 context を 14% 削った話。disabledTools は存在しなかった」](https://zenn.dev/ishizakahiroshi/articles/20260622-claude-code-context-diet)
- 図解版・詳細版: [ishizakahiroshi.github.io/articles/2026-06-22-claude-code-context-diet/](https://ishizakahiroshi.github.io/articles/2026-06-22-claude-code-context-diet/)

## 注意

- 環境依存度がそこそこ高いです。すでにスリムな人はほぼ変化なし
- `disableWorkflows: true` を入れるとローカル Workflow ツール（ultracode モードや一部 skill）が使えなくなる。1 行戻せば即復活
- `~/.claude/CLAUDE.md` を編集する前に **必ずバックアップ**（`*.bak.YYYY-MM-DD` 推奨）

## ライセンス

MIT。Copyright (c) 2026 Hiroshi Ishizaka (ishizakahiroshi)
