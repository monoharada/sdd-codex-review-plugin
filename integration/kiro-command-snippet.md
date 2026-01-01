# Kiroコマンド連携スニペット

このスニペットは、`/kiro:spec-impl` コマンドに Codex レビューの自動呼び出しを追加するためのものです。

---

## 概要

`/kiro:spec-impl` コマンドで実装が完了した後、**セクション単位**で Codex レビューを実行します。
タスクごとではなく、メジャーセクション完了時にのみレビューを実行することで、効率的なレビューサイクルを実現します。

---

## セクション単位レビューの考え方

### 従来方式（非推奨）
```
Task 1.1完了 → レビュー → Task 1.2完了 → レビュー → ...
```
→ オーバーヘッドが大きい

### セクション単位（推奨）
```
Task 1.1完了 → Task 1.2完了 → Task 1.3完了 → Section 1完了 → レビュー
```
→ セクション単位でまとめてレビュー

---

## 追加するセクション

Kiroコマンド定義ファイル（例: `.claude/commands/kiro/spec-impl.md`）の最後に以下を追加：

```markdown
## 実装完了後の自動レビュー（セクション単位）

### 重要: セクション完了時のみCodexレビューを実行

タスク実装が完了したら、**セクション完了を確認**してからレビューを実行。
タスクごとにレビューを回さないこと。

### セクション完了の確認ロジック

```
1. 現在のタスクが属するセクションを特定
   - tasks.mdの`##`見出しでセクションを識別

2. セクション内の全タスクの期待ファイルを確認
   - 各タスクの`**Creates:**`/`**Modifies:**`を参照

3. 全ファイルが存在するか確認

4. 完了判定
   - 全ファイル存在 AND 未レビュー → レビュー実行
   - それ以外 → 次のタスクへ続行
```

### 呼び出し条件

```
IF section_complete AND NOT section_reviewed:
    // Codexレビュー実行（まずレビューを行う）
    Skill tool invoke: "sdd-codex-review"
    args: impl-section $feature $section_id
    APPROVED が返されるまでループ

    // E2Eエビデンス収集（レビュー承認後、[E2E]タグ付きセクションのみ）
    IF section.e2e_required AND section.e2e_evidence.status == "pending":
        Playwright MCPでE2Eエビデンス収集
        結果を .context/e2e-evidence/ に保存
        ユーザーにスクリーンショットパスを報告
        E2E失敗でも次へ続行

ELSE:
    次のタスクへ続行（レビューはスキップ）
```

### 期待される動作

1. タスク実装が完了
2. セクション完了をチェック（全期待ファイル存在確認）
3. セクション完了 AND 未レビュー の場合：
   a. **Codexレビュー実行**：
      - Codex CLI を使用してセクションレビューを実行
      - レビュー結果を解析
      - NEEDS_REVISION の場合は修正を適用して再レビュー
      - APPROVED になるまでループ
   b. **E2Eエビデンス収集**（APPROVED後、`[E2E]`タグ付きセクションのみ）：
      - Playwright MCPでスクリーンショット収集
      - `.context/e2e-evidence/`に保存
      - ユーザーにパスを報告
      - E2E失敗でもセクション完了として扱う
   c. 次のセクション/タスクへ進む
4. セクション未完了の場合：
   - レビューをスキップして次のタスクへ

### シミュレート禁止

- Codex CLI を実際に実行すること
- レビュー結果を自分で生成しないこと
- Session ID を報告に含めること
```

---

## 完全なコマンド定義例

以下は `/kiro:spec-impl` コマンドの完全な定義例です：

```markdown
---
name: kiro:spec-impl
description: Execute spec tasks using TDD methodology
arguments:
  - name: feature-name
    description: The feature to implement
  - name: task-numbers
    description: Optional specific task numbers (e.g., "1.1,1.2")
---

# Spec Implementation Command

## Overview

Implement tasks from `.kiro/specs/[feature]/tasks.md` using TDD methodology.

## Workflow

1. Read spec files (requirements.md, design.md, tasks.md)
2. Identify target tasks
3. For each task:
   a. Write failing test (RED)
   b. Implement minimal code (GREEN)
   c. Refactor (REFACTOR)
4. Update spec.json progress

## 実装完了後の自動レビュー（セクション単位）

### 重要: セクション完了時のみCodexレビューを実行

タスク実装完了後、**セクション完了を確認**してからレビューを実行。

### セクション完了チェック

1. 現在のタスクが属するセクションを特定
2. セクション内の全タスクの期待ファイルが存在するか確認
3. 全ファイル存在 AND 未レビュー → レビュー実行

### 呼び出し条件

```
IF section_complete AND NOT section_reviewed:
    Skill tool invoke: "sdd-codex-review"
    args: impl-section $1 $section_id
ELSE:
    次のタスクへ続行
```

### 条件

- セクション完了時のみレビュー実行
- Codexレビューを先に実行、APPROVED後にE2Eエビデンス収集
- `[E2E]`タグ付きセクションでのみE2Eエビデンスを収集
- E2E失敗でもセクション完了として扱う（ブロッキングではない）
- APPROVED が返されるまで修正・再レビューをループ
- 最大6回のリトライ
- 6回超過時はユーザー介入を要求
```

---

## 注意事項

1. **Skill tool の使用**: `/sdd-codex-review` は Skill tool で呼び出す必要があります
2. **セクション単位**: タスクごとではなく、セクション完了時のみレビューを実行
3. **シミュレート禁止**: Codex CLI を実際に実行し、Session ID を報告に含めること
3. **ループ処理**: APPROVED になるまでレビュー→修正→再レビューのループを継続
4. **ユーザー介入**: 6回超過時は自動処理を停止し、ユーザーに判断を委ねる
