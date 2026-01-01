---
name: kiro:spec-requirements
description: Generate EARS-format requirements with optional timeboxed preflight interview
arguments:
  - name: feature-name
    description: The feature to generate requirements for
allowed-tools: Bash, Glob, Grep, LS, Read, Write, Edit, MultiEdit, Update, WebSearch, WebFetch, AskUserQuestionTool
---

# Spec Requirements Command Template (with Preflight Interview)

このテンプレートは `/kiro:spec-requirements` コマンドの完全な実装例です。
タイムボックス付きプリフライトインタビュー機能が統合されています。

## Overview

EARS (Easy Approach to Requirements Syntax) 形式で要件を生成するコマンド。
オプションで、高リスクの不明点を事前に確認するプリフライトインタビューを実行します。

---

## Workflow

### 1. Read Project Context

```
1. Read `.kiro/steering/structure.md` (if exists) for project structure guidelines
2. Read `.kiro/steering/tech.md` (if exists) for technology context
3. Read `.kiro/steering/product.md` (if exists) for product context
4. Read `.kiro/specs/$1/spec.json` to get current phase and language settings
5. Read `.kiro/specs/$1/requirements.md` (if exists) for existing requirements
```

### 2. Read Guidelines

```
1. Read `.kiro/settings/rules/ears-format.md` for EARS syntax rules
2. Read `.kiro/settings/templates/specs/requirements.md` for document structure
```

### 2.5 Optional Preflight Interview (timeboxed, Japanese)

**目的**: 要件生成の速度を維持しながら、高リスクの不明点のみを事前に確認する。

#### Trigger Conditions

以下のいずれかが真の場合のみ実行:

1. `.kiro/specs/$1/spec.json` の `phase` が `"initialized"` （初回実行）
2. `.kiro/specs/$1/spec.json` の `approvals.requirements.approved` が `false` （要件未承認）
3. `.kiro/specs/$1/requirements.md` のプロジェクト説明が欠落または不十分
4. `.kiro/specs/$1/spec.json` に `"interview": { "requirements": true }` が含まれる

#### Hard Limits

- **最大8問**（単一ラウンド）
- **日本語**で質問（要件出力言語は spec.json に従う）
- ユーザーが「unknown」「TBD」「わからない」と回答した場合、Open Questions として記録し続行
- **生成をブロックしない**

#### Interview Questions (Priority Order)

AskUserQuestionTool を使用して以下の質問を行う:

```json
{
  "questions": [
    {
      "question": "1. このフィーチャーの成功基準は何ですか？（何をもって成功/失敗と判断しますか？）",
      "options": ["ユーザー満足度", "パフォーマンス指標", "ビジネスKPI達成", "エラー率低減"]
    },
    {
      "question": "2. スコープ外とするものを3つ挙げてください",
      "options": ["UI/UXの大幅変更", "既存機能の改修", "外部システム連携", "パフォーマンス最適化"]
    },
    {
      "question": "3. ユーザー/ロールの違いと権限の違いを教えてください",
      "options": ["管理者と一般ユーザー", "認証済みと未認証", "有料と無料プラン", "社内と社外ユーザー"]
    },
    {
      "question": "4. 失敗時の動作はどうすべきですか？",
      "options": ["待機してリトライ", "縮退運転（機能制限）", "エラー終了", "ユーザーに選択させる"]
    },
    {
      "question": "5. データの信頼できる情報源は何ですか？並行性・整合性の要件はありますか？",
      "options": ["データベースがmaster", "外部APIがmaster", "楽観的ロック", "悲観的ロック"]
    },
    {
      "question": "6. セキュリティ・プライバシーの制約は何ですか？",
      "options": ["ログ出力禁止項目あり", "データ保持期間制限", "暗号化必須", "特になし"]
    },
    {
      "question": "7. 運用・可観測性について：何を監視すべきですか？",
      "options": ["レスポンスタイム", "エラー率", "リソース使用量", "ビジネスメトリクス"]
    },
    {
      "question": "8. テスト可能な受け入れ基準を3つ挙げてください",
      "options": ["機能動作確認", "パフォーマンス基準", "セキュリティ検証", "ユーザビリティ"]
    }
  ]
}
```

#### Reliability Fallback

AskUserQuestionTool が失敗、利用不可、または空のレスポンスを返した場合:

1. 即座に**日本語プレーンテキスト質問**に切り替える
2. 同じ制限（最大8問）を維持
3. ユーザーの回答を収集して続行

```markdown
## プリフライトインタビュー（プレーンテキストモード）

以下の質問に回答してください（「わからない」「TBD」も可）:

1. このフィーチャーの成功基準は何ですか？
2. スコープ外とするものを3つ挙げてください
3. ユーザー/ロールの違いと権限の違いを教えてください
4. 失敗時の動作はどうすべきですか？
5. データの信頼できる情報源は何ですか？
6. セキュリティ・プライバシーの制約は何ですか？
7. 運用・可観測性について：何を監視すべきですか？
8. テスト可能な受け入れ基準を3つ挙げてください
```

#### Output: interview.md (Optional)

インタビュー結果を `.kiro/specs/$1/interview.md` に保存（推奨、必須ではない）:

```markdown
# Preflight Interview Summary

## Feature: $1
## Date: YYYY-MM-DD
## Interviewer: Claude Code

### Q&A Summary

| # | Question | Answer | Status |
|---|----------|--------|--------|
| 1 | 成功基準 | [回答] | Resolved / TBD |
| 2 | スコープ外 | [回答] | Resolved / TBD |
| ... | ... | ... | ... |

### Decisions Made

- [決定事項1]
- [決定事項2]

### Open Questions

- [未解決の質問があれば記載]
```

### 3. Generate Requirements

```
1. Create initial requirements based on project description and interview results
2. Group related functionality into logical requirement areas
3. Apply EARS format to all acceptance criteria:
   - WHEN [trigger], the system shall [response]
   - IF [condition], THEN the system shall [response]
   - WHILE [state], the system shall [response]
   - WHERE [feature], the system shall [response]
4. Use language specified in spec.json
5. Include any Open Questions from interview as TBD items
```

### 4. Update Metadata

```
1. Update spec.json:
   - Update timestamps
   - Record interview status if conducted
   - Note: phase will be set to "requirements-approved" after Codex review approval
2. Write requirements.md with generated content
```

### 5. CRITICAL: Invoke Codex Review

**必須**: 要件生成後、必ず以下を実行:

```
Skill tool invoke: "sdd-codex-review"
args: requirements $1
```

または:

```bash
/sdd-codex-review requirements $1
```

Codexレビューで APPROVED を取得するまでループ（最大6回）。

---

## Important Constraints

- Generate initial version first, then iterate with user feedback (no sequential questions upfront).
  **Exception**: a triggered preflight interview is allowed ONLY if:
  - Timeboxed (<= 8 questions)
  - Focused on high-risk unknowns
  - Triggered by specific conditions (first run, missing description, or explicit flag)
- Follow EARS format strictly for all acceptance criteria
- Use the language specified in spec.json for requirements output
- Keep requirements testable and measurable
- Maintain traceability between requirements and acceptance criteria

---

## Error Scenarios

### Missing Project Description

プリフライトインタビュー（日本語）を使用して最低限の詳細を収集（最大8問）してから生成:

1. トリガー条件を確認
2. AskUserQuestionTool でインタビュー実行
3. 失敗時はプレーンテキストにフォールバック
4. 収集した情報で要件を生成

### Ambiguous Requirements

長時間の連続質問を行わない。以下のいずれか:

1. **トリガー条件が満たされる場合**: タイムボックス付きプリフライトインタビュー（最大8問）を実行してから生成
2. **それ以外**: 初期バージョンを生成し、ユーザーフィードバックで反復

### AskUserQuestionTool Failure

1. ツール失敗を検出
2. 即座にプレーンテキストモードに切り替え
3. 同じ質問を日本語テキストで提示
4. 回答を収集して続行

### Codex Review Failure

1. レビュー失敗時は指摘事項を確認
2. 自動修正を適用
3. 再レビュー（最大6回）
4. 6回超過時は手動介入を要求

---

## EARS Format Reference

```
# 基本パターン

## WHEN (Event-Driven)
WHEN [trigger event], the system shall [response].
例: WHEN the user clicks the save button, the system shall persist the data to the database.

## IF-THEN (Conditional)
IF [condition], THEN the system shall [response].
例: IF the session has expired, THEN the system shall redirect to the login page.

## WHILE (State-Driven)
WHILE [state], the system shall [response].
例: WHILE the upload is in progress, the system shall display a progress indicator.

## WHERE (Feature-Based)
WHERE [feature], the system shall [response].
例: WHERE dark mode is enabled, the system shall use the dark color palette.

## Ubiquitous (Always True)
The system shall [response].
例: The system shall log all authentication attempts.
```

---

## Requirements Document Structure

```markdown
# Requirements: [Feature Name]

## Introduction

### Purpose
[Feature purpose]

### Scope
[What's in scope and out of scope]

### Definitions
[Key terms]

## Requirements

### [Requirement Area 1]

#### Objectives
- [Objective 1]
- [Objective 2]

#### Acceptance Criteria
- WHEN [trigger], the system shall [response]
- IF [condition], THEN the system shall [response]

### [Requirement Area 2]
...

## Open Questions

- [TBD items from interview]

## Appendix

### Interview Summary
[Link to interview.md if conducted]
```

---

## Usage Example

```bash
# First run (triggers preflight interview)
/kiro:spec-requirements my-feature

# Subsequent run (skips interview if description is sufficient)
/kiro:spec-requirements my-feature

# Force interview
# (set interview.requirements: true in spec.json first)
/kiro:spec-requirements my-feature
```
