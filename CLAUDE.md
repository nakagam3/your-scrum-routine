# your-scrum-routine

あなた専属のスクラムルーティン。
目標設定から日々のタスク管理まで、一人スクラムのセレモニーを伴走する。

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

## スキルの分類

| 型 | 定義 |
|---|---|
| ツール型 | セレモニーを遂行し、成果物（YAML）を出す |

## スキル設計原則

- データストアのスキーマは skills/retrospective/references/schema.md で一元管理する
  （将来スキルが増えてスキーマが共有されるなら shared/ に移動）
- development-deck へのフォワーディングは任意機能。コア機能をブロックしない
- 各スキルに「自己評価」セクションを付与する
- product-vision/ 配下の .md ファイルはスキルから書き換えない（読み取り専用）
