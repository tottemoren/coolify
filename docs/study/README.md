# 学習ノート — Coolify コードリーディング

coolLabs のセルフホスト PaaS「Coolify」を題材に、**管理画面と API からサーバー・アプリを操作する実務コード** を観点ごとに読み解いたノートです。
各ノートには **該当コードへのリンク** と **自分の考え** を記録しています。

> `docs/` 直下の `cloud-horizon-workers.md` などはフォーク元の公式ドキュメントです。
> 自分の学習ノートはすべてこの `docs/study/` にまとめています。

## 目次

| # | ノート | 内容 |
|---|--------|------|
| 0 | [アプリの全体像](00-app-overview.md) | Coolify とは、チーム → プロジェクト → 環境 → リソースのデータ構造、規模感 |
| 1 | [設計](01-architecture.md) | Livewire / Action / Job / Service の役割分担、管理画面と API が同じ Action を呼ぶ構造 |
| 2 | [管理画面](02-control-panel.md) | Livewire の仕組み、画面の出し分けと操作時の権限確認、`#[Locked]`、リアルタイム更新 |
| 3 | [API 設計](03-api-design.md) | `/api/v1` と UUID、公開 API を壊さない工夫、トークンの権限、機密情報の出し分け、OpenAPI |
| 4 | [セキュリティ](04-security.md) | チームでのデータ分離、Role と Policy、何重ものトークン確認、他チームを使った権限テスト |
| 5 | [リファクタリング](05-refactoring.md) | Conventional Commits、数値 ID から UUID への互換性を保った移行、権限の仕組みの成長 |

## 読み進め方
1. [routes/api.php](../../routes/api.php) と [routes/web.php](../../routes/web.php) から機能の全体像を把握する
2. 1つの操作（例：アプリを停止する）を「ルート → Livewire / API コントローラー → Policy → Action / Job → テスト」と縦に追う
3. [AGENTS.md](../../AGENTS.md) で、開発チームが決めている設計とセキュリティのルールを確認する
4. `git log` / `git show` で設計の変遷を確認する
5. 学んだことを自分の言葉でノートにまとめる

## 環境
- フォーク時点のコミット：`86d84e21c`
- ローカルでの起動は未実施（コードリーディングのみ）
