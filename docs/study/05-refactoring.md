# 5. リファクタリング

## git 履歴という教材
- 最初のコミットは 2021-03-24 の「initial production release」
- コミット数は約 17,700。5年以上、設計の変更が積み重なっている

## コミットメッセージの規則（Conventional Commits）

```
fix(ui): deny members on cloud tokens and OAuth settings without a 500
perf(databases): load Redis-family passwords once for database lists
refactor(docker): identify containers by resource UUID instead of numeric id
```

`種類(範囲): 内容` の形で書かれている。2026-06 以降のコミットの種類を数えると：

| 種類 | 意味 | 件数（2026-06 以降） |
|------|------|-------------------|
| `fix` | バグ修正 | 約 890 |
| `feat` | 機能追加 | 約 390 |
| `chore` | 雑務（依存更新など） | 約 120 |
| `test` | テスト | 約 26 |
| `refactor` | リファクタリング | 約 26 |

```sh
git log --since=2026-06-01 --format='%s' | sed -E 's/^([a-z]+)(\(.*\))?!?:.*/\1/' | sort | uniq -c | sort -rn
```

**学び**
- 種類と範囲が先頭にあるので、**履歴を見ただけで何の変更か分かる**。リリースノートの自動生成にも使える（[cliff.toml](../../cliff.toml)）
- リファクタリングは「まとめて大きくやる」のではなく、修正や機能追加の合間に **小さく、目的を1つに絞って** 行われている

## 発見したこと

### 数値の ID から UUID へ（互換性を保った変更）

コミット `af983a29d refactor(docker): identify containers by resource UUID instead of numeric id`

```sh
git show af983a29d
```

コミットの説明を要約すると：
- コンテナに付けるラベルを、数値の ID から UUID に変えた
- 理由：数値の ID は Coolify のインスタンスごとに違うため、**サーバーを別のインスタンスに移すと、既存のコンテナを見つけられなかった**
- 探す順番を「UUID のラベル → Compose のプロジェクト名 → 古い数値 ID のラベル」にして、**既存のコンテナは作り直さなくても動き続け**、次のデプロイで新しいラベルが付く

**学び**
- 「なぜ変えるのか」「既存のものはどうなるのか」がコミットの本文に書かれている。**良いコミットメッセージの見本**
- 大きな変更でも、**古い形と新しい形を一時的に両方受け付ける** ことで、利用者の環境を壊さずに移行できる（[03-api-design.md](03-api-design.md) の API の互換性と同じ考え方）
- 数値の ID より UUID の方が、環境をまたいでも一意で、推測もされにくい

### 1つのファイルの履歴から、設計の成長を見る

```sh
git log --oneline --follow -- app/Http/Middleware/ApiAbility.php
```

[ApiAbility.php](../../app/Http/Middleware/ApiAbility.php) の履歴を古い順に並べると、権限の仕組みが段階的に強くなっている：

1. 権限（ability）ごとのミドルウェアを追加
2. `root`（すべての権限）を導入
3. チームメンバーの権限を、アプリ全体で強制（`refactor(auth): ...`）
4. メンバーには機密情報を見せない（`feat(auth): ...`）
5. 認証・権限の失敗を監査ログに記録（`feat(observability): ...`）

途中には `rector: arrrrr` というコミットと、その `Revert`（取り消し）もある。自動リファクタリングツール（Rector）の変更を一度入れて、取り消したと思われる。

**学び**
- セキュリティは最初から完成していたわけではなく、**必要になるたびに少しずつ足されている**
- うまくいかなかった変更は `git revert` で、履歴を残したまま取り消せる

## 次に追ってみたいもの
- [ ] `ApplicationDeploymentJob.php`（約 5,900 行）が、どのように大きくなってきたか
- [ ] 「アプリをデプロイする」を、API の `POST /deploy` から Job の完了まで縦に追う
- [ ] （自分の言葉で追記する）
