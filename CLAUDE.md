# your-scrum-routine

あなた専属のスクラムルーティン。
目標設定から日々のタスク管理まで、一人スクラムのセレモニーを伴走する。

全体設計は [docs/cycle-architecture.md](docs/cycle-architecture.md) を参照。

## データストア

    ~/.claude/your-scrum-routine/
    ├── config.yml                 ユーザー設定（slack_user_id, github_username, timezone）
    ├── product-vision/            目標階層（Q目標、マイルストーン、North Star）
    ├── sprint-log/                日次記録（計画・実績・照合）
    └── increments/                セッションサマリ（session-closer が書き込む）

スキルがデータストアを参照する際は `~/.claude/your-scrum-routine/` をルートとする。

## peerDependencies

| デッキ | 用途 | 必須 |
|---|---|---|
| development-deck | 深掘り・設計相談（dp, design-partner 経由） | No |

development-deck が未インストールの場合、フォワーディング先のスキルは利用不可。
各スキルは development-deck なしでもコア機能が動作すること。

## 設計哲学

### 痛み駆動

スキルは具体的な痛みを解決するために存在する。セレモニーのカタログ移植はしない。
一人スクラムに「報告」は不要。価値は「判断精度の向上」にしかない。
痛みの無い改善に価値はない。

### PDCA フラクタル

Scrum は PDCA を回すためのフレームワークであり、全て P から始まる。
同じ C→A→P 構造が Q → 月 → 週 → 日 → 着手のスケールで相似形を成す。
各スキルはフラクタルの特定スケール・特定フェーズを担当する。

### スキルの境界 = タイミング × 責務

スキルはセレモニー（ルーティン）を担う。タスクの種類（凡事/プロジェクト/個人依頼）で分けない。
Scrum の Sprint Backlog にストーリーもバグもタスクも区別なく載るのと同じ。

### sprint-log が共通インターフェース

git の `.git/` と同様、sprint-log YAML が全スキルの共通データモデルである。
一つのスキルの出力が別のスキルの入力になる設計にすること。
スキルを増やしてもデータストアのスキーマが膨張しないこと。

## スキルの分類

| 型 | 定義 |
|---|---|
| ツール型 | セレモニーを遂行し、成果物（YAML）を出す |

## スキル設計原則

- データストアのスキーマは skills/retrospective/references/schema.md で一元管理する
  （複数スキルがスキーマを共有する段階で shared/schema.md に移動する）
- development-deck へのフォワーディングは任意機能。コア機能をブロックしない
- 各スキルに「自己評価」セクションを付与する
- product-vision/ 配下の .md ファイルはスキルから書き換えない（読み取り専用）
