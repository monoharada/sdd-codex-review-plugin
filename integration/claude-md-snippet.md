# CLAUDE.md 統合スニペット

このスニペットをプロジェクトの `CLAUDE.md` に追加してください。

---

## 追加するセクション

```markdown
### Codex Review Workflow (必須)

**重要**: 各kiro:specフェーズ完了後、Codexレビューを実行すること。

```bash
# 要件・設計・タスクフェーズ：フェーズ完了後すぐにレビュー
/sdd-codex-review requirements [feature]  # kiro:spec-requirements後
/sdd-codex-review design [feature]        # kiro:spec-design後
/sdd-codex-review tasks [feature]         # kiro:spec-tasks後

# 実装フェーズ：セクション完了時のみレビュー（タスクごとではない）
/sdd-codex-review impl-section [feature] [section-id]  # セクション完了時
/sdd-codex-review impl-pending [feature]               # 完了済み未レビューセクションを一括

# E2Eエビデンス収集（[E2E]タグ付きセクションで自動実行）
/sdd-codex-review e2e-evidence [feature] [section-id]  # 手動実行用
```

#### ワークフロー

1. `/kiro:spec-requirements [feature]` → `/sdd-codex-review requirements [feature]`
2. `/kiro:spec-design [feature]` → `/sdd-codex-review design [feature]`
3. `/kiro:spec-tasks [feature]` → `/sdd-codex-review tasks [feature]`
4. `/kiro:spec-impl [feature] [task]` → **セクション完了時のみ** → `/sdd-codex-review impl-section [feature] [section-id]`

各フェーズでAPPROVEDを取得してから次へ進む。

#### セクション単位レビューの仕組み

実装フェーズでは、**タスクごと**ではなく**セクション単位**でレビューを実行：

1. tasks.mdの`##`見出しでセクションを定義
2. 各タスクの`**Creates:**`で期待ファイルを指定
3. セクション内の全ファイルが存在 → セクション完了
4. 完了 AND 未レビュー → Codexレビュー実行

これにより、レビューオーバーヘッドを削減しつつ、セクション間の整合性を確保。

#### E2Eエビデンス収集

`[E2E]` タグ付きセクションでは、Playwright MCPを使用してE2Eエビデンスを自動収集：

```markdown
## Section 2: User Dashboard [E2E]    ← [E2E]タグでE2E必要を示す

### Task 2.1: Build dashboard
**Creates:** `src/Dashboard.tsx`
**E2E:** ダッシュボード初期表示確認    ← 各タスクのE2Eシナリオ
```

フロー:
1. セクション完了 + `[E2E]`タグ検出
2. Playwright MCPでスクリーンショット収集
3. `.context/e2e-evidence/`に保存
4. Codexレビュー実行

エビデンスは `.context/` に保存（gitignore推奨）。E2E失敗でもレビューは続行。
```

---

## 配置例

`CLAUDE.md` の構造例：

```markdown
# CLAUDE.md

## Project Overview
...

## Development Commands
...

## Spec-Driven Development

### Codex Review Workflow (必須)

**重要**: 各kiro:specフェーズ完了後、Codexレビューを実行すること。

```bash
# 要件・設計・タスクフェーズ
/sdd-codex-review requirements [feature]
/sdd-codex-review design [feature]
/sdd-codex-review tasks [feature]

# 実装フェーズ：セクション完了時のみ
/sdd-codex-review impl-section [feature] [section-id]
```

ワークフロー:
1. `/kiro:spec-requirements` → `/sdd-codex-review requirements`
2. `/kiro:spec-design` → `/sdd-codex-review design`
3. `/kiro:spec-tasks` → `/sdd-codex-review tasks`
4. `/kiro:spec-impl` → セクション完了時のみ → `/sdd-codex-review impl-section`

各フェーズでAPPROVEDを取得してから次へ進む。

## Architecture
...
```

---

## 注意事項

- このスニペットを追加することで、Claude Codeは各フェーズ完了後にCodexレビューを実行するよう認識します
- **実装フェーズはセクション単位**: タスクごとではなく、セクション完了時にレビューを実行
- スキルが正しくインストールされていない場合、`/sdd-codex-review` コマンドは認識されません
- スキルのインストールは `~/.claude/skills/sdd-codex-review/` に配置することで完了します
