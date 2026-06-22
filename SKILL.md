---
name: context-diet
description: Claude Code の常駐 context を棚卸し・削減する手順。「コンテキスト棚卸して」「context 確認」「常駐減らしたい」「/context 見て無駄ないか」「memory ファイル分割したい」「disabledTools」「settings.json の context」等で起動。
---

# context-diet スキル

Claude Code セッションの常駐 context を実測ベースで減らす。`/context` で内訳を見て、Memory files / System tools を必要に応じてスリム化する。

## 起動条件

ユーザーが下記いずれかに該当する発話をしたら起動:

- 「コンテキスト棚卸して」「context 確認」「context 減らしたい」
- 「`/context` の結果これでいい？」「無駄に常駐してる？」
- 「`disabledTools` 効く？」「settings.json で context 削減できる？」
- 「memory ファイル分割したい」「CLAUDE.md 大きすぎる」

## 手順

### Step 1: 現状把握

ユーザーに `/context` を実行してもらい、結果（Total / System prompt / System tools / Memory files / Skills の内訳）を貼ってもらう。

### Step 2: Memory files の棚卸し

`~/.claude/CLAUDE.md` と参照されている @import 群を Read し、章ごとに 3 分類:

| 分類 | 対応 |
|---|---|
| メタトリガー（残す） | 毎ターン判定材料として必要。`P` 略記、ツール選択ルール、出力フォーマット規約など |
| サブファイル化（移す） | トリガー時にだけ意味があるルール本文。コーディング規範、ドメイン参照ルール、リリース手順など |
| 削除候補 | 既に冗長・他で代替されている記述 |

「サブファイル化」対象は `~/.claude/guides/<name>.md` に切り出し、CLAUDE.md には「<トリガー条件> のとき必ず Read」の 3〜5 行に圧縮する。

### Step 3: settings.json の `system tools` 系チューニング

⚠️ **`disabledTools` / `enabledTools` / `skillOverrides` は公式に存在しない**。`permissions.deny` は呼び出しブロックのみで context 削減効果なし。

実在する公式キー（[settings-keys-reference.md](docs/settings-keys-reference.md) 参照）:

- `disableArtifact: true` — 安全・効果小
- `disableWorkflows: true` — 効果大（5〜10k tokens）。Workflow ツール定義削除。多 agent / ultracode 等が使えなくなる。1 行削除で即復活
- `disableBundledSkills: true` — 過激。`/loop` `/schedule` 等の bundled skill が全部封印される
- `maxSkillDescriptionChars: 512` — skill description を圧縮（精度が落ちる副作用あり）
- `includeGitInstructions: false` — git status snapshot を除去

**ユーザーの利用パターンを確認してから提案**:
- Workflow / ultracode 使う → `disableWorkflows` は入れない
- bundled skill 多用 → `disableBundledSkills` は入れない
- git 操作が中心 → `includeGitInstructions: false` は入れない

最小リスク 2 行（`disableArtifact: true` + `disableWorkflows: true`）から始めて、新セッションで `/context` を再測。

### Step 4: バックアップ必須

CLAUDE.md / settings.json を編集する **前に必ずバックアップ**（`*.bak.YYYY-MM-DD`）。事故時に即戻せる状態を作ってから着手する。

### Step 5: 効果実測と報告

新セッションで `/context` を再実行 → Before/After 表を作って報告。期待値:

| 介入レベル | 期待削減量 |
|---|---|
| Memory 分割のみ | -2〜-4k tokens |
| Memory 分割 + `disableArtifact` + `disableWorkflows` | -7〜-15k tokens |

環境依存が大きい。すでにスリムな環境ではほぼ効果なし。

## やらないこと

- 公式 docs に未記載のキー（`disabledTools` 等）を勝手に追加しない
- バックアップなしで CLAUDE.md / settings.json を編集しない
- ユーザーの利用パターンを聞かずに「強い」設定（`disableBundledSkills` 等）を勝手に当てない
- 削減幅を大げさに言わない（環境依存）

## 関連

- [docs/playbook.md](docs/playbook.md): 元記事の詳細手順
- [docs/settings-keys-reference.md](docs/settings-keys-reference.md): settings.json 公式キーの完全リファレンス
- [examples/claude-md-split-pattern.md](examples/claude-md-split-pattern.md): CLAUDE.md 分割の実例
