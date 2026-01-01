# セクション検出アルゴリズム

本ドキュメントでは、tasks.mdからセクションを検出し、セクション単位でレビューを実行するための仕様を定義します。

---

## tasks.md セクションフォーマット

### 推奨構造

```markdown
## Section 1: Core Foundation

### Task 1.1: Define base types
**Creates:** `src/types/base.ts`
**Implements:** REQ-1.1

[タスク詳細...]

### Task 1.2: Implement utilities
**Creates:** `src/utils/helpers.ts`, `src/utils/helpers.test.ts`
**Modifies:** `src/config/index.ts`
**Implements:** REQ-1.2

[タスク詳細...]

## Section 2: Feature Implementation [E2E]

### Task 2.1: Build main component
**Creates:** `src/components/Main.tsx`, `src/components/Main.test.tsx`
**Implements:** REQ-2.1, REQ-2.2
**E2E:** コンポーネントの初期表示確認、ユーザー操作テスト

[タスク詳細...]

### Task 2.2: Add user interaction
**Creates:** `src/components/UserForm.tsx`
**Implements:** REQ-2.3
**E2E:** フォーム入力、バリデーション表示、送信確認

[タスク詳細...]
```

### フォーマット要素

| 要素 | パターン | 説明 |
|------|---------|------|
| セクション見出し | `## Section X: Name` | セクション境界を定義 |
| E2Eセクション | `## Section X: Name [E2E]` | E2Eエビデンス収集が必要なセクション |
| タスク見出し | `### Task X.Y: Title` | セクション内のタスク |
| 作成ファイル | `**Creates:** \`path\`` | タスクが作成するファイル |
| 変更ファイル | `**Modifies:** \`path\`` | タスクが変更するファイル |
| 要件トレース | `**Implements:** REQ-X.X` | 対応する要件ID |
| E2Eシナリオ | `**E2E:** シナリオ説明` | そのタスクのE2E検証シナリオ |

---

## セクション解析アルゴリズム

### Step 1: セクション境界の検出

```
INPUT: tasks.md content
OUTPUT: sections[]

1. 行ごとにスキャン
2. `^## ` で始まる行をセクション境界として検出
3. 各セクションの範囲（開始行〜次のセクション or EOF）を記録
```

### Step 2: タスクの抽出

```
FOR each section:
    1. セクション範囲内で `^### Task ` パターンを検索
    2. タスクID（X.Y形式）を抽出
    3. タスクをセクションにマッピング
```

### Step 3: 期待ファイルの抽出

```
FOR each task:
    1. `**Creates:** ` の後のファイルパスを抽出
    2. `**Modifies:** ` の後のファイルパスを抽出
    3. バッククォート内のパス、カンマ区切り複数パス対応

    正規表現例:
    Creates: /\*\*Creates:\*\*\s*(.+)$/
    ファイル抽出: /`([^`]+)`/g
```

### Step 4: E2Eタグの検出

```
FOR each section:
    1. セクション見出しに `[E2E]` サフィックスがあるか確認
    2. 正規表現: /\[E2E\]\s*$/
    3. マッチした場合: section.e2e_required = true
    4. セクション内の各タスクから `**E2E:**` を抽出

E2E検出の正規表現:
    セクションタグ: /^##\s+.+\s*\[E2E\]\s*$/
    タスクシナリオ: /\*\*E2E:\*\*\s*(.+)$/

E2Eシナリオの集約:
    section.e2e_scenarios = [
        { task: "2.1", scenario: "コンポーネントの初期表示確認、ユーザー操作テスト" },
        { task: "2.2", scenario: "フォーム入力、バリデーション表示、送信確認" }
    ]
```

---

## セクションIDの生成

セクション見出しからセクションIDを自動生成：

```
入力: "## Section 1: Core Foundation"
出力: "section-1-core-foundation"

変換ルール:
1. "## " プレフィックスを除去
2. 小文字に変換
3. 英数字以外をハイフンに置換
4. 連続ハイフンを単一に圧縮
5. 先頭・末尾のハイフンを除去
```

---

## 完了判定アルゴリズム

### isSectionComplete(sectionId)

```
FUNCTION isSectionComplete(sectionId):
    section = getSectionById(sectionId)

    // 1. Creates ファイル: 存在確認
    FOR each file in section.creates_files:
        IF NOT fileExists(file):
            RETURN false

    // 2. Modifies ファイル: ベースブランチからの差分で実変更を確認
    FOR each file in section.modifies_files:
        IF NOT fileExists(file):
            RETURN false
        IF NOT hasChangesFromBaseBranch(file):
            // ベースブランチからの変更がない = まだ実装されていない
            RETURN false

    RETURN true

FUNCTION hasChangesFromBaseBranch(file, baseBranch = "main"):
    // ベースブランチからの差分でファイルに変更があるか確認
    // コミット済みの変更も検出するため、HEAD比較ではなくベースブランチ比較を使用
    result = exec("git diff " + baseBranch + "..HEAD -- " + file)
    RETURN result.length > 0
```

### 重要: Creates vs Modifies の区別

| タイプ | 完了条件 | 理由 |
|--------|----------|------|
| `**Creates:**` | ファイル存在 | 新規ファイルなので存在=実装完了 |
| `**Modifies:**` | ファイル存在 + ベースブランチからの差分あり | 既存ファイルなので変更検知が必要 |

### ベースブランチ差分の理由

`git diff main..HEAD` を使用する理由：

1. **コミット済み変更の検出**: `git diff HEAD` はuncommittedな変更のみ検出するが、`main..HEAD` はブランチ作成後の全コミットを含む
2. **レビュー単位との整合性**: 実装レビューはブランチ単位で行うため、ベースブランチからの全変更を対象とするのが自然
3. **シンプルさ**: 直近レビューコミットを追跡するより、ベースブランチを基準とする方が実装が単純

### 代替案: タスク完了フラグ併用

git diff に依存しない場合は、spec.json にタスク完了フラグを追加：

```json
{
  "section_tracking": {
    "sections": {
      "section-1": {
        "tasks_completed": {
          "1.1": true,
          "1.2": false
        }
      }
    }
  }
}
```

この場合、実装コマンド（`/kiro:spec-impl`）がタスク完了時にフラグを更新する。

### shouldTriggerReview(sectionId)

```
FUNCTION shouldTriggerReview(sectionId):
    // セクションが完了していること
    IF NOT isSectionComplete(sectionId):
        RETURN false

    // まだレビューされていないこと
    IF sectionAlreadyReviewed(sectionId):
        RETURN false

    RETURN true
```

---

## 状態管理

### 正本（Source of Truth）: spec.json

**重要**: セクション状態の正本は `spec.json` の `section_tracking` フィールドです。

| ファイル | 役割 | 更新タイミング |
|----------|------|---------------|
| `spec.json.section_tracking` | **正本** - レビュー状態、完了状態 | レビュー完了時、E2E完了時 |
| `.context/section-completion-state.json` | **キャッシュ** - tasks.md解析結果 | tasks.md変更検知時に再生成 |

### キャッシュファイル

`.context/section-completion-state.json` は tasks.md の解析結果をキャッシュするためのもので、
`spec.json` と矛盾する場合は **spec.json が優先** されます。

キャッシュ内容（tasks.md 解析結果のみ）：

```json
{
  "feature": "my-feature",
  "parsed_at": "2025-12-30T10:00:00Z",
  "tasks_md_hash": "abc123...",
  "sections": {
    "section-1-core-foundation": {
      "name": "Section 1: Core Foundation",
      "line_range": [1, 25],
      "tasks": ["1.1", "1.2"],
      "creates_files": [
        "src/types/base.ts",
        "src/utils/helpers.ts"
      ],
      "modifies_files": [
        "src/config/index.ts"
      ],
      "e2e_required": false,
      "e2e_scenarios": [],
      "status": "complete",
      "completed_at": "2025-12-30T12:00:00Z",
      "reviewed": true,
      "review_session_id": "codex-xyz"
    },
    "section-2-feature-impl": {
      "name": "Section 2: Feature Implementation [E2E]",
      "line_range": [26, 50],
      "tasks": ["2.1", "2.2"],
      "creates_files": [
        "src/components/Main.tsx",
        "src/components/Main.test.tsx",
        "src/components/UserForm.tsx"
      ],
      "modifies_files": [],
      "e2e_required": true,
      "e2e_scenarios": [
        { "task": "2.1", "scenario": "コンポーネントの初期表示確認、ユーザー操作テスト" },
        { "task": "2.2", "scenario": "フォーム入力、バリデーション表示、送信確認" }
      ],
      "e2e_evidence": {
        "status": "pending",
        "video_path": null,
        "screenshots": [],
        "executed_at": null,
        "error_message": null
      },
      "status": "in_progress",
      "completed_at": null,
      "reviewed": false,
      "review_session_id": null
    }
  }
}
```

---

## 変更検知

### tasks.mdの変更検知

```
現在のハッシュ = hash(tasks.md内容)
保存済みハッシュ = state.tasks_md_hash

IF 現在のハッシュ != 保存済みハッシュ:
    // tasks.mdが変更された
    1. セクション構造を再解析
    2. 既存のレビュー状態を可能な限り維持
    3. 新しいセクションは未レビュー状態で追加
    4. 削除されたセクションは状態から除去
```

---

## エッジケース

### セクションがない場合

tasks.mdに `##` 見出しがない場合：

```
1. 全体を単一セクション "default" として扱う
2. 全タスク完了時にレビュー実行
```

### ファイルパス指定がない場合

タスクに `**Creates:**` / `**Modifies:**` がない場合：

```
1. 警告ログを出力
2. そのタスクは常に「完了」として扱う
3. セクション内の他タスクの完了で判定
```

### 複数ファイルの記述

カンマ区切りで複数ファイルを指定可能：

```markdown
**Creates:** `src/a.ts`, `src/b.ts`, `src/c.ts`
```

### E2Eタグがあるがシナリオがない場合

セクションに `[E2E]` タグがあるが、タスクに `**E2E:**` がない場合：

```
1. 警告ログを出力
2. e2e_required = true のまま維持
3. e2e_scenarios = [] (空配列)
4. E2Eエビデンス収集時に汎用シナリオを使用
```

### E2Eエビデンス収集失敗時

Playwright MCPでの実行が失敗した場合：

```
1. e2e_evidence.status = "failed"
2. e2e_evidence.error_message にエラー内容を記録
3. 警告をユーザーに通知
4. Codexレビューは続行（E2E失敗でブロックしない）
```

---

## E2Eエビデンスワークフロー

### トリガー条件

```
FUNCTION shouldTriggerE2EEvidence(sectionId):
    section = getSectionById(sectionId)

    // Codexレビューで承認されていること（最重要）
    IF NOT sectionReviewApproved(sectionId):
        RETURN false

    // E2Eが必要なセクションであること
    IF NOT section.e2e_required:
        RETURN false

    // まだE2Eエビデンスが収集されていないこと
    IF section.e2e_evidence.status != "pending":
        RETURN false

    RETURN true
```

### 実行フロー（レビュー後にE2E）

```
セクション完了
    ↓
Codexレビュー実行
    ↓
APPROVED ?
    ├── NO → 修正 → 再レビュー
    ↓ YES
shouldTriggerE2EEvidence(sectionId) = true?
    ├── NO → セクション完了、次へ進行
    ↓ YES
E2Eエビデンス収集（Playwright MCP）
    ↓
結果を e2e_evidence に保存
    ↓
エビデンスディレクトリを自動オープン
    ↓
セクション完了、次へ進行
```

### なぜレビュー後にE2Eを実行するのか

1. **品質優先**: レビュー済みの承認されたコードでエビデンスを取得
2. **効率性**: レビューで修正が入る可能性があるため、レビュー前のE2Eは無駄
3. **エビデンス目的**: 最終的な品質保証済みコードの動作を記録

### エビデンス保存先

```
.context/e2e-evidence/
└── [feature-name]/
    └── [section-id]/
        ├── step-01-initial.png     # 初期状態
        ├── step-02-action.png      # 操作後
        └── step-03-complete.png    # 完了状態
```

**注意**: Playwright MCPは直接録画機能を提供しないため、スクリーンショットのみがエビデンスとして収集されます。

---

## 実装例

### Codex実行時のセクションコンテキスト

```bash
codex exec --sandbox read-only "
## セクション実装レビュー

### レビュー対象
セクション: {{SECTION_NAME}}
タスク: {{SECTION_TASK_IDS}}

### 変更ファイル
$(for file in {{SECTION_FILES}}; do echo "- $file"; done)

### セクション境界
- このセクションで完了すべき: {{IN_SCOPE_FEATURES}}
- 後続セクションで対応: {{OUT_OF_SCOPE_FEATURES}}

[プロンプト本文...]
"
```
