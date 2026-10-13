# 4. セキュリティ

他人のサーバーに SSH でつないでコマンドを実行するアプリなので、**権限の漏れがそのまま他人のサーバーの乗っ取りにつながる**。
セキュリティのルールは [AGENTS.md](../../AGENTS.md) の Authorization の節にも、開発者（と AI）向けに明文化されている。

## 4-1. チームでデータを分ける（マルチテナント）

```php
// app/Models/Application.php
public static function ownedByCurrentTeamAPI(int $teamId)
{
    return Application::whereRelation('environment.project.team', 'id', $teamId)->orderBy('name');
}
```

```php
// app/Http/Controllers/Api/ApplicationsController.php
$application = Application::ownedByCurrentTeamAPI($teamId)
    ->where('uuid', $request->route('uuid'))
    ->first();
if (! $application) {
    return response()->json(['message' => 'Application not found.'], 404);
}
$this->authorize('view', $application);
```

- [app/Models/Application.php](../../app/Models/Application.php) / [ApplicationsController.php](../../app/Http/Controllers/Api/ApplicationsController.php)
- **検索の時点で自分のチームのものに絞る**。他のチームの UUID を指定しても「見つからない（404）」になる
- さらに、見つかった後で `authorize`（Policy）でも確認する。**二重の確認**

**学び**
- Fizzy の `Current.user.accessible_cards.find_by!` と同じ考え方（「後から権限を確認する」のではなく「最初から絞る」）
- Rebuild は `findById(id)` で全員のデータから探していたので、IDOR の穴になっていた

## 4-2. 役割（Role）と Policy

### 役割には順位がある

```php
// app/Enums/Role.php
enum Role: string
{
    case MEMBER = 'member';   // 1
    case ADMIN = 'admin';     // 2
    case OWNER = 'owner';     // 3
}
```

[app/Enums/Role.php](../../app/Enums/Role.php) では `rank()` と `lt()` / `gt()` で「どちらが上か」を比較できる。

### 操作ごとに Policy で判定する

```php
// app/Policies/ApplicationPolicy.php
public function view(User $user, Application $application): bool
{
    // チームのメンバーなら見られる
    return $teamId !== null && $user->teams->contains('id', $teamId);
}

public function delete(User $user, Application $application): bool
{
    // チームの管理者以上でないと削除できない
    return $teamId !== null && $user->isAdminOfTeam($teamId);
}

public function forceDelete(User $user, Application $application): bool
{
    return false;   // 誰にも許さない
}
```

- [app/Policies/ApplicationPolicy.php](../../app/Policies/ApplicationPolicy.php)
- **「見る」はメンバー、「変える・消す・デプロイする」は管理者以上**
- チームが分からない場合は `false`（**迷ったら拒否**）。AGENTS.md にも「ポリシーや所有関係が無い・曖昧なときは、アクセスを拒否するのを基本にする」とある

## 4-3. API トークンは、ミドルウェアで何重にも確かめる

API のルートには、次のミドルウェアが順に掛かっている（[app/Http/Kernel.php](../../app/Http/Kernel.php) で名前を登録）。

```php
'middleware' => ['auth:sanctum', 'api.token.team', ApiAllowed::class, 'api.sensitive'],
// さらにルートごとに 'api.ability:read' など
```

| ミドルウェア | 確かめること |
|-------------|-------------|
| `auth:sanctum` | トークンが有効か（認証） |
| `api.token.team`（[EnsureTokenBelongsToCurrentTeamMember](../../app/Http/Middleware/EnsureTokenBelongsToCurrentTeamMember.php)） | トークンを作った人が、**今も** そのチームのメンバーか |
| `ApiAllowed` | このインスタンスで API が有効になっているか |
| `api.sensitive`（[ApiSensitiveData](../../app/Http/Middleware/ApiSensitiveData.php)） | 機密情報を見てよいか |
| `api.ability`（[ApiAbility](../../app/Http/Middleware/ApiAbility.php)） | このルートに必要な権限を、トークンが持っているか |

### 特に良いと思った点：役割が下がったときにも効く

```php
// app/Http/Middleware/ApiAbility.php
private const MEMBER_DISALLOWED_ABILITIES = ['root', 'write', 'write:sensitive', 'deploy', 'read:sensitive'];

if ($teamId !== null && ! $request->user()->isAdminOfTeam((int) $teamId)) {
    // 管理者でない人のトークンに、書き込み系の権限が付いていたら拒否する
}
```

- 管理者だったときに書き込み権限付きのトークンを作り、その後メンバーに降格された場合でも、**そのトークンで書き込みはできない**
- エラーメッセージで「読み取り専用のトークンを作り直してください」と **次にすべきことを教えている**
- 認証や権限の失敗は `auditLog` に記録している

**学び**
- トークンは「発行したときの状態」ではなく、**使われるたびに今の状態で確かめる** 必要がある
- 1つの仕組みに頼らず、**認証 → チーム所属 → 権限 → 機密情報** と段階的に確認している

## 4-4. 権限のテストを「他のチーム」で書く

[tests/Feature/ActivityMonitorCrossTeamTest.php](../../tests/Feature/ActivityMonitorCrossTeamTest.php)

```php
test('hydrateActivity blocks access to another teams activity via team_id', function () {
    $otherActivity = Activity::create([... 'properties' => ['team_id' => $this->otherTeam->id]]);

    $this->actingAs($this->user);
    session(['currentTeam' => ['id' => $this->team->id]]);

    Livewire::test(ActivityMonitor::class)
        ->call('newMonitorActivity', $otherActivity->id)
        ->assertSet('activity', null);   // 他のチームのものは見えない
});
```

- `tests/Feature/` には `...CrossTeamTest.php`、`AccessControlRegressionTest.php`、`ApiDeploymentPermissionsTest.php` など、**権限専用のテスト** が多数ある
- AGENTS.md では「許可される場合・メンバーの制限・他チームからのアクセス」の3つをテストするよう求めている

**学び**
- 「自分のデータが見える」テストだけでは足りない。**「他人のデータが見えない」テスト** を書くことで、認可の漏れを防ぐ
- Rebuild のテストには、この「他人のデータ」の観点がまだない

## 4-5. 大きな OSS でも、認可の穴は後から見つかる

2026-02-27 のコミット `refactor(auth): enforce team member authorization across app` では、
「プロジェクト管理・サーバー管理・設定のコンポーネントにあった **認可の抜け（authorization gaps）を塞いだ**」と書かれている。

```sh
git show b878dc810 --stat
```

**学び**
- 何年も開発されている人気の OSS でも、認可の漏れは起きる。だからこそ、**ルールの明文化（AGENTS.md）・ポリシー・テスト** で仕組みとして防いでいる
- 自分のコードの穴（Rebuild の認可漏れ）を見つけて直すことは、実務と同じ種類の仕事

## 自分の言葉で追記する
- [ ] （ここに自分の気づきを書く）
