# 3. API 設計

Coolify は管理画面とは別に、**外部から操作するための REST API**（`/api/v1/...`）を公開している。
ルートは [routes/api.php](../../routes/api.php)、コントローラーは [app/Http/Controllers/Api/](../../app/Http/Controllers/Api/)。

## 3-1. URL の設計

```
GET    /api/v1/applications                 一覧
GET    /api/v1/applications/{uuid}          1件取得
PATCH  /api/v1/applications/{uuid}          更新
DELETE /api/v1/applications/{uuid}          削除
POST   /api/v1/applications/{uuid}/start    起動（操作）
POST   /api/v1/deploy                       デプロイ
```

- **`v1` というバージョンを URL に入れている**：将来、互換性のない変更が必要になったら `v2` を作れる
- **ID ではなく UUID を使う**：`/applications/1` のような連番ではないので、他のリソースの ID を推測されにくい
  - レスポンスからも数値の `id` は隠している（`removeSensitiveData()` の `makeHidden(['id', ...])`）
- 「起動」「停止」のような **CRUD に当てはまらない操作** は、`/start` のようにリソースの下の動詞で表している

## 3-2. 公開した API を壊さない工夫

### GET で受け付けていた操作を、POST に変えた

```php
// routes/api.php
Route::get('/deploy', [OtherController::class, 'post_required'])->middleware(['api.ability:deploy']);
Route::post('/deploy', [DeployController::class, 'deploy'])->middleware(['api.ability:deploy']);

// app/Http/Controllers/Api/OtherController.php
public function post_required(): JsonResponse
{
    return response()
        ->json(['message' => 'This endpoint has changed to a POST request.'], 405)
        ->header('Allow', 'POST');
}
```

- デプロイ・起動・停止のような **状態を変える操作は、本来 GET で受けるべきではない**（GET は「取得するだけ」の意味。リンクのプリフェッチなどで勝手に実行される危険もある）
- ただし、すでに GET で呼んでいる利用者がいるので、**古い呼び方を消すのではなく、「POST に変わりました」と教える `405` を返す** ようにしている
  - `405 Method Not Allowed` と `Allow: POST` ヘッダーは、HTTP の意味そのまま

### 古い値を「互換のための別名」として残す
ログ取得 API の `lines` パラメータは、新しく `all` という値を受け付けるようになったが、古い `-1` も「compatibility alias（互換のための別名）」として残している（OpenAPI の説明文に明記）。

**学び**
- **API は公開した瞬間に「約束」になる**。CLI・Terraform・利用者のスクリプトが依存しているので、黙って変えると壊れる
- 変えるときは「古い呼び方をしばらく残す」「何をすればいいかをエラーで教える」

## 3-3. トークンの権限（ability）

API はトークンで認証する（Laravel Sanctum）。トークンには **できることの範囲（ability）** を付けて発行する。

| ability | できること |
|---------|-----------|
| `read` | 読み取り |
| `write` | 作成・更新・削除 |
| `deploy` | デプロイ・起動・停止 |
| `read:sensitive` | パスワードなど機密情報も読める |
| `root` | すべて |

ルートごとに必要な ability をミドルウェアで指定している：

```php
Route::get('/applications/{uuid}', [...])->middleware(['api.ability:read']);
Route::patch('/applications/{uuid}', [...])->middleware(['api.ability:write']);
```

**学び**
- 「読むだけの監視ツール用」「デプロイだけする CI 用」のように、**用途に合わせて最小限の権限のトークンを作れる**（最小権限の原則）
- 権限の確認はルート定義で一覧でき、書き忘れを見つけやすい

## 3-4. 機密情報を返し分ける

[ApiSensitiveData](../../app/Http/Middleware/ApiSensitiveData.php) ミドルウェアが「機密情報を見てよいか」を判定し、コントローラーの `removeSensitiveData()` がそれに従って項目を出し分ける。

- 機密情報を見られるのは「トークンに `read:sensitive` か `root` がある」**かつ**「その人がチームの管理者以上」のときだけ
- それ以外は、Webhook の秘密鍵や環境変数の値などをレスポンスから隠す

**学び**
- **トークンの権限と、人の役割の両方** を確認している（どちらか片方だけでは足りない）
- Rebuild の `@JsonIgnore` は「パスワードは常に隠す」だったが、Coolify は「誰が見るかによって出し分ける」

## 3-5. ドキュメントをコードと一緒に書く（OpenAPI）

各メソッドの上に、OpenAPI の注釈（`#[OA\Get(...)]`）で、パス・パラメータ・レスポンス・ステータスコードを書いている。
ここから [openapi.yaml](../../openapi.yaml) / [openapi.json](../../openapi.json) が生成され、公式の API ドキュメントになる。

```php
#[OA\Get(
    path: '/applications/{uuid}',
    responses: [
        new OA\Response(response: 200, ...),
        new OA\Response(response: 401, ref: '#/components/responses/401'),
        new OA\Response(response: 404, ref: '#/components/responses/404'),
    ]
)]
public function application_by_uuid(Request $request)
```

**学び**
- **ドキュメントがコードの真上にある** ので、コードを変えたときにドキュメントの更新を忘れにくい
- 一方で、コントローラーが非常に長くなる原因にもなっている（[01-architecture.md](01-architecture.md)）

## 3-6. 操作の記録（監査ログ）

アプリの作成や認証の失敗などを `auditLog(...)` で記録している。

```php
auditLog('api.application.created', [
    'team_id' => $teamId,
    'application_uuid' => data_get($application, 'uuid'),
    ...
]);
```

**学び**
- 外部から操作される API では、「**いつ、誰が、何をしたか**」を後から追えることが重要（障害調査、不正利用の調査）

## 自分の言葉で追記する
- [ ] （ここに自分の気づきを書く）
