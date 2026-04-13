# データスキーマ定義

## ストレージ構成

```
~/.claude/your-scrum-routine/
├── config.yml                     # ユーザー設定
├── product-vision/
│   ├── 中期経営計画.md          # Markdown（5年周期）
│   ├── 輝かしい未来.md          # Markdown（3年周期）
│   ├── PD室年間テーマ.md        # Markdown（1年周期）
│   ├── 目指す姿.md              # Markdown（半年周期）
│   ├── 今Q目標.yml              # YAML（3ヶ月周期）
│   └── マイルストーン.yml        # YAML（週次+月次）
├── sprint-log/
│   └── YYYY-MM-DD.yml           # YAML（日次）
└── increments/
    └── YYYY-MM-DD/
        └── {session_id}.md      # Markdown（セッション単位。session-closer が書き込む）
```

## Markdown層（North Star: 中期経営計画〜目指す姿）

スキルは**読み取り専用**。日報末尾への表示と、Q初日の接続確認に使う。

```markdown
---
horizon: 3年
updated_at: 2026-04-01
---

# 輝かしい未来

ストリームアラインドのイネイブラー。
プロダクト開発チームが自律的に成果を出せる土台を設計し、
仕組みとして機能させる人間になる。

## なぜこの方向か

- 技術で直接価値を出すフェーズから、構造で価値を出すフェーズへ
- 個人のアウトプットではなくチームのスループットを最大化する
```

### frontmatter

| フィールド | 型 | 必須 | 説明 |
|---|---|---|---|
| horizon | string | Yes | 周期ラベル（「5年」「3年」「1年」「半年」） |
| updated_at | date | Yes | 最終更新日 |

本文は自由記述。見出し・箇条書き・散文いずれも可。

---

## 今Q目標.yml

```yaml
period: "2026-Q2"
updated_at: "2026-04-04"

context:
  unit: "Engagement Product Unit > 3G"
  objective: "新たな価値創出 > 実用的な診断体験の提供"
  role: "MCEのハーネス"
  vision: "ストリームアラインドのイネイブラー"
  org_theme: "Rising Spiral ~PDCAを回し、螺旋状に上昇する~"
  pd_theme: "差分を創造する一年"
  q_theme: "たくさん試そう（試行）"

goals:
  # --- roof/moon が単一文字列のパターン ---
  - id: 2026q2-perf-pickup
    summary: "ピックアップ分析"
    type: performance
    calendar_color_id: "7"
    roof: "リリース完了(64問版)"
    moon: "リリース完了(32問版/英語)"

  # --- roof/moon がリストのパターン ---
  - id: 2026q2-perf-ds
    summary: "AI-Nativeなデザインシステムの構築"
    type: performance
    calendar_color_id: "5"
    roof:
      - "[デザイン] re:noコンポーネント全53個の設計完了"
      - "[コード] re:noコンポーネント全53個の実装完了"
    moon:
      - "[デザイン] re:noコンポーネント全53個のFigmaコンポーネント作成完了"
      - "[コード] re:noデザインパターンのAI-Native化を1件実戦で成功"

  # --- criteria ベースのパターン（roof/moon なし） ---
  - id: 2026q2-perf-mentee
    summary: "弟子の目標達成"
    type: performance
    criteria:
      "4": "弟子の評価が9である"
      "5": "弟子の評価が10である"
      "6": "弟子の評価が12であり、周囲からも変化感があったと評価されていること"

  # --- ストレッチ目標 ---
  - id: 2026q2-stretch
    summary: "AXコンサルタント"
    type: stretch
    status: in_progress    # in_progress | confirmed
    prior_q_reflection: |
      前Q「成果を成立させる設計者」で計画力にフォーカスしたが未達(評価4)。
      判断基準を立てること・意図して行動することが課題として残った。
    skills_to_acquire: null
    action_plan: null
    criteria: null
```

### フィールド定義

**トップレベル**

| フィールド | 型 | 必須 | 説明 |
|---|---|---|---|
| period | string | Yes | 四半期ラベル（"YYYY-QN"） |
| updated_at | date | Yes | 最終更新日 |
| context | object | No | 組織目標・役割・テーマ等の背景情報 |

**context**

| フィールド | 型 | 必須 | 説明 |
|---|---|---|---|
| unit | string | No | 所属組織 |
| objective | string | No | 組織目標 |
| role | string | No | 担う役割 |
| vision | string | No | 半年後に目指す姿 |
| org_theme | string | No | 年間テーマ |
| pd_theme | string | No | PD室テーマ |
| q_theme | string | No | 四半期テーマ |

**goals[]**

| フィールド | 型 | 必須 | 説明 |
|---|---|---|---|
| id | string | Yes | 一意ID。"{year}q{N}-perf-{name}" or "{year}q{N}-stretch"。年を含めて跨Q衝突を防ぐ |
| summary | string | Yes | ゴールの要約（プロジェクト名レベル。roof/moon の詳細は別フィールド） |
| type | enum | Yes | performance \| stretch |
| calendar_color_id | string | No | Google Calendar の colorId。プロジェクト単位で色を振る |
| roof | string \| string[] | No | 確実に達成する水準。文字列 or リスト |
| moon | string \| string[] | No | ストレッチ達成水準。roof の先にある到達点 |
| criteria | map | No | 評価基準。キーは評価点("4","5","6")、値は基準の説明 |
| note | string | No | 補足情報（進め方の方針等） |
| prior_q_reflection | string | No | 前Qからの引き継ぎ課題（stretch で主に使用） |
| skills_to_acquire | string \| null | No | 習得したい能力（stretch 用） |
| action_plan | string \| null | No | アクションプラン（stretch 用） |
| status | enum | No | in_progress \| confirmed。ファクト確定ルールで使用。日報の North Star 行で confirmed のみ summary を表示し、それ以外は「設定中」に置換する。stretch から導入、必要に応じて performance にも広げる |

### roof / moon の設計意図

roof と moon は同一ゴール内の達成段階であり、別ゴールではない。

```
1ゴール = 1プロジェクト
  ├── roof: ここまでは確実に（OKR の Target 相当）
  └── moon: ここまで行けたら（OKR の Stretch 相当）
```

- calendar_color_id はゴール（=プロジェクト）単位で1つ振る
- Phase 4 の Q 振り返り時、roof 達成 / moon 達成を個別に判定する
- roof のみのゴール（moon なし）も許容する

---

## マイルストーン.yml

月次マイルストーン（type: monthly）と週次マイルストーン（type: weekly）を1ファイルに同居させる。
週次は月次の子としてぶら下がる。

```yaml
quarter: "2026-Q2"
updated_at: "2026-04-04"

# プロジェクト名の辞書置換（日報出力時の Lint で使用）
project_aliases:
  {内部ID or 略語}: "{読者向けの正式名称}"

milestones:
  # --- 月次マイルストーン（KR単位で分離） ---
  - id: ms-apr-pickup
    summary: "詳細設計完了 + 実装着手支援"
    deadline: "2026-04-30"
    parent: "今Q目標/2026q2-perf-pickup"
    type: monthly
    status: in_progress    # not_started | in_progress | achieved | missed
    done_condition: "若手が迷わず実装開始できる状態"

  - id: ms-apr-ds
    summary: "計測開始 + ハーネスv0"
    deadline: "2026-04-30"
    parent: "今Q目標/2026q2-perf-ds"
    type: monthly
    status: in_progress
    done_condition: "計測が回っている。ハーネスv0が動作する状態"

  # --- 週次マイルストーン（"ms-{kr}-w{week}" 命名） ---
  - id: ms-pickup-w15
    summary: "比較分析: 詳細設計完了、モブQA設計、カスタム権限認識合わせ"
    deadline: "2026-04-11"
    parent: ms-apr-pickup
    type: weekly
    status: in_progress

  - id: ms-pickup-w16
    summary: "比較分析: 新規登録、カスタム権限設定、ベロシティ再見積もり"
    deadline: "2026-04-18"
    parent: ms-apr-pickup
    type: weekly
    status: not_started

changelog:
  - date: "2026-04-04"
    description: "今Q目標のID体系変更(2026q2-*)に追従。月次マイルストーンをKR単位に分離"
```

### フィールド定義

| フィールド | 型 | 必須 | 説明 |
|---|---|---|---|
| quarter | string | Yes | 四半期ラベル |
| updated_at | date | Yes | 最終更新日 |
| project_aliases | map | No | codename → 正式名称の authoritative mapping。日報 Lint 時、document-reviewer に渡して読者向け置換を行う。キーは内部語彙、値は正式名称。**原則**: マイルストーンの summary/done_condition は最初から正式名称で書く。`project_aliases` は codename が混入した場合の救済であり、恒久的な翻訳レイヤーではない |
| milestones[].id | string | Yes | 一意ID。月次: "ms-{month}-{kr}", 週次: "ms-{kr}-w{week}" |
| milestones[].summary | string | Yes | マイルストーンの要約 |
| milestones[].deadline | date | Yes | 期限 |
| milestones[].parent | string | Yes | 親への参照。月次→"今Q目標/{id}"（例: "今Q目標/2026q2-perf-pickup"）, 週次→月次id |
| milestones[].type | enum | Yes | monthly \| weekly |
| milestones[].status | enum | Yes | not_started \| in_progress \| achieved \| missed |
| milestones[].done_condition | string | No | 達成条件の記述。月次マイルストーンで主に使用 |
| changelog[] | array | No | 変更履歴。date + description |

---

## sprint-log/YYYY-MM-DD.yml

```yaml
date: "2026-04-06"

# === Plan（朝 or 前日夕方に設定） ===
goals:
  - id: d0406-1
    summary: "ピックアップ分析: マイルストーン・チケット認識合わせ"
    done_condition: "エピック構造とマイルストーンが全員で合意される"
    parent: ms-pickup-w15
    priority_type: must        # must | win（内部分類。出力には含めない）
    rationale: "09:30のMTG。金曜にマイルストーン策定済みで叩き台あり。プランニングの前提条件"
    status: null
    result: null
  - id: d0406-2
    summary: "survey_selections rename フォロー"
    done_condition: "篠原・中崎の回答を確認し、合意 or 調整方針が決まる"
    parent: ms-pickup-w15
    priority_type: win
    rationale: "金曜にSlack投稿済み。週明けにフォローしないと流れる"
    status: null
    result: null

carryover: []

# === Check（夕方に埋まる） ===
reconciliation:
  achieved: []       # goal id のリスト
  partial: []        # { id, progress, remaining }
  missed: []         # { id, reason }
  unexpected: []     # { summary, time_spent, parent_if_any }

alignment_check:
  orphan_goals: []       # parent がない日次ゴール
  stale_milestones: []   # 今週まだ日次ゴールが紐づかない週次マイルストーン

# === 実績ログ（自動収集） ===
activity_log:
  calendar: []       # MTG等のサマリ
  slack: []          # 投稿のサマリ
  github: []         # PR/レビュー等のサマリ

```

### フィールド定義

| フィールド | 型 | 必須 | 説明 |
|---|---|---|---|
| date | date | Yes | 日付 |
| goals[].id | string | Yes | 一意ID（"dMMDD-N" 形式） |
| goals[].summary | string | Yes | ゴールの要約 |
| goals[].done_condition | string | Yes | 完了条件。「作業継続」は不可 |
| goals[].parent | string | Yes | 週次マイルストーンIDへの参照 |
| goals[].priority_type | enum | Yes | must \| win（内部分類。出力には含めない） |
| goals[].rationale | string | Yes | このゴールが明日のゴールとして効く理由 |
| goals[].status | enum | Null可 | null → achieved \| partial \| missed \| unexpected |
| goals[].result | string | Null可 | 実績の簡潔な記述 |
| carryover[].from_date | date | Yes | 繰越元の日付 |
| carryover[].original_id | string | Yes | 繰越元のゴールID |
| carryover[].reason | string | Yes | 繰越理由 |
| reconciliation | object | No | 夕方の照合結果 |
| alignment_check | object | No | 接続検証結果 |
| activity_log | object | No | MCP経由で自動収集した実績データ |

### status の判定基準

| status | 判定基準 |
|---|---|
| achieved | done_condition を満たした |
| partial | 着手したが done_condition 未達。progress と remaining を記録 |
| missed | 未着手、または着手したが成果なし。reason を記録 |
| unexpected | 計画外だが実施した作業。parent があれば記録 |

---

## increments/YYYY-MM-DD/{session_id}.md

session-closer が書き込むセッションサマリ。retrospective は Phase 1 で当日分を Glob (`increments/{date}/*.md`) で取得し、予実照合の参考データとして使用する。

ファイルフォーマットは session-closer の出力仕様に従う（6カテゴリ構造）。retrospective が参照するのは主に以下:

- **成果・到達点**: Phase 2 の予実照合で、ゴールの達成判定の根拠として使用
- **未解決・Next Actions**: Phase 3 の明日のGoal提案で、繰越・新規ゴール候補として使用
- **コンテキスト**: Phase 1 の activity_log に「セッション作業」として記録
