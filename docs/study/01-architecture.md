# 1. 設計

Coolify は Laravel の上に、**画面は Livewire、処理は Action / Job / Service** という役割分担で組まれている。
構成の説明は開発者向けの [AGENTS.md](../../AGENTS.md)（Architecture の節）にまとまっている。

## 1-1. `app/` の主な役割

| フォルダ | 役割 | Spring Boot でいうと |
|---------|------|-------------------|
| `Livewire/` | **画面（UI）のすべて**。従来の Blade 用コントローラーはほぼない | Controller + 画面 |
| `Http/Controllers/Api/` | REST API | `@RestController` |
| `Actions/` | 1つの目的だけを持つ処理（停止する、起動するなど） | Service の小さな1メソッド |
| `Jobs/` | キューで非同期に動く重い処理（デプロイ、バックアップなど） | `@Async` / バッチ |
| `Services/` | 複数の処理を組み合わせる複雑な処理 | Service |
| `Models/` | Eloquent モデル（Active Record） | Entity + Repository |
| `Policies/` | 「この人はこの操作をしてよいか」の判定 | Spring Security の認可ルール |

AGENTS.md には使い分けの方針も書かれている：
**「複雑な組み立てには Services、単一目的のドメイン操作には Actions を使う」**

## 1-2. 管理画面と API が、同じ Action を呼ぶ

「アプリを停止する」処理は [app/Actions/Application/StopApplication.php](../../app/Actions/Application/StopApplication.php) に1つだけ書かれていて、複数の入口から呼ばれている。

```php
// 管理画面（Livewire）：app/Livewire/Project/Application/Heading.php
StopApplication::dispatch($this->application, false, $this->docker_cleanup);

// REST API：app/Http/Controllers/Api/ApplicationsController.php
StopApplication::dispatch($application, false, $dockerCleanup);
```

- 他にも MCP（AI 連携）のツールや、別の Action・Job からも呼ばれている
- `AsAction` トレイト（`lorisleiva/laravel-actions`）により、同じクラスを **その場で実行・キューに入れて非同期実行** のどちらでも使える
- `dispatch` はキューに入れるので、画面や API はすぐに応答を返せる

**学び**
- 入口（画面・API・AI）が増えても、**処理の本体は1か所**。修正が1回で全入口に効く
- クラウドのコントロールパネルも「画面」と「API」の2つの入口を持つので、この構造はそのまま参考になる
- 自作アプリ Rebuild で、ファイルアップロード処理が2か所に重複していた問題の、解決の形の1つ

## 1-3. 大きすぎるファイルもある

| ファイル | 行数（フォーク時点） |
|---------|-------------------|
| [ApplicationsController.php](../../app/Http/Controllers/Api/ApplicationsController.php) | 約 6,400 行 |
| [ApplicationDeploymentJob.php](../../app/Jobs/ApplicationDeploymentJob.php) | 約 5,900 行 |
| [DatabasesController.php](../../app/Http/Controllers/Api/DatabasesController.php) | 約 5,200 行 |

- API コントローラーが大きい理由の多くは、メソッドごとに付いている **OpenAPI の注釈**（[03-api-design.md](03-api-design.md)）と、メソッドごとに書かれた入力検証
- デプロイの Job は、ビルドパックの種類（Dockerfile、Nixpacks、Docker Compose など）ごとの処理が1つのクラスに集まっている

**学び**
- 実務の人気 OSS でも、**機能追加を重ねるとファイルは大きくなる**。Fizzy の「1ファイル 50 行程度」とは対照的
- 大きいファイルは全体を読まず、`grep` で目的のメソッドを探し、そこから縦に追う
- 「Action に小さく分ける」方針と「巨大な Job」が同じリポジトリに共存している。**設計方針は、作られた時期や部分によってばらつく**

## 1-4. Fizzy・Rebuild との比較

| 観点 | Fizzy（Rails） | Coolify（Laravel） | Rebuild（Spring Boot） |
|------|---------------|-------------------|----------------------|
| 処理の置き場所 | Model と concern | Action / Job / Service | Service（一部は Controller に直接） |
| 画面 | サーバーで HTML（Hotwire） | サーバーで HTML（Livewire） | React（別アプリ） |
| API | ほぼ画面用 | 画面とは別に公開 REST API | 画面用の API のみ |
| ファイルの大きさ | 小さくそろっている | 小さい Action と巨大な Controller / Job が混在 | 小さい |

## 自分の言葉で追記する
- [ ] （ここに自分の気づきを書く）
