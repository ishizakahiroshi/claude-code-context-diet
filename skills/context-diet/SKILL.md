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

ユーザーに `/context` を実行してもらい、結果（Total / System prompt / System tools / Memory files / Skills の内訳）を貼ってもらう。加えて `/context all`（展開表示）と `/skill-doctor`（skill ごとの listing cost・直近 7 日間の使用回数・一度も呼ばれていない skill の一覧）も貼ってもらう。**Total 行のモデル名・トークン上限も必ず見る。** `[1m]` サフィックス付きモデルなど 1M トークン級の環境では、Memory files が数万トークンあっても Total に対しては 1% 前後のことがあり、削減の優先度判断が変わる（2026-08-18 実測: Memory files -22.7% でも Total 比では 1% 未満）。

`/context` で "deferred" と表示される MCP ツールは、Tool Search により定義（スキーマ）が既に遅延ロードされていて常駐していないので、削減対象にしない。

### Step 2: Memory files の棚卸し

`~/.claude/CLAUDE.md` と参照されている @import 群を Read し、章ごとに 3 分類:

| 分類 | 対応 |
|---|---|
| メタトリガー（残す） | 毎ターン判定材料として必要。`P` 略記、ツール選択ルール、出力フォーマット規約など |
| サブファイル化（移す） | トリガー時にだけ意味があるルール本文。コーディング規範、ドメイン参照ルール、リリース手順など |
| 削除候補 | 既に冗長・他で代替されている記述 |

「サブファイル化」対象は `~/.claude/guides/<name>.md` に切り出し、CLAUDE.md には「<トリガー条件> のとき必ず Read」の 3〜5 行に圧縮する。

Memory files には CLAUDE.md 系だけでなく auto-memory の `MEMORY.md`（`~/.claude/projects/<project>/memory/MEMORY.md`。先頭 200 行または 25KB が毎セッション常駐。公式 docs memory.md による）も含まれる。棚卸しは 2 段階:

1. `MEMORY.md` は索引だけ残し、各エントリの本文は同じディレクトリの topic ファイルへ移す
2. auto-memory 自体が不要なら `autoMemoryEnabled: false`（settings.json）で読み書きを止める

### Step 3: settings.json の `system tools` 系チューニング

⚠️ **`disabledTools` / `enabledTools` は公式に存在しない（Claude Code 2.1.263 で確認）**。`permissions.deny` は呼び出しブロックのみで context 削減効果なし。**`skillOverrides` は実在する**（下記）。

実在する公式キー（[settings-keys-reference.md](references/settings-keys-reference.md) 参照）:

- `enableArtifact: false` — 安全・効果小（旧キー `disableArtifact: true` も引き続き動作）
- `enableWorkflows: false` — 効果大（5〜10k tokens、2026-06-22 時点の推定・2.1.263 未再測）。Workflow ツール定義削除。多 agent / ultracode 等が使えなくなる。1 行削除で即復活（旧キー `disableWorkflows: true` も引き続き動作）
- `disableBundledSkills: true` — 過激。`/loop` `/schedule` 等の bundled skill が全部封印される
- `skillListingMaxDescChars`（既定 1536） — skill listing で Claude に送る各 skill の description の文字数上限。全体の予算は `skillListingBudgetFraction`（既定 0.01 = context window の 1%）で決まる。Step 1 で見た Total 行の上限が大きい環境（1M トークン級等）では、既定のままでも listing 予算が自動的に広いので、下げても効果が薄いことがある
- `skillOverrides` — skill ごとに listing 表示を `name-only`（description を隠して名前だけ残す）/ `user-invocable-only`（AI からは隠すが `/name` は使える）/ `off`（完全に隠す）へ落とせる。滅多に使わない skill を完全には消したくないとき、`disableBundledSkills` の一括封印より先にこちらを検討する
- `includeGitInstructions: false` — git status snapshot を除去

**ユーザーの利用パターンを確認してから提案**:
- Workflow / ultracode 使う → `enableWorkflows: false` は入れない
- bundled skill 多用 → `disableBundledSkills` は入れない
- git 操作が中心 → `includeGitInstructions: false` は入れない
- 既に `enableWorkflows: true` 等で能動利用が明示されている設定があれば、そもそも変更候補から外す

**提案前に Step 1 で見た Total 行の上限も踏まえる。** 1M トークン級の環境では `enableWorkflows: false` の削減量（5〜10k tokens、2026-06-22 時点の推定・2.1.263 未再測）ですら Total の 1% 未満のことがあり、失う機能に対して割に合わないことがある。その場合は「今は変更しない」を選択肢として案内し、強い設定を既定の推奨にしない。

最小リスク 2 行（`enableArtifact: false` + `enableWorkflows: false`。旧キー `disableArtifact: true` / `disableWorkflows: true` もまだ動く）から始めて、新セッションで `/context` を再測。滅多に使わない skill が個別にあれば `skillOverrides` で `name-only` / `off` に落とす手も併せて案内する。

### Step 4: バックアップ必須

CLAUDE.md / settings.json を編集する **前に必ずバックアップ**（`*.bak.YYYY-MM-DD`）。事故時に即戻せる状態を作ってから着手する。

### Step 5: 効果実測と報告

新セッションで `/context` を再実行 → Before/After 表を作って報告。期待値（2026-06-22 時点の推定・2.1.263 未再測）:

| 介入レベル | 期待削減量 |
|---|---|
| Memory 分割のみ | -2〜-4k tokens |
| Memory 分割 + `enableArtifact: false` + `enableWorkflows: false` | -7〜-15k tokens |

環境依存が大きい。すでにスリムな環境ではほぼ効果なし。

## やらないこと

- 公式に存在しないキー（`disabledTools` / `enabledTools` 等）を勝手に追加しない
- バックアップなしで CLAUDE.md / settings.json を編集しない
- ユーザーの利用パターンを聞かずに「強い」設定（`disableBundledSkills` 等）を勝手に当てない
- 削減幅を大げさに言わない（環境依存）
- このスキル自身の frontmatter に `when_to_use` を足して起動語を分離しない。listing では description と合算で 1536 文字に切り詰められるため効果が無い（`name` / `description` のみで運用する）

## 関連

- [references/playbook.md](references/playbook.md): 元記事の詳細手順
- [references/settings-keys-reference.md](references/settings-keys-reference.md): settings.json 公式キーの完全リファレンス
- [references/claude-md-split-pattern.md](references/claude-md-split-pattern.md): CLAUDE.md 分割の実例
