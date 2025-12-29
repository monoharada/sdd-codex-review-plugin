# SDD Codex Review Plugin

Claude Code用 Spec-Driven Development (SDD) 外部レビュースキル

## 概要

このプラグインは、[OpenAI Codex CLI](https://github.com/openai/codex) を使用してClaude Codeの成果物を**独立レビュー**するスキルです。同一モデルによる自己レビューの盲点を排除し、品質を第三者保証します。

### なぜ外部レビューが重要か

- **別視点からの問題発見**: 異なるモデル（Codex/GPT）が別の角度から検証
- **確証バイアスの排除**: 生成者と異なるレビュアーによる客観的評価
- **追跡可能性**: Codex Session IDによるレビュー履歴の証跡

## 特徴

- 4フェーズ対応: requirements, design, tasks, impl
- 自動修正・再レビューループ（最大6回）
- Kiroコマンドとのシームレスな統合
- 構造化JSONレスポンスによる自動判定

## 前提条件

### 必須

1. **Claude Code** がインストール済み
2. **Codex CLI** がインストール済み
   ```bash
   npm install -g @openai/codex
   ```
3. **Kiro spec構造** (`.kiro/specs/[feature]/`)
   - requirements.md
   - design.md
   - tasks.md
   - spec.json

### 推奨

- Kiroコマンド群 (`/kiro:spec-*`) が利用可能

## インストール

### 方法1: git clone（推奨）

```bash
# リポジトリをクローン
git clone https://github.com/monoharada/sdd-codex-review-plugin.git

# スキルディレクトリにコピー
cp -r sdd-codex-review-plugin/skill ~/.claude/skills/sdd-codex-review
```

### 方法2: 手動コピー

1. このリポジトリの `skill/` ディレクトリをダウンロード
2. `~/.claude/skills/sdd-codex-review/` に配置

### CLAUDE.md への統合

プロジェクトの `CLAUDE.md` に以下のスニペットを追加してください：

```markdown
### Codex Review Workflow (必須)

**重要**: 各kiro:specフェーズ完了後、必ずCodexレビューを実行すること。

\`\`\`bash
# 各フェーズ完了後にCodexレビューを実行
/sdd-codex-review requirements [feature]  # kiro:spec-requirements後
/sdd-codex-review design [feature]        # kiro:spec-design後
/sdd-codex-review tasks [feature]         # kiro:spec-tasks後
/sdd-codex-review impl [feature]          # kiro:spec-impl後
\`\`\`

ワークフロー:
1. `/kiro:spec-requirements [feature]` → `/sdd-codex-review requirements [feature]`
2. `/kiro:spec-design [feature]` → `/sdd-codex-review design [feature]`
3. `/kiro:spec-tasks [feature]` → `/sdd-codex-review tasks [feature]`
4. `/kiro:spec-impl [feature] [task]` → `/sdd-codex-review impl [feature]`

各フェーズでAPPROVEDを取得してから次へ進む。
```

詳細なスニペットは `integration/` ディレクトリを参照してください。

## 使用方法

### 基本コマンド

```bash
# 要件レビュー
/sdd-codex-review requirements [feature-name]

# 設計レビュー
/sdd-codex-review design [feature-name]

# タスクレビュー
/sdd-codex-review tasks [feature-name]

# 実装レビュー
/sdd-codex-review impl [feature-name]

# 自動進行モード（現在のフェーズから順次レビュー）
/sdd-codex-review auto [feature-name]
```

### 4フェーズの詳細

#### Phase 1: 要件レビュー (requirements)

**対象**: `.kiro/specs/[feature]/requirements.md`

**レビュー基準**:
- 完全性: 全機能要件のカバー
- 明確性: 曖昧な表現の排除
- テスト可能性: 検証可能な受け入れ基準
- EARS形式準拠: When/If/The system shall

**判定**: `OK` or `NEEDS_REVISION`

#### Phase 2: 設計レビュー (design)

**対象**: `.kiro/specs/[feature]/design.md`

**レビュー基準**:
- 既存アーキテクチャとの整合性
- 設計の一貫性と標準
- 拡張性と保守性
- 型安全性とインターフェース設計

**判定**: `GO` or `NO_GO`

#### Phase 3: タスクレビュー (tasks)

**対象**: `.kiro/specs/[feature]/tasks.md`

**レビュー基準**:
- 要件カバレッジ
- タスク粒度（1-4時間）
- 依存関係の明確さ
- 並列実行可能性

**判定**: `APPROVED` or `NEEDS_REVISION`

#### Phase 4: 実装レビュー (impl)

**対象**: 変更されたソースファイル

**レビュー基準**:
- 要件準拠
- 設計準拠
- コード品質
- テストカバレッジ
- セキュリティ

**判定**: `APPROVED` or `NEEDS_REVISION`

### 判定基準

| フェーズ | PASS条件 |
|---------|----------|
| requirements | critical = 0 かつ medium ≦ 2 |
| design | critical issues ≦ 3 |
| tasks | critical = 0 かつ medium ≦ 2 かつ 未カバー要件 = 0 |
| impl | critical = 0 かつ medium ≦ 2 かつ 未達成要件 = 0 |

## Kiroワークフローとの統合

### 推奨ワークフロー

```
/kiro:spec-requirements [feature]
    ↓
/sdd-codex-review requirements [feature]  ← APPROVED
    ↓
/kiro:spec-design [feature]
    ↓
/sdd-codex-review design [feature]        ← GO
    ↓
/kiro:spec-tasks [feature]
    ↓
/sdd-codex-review tasks [feature]         ← APPROVED
    ↓
/kiro:spec-impl [feature] [task]
    ↓
/sdd-codex-review impl [feature]          ← APPROVED
    ↓
次のタスクまたは完了
```

### spec-impl からの自動呼び出し

`/kiro:spec-impl` コマンドで実装完了後、自動的にCodexレビューを呼び出すには、Kiroコマンド定義に以下を追加：

```markdown
## 実装完了後の処理

実装が完了したら、必ず Skill tool を使用して sdd-codex-review impl を実行：

\`\`\`
Skill tool invoke: "sdd-codex-review"
args: impl $1
\`\`\`
```

詳細は `integration/kiro-command-snippet.md` を参照。

## spec.json の更新

レビュー完了時、`spec.json` に以下の情報が追加されます：

```json
{
  "phase": "requirements-approved",
  "approvals": {
    "requirements": { "approved": true },
    "design": { "approved": false },
    "tasks": { "approved": false }
  },
  "codex_reviews": {
    "requirements": {
      "review_count": 2,
      "final_verdict": "OK",
      "resolved_issues": 3,
      "timestamp": "2025-12-29T12:00:00.000Z"
    }
  }
}
```

## トラブルシューティング

### Codex CLIが見つからない

```bash
# インストール確認
which codex

# 未インストールの場合
npm install -g @openai/codex

# 認証確認
codex auth status
```

### レビューが6回超過

最大6回のレビューサイクル後も承認されない場合、手動介入が必要です：

1. Codexの指摘事項を確認
2. 手動で修正を適用
3. 再度 `/sdd-codex-review [phase] [feature]` を実行

### JSON解析エラー

Codexの出力が期待形式でない場合：

1. raw出力が表示されます
2. 手動で判定を確認
3. 必要に応じて再レビュー

### Session IDがない

レビュー報告にSession IDがない場合、シミュレートの可能性があります：

```
「本当にcodexにレビューしてもらった？」
```

と質問して確認してください。

## ディレクトリ構造

```
sdd-codex-review-plugin/
├── README.md
├── LICENSE
├── skill/
│   ├── SKILL.md                  # メインスキル定義
│   ├── prompts/
│   │   ├── requirements-review.md
│   │   ├── design-review.md
│   │   ├── tasks-review.md
│   │   └── impl-review.md
│   └── templates/
│       └── review-report.md
├── integration/
│   ├── claude-md-snippet.md      # CLAUDE.md用スニペット
│   └── kiro-command-snippet.md   # Kiroコマンド連携用
└── examples/
    └── spec-json-example.json    # spec.jsonサンプル
```

## ライセンス

MIT License

## 関連リンク

- [OpenAI Codex CLI](https://github.com/openai/codex)
- [Claude Code](https://claude.ai/code)
- [Kiro](https://kiro.dev/) (Spec-Driven Development)


## コンテキスト節約モード（自動実行）

このスキルは、Codexレビュー時に**自動的に**コンテキスト節約を行います。
オプション指定は不要です。

### 自動で行われること

1. **状態保存**: `.context/sdd-review-state.json` に復元用情報を保存
2. **バックグラウンド実行**: Codexをバックグラウンドで起動
3. **圧縮提案**: 「コンテキストを圧縮しますか？」と提案
4. **自動復元**: 結果取得後、状態から復元して続行
5. **クリーンアップ**: 完了後に一時ファイルを削除

### ユーザー体験

```
/sdd-codex-review requirements my-feature
    │
    ▼
「Codexレビューをバックグラウンドで実行中...」
「コンテキストを圧縮しますか？」
    │
    ├─ 「圧縮して」→ コンテキスト圧縮、メモリ解放
    ├─ 別作業を続ける → 他のファイル編集など
    └─ 待機 → そのまま待つ
    │
    ▼
（レビュー完了後、自動的に結果を取得・処理）
「レビュー完了: OK」
```

### メリット

- **自動化**: ユーザーが何も指定しなくても最適化
- **メモリ効率**: 長時間セッションでも快適に動作
- **中断耐性**: セッションが切れても状態から復元可能

詳細は `skill/SKILL.md` の「コンテキスト節約モード」セクションを参照。
