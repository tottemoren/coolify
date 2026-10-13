# 2. 管理画面（Livewire）

Coolify の画面は、ほぼすべて **Livewire コンポーネント**（[app/Livewire/](../../app/Livewire/)、約 250 個）でできている。

## 2-1. Livewire の仕組み

```
ブラウザ（Blade で作られた HTML + Alpine.js）
   │ ボタンを押す（wire:click / submitAction）
   ▼
サーバーの Livewire コンポーネント（PHP クラス）のメソッドが呼ばれる
   │ プロパティを変更 → HTML を作り直す
   ▼
ブラウザの画面の差分だけが書き換わる
```

- React のように「フロント用のアプリ」と「API」を分けず、**PHP のクラス1つが画面の状態と操作を持つ**
- `mount()`：画面を開いたときに1回だけ動く（Spring の画面表示の Controller に近い）
- それ以外の public メソッド：画面の操作（ボタンなど）で呼ばれる

**Rebuild（React + Spring Boot）との違い**：Rebuild はボタンの処理を React 側に書き、`fetch` で API を呼んでいた。Livewire ではボタンの処理がそのままサーバー側の PHP メソッドになる。

## 2-2. 例：リソースの削除画面（Danger）

[app/Livewire/Project/Shared/Danger.php](../../app/Livewire/Project/Shared/Danger.php) と、画面の [danger.blade.php](../../resources/views/livewire/project/shared/danger.blade.php)。

### ① 画面を開いたとき：ボタンを出すかどうか（UX）

```php
public function mount()
{
    // URL のパラメータから、自分のチームのリソースだけを探す
    $this->resource = Service::ownedByCurrentTeam()->where('uuid', $parameters['service_uuid'])->first();
    ...
    // 削除できる人かどうか（画面の出し分け用）
    $this->canDelete = auth()->user()->can('delete', $this->resource);
}
```

```blade
@if ($canDelete)
    {{-- 削除ボタンを表示 --}}
@endif
```

### ② ボタンが押されたとき：もう一度確かめる（セキュリティ）

```php
public function delete($password, $selectedActions = [])
{
    if (! verifyPasswordConfirmation($password, $this)) {   // パスワードの再入力を確認
        return 'The provided password is incorrect.';
    }
    ...
    $this->authorize('delete', $this->resource);           // 操作の瞬間にも権限を確認
    DeleteResourceJob::dispatch($this->resource, ...)->afterResponse();
    ...
}
```

**学び**
- **画面の出し分け（`$canDelete`）と、操作時の権限確認（`authorize`）を両方やっている**。ボタンを隠すのは使いやすさのため、守りの本体はサーバー側の `authorize`
  - AGENTS.md にも「`@can` などで画面の部品を隠すことに、セキュリティを頼ってはいけない」と明記されている
- 取り消せない操作（削除）は **パスワードの再入力** を求める
- 実際の削除（コンテナ・ボリューム・ネットワークの片付け）は時間がかかるので、**Job にして応答の後で実行**（`afterResponse()`）し、画面はすぐに一覧へ戻す

## 2-3. ブラウザから書き換えられないようにする（`#[Locked]`）

Livewire のコンポーネントの public プロパティは、ブラウザとサーバーの間でやりとりされる。
そのため、**利用者が書き換えてはいけない値**（ID など）には `#[Locked]` 属性を付けて、ブラウザ側から変更できないようにしている（`app/Livewire` 内で 120 か所以上）。

**学び**
- 「画面に持たせている値は、利用者が書き換えられる」という前提は、React でも Livewire でも同じ
- Rebuild で `userId` をリクエストに入れて送っていたのと同じ種類の危険を、フレームワークの機能で防いでいる

## 2-4. リアルタイム更新

- デプロイの進行状況やコンテナの状態は、Laravel Reverb（WebSocket）でサーバーから画面へ届く
- 各コンポーネントは **チーム単位のプライベートチャンネル** を購読するので、他のチームのイベントは届かない

**学び**
- 管理画面は「押したら終わり」ではなく、**時間のかかる処理の進み具合を見せ続ける** 必要がある（デプロイ、バックアップなど）
- その通知の経路にも、チームごとの分離（[04-security.md](04-security.md)）が効いている

## 自分の言葉で追記する
- [ ] （ここに自分の気づきを書く）
