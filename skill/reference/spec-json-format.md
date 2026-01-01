# spec.json フォーマット仕様

レビュー結果とフェーズ進行状況を記録するspec.jsonの仕様です。

---

## 基本構造

```json
{
  "phase": "requirements-approved",
  "approvals": {
    "requirements": { "approved": true },
    "design": { "approved": false },
    "tasks": { "approved": false }
  },
  "updated_at": "2025-12-25T12:00:00.000Z",
  "interview": {
    "requirements": false
  },
  "codex_reviews": {
    "requirements": {
      "review_count": 2,
      "final_verdict": "OK",
      "resolved_issues": 3,
      "session_id": "codex-session-xyz",
      "timestamp": "2025-12-25T12:00:00.000Z"
    }
  }
}
```

---

## フィールド説明

### phase

現在のフェーズを示す文字列。

| 値 | 意味 |
|----|------|
| `initialized` | 初期状態 |
| `requirements-approved` | 要件レビュー承認済み |
| `design-approved` | 設計レビュー承認済み |
| `ready-for-implementation` | タスクレビュー承認済み、実装可能 |
| `impl-approved` | 実装レビュー承認済み |
| `completed` | 全フェーズ完了 |

### approvals

各フェーズの承認状況。

```json
{
  "requirements": { "approved": boolean },
  "design": { "approved": boolean },
  "tasks": { "approved": boolean }
}
```

### codex_reviews

Codexレビューの履歴。

```json
{
  "[phase]": {
    "review_count": number,        // レビュー回数
    "final_verdict": string,       // 最終判定（OK, GO, APPROVED等）
    "resolved_issues": number,     // 解決した指摘件数
    "session_id": string,          // Codex Session ID（必須）
    "timestamp": string            // ISO8601形式タイムスタンプ
  }
}
```

### interview（オプション）

プリフライトインタビューの設定。各フェーズでインタビューを強制実行するかを制御します。

```json
{
  "interview": {
    "requirements": boolean,  // true: 要件生成時にインタビューを強制実行
    "design": boolean,        // true: 設計生成時にインタビューを強制実行（将来用）
    "tasks": boolean          // true: タスク生成時にインタビューを強制実行（将来用）
  }
}
```

**使用例**: 既に `requirements-approved` 状態でも、明示的にインタビューを再実行したい場合に `interview.requirements: true` を設定。

---

## フェーズ別更新例

### 要件レビュー承認時

```javascript
spec.approvals.requirements.approved = true;
spec.phase = "requirements-approved";
spec.updated_at = new Date().toISOString();
spec.codex_reviews.requirements = {
  review_count: 2,
  final_verdict: "OK",
  resolved_issues: 3,
  session_id: "abc123",
  timestamp: new Date().toISOString()
};
```

### 設計レビュー承認時

```javascript
spec.approvals.design.approved = true;
spec.phase = "design-approved";
```

### タスクレビュー承認時

```javascript
spec.approvals.tasks.approved = true;
spec.phase = "ready-for-implementation";
```

---

## 判定値マッピング

| フェーズ | PASS | FAIL |
|---------|------|------|
| requirements | OK | NEEDS_REVISION |
| design | GO | NO_GO |
| tasks | APPROVED | NEEDS_REVISION |
| impl | APPROVED | NEEDS_REVISION |

---

## 完全な例

```json
{
  "name": "user-authentication",
  "phase": "impl-approved",
  "approvals": {
    "requirements": { "approved": true },
    "design": { "approved": true },
    "tasks": { "approved": true }
  },
  "updated_at": "2025-12-25T15:30:00.000Z",
  "interview": {
    "requirements": false,
    "design": false,
    "tasks": false
  },
  "codex_reviews": {
    "requirements": {
      "review_count": 1,
      "final_verdict": "OK",
      "resolved_issues": 0,
      "session_id": "codex-req-001",
      "timestamp": "2025-12-25T10:00:00.000Z"
    },
    "design": {
      "review_count": 2,
      "final_verdict": "GO",
      "resolved_issues": 2,
      "session_id": "codex-design-002",
      "timestamp": "2025-12-25T12:00:00.000Z"
    },
    "tasks": {
      "review_count": 1,
      "final_verdict": "APPROVED",
      "resolved_issues": 0,
      "session_id": "codex-tasks-003",
      "timestamp": "2025-12-25T14:00:00.000Z"
    },
    "impl": {
      "review_count": 3,
      "final_verdict": "APPROVED",
      "resolved_issues": 5,
      "session_id": "codex-impl-004",
      "timestamp": "2025-12-25T15:30:00.000Z"
    }
  }
}
```
