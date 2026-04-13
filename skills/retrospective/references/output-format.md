# 日報出力フォーマット

## 設計原則

- **他メンバーが読む前提（部署全員）**。内部ID・内部語彙を出力に含めない
- シンプルに。Must/Win の内部分類は出力に含めない。全てゴール
- 提案理由は YAML に残し、出力には出さない

---

## フォーマット

```
■ GoalSetting & Done

- [x] {summary}
- [ ] {summary}
      … {結果 or 残タスク}

■ 明日のGoalSetting

- [ ] {summary}
      → {done_condition}

■ 今週のGoalSetting

- [ ] {プロジェクト名}: {週末に達成したい状態}
- [x] {プロジェクト名}: {週末に達成したい状態}

------------------------------------------
■ 今期ストレッチ / Q目標 / 目指す姿 / 輝かしい未来
{今期ストレッチのキーワード} / {Q目標のキーワード} / {目指す姿のキーワード} / {輝かしい未来のキーワード}
```

---

## 表示ルール

### GoalSetting & Done

- 2行構造: 1行目 `- [x]/[ ] {summary}`、2行目（未達時のみ）`      … {結果 or 残タスク}`
- achieved は1行で完結。done_condition は表示しない（結果で上書きされるため）
- partial/missed のみ2行目に結果・残タスクを記載
- 計画外の作業は出力しない（YAML の reconciliation.unexpected に記録されるが、日報には含めない）
- 初回実行（前営業日のゴールなし）の場合、このセクション自体をスキップ

### 明日のGoalSetting

- 2行構造: 1行目 `- [ ] {summary}`、2行目 `      → {done_condition}`
- done_condition は明日の目標として2行目に表示する（Done 欄との違い）
- Must/Win の区別は出力しない。全てフラットに並べる
- 並び順は内部的に Must → Win の優先度順

### 今週のGoalSetting

- 今週の weekly マイルストーンをプロジェクト単位で集約し、「週末に達成したい状態」として表示する
- 週次マイルストーンの summary（タスクの羅列）をそのまま出さず、抽象度を上げて「どういう状態になっていたいか」に変換する
  - 変換元: マイルストーンの summary + 親の月次マイルストーンの done_condition
  - 例: `比較分析: 詳細設計完了、モブQA設計、…` → `比較分析: 実装着手できる状態`
- 達成済み → `[x]`、未達成 → `[ ]`
- ID、進捗インジケータは表示しない

### North Star 1行サマリ

- 今期ストレッチ / Q目標 / 目指す姿 / 輝かしい未来 の4層をスラッシュ区切りで1行にまとめる
- Q目標は context.objective の値を使用する（例: `新たな価値創出 > 実用的な診断体験の提供`）
- 毎日同じ内容が出る（方向性の定着が目的）

**ファクト確定ルール**:
- 今Q目標.yml の `goals[].status` が `confirmed` の場合のみ当該 goal の summary を表示する
- `status` が未設定 or `in_progress` の場合は「設定中」と表示する
- 現在は stretch 目標のみ status を運用。performance 目標は従来通り表示する

---

## 変換ルール

日報は部署全員（PD室）配信。内部語彙の出力を防ぐため Phase 5 の Lint で以下を検証・修正する。
Lint は document-reviewer に一任する（Agent ツール経由）。マイルストーン.yml の `project_aliases` を authoritative mapping として渡すことで、codename の正解を担保する。

### 1. codename・内部IDの置換

summary / done_condition / result / 残タスク記述内の内部ID・略語を、**マイルストーン.yml の `project_aliases`** に基づき正式名称に置換する。

辞書にない codename が検出された場合は原文維持で指摘（勝手に文脈推定しない。誤訳のほうが有害）。

**運用原則**: マイルストーン側で最初から正式名称で書く習慣をつける。`project_aliases` は作業中の codename が混ざってしまった場合の救済策であり、恒久的な翻訳レイヤーにはしない。

### 2. 週番号の相対化

`W{N}` は読者に意味が通らないため、基準日（生成対象日）の ISO 週と比較し相対表現に変換する。

- 前週 → 「先週」 / 当週 → 「今週」 / 翌週 → 「来週」
- 範囲外（2週以上離れている）は原文維持

例: `W15完了済み、W16着手` → `先週完了済み、今週着手`

### 3. 時刻・付帯情報の削除

summary / done_condition から時刻情報を削除する。時刻が必要な文脈は YAML の `rationale` に退避する。

- 範囲: `15:00-15:45 面接2次` → `面接2次`
- 所属: `09:30のMTG` → `MTG`

### 4. self-evident な補足の削除

同義反復・自明な補足語を削除する。

- `面接（評価入力）` → `面接`（評価入力は面接の一部であり冗長）
- `コードレビュー（PR確認）` → `コードレビュー`

### 5. summary と done_condition の同義反復回避

done_condition が summary の単純な言い換えにならないようにする。done_condition は「どの状態になれば完了か」を検証可能な粒度で書く。

- NG: `summary: 詳細設計` / `done_condition: 詳細設計を完了する`
- OK: `summary: 詳細設計` / `done_condition: 全エンドポイントのスキーマが定義され、レビュー待ち状態になる`

### 6. 個人内省フレーズの退避

「〜パターン」「自分の〜」など、読者にとって意味が薄い個人内省フレーズは日報本文に出さず、YAML の `rationale` フィールドに退避する。

- NG: `summary: 全部やろうとするパターンを回避しつつ進める`
- OK: `summary: 比較分析の新規登録実装を進める` / `rationale: 全部やろうとするパターンを回避する`

---

## 周期境界チェック時の追加出力

### 月末付近（月次マイルストーン照合）

GoalSetting & Done の後に挿入:

```
■ 月次ふりかえり

- {summary} → {達成見込み / 要調整}
```

### Q初日（四半期照合）

通常の日報の前に挿入:

```
■ 四半期ふりかえり

- {今Q目標のsummary} → {達成 / 未達 / 部分達成}
```

---

## 出力に含めないもの（YAML に残る）

| 情報 | 保存先 |
|---|---|
| Must/Win の内部分類 | days/*.yml の goals[].priority_type |
| 提案理由 | days/*.yml の goals[].rationale |
| 接続チェーン（日次→週次→四半期） | days/*.yml の parent フィールド |
| 放置マイルストーン警告 | Phase 3 の対話中にユーザーに伝える |
| 実績ログ（Calendar, Slack, GitHub） | days/*.yml の activity_log |
| 週次マイルストーン更新の詳細 | Phase 4 の対話中に完結。出力には含めない |

---

## 配信

preprocess → Lint を通過した最終出力を、チャット表示に加えて **Slack の自分宛 DM** に送信する。

- 送信先: config.yml の `slack_user_id` を `channel_id` に指定（自分宛 DM）
- フォーマット: ` ``` ` で囲んだ pure markdown テキスト
- 送信ツール: `mcp__claude_ai_Slack__slack_send_message`

具体的な実行手順（スクリプトのコマンド、Agent プロンプト等）は SKILL.md Phase 5 を参照。
