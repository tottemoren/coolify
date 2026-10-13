# 0. アプリの全体像

## Coolify とは
自分のサーバーに、アプリ・データベース・各種サービスをデプロイして管理できる、オープンソースのセルフホスト型 PaaS。
Heroku・Vercel・Netlify のような「デプロイと運用の管理画面」を、自分のサーバー上で動かせる。
開発元は coolLabs で、[coolify.io](https://coolify.io) ではクラウド版も提供している。ライセンスは Apache 2.0。

**学習の題材に選んだ理由**：サーバーやアプリを **管理画面と API の両方から操作する** 構造が、クラウドサービスのコントロールパネルに近いため。

## データの構造

```
チーム（Team）… マルチテナントの単位。ユーザーは複数のチームに所属できる
 ├ サーバー（Server）… SSH で接続する実際のマシン
 │   └ デスティネーション（StandaloneDocker / SwarmDocker）… そのサーバー上の Docker 環境
 └ プロジェクト（Project）
     └ 環境（Environment）（例：production / staging）
         └ リソース
             ├ アプリケーション（Application）… Git リポジトリなどからビルドしてデプロイ
             ├ サービス（Service）… Docker Compose で構成されたワンクリックのサービス
             └ データベース（StandalonePostgresql / StandaloneMysql / StandaloneRedis …）
```

- リソースは「どのチームのものか」を **環境 → プロジェクト → チーム** とたどって判断する
  - 例：[app/Models/Application.php](../../app/Models/Application.php) の `ownedByCurrentTeam()` は `environment.project.team` で絞り込む
- リソースは「どこで動くか」をデスティネーション（どのサーバーのどの Docker か）で持つ

## 技術スタック（[TECH_STACK.md](../../TECH_STACK.md)）

| 層 | 使っているもの |
|----|---------------|
| 画面 | Livewire 3 + Alpine.js + Blade + Tailwind CSS |
| バックエンド | Laravel（PHP 8.4） |
| DB / キャッシュ | PostgreSQL / Redis |
| リアルタイム更新 | Laravel Reverb（WebSocket） |
| 非同期処理 | Redis キュー + Horizon |
| テスト | Pest（PHPUnit ベース） |

## 規模感（フォーク時点）

| 項目 | 数 |
|------|----|
| コミット数 | 約 17,700（2021-03 の初回リリースから） |
| `app/` 配下の PHP ファイル | 約 1,000（うち Livewire コンポーネント約 250） |
| API のルート | 約 300（[routes/api.php](../../routes/api.php)） |
| テストファイル | 約 1,000 |

**Fizzy との違い**：Fizzy は小さく一貫したアプリだったが、Coolify は **機能が非常に多い中〜大規模のアプリ**。
全体を読むのではなく、1つの操作（例：アプリを停止する、リソースを削除する）を縦に追う読み方が必要になる。
