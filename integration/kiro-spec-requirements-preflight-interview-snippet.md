# Integration Snippet: Timeboxed Preflight Interview for /kiro:spec-requirements

## 概要

既存の `/kiro:spec-requirements` コマンド定義にタイムボックス付きプリフライトインタビュー機能を追加するための最小diffパッチです。

---

## Patch Instructions

### Step 1: Add AskUserQuestionTool to allowed-tools

```diff
 ---
 name: kiro:spec-requirements
 description: Generate comprehensive requirements for a specification
-allowed-tools: Bash, Glob, Grep, LS, Read, Write, Edit, MultiEdit, Update, WebSearch, WebFetch
+allowed-tools: Bash, Glob, Grep, LS, Read, Write, Edit, MultiEdit, Update, WebSearch, WebFetch, AskUserQuestionTool
 argument-hint: <feature-name>
 ---
```

---

### Step 2: Insert Preflight Interview Step (2.5)

ステップ2「Read Guidelines」とステップ3「Generate Requirements」の間に挿入:

```diff
 2. **Read Guidelines**:
    - Read `.kiro/settings/rules/ears-format.md` for EARS syntax rules
    - Read `.kiro/settings/templates/specs/requirements.md` for document structure

+2.5 **Optional Preflight Interview (timeboxed, Japanese)**:
+   - Purpose: clarify only high-risk unknowns WITHOUT slowing down generation.
+   - Trigger ONLY if any of the following is true:
+     - `.kiro/specs/$1/spec.json` phase is `"initialized"` (first run), OR
+     - `.kiro/specs/$1/spec.json` approvals.requirements.approved is `false`, OR
+     - `.kiro/specs/$1/requirements.md` project description is missing/thin, OR
+     - `.kiro/specs/$1/spec.json` contains `interview.requirements: true`
+   - Use AskUserQuestionTool in Japanese. Keep the requirements output language unchanged (still follow spec.json).
+   - Hard limits:
+     - Max 8 questions total (single round).
+     - If user answers "unknown/TBD", record it as Open Questions and continue.
+   - Interview questions (priority order):
+     1. このフィーチャーの成功基準は何ですか？
+     2. スコープ外とするものを3つ挙げてください
+     3. ユーザー/ロールの違いと権限の違いを教えてください
+     4. 失敗時の動作はどうすべきですか？
+     5. データの信頼できる情報源は何ですか？
+     6. セキュリティ・プライバシーの制約は何ですか？
+     7. 運用・可観測性について：何を監視すべきですか？
+     8. テスト可能な受け入れ基準を3つ挙げてください
+   - Write a concise Q&A + decisions summary to `.kiro/specs/$1/interview.md` (recommended).
+     - Do NOT block requirements generation if interview.md writing is skipped.
+   - Reliability fallback:
+     - If AskUserQuestionTool fails/unavailable/returns empty, immediately switch to plain text questions (still Japanese), same limits.

 3. **Generate Requirements**:
    - Create initial requirements based on project description
    - Group related functionality into logical requirement areas
    - Apply EARS format to all acceptance criteria
    - Use language specified in spec.json
```

---

### Step 3: Update Important Constraints

```diff
 ## Important Constraints

-- Generate initial version first, then iterate with user feedback (no sequential questions upfront)
+- Generate initial version first, then iterate with user feedback (no sequential questions upfront).
+  Exception: a triggered preflight interview is allowed ONLY if timeboxed (<= 8 questions) and focused on high-risk unknowns.
 - Follow EARS format strictly for all acceptance criteria
 - Use the language specified in spec.json for requirements output
```

---

### Step 4: Update Error Scenarios

```diff
 ### Error Scenarios

-- **Missing Project Description**: If requirements.md lacks project description, ask user for feature details
-- **Ambiguous Requirements**: Propose initial version and iterate with user rather than asking many upfront questions
++ **Missing Project Description**: If requirements.md lacks project description, use preflight interview (Japanese) to gather minimum viable details (<= 8 questions), then generate.
++ **Ambiguous Requirements**: Do NOT run long sequential questioning. Either:
++   - (If triggered) run the timeboxed preflight interview (<= 8 questions), then generate, OR
++   - Generate an initial version and iterate with user feedback.
```

---

## Triggers Summary

| Condition | Check Method | Action |
|-----------|--------------|--------|
| First run | `spec.json` phase == `"initialized"` | Run interview |
| Requirements not approved | `approvals.requirements.approved` == `false` | Run interview |
| Missing description | `requirements.md` project description missing/thin | Run interview |
| Explicit flag | `spec.json` contains `interview.requirements: true` | Run interview |
| None of above | - | Skip interview |

---

## Note: Mandatory Codex Review

**重要**: `/sdd-codex-review requirements $1` の呼び出しは引き続き必須です。

プリフライトインタビューはCodexレビューの代替ではありません。

```
[Preflight Interview] → [Requirements Generation] → [Codex Review] → [Approval]
```

---

## AskUserQuestionTool Usage Example

```json
{
  "questions": [
    {
      "question": "1. このフィーチャーの成功基準は何ですか？",
      "options": ["ユーザー満足度", "パフォーマンス指標", "ビジネスKPI達成", "エラー率低減"]
    },
    {
      "question": "2. スコープ外とするものを3つ挙げてください",
      "options": ["UI/UXの大幅変更", "既存機能の改修", "外部システム連携", "パフォーマンス最適化"]
    }
  ]
}
```

**注意**:
- 選択肢は参考例であり、ユーザーは自由入力も可能
- 各質問は単一トピックに限定
- 最大8問を超えないこと

---

## Fallback Implementation

```markdown
## プリフライトインタビュー（プレーンテキストモード）

AskUserQuestionToolが利用できないため、テキストで質問します。
以下の質問に回答してください（「わからない」「TBD」も可）:

1. このフィーチャーの成功基準は何ですか？（何をもって成功/失敗と判断しますか？）

2. スコープ外とするものを3つ挙げてください

3. ユーザー/ロールの違いと、それぞれの権限の違いを教えてください

4. 失敗時の動作はどうすべきですか？（待機/縮退運転/エラー終了）

5. データの信頼できる情報源は何ですか？並行性・整合性の要件はありますか？

6. セキュリティ・プライバシーの制約は何ですか？（ログ出力可否、データ保持期間など）

7. 運用・可観測性について：何を監視すべきですか？誰が運用しますか？

8. テスト可能な受け入れ基準を3つ挙げてください
```

---

## Complete Template Reference

完全なテンプレートは以下を参照:

- [examples/spec-requirements-template.md](../examples/spec-requirements-template.md)

詳細なドキュメントは以下を参照:

- [docs/kiro-spec-requirements-preflight-interview.md](../docs/kiro-spec-requirements-preflight-interview.md)
