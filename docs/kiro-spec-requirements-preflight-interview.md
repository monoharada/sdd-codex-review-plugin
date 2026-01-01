# Interview-first (timeboxed) for /kiro:spec-requirements

## 概要

このドキュメントは、`/kiro:spec-requirements` コマンドにオプションのプリフライトインタビュー機能を追加する方法を説明します。

このリポジトリは、`.kiro/specs/<feature>/` 配下のKiro互換スペックに対して外部Codexレビューを統合します。
再作業を減らすため、`requirements.md` 生成前に高リスクの不明点を明確化することが効果的です。

---

## Goal

- `/kiro:spec-requirements` の速度を維持（まず生成、後で反復）
- **オプション**、**トリガー式**、**タイムボックス付き**のプリフライトインタビューを追加
- インタビューには `AskUserQuestionTool`（日本語）を使用
- 要件出力言語は変更しない（引き続き `.kiro/specs/<feature>/spec.json` の設定に従う）
- 生成後は通常通り `/sdd-codex-review requirements <feature>` を実行

---

## Why Timeboxed?

上流のコマンドは、速度のために長時間の連続質問を明示的に避けています。
そのため、プリフライトインタビューは以下の制約を持ちます:

| 制約 | 値 |
|------|-----|
| 実行条件 | トリガー条件を満たす場合のみ |
| 最大質問数 | 8問 |
| ラウンド数 | 1回のみ（連続質問禁止） |
| 対象 | 高リスクの不明点のみ |

**高リスクの不明点とは**:
- 認証/ロール/権限
- データの信頼できる情報源
- 失敗モード
- 運用/可観測性
- セキュリティ/プライバシー
- アクセシビリティ

---

## Trigger Conditions

プリフライトインタビューは以下のいずれかが真の場合のみ実行:

| # | 条件 | 説明 |
|---|------|------|
| 1 | `spec.json` phase == `"initialized"` | 初回実行（要件未承認） |
| 2 | `approvals.requirements.approved` == `false` | 要件が未承認 |
| 3 | `requirements.md` プロジェクト説明が欠落/不十分 | コンテキスト不足 |
| 4 | `spec.json` に `interview.requirements: true` | 明示的なフラグ |

### 条件判定ロジック

```
IF phase == "initialized" OR approvals.requirements.approved == false THEN
  RUN preflight interview
ELSE IF requirements.md project description is missing/thin THEN
  RUN preflight interview
ELSE IF spec.json contains interview.requirements: true THEN
  RUN preflight interview
ELSE
  SKIP preflight interview (proceed directly to generation)
END
```

---

## Interview Content (8 Questions Priority Order)

高リスクの不明点を優先順位順に確認します。各質問は単一トピックに限定:

| # | 質問（日本語） | 目的 |
|---|---------------|------|
| 1 | このフィーチャーの成功基準は何ですか？（何をもって成功/失敗と判断しますか？） | ゴールの明確化 |
| 2 | スコープ外とするものを3つ挙げてください | 境界の明確化 |
| 3 | ユーザー/ロールの違いと、それぞれの権限の違いを教えてください | 認可要件の把握 |
| 4 | 失敗時の動作はどうすべきですか？（待機/縮退運転/エラー終了） | 障害対応の設計 |
| 5 | データの信頼できる情報源は何ですか？並行性・整合性の要件はありますか？ | データ整合性 |
| 6 | セキュリティ・プライバシーの制約は何ですか？（ログ出力可否、データ保持期間など） | セキュリティ要件 |
| 7 | 運用・可観測性について：何を監視すべきですか？誰が運用しますか？ | 運用設計 |
| 8 | テスト可能な受け入れ基準を3つ挙げてください | 検証可能性 |

### 回答ハンドリング

- ユーザーが「unknown」「TBD」「わからない」と回答した場合:
  - **Open Questions** として記録
  - 生成を**ブロックしない**
  - 要件に TBD セクションとして含める

---

## How to Integrate

詳細なパッチ手順は以下を参照:

**[integration/kiro-spec-requirements-preflight-interview-snippet.md](../integration/kiro-spec-requirements-preflight-interview-snippet.md)**

### Quick Start

1. `allowed-tools` に `AskUserQuestionTool` を追加
2. ステップ 2.5「Optional Preflight Interview」を挿入
3. 「Important Constraints」を更新（トリガードpreflight例外を許可）
4. 「Error Scenarios」を更新
5. 必須の `/sdd-codex-review requirements <feature>` 呼び出しを維持

完全なテンプレートは以下を参照:

**[examples/spec-requirements-template.md](../examples/spec-requirements-template.md)**

---

## Fallback Behavior

### AskUserQuestionTool 失敗時

```
IF AskUserQuestionTool fails/unavailable/returns empty THEN
  1. 即座にプレーンテキストモードに切り替え
  2. 同じ質問を日本語テキストで提示
  3. 同じ制限を維持（最大8問）
  4. 回答を収集して続行
END
```

### プレーンテキストフォールバック例

```markdown
## プリフライトインタビュー

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

---

## interview.md Output Format

インタビュー結果は `.kiro/specs/<feature>/interview.md` に保存（推奨、必須ではない）:

```markdown
# Preflight Interview Summary

## Feature: [feature-name]
## Date: YYYY-MM-DD
## Interviewer: Claude Code

### Q&A Summary

| # | Question | Answer | Status |
|---|----------|--------|--------|
| 1 | 成功基準 | ユーザー満足度とパフォーマンス指標 | Resolved |
| 2 | スコープ外 | UI大幅変更、外部連携、パフォーマンス最適化 | Resolved |
| 3 | ユーザー/ロール | 管理者と一般ユーザー、権限は管理者のみ設定変更可 | Resolved |
| 4 | 失敗時の動作 | 縮退運転でエラーメッセージ表示 | Resolved |
| 5 | データ信頼源 | データベースがmaster、楽観的ロック | Resolved |
| 6 | セキュリティ | パスワードはログ出力禁止、データ保持30日 | Resolved |
| 7 | 運用/可観測性 | レスポンスタイムとエラー率を監視、SREチームが運用 | Resolved |
| 8 | 受け入れ基準 | わからない | TBD |

### Decisions Made

- ユーザー権限は管理者/一般の2階層
- 失敗時は縮退運転を採用
- 楽観的ロックでデータ整合性を担保

### Open Questions

- テスト可能な受け入れ基準は要件生成後に再確認

### Notes

- インタビュー方法: AskUserQuestionTool
- 所要時間: 約5分
```

---

## Recommended Workflow

```
1. /kiro:spec-requirements <feature>
   │
   ├─ [トリガー条件を満たす] → プリフライトインタビュー実行
   │                         ↓
   │                         interview.md に保存（オプション）
   │                         ↓
   └─────────────────────────→ requirements.md 生成
                               ↓
2. /sdd-codex-review requirements <feature>
   │
   ├─ [OK] → 次のフェーズへ
   └─ [NEEDS_REVISION] → 自動修正 → 再レビュー（最大6回）
                          ↓
3. requirements.md レビュー・確認
   ↓
4. 必要に応じて再実行（/kiro:spec-requirements <feature>）
   ↓
5. 承認後、設計フェーズへ進む
```

---

## Acceptance Tests (Self-check)

### Test 1: 初回実行

**シナリオ**: フィーチャーの初回要件生成

**手順**:
1. `/kiro:spec-requirements <feature>` を実行
2. `spec.json` の phase が `"initialized"` または `approvals.requirements.approved` が `false` であることを確認

**期待結果**:
- プリフライトインタビューが実行される（日本語、最大8問）
- `requirements.md` が spec.json で指定された言語で生成される
- メタデータ更新が行われる（phase: `"requirements-approved"` after Codex review）
- `/sdd-codex-review requirements <feature>` が呼び出される

### Test 2: 十分な説明がある場合の再実行

**シナリオ**: 既に十分なプロジェクト説明がある状態での再実行

**手順**:
1. `requirements.md` に十分なプロジェクト説明があることを確認
2. `spec.json` の phase が `"requirements-approved"` 以降であることを確認
3. `spec.json` に `interview.requirements: true` がないことを確認
4. `/kiro:spec-requirements <feature>` を実行

**期待結果**:
- プリフライトインタビューは**実行されない**
- コマンドは高速に維持される

### Test 3: AskUserQuestionTool 失敗

**シナリオ**: AskUserQuestionTool が利用不可または失敗

**手順**:
1. AskUserQuestionTool が失敗する状況を作成
2. `/kiro:spec-requirements <feature>` を実行

**期待結果**:
- 即座にプレーンテキスト質問（日本語）にフォールバック
- 最大8問の制限を維持
- 生成が続行される

---

## Related Files

- [examples/spec-requirements-template.md](../examples/spec-requirements-template.md) - 完全なコマンドテンプレート
- [integration/kiro-spec-requirements-preflight-interview-snippet.md](../integration/kiro-spec-requirements-preflight-interview-snippet.md) - 統合スニペット
- [skill/SKILL.md](../skill/SKILL.md) - sdd-codex-review スキル定義

---

## Version History

| Date | Version | Changes |
|------|---------|---------|
| 2026-01-01 | 1.0.0 | Initial release |
