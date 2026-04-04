# 日報出力フォーマット

## 設計原則

- **他メンバーが読む前提**。内部ID（ms-w14-1, d0402-1 等）は出力に含めない
- シンプルに。Must/Win の内部分類は出力に含めない。全てゴール
- 提案理由は YAML に残し、出力には出さない

---

## フォーマット

```
■ GoalSetting & Done

- {summary} → {done_condition} … {結果}
- {summary} → {done_condition} … {結果}
+ 【計画外】{summary}

■ 明日のGoalSetting

- [ ] {summary} → {done_condition}
- [ ] {summary} → {done_condition} [繰越]

■ 今週のGoalSetting

- {週次マイルストーンのsummary}
- {週次マイルストーンのsummary}

------------------------------------------
■ 今期ストレッチ / Q目標 / 目指す姿 / 輝かしい未来
{今期ストレッチのキーワード} / {今Q目標のキーワード} / {目指す姿のキーワード} / {輝かしい未来のキーワード}
```

---

## 表示ルール

### GoalSetting & Done

- 前営業日の goals を1行ずつ表示
- 結果の表記: 「達成」「一部達成: {remaining}」「未達: {reason}」
- 計画外の作業は `+` プレフィックスで末尾に追加
- 初回実行（前営業日のゴールなし）の場合、このセクション自体をスキップ

### 明日のGoalSetting

- チェックボックス付きで表示
- 繰越ゴールには `[繰越]` を付与
- Must/Win の区別は出力しない。全てフラットに並べる
- 並び順は内部的に Must → Win の優先度順

### 今週のGoalSetting

- マイルストーン.yml の今週 weekly マイルストーンの summary を箇条書き
- ID、status、進捗インジケータは表示しない

### North Star 1行サマリ

- horizons/ 配下の .md と .yml からキーワードを抽出し、スラッシュ区切りで1行にまとめる
- 毎日同じ内容が出る（方向性の定着が目的）

---

## 周期境界チェック時の追加出力

### 月曜（週次マイルストーン更新）

GoalSetting & Done の後に挿入:

```
■ 週次マイルストーン更新

先週:
- {summary} → {達成 / 未達}

今週:
- {新規マイルストーンのsummary}
```

### 月末付近（月次マイルストーン照合）

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
