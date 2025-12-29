---
name: sdd-codex-review
description: "Codex統合レビュースキル。kiro:spec-requirements/design/tasks/impl完了後に実行。/sdd-codex-review [phase] [feature]で各フェーズをCodexレビュー。APPROVEDまでループ。"
---

# SDD-Codex-Review: Codex統合レビューワークフロー

cc-sddワークフローの各フェーズ完了時にOpenAI Codexを使用して自動レビューを実行し、
OKが出るまでフィードバックループを繰り返すスキルです。

## 前提条件

- `codex` CLIがインストール済み
- プロジェクトがcc-sdd仕様に準拠（`.kiro/specs/[feature]/`）
- spec.jsonが存在し、フェーズ情報が正しく設定されている

---

## ⚠️ 重要: シミュレート禁止・外部レビュー必須

**このスキルでは、Codex CLIを実際に実行することが必須です。**

### 禁止事項
- ❌ Codexのレビュー結果を自分で生成（シミュレート）すること
- ❌ `codex exec`を実行せずにJSON verdictを返すこと
- ❌ Codex session IDなしでAPPROVEDを報告すること

### 必須事項
- ✅ 必ず`codex exec --full-auto "[prompt]"`を実行すること
- ✅ Codex出力の`session id:`をユーザー報告に含めること
- ✅ 外部LLM（Codex/GPT）による独立レビューの価値を尊重すること

### 理由
同一モデル（Claude）による自己レビューは盲点を見逃すリスクがあります。
異なるモデル（Codex/GPT）による独立レビューは、以下の価値を提供します：
- 別視点からの問題発見
- 確証バイアスの排除
- レビュー品質の第三者保証

### 検証方法
ユーザーは以下の方法でCodex実行を確認できます：
1. レビュー報告に`Codex Session ID`が含まれているか確認
2. 疑問がある場合は「本当にcodexにレビューしてもらった？」と質問
3. 必要に応じてCodexログの提示を要求

---

## 使用方法

```bash
# 特定フェーズのレビュー実行
/sdd-codex-review requirements [feature-name]
/sdd-codex-review design [feature-name]
/sdd-codex-review tasks [feature-name]
/sdd-codex-review impl [feature-name]

# 自動進行モード（現在のフェーズから順次レビュー）
/sdd-codex-review auto [feature-name]
```

## ワークフロー概要

```
┌─────────────┐     ┌──────────────┐     ┌─────────────┐
│ kiro:spec-* │ ──▶ │ Codex Review │ ──▶ │ 自動修正    │
│ 完了        │     │ (JSON出力)   │     │             │
└─────────────┘     └──────────────┘     └──────┬──────┘
                                                 │
       ┌──────────────────────────────┬──────────┴──────────┐
       ▼                              ▼                      ▼
┌──────────────┐                ┌──────────┐          ┌──────────┐
│ OK: 次フェーズ│                │ 再レビュー│          │ 6回超過: │
│ spec.json更新│                │ (ループ) │          │ ユーザー │
│ ユーザー報告 │                └──────────┘          │ 介入要求 │
└──────────────┘                                      └──────────┘
```

---

## Phase 1: 要件レビュー (spec-requirements完了後)

### トリガー
```bash
/sdd-codex-review requirements [feature-name]
```

### 処理フロー

1. **ファイル読み込み**
   ```bash
   # 対象ファイル
   .kiro/specs/[feature]/requirements.md
   .kiro/specs/[feature]/spec.json
   ```

2. **Codexレビュー実行**
   ```bash
   codex exec --full-auto "
   あなたはcc-sdd要件レビュアーです。以下の要件定義をレビューしてください。

   ## レビュー基準
   1. 完全性: 全機能要件がカバーされているか
   2. 明確性: 曖昧な表現がないか
   3. テスト可能性: 受け入れ基準が検証可能か
   4. 一貫性: 要件間の矛盾がないか
   5. EARS形式準拠: When/If/The system shall 形式か

   ## 要件ファイル内容
   $(cat .kiro/specs/[feature]/requirements.md)

   ## 出力形式（JSON）
   {
     \"verdict\": \"OK\" | \"NEEDS_REVISION\",
     \"issues\": [
       {
         \"severity\": \"critical\" | \"medium\" | \"minor\",
         \"location\": \"Requirement X.X\",
         \"issue\": \"問題の説明\",
         \"suggestion\": \"修正案\"
       }
     ],
     \"summary\": \"全体評価\"
   }
   "
   ```

3. **結果解析と判定**
   - `verdict === "OK"` → 承認、次フェーズへ
   - `verdict === "NEEDS_REVISION"` → 自動修正適用、再レビュー

4. **自動修正（NEEDS_REVISION時）**
   - `issues`配列を走査
   - 各`suggestion`をrequirements.mdに適用
   - 再度Codexレビューを実行（最大6回）

5. **spec.json更新（OK時）**
   ```javascript
   spec.approvals.requirements.approved = true;
   spec.phase = "requirements-approved";
   spec.updated_at = new Date().toISOString();
   ```

6. **ユーザー報告**
   ```markdown
   ## Codexレビュー完了: 要件定義

   | 項目 | 値 |
   |------|-----|
   | フィーチャー | [feature-name] |
   | レビュー回数 | X / 6 |
   | 最終判定 | OK |

   ### 解決済み指摘
   - [修正内容1]
   - [修正内容2]

   ### 次のステップ
   `/kiro:spec-design [feature-name]` で設計フェーズへ進んでください。
   ```

---

## Phase 2: 設計レビュー (spec-design完了後)

### トリガー
```bash
/sdd-codex-review design [feature-name]
```

### 処理フロー

1. **ファイル読み込み**
   ```bash
   .kiro/specs/[feature]/design.md
   .kiro/specs/[feature]/requirements.md  # 参照用
   ```

2. **Codexレビュー実行**
   - 設計レビュー基準（`.kiro/settings/rules/design-review.md`準拠）
   - GO/NO-GO判定
   - Critical Issues ≦ 3件

3. **判定基準**
   - `verdict === "GO"` → 承認
   - `verdict === "NO_GO"` → 自動修正 or ユーザー介入

4. **spec.json更新（GO時）**
   ```javascript
   spec.approvals.design.approved = true;
   spec.phase = "design-approved";
   ```

---

## Phase 3: タスクレビュー (spec-tasks完了後)

### トリガー
```bash
/sdd-codex-review tasks [feature-name]
```

### レビュー基準
- 要件カバレッジ: 全要件IDがタスクにトレースされているか
- タスク粒度: 各タスクが適切なサイズか
- 依存関係: タスク間の依存が正しく定義されているか
- 並列実行可能性: (P)マークが適切に付与されているか

### spec.json更新（APPROVED時）
```javascript
spec.approvals.tasks.approved = true;
spec.phase = "ready-for-implementation";
```

---

## Phase 4: 実装レビュー (spec-impl完了後)

### トリガー
```bash
/sdd-codex-review impl [feature-name]
```

### レビュー対象
- 変更されたソースファイル
- 対象タスクのtasks.md内の定義
- design.mdのインターフェース定義

### レビュー基準
- 要件準拠: 対象タスクの要件IDが満たされているか
- 設計準拠: design.mdのインターフェース定義に従っているか
- コード品質: TypeScript strict mode準拠、テストカバレッジ

### 設計コンテキストの提供（重要）

**NEEDS_REVISION時は、タスク範囲と設計意図を明示的に伝えて再レビューする**

段階的リファクタリングなど、タスクが分割されている場合、Codexが全体設計を理解せずに指摘することがある。この場合、以下の情報を補足して`resume`で再レビューを依頼する：

```bash
codex exec --full-auto resume [SESSION_ID] "
## 設計コンテキストの補足

### タスク分割の設計意図
- タスクX.X: [このタスクの範囲]を実施
- タスクY.Y: [関連する後続タスク]で対応予定

### 現タスクの完了条件
1. [条件1] ✅
2. [条件2] ✅
...

### 指摘への対応方針
[指摘内容]は[後続タスク]で対応予定。これは[理由]という設計判断。

この設計コンテキストを踏まえて再評価をお願いします。
"
```

**効果**: Codexがタスク範囲を正しく理解し、適切な判定（APPROVED）を返す

### 実装レビュー専用コマンド（オプション）
```bash
# mainブランチとの差分をベースにレビュー
codex exec --full-auto review --base main "
[追加のレビュー指示（要件ID、テスト基準など）]
"

# 未コミットの変更をレビュー
codex exec --full-auto review --uncommitted "
[追加のレビュー指示]
"
```
**メリット**: コード差分が自動抽出されるため、ファイル内容を手動で渡す必要がない

---

## 自動進行モード

### トリガー
```bash
/sdd-codex-review auto [feature-name]
```

### 処理フロー

1. spec.jsonの現在`phase`を確認
2. 現在フェーズの成果物をCodexでレビュー
3. OKなら次フェーズへ自動進行
4. 全フェーズ完了までループ

```
initialized → requirements → design → tasks → impl → completed
```

---

## レビュー判定基準

| 判定 | 条件 |
|------|------|
| OK/GO/APPROVED | critical = 0件 かつ medium ≦ 2件 |
| NEEDS_REVISION/NO_GO | critical ≧ 1件 または medium ≧ 3件 |

---

## エラーハンドリング

### Codex呼び出し失敗
- 3回リトライ（指数バックオフ: 5s, 15s, 45s）
- 失敗継続時はユーザーに通知

### 無限ループ防止
- 各フェーズ最大6回のレビューサイクル
- 6回超過時はユーザー手動介入を要求

### JSON解析失敗
- Codex出力が期待形式でない場合
- raw出力をユーザーに提示して判断を委ねる

---

## Codex呼び出しパターン

### 初回レビュー
```bash
# 基本形式
codex exec --full-auto "[prompt]"

# 実装レビュー専用（コード差分ベース）
codex exec --full-auto review --base main "[追加の指示]"
codex exec --full-auto review --uncommitted "[追加の指示]"
```

### 再レビュー（2回目以降）
```bash
# 前回セッションを継続（コンテキスト保持、高速）
codex exec --full-auto resume --last "[修正内容と再レビュー依頼]"

# 特定セッションを継続
codex exec --full-auto resume [SESSION_ID] "[修正内容と再レビュー依頼]"
```

**`resume`のメリット**:
- 前回のレビューコンテキストが保持される（ファイル再読み込み不要）
- トークン消費が少なく、レスポンスが高速
- 同一セッションIDで一貫性のあるレビュー

### プロンプトテンプレート参照
- 要件: `prompts/requirements-review.md`
- 設計: `prompts/design-review.md`
- タスク: `prompts/tasks-review.md`
- 実装: `prompts/impl-review.md`

---

## spec.json更新フォーマット

```json
{
  "phase": "requirements-approved",
  "approvals": {
    "requirements": { "approved": true },
    "design": { "approved": false },
    "tasks": { "approved": false }
  },
  "updated_at": "2025-12-25T12:00:00.000Z",
  "codex_reviews": {
    "requirements": {
      "review_count": 2,
      "final_verdict": "OK",
      "resolved_issues": 3
    }
  }
}
```

---

## ユーザー報告フォーマット

```markdown
## Codexレビュー完了報告

### フェーズ: [PHASE] ([FEATURE])

| 項目 | 状態 |
|------|------|
| **Codex Session ID** | `[SESSION_ID]` |
| レビュー回数 | X / 6 |
| 最終判定 | [VERDICT] |
| 解決済み指摘 | X件 |

**注意**: Codex Session IDがない報告はシミュレートの可能性があります。

### サマリー
[Codexからのsummary]

### 解決済み指摘事項
1. [issue1] → [suggestion1適用]
2. [issue2] → [suggestion2適用]

### 残存警告（承認済み）
- [minor issue] (対応不要)

### 更新ファイル
- `.kiro/specs/[FEATURE]/[FILE]`
- `.kiro/specs/[FEATURE]/spec.json`

### 次のステップ
[次に実行すべきコマンド]
```

---

## 実装時の注意事項

1. **ファイル読み込み**: 必ず対象ファイルの存在確認を行う
2. **JSON抽出**: Codex出力から```json...```ブロックを正規表現で抽出
3. **差分適用**: suggestionは可能な限り自動適用、複雑な場合はユーザー確認
4. **履歴保持**: 各レビュー結果をspec.jsonのcodex_reviewsに記録
5. **並列処理禁止**: 1フェーズずつ順次処理（依存関係のため）

---


---

## コンテキスト節約モード（自動実行）

**このスキルは、Codexレビュー実行時に自動的にコンテキスト節約を行います。**
オプション指定は不要です。Claude Codeが自律的に判断して実行します。

### 自律的な動作フロー

```
┌─────────────────────────────────────────────────────────────────┐
│ 1. 状態保存（自動）                                              │
│    レビュー開始前に .context/sdd-review-state.json を作成       │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 2. バックグラウンド実行（自動）                                  │
│    codex exec を run_in_background で実行                       │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 3. コンテキスト圧縮を提案（自動）                                │
│    「レビュー中。コンテキスト圧縮しますか？」                   │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 4. 結果取得・復元（自動）                                        │
│    TaskOutput で結果取得 → 状態から復元 → 処理続行              │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 5. クリーンアップ（自動）                                        │
│    状態ファイルを削除                                           │
└─────────────────────────────────────────────────────────────────┘
```

### スキル実装時の必須動作

**重要**: このスキルを発動する際、Claude Codeは以下の手順を**必ず自動実行**すること。

#### Step 1: 状態保存（レビュー開始前に自動実行）

```bash
# .context/ ディレクトリがなければ作成
mkdir -p .context

# 状態ファイルを作成
cat > .context/sdd-review-state.json << STATEEOF
{
  "feature_name": "[FEATURE]",
  "phase": "[PHASE]",
  "spec_path": ".kiro/specs/[FEATURE]/",
  "started_at": "[TIMESTAMP]",
  "restore_hints": {
    "spec_json": ".kiro/specs/[FEATURE]/spec.json",
    "steering": ".kiro/steering/"
  }
}
STATEEOF
```

#### Step 2: バックグラウンド実行

```python
# Bash tool を run_in_background=true で使用
result = Bash(
    command='codex exec --full-auto "[レビュープロンプト]"',
    run_in_background=True
)
task_id = result.task_id  # 状態ファイルに追記
```

#### Step 3: ユーザーへの通知と提案（自動表示）

レビュー開始後、以下のメッセージを**必ず**表示：

```markdown
## Codexレビューをバックグラウンドで実行中

| 項目 | 値 |
|------|-----|
| フェーズ | [PHASE] |
| フィーチャー | [FEATURE] |
| タスクID | [TASK_ID] |

### コンテキスト節約の提案

レビュー完了まで数分かかります。この間に以下が可能です：

1. **コンテキストを圧縮する**（推奨）
   - 「圧縮して」と言ってください
   - 現在の会話を要約してメモリを解放します

2. **別の作業を続ける**
   - 他のファイルの確認や編集
   - 別フィーチャーの調査

3. **結果を待つ**
   - このまま待機

レビュー結果は自動的に取得して処理します。
```

#### Step 4: コンテキスト圧縮（ユーザーが希望した場合）

ユーザーが「圧縮して」「コンテキスト整理」等と言った場合：

1. 現在のセッションの要約を生成
2. 重要ポイントを `.context/session-summary.md` に保存
3. 以下を通知：

```markdown
## コンテキストを圧縮しました

要約を `.context/session-summary.md` に保存しました。

レビュー結果は自動取得します。
結果が返ってきたら、状態ファイルから自動復元して続行します。
```

#### Step 5: 結果取得と自動復元

バックグラウンドタスク完了後：

```python
# 結果取得
result = TaskOutput(task_id=task_id, block=True, timeout=300000)

# 状態復元（自動）
state = read(".context/sdd-review-state.json")
spec_json = read(state.restore_hints.spec_json)

# 必要に応じて steering/ を再読み込み
if context_was_compressed:
    read_all(state.restore_hints.steering)

# 通常のレビュー後処理を継続
# - JSON解析
# - verdict判定
# - 自動修正（必要時）
# - spec.json更新
```

#### Step 6: クリーンアップ（自動）

レビュー正常完了後：

```bash
rm .context/sdd-review-state.json
rm -f .context/session-summary.md  # 存在する場合
```

### セッション再開時の自動復元

新しいセッション開始時に `.context/sdd-review-state.json` が存在する場合：

```markdown
## 前回のレビューが中断されています

| 項目 | 値 |
|------|-----|
| フィーチャー | [FEATURE] |
| フェーズ | [PHASE] |
| 開始時刻 | [TIMESTAMP] |

レビュー結果を確認しますか？
1. 結果を確認して続行
2. 最初からやり直す
3. キャンセル
```

### エラーハンドリング

#### タスクIDが無効（セッション終了）

```markdown
バックグラウンドタスクが見つかりません。
セッションが終了した可能性があります。

状態ファイルから復元して、レビューを再実行しますか？
```

#### Codex実行エラー

```markdown
Codexの実行でエラーが発生しました。

エラー: [ERROR_MESSAGE]

リトライしますか？（3回までリトライ可能）
```

### 注意事項

1. **シミュレート禁止は維持**
   - バックグラウンド実行でも、必ずCodex CLIを実際に実行
   - Session IDを報告に含める

2. **自動実行が基本**
   - ユーザーがオプションを指定する必要なし
   - スキル発動時に自動的にこのフローを実行

3. **状態ファイルは一時的**
   - レビュー完了後は自動削除
   - 永続化が必要な情報は spec.json に記録
