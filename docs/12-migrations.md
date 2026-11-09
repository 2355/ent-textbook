# 12. マイグレーション

> **この章でわかること**
> ent のスキーマを実際の DB に反映する 2 つの方法。自動マイグレーションとバージョン管理マイグレーションの違い。Atlas の役割。題材アプリで「スキーマを変えてから本番に反映されるまで」の全行程。型を変えるマイグレーションの注意点。

---

## 12.1 スキーマと DB はずれる

ここまで、スキーマ（`ent/schema`）を書くと Go のコードが生成される、という話をしてきました。

しかし生成されるのは**Go のコード**だけです。MySQL のテーブルは勝手には変わりません。

- スキーマ: 「`users` には `nickname` 列があるべき」
- DB: まだ `nickname` 列が無い

このずれを埋めるのが**マイグレーション**です。

Prisma を使ったことがあれば、次の 2 つの違いを思い出してください。

| Prisma | やること |
| --- | --- |
| `prisma db push` | スキーマに合わせて DB を直接いじる。履歴は残らない |
| `prisma migrate dev` | 差分を SQL ファイルとして残し、それを順に流す |

ent にもまったく同じ 2 系統があります。

| ent | Prisma での相当 |
| --- | --- |
| 自動マイグレーション | `db push` |
| バージョン管理マイグレーション | `migrate dev` / `migrate deploy` |

題材アプリは**本番では後者だけ**を使っています。前者はテストで使われています。

---

## 12.2 自動マイグレーション

一番簡単な方法です。アプリの起動時にこう書きます。

```go
err := client.Schema.Create(ctx)
```

ent が今の DB を調べ、スキーマとの差を埋める DDL をその場で実行します。

### 既定は「足すだけ」

公式ドキュメント（`migrate.md`）によると、`Create` は既定で **append-only** です。

- テーブル・インデックスの作成
- 列の追加
- 型の拡張（`int` → `bigint` など）

だけを行い、**列やインデックスの削除はしません**。削除もさせたいときは `migrate.WithDropColumn(true)` や `migrate.WithDropIndex(true)` を渡します。

### なぜ本番で使わないのか

手軽な反面、本番運用には向きません。

- **何が実行されるか事前にレビューできない**。起動して初めて DDL が決まる
- **履歴が残らない**。いつ何を変えたか、Git を見てもわからない
- **細かい制御ができない**。データの移し替え（列の分割など）を挟めない

Prisma のドキュメントが `db push` をプロトタイプ向けとしているのと同じ理由です。

### 題材アプリでの使い道: テスト

題材アプリのアプリ本体は `Schema.Create` を呼んでいません。

使っているのはテスト用の `ent/enttest` です。`enttest.Open` は内部で `migrate.Create` を呼び、空の DB（SQLite のメモリ DB など）にテーブルを一気に作ります。第 15 章で詳しく扱います。

テストでは「毎回まっさらな DB に、今のスキーマどおりのテーブルがある」ことだけが重要なので、自動マイグレーションがぴったりです。

---

## 12.3 バージョン管理マイグレーション

もう 1 つの方法は、差分を **SQL ファイルとして保存**し、それを順番に流すやり方です。

```text
ent/migrate/migrations/
├── 20250101000000_init.sql
├── 20250102000000_add_xxx.sql
├── ...
└── atlas.sum
```

ファイル名の先頭はタイムスタンプで、これがそのまま実行順になります。各ファイルの中身は普通の DDL です。

```sql
-- Modify "users" table
ALTER TABLE `users`
  ADD COLUMN `nickname` varchar(255) NOT NULL;
```

**SQL がファイルとして PR に載る**ので、何が実行されるかをレビューできます。

---

## 12.4 Atlas — 差分計算と適用の担当

バージョン管理マイグレーションでは、ent は **Atlas** というツールと組んで動きます。

Atlas は ent と同じ Ariga 社が作っている、DB スキーマ管理ツールです。役割分担はこうです。

| 担当 | 仕事 |
| --- | --- |
| ent | 「あるべきスキーマ」を Atlas に渡す |
| Atlas | 差分の計算、SQL ファイルの生成、DB への適用、整合性チェック |

Prisma Migrate は Prisma に内蔵されていますが、ent はこの部分を Atlas に任せています。

`entc.go` の feature flag `sql/versioned-migration` は、この連携に必要なコード（`ent/migrate` パッケージの `NamedDiff` など）を生成させるためのものです。

---

## 12.5 差分はどうやって計算するのか

差分を出すには「今の状態」と「あるべき状態」の 2 つが要ります。

- **あるべき状態**: ent のスキーマ。これはコードにある
- **今の状態**: ……どこから取る？

公式ドキュメント（`versioned-migrations.mdx`）は 2 つの方法を挙げています。

1. **既存のマイグレーションファイルを再生（replay）して求める**（公式ドキュメントの推奨。ただし v0.14.1 の Go API の既定は 2 の ModeInspect なので、`WithMigrationMode(ModeReplay)` を明示する必要があります。`dialect/sql/schema/atlas.go`）
2. 既存の DB に接続して調べる

### Replay モード

題材アプリが使うのは 1 です。

```text
migrations/*.sql
   │ 空の DB に順番に流す
   ▼
dev DB（使い捨て）
   │ 中身を調べる → 「今の状態」
   ▼
ent スキーマと比較 → 差分 SQL を出力
```

ここで使う空の DB を Atlas は **dev database** と呼びます。本番でも開発用 DB でもなく、**計算のためだけに使い捨てる DB**です。題材アプリではローカルの MySQL コンテナに、計算専用のデータベースを 1 つ用意しています。

### なぜ Replay がうれしいのか

公式ドキュメントの説明は 2 点です。

- 差分を出すために**本番 DB に接続しなくていい**
- 環境ごとに適用状況が違っていても**同じ差分が出る**

ファイルが唯一の正、という考え方です。手元の DB を手でいじっていても、差分計算には影響しません。

---

## 12.6 差分生成スクリプト

差分生成は `ent/migrate/main.go` が担当します。`make migration` から呼ばれます。

```go
dir, err := atlas.NewLocalDir(
  "ent/migrate/migrations",
)
opts := []schema.MigrateOption{
  schema.WithDir(dir),
  schema.WithMigrationMode(
    schema.ModeReplay),
  schema.WithDialect(dialect.MySQL),
  schema.WithFormatter(
    atlas.DefaultFormatter),
  schema.WithDropColumn(true),
  schema.WithDropIndex(true),
}
err = migrate.NamedDiff(ctx,
  "mysql://root@db:3306/dev_db",
  os.Args[1], opts...)
```

オプションを 1 つずつ見ます。

| オプション | 意味 |
| --- | --- |
| `WithDir` | SQL ファイルの置き場所 |
| `WithMigrationMode(ModeReplay)` | 12.5 の Replay 方式で今の状態を求める |
| `WithDialect(MySQL)` | MySQL 用の SQL を出す |
| `WithFormatter(DefaultFormatter)` | Atlas 標準のファイル形式で書き出す |
| `WithDropColumn(true)` | スキーマから消した列を `DROP` する |
| `WithDropIndex(true)` | スキーマから消したインデックスを `DROP` する |

`NamedDiff` の第 2 引数が dev database の URL、第 3 引数がファイル名に付く名前です。

`WithDropColumn` / `WithDropIndex` を有効にしているので、**スキーマから列を消すと DROP 文が生成されます**。生成された SQL は必ず目で確認してください。

ファイル先頭の `//go:build ignore` は「通常のビルドには含めない」という印です。`go run` で直接実行するスクリプトだからです（`entc.go` も同じ）。

---

## 12.7 atlas.sum — 整合性の番人

マイグレーションディレクトリには `atlas.sum` というファイルがあります。

```text
h1:...=（全体のハッシュ）
20250101000000_init.sql h1:...=
20250102000000_add_xxx.sql h1:...=
```

各ファイルのハッシュと、全体のハッシュです。`package-lock.json` の `integrity` に近いものです。

公式ドキュメントによれば、目的は「マイグレーションの順序や中身が**後から変更された**状況に、開発者を向き合わせる」ことです。

### チーム開発で効く

A さんと B さんが別ブランチでそれぞれマイグレーションを足したとします。

- 2 人とも `atlas.sum` の末尾を書き換える
- マージ時に **`atlas.sum` が Git でコンフリクトする**

これがありがたいのです。コンフリクトしなければ、2 つのマイグレーションが同じ列を追加していても誰も気づけません。

### 直し方

`atlas.sum` は**手で編集しません**。中身を確認・調整したあと、再計算させます。

```bash
make db-hash   # atlas migrate hash
```

題材アプリのチームの規約でも手動編集は禁止です。

---

## 12.8 題材アプリの運用フロー

### 普段の流れ

規約に書かれた順序はこうです。

```bash
# 1. スキーマを編集してコード生成
make gen
# 2. 差分 SQL を生成
make migration name=add_nickname_to_users
# 3. ローカル DB に適用
make db-apply
# 4. 検証
make db-lint
```

コマンドはすべて開発用の Go コンテナの中で実行します。

### Makefile のターゲット一覧

マイグレーション関連のターゲットです。

| ターゲット | 中身 | 用途 |
| --- | --- | --- |
| `migration` | `ent/migrate/main.go` | スキーマとの差分 SQL を生成 |
| `empty-migration` | `atlas migrate new` | 空ファイルを作る（データ移行など手書き用） |
| `db-apply` | `atlas migrate apply` | 未適用分をローカル DB に流す |
| `db-apply-nonlinear` | `apply --exec-order non-linear` | 順序がずれたファイルも適用する |
| `db-down` | `atlas migrate down` | 直前の適用を取り消す |
| `db-migrate-rm` | `atlas migrate rm` | ファイルを削除し `atlas.sum` も更新 |
| `db-status` | `atlas migrate status` | 適用状況を表示 |
| `db-hash` | `atlas migrate hash` | `atlas.sum` を再計算 |
| `db-lint` | `atlas migrate lint --latest=1` | 最新 1 本の危険な変更を検査 |
| `db-validate` | `atlas migrate validate` | ディレクトリの整合性を検査 |

### 空マイグレーションの使いどころ

`make migration` が出せるのは**スキーマの差分だけ**です。「既存データを新しい列に移す」ような UPDATE 文は出せません。

そういうときは `make empty-migration` で空ファイルを作り、SQL を手で書きます。

### やり直したいとき

PR の途中で同じテーブルへの変更を何度も足すと、ファイルが散らばります。規約では 1 本にまとめることになっています。

```bash
make db-down
make db-migrate-rm name=20250102000000_xxx.sql
make migration name=...   # 作り直す
```

---

## 12.9 本番への適用

### コンテナ起動時に流す

本番では、API サーバーのコンテナが**起動するたびに**マイグレーションを流します。コンテナのエントリーポイントはこうなっています。

```sh
# DB のマイグレーションを適用
atlas migrate apply --dir file://migrations \
  --url mysql://$DB_USERNAME:...
# API サーバーを起動
/bin/server
```

`atlas migrate apply` は未適用のファイルだけを流します。適用済みかどうかは、Atlas が DB 内に持つ履歴テーブルで管理されます。Prisma の `migrate deploy` と `_prisma_migrations` テーブルの関係と同じです。

この方式だと、「デプロイ = マイグレーション適用」になります。アプリのリリースとスキーマ変更が必ず同じタイミングで起きるので、**古いアプリが新しいスキーマで動く時間**が生まれることを意識してください。列の削除や改名は、アプリ側の参照を先に外してから別リリースで行うのが安全です。

### CI で検査する

マイグレーションファイルを変更した PR では、GitHub Actions が動きます。

- `migrate/lint`: MySQL 8 のコンテナを dev database にして、危険な変更（データ消失など）を検査
- `migrate/push`: メインブランチへの push 時に、ディレクトリを Atlas Cloud へ送る

lint が警告を出すこともあります。警告は「必ず失敗する」という意味ではなく、「データ消失や失敗の可能性がある」という知らせです。中身を読み、通常のデプロイで問題ないかを判断してから進めます。

---

## 12.10 落とし穴: 外部キー付きの列を変える

チームのナレッジに記録されている実例です。

外部キー制約が付いた列の型や NULL 許容を変えると、ent はこういう SQL を出します。

```sql
ALTER TABLE `article_views`
  MODIFY COLUMN `article_id`
  bigint unsigned NOT NULL, ...;
```

これが**デプロイ時のマイグレーションでエラー**になりました。

```text
Error 1832: Cannot change column
'article_id': used in a foreign key
constraint ...
```

しかも**ローカルではエラーにならない**ため、事前に気づけません。

### 対処: 3 段階に書き換える

生成された SQL を手で直します。

1. 外部キー制約を `DROP FOREIGN KEY`
2. 列を `MODIFY COLUMN`
3. 外部キー制約を `ADD CONSTRAINT` で付け直す

SQL を書き換えたら `make db-hash` を忘れずに。題材アプリの規約にもこの 3 段階が明記されています。

「生成された SQL はそのまま信じず、読んでから使う」の典型例です。

---

## 12.11 落とし穴: 型を変える前にデータを確かめる

スキーマの型を変えるマイグレーションは、**DDL が通っても安全とは限りません**。

> **現場メモ: 0/1 の数値列を bool に変えた**
>
> 既存 DB から取り込んだテーブルに、「注目記事なら 1、そうでなければ 0」を入れる列（`is_featured`）がありました。DB は整数型（`bigint`）、ent と GraphQL は `Int` という、ちぐはぐな状態です（既存 DB からの逆生成で、DB 都合の型がそのまま残っていました。12.12）。
>
> これを `field.Bool` に揃えることにしました。マイグレーションは `MODIFY COLUMN ... bool` の DDL になり、それ自体は通ります。ところが調べてみると、**0/1 以外の値（3 など）が入った行**が残っていました。
>
> ent は MySQL の bool 列を読むとき、値を `bool` に変換します。0/1 以外の値があると、その行のスキャンに失敗します。そして一覧取得は 1 行の失敗で**全体がエラー**になります。

教訓は 2 つです。

- 型を狭める変更（数値 → bool、文字列 → enum、NULL 可 → NOT NULL）の前に、**新しい型に収まらない行が無いか**を SQL で確認する
- 確認と修正を**リリースの前提条件**として PR に明記する。データ修正は本番 DB に対して行うので、マイグレーションと同じくらい慎重に
- 確認のタイミングは DDL の適用前だけではない。読み込みエラーは **ent の型を変えたアプリ**が動いた時点で起きるので、そのアプリのリリース前にも収まらない行が無いことを確かめる

型変更が GraphQL の型にも波及する場合、配布済みのアプリを壊さないための工夫が要ります。それは第 13 章（13.10）で扱います。

---

## 12.12 周辺の話

### schema/snapshot

`entc.go` の feature flag に `schema/snapshot` があります。公式ドキュメント（`features.md`）によると、これは**最新のスキーマのスナップショットを内部パッケージ（`ent/internal`）に保存**し、スキーマがビルドできなくなったとき（Git のマージ衝突など）に自動で解決するための機能です。

DB のマイグレーションとは関係ありません。名前が似ているので混同しないでください。

`ent/internal/schema.go` は生成のたびに変わり、どこからも import されません。生成物の変更を検知して再生成を省く仕組みを作るなら、このファイルは対象から外しておくとよいでしょう。

### entimport — 既存 DB からスキーマを作る

題材アプリの ent スキーマの多くは、先頭に `Code generated by entimport` と書かれています。

これは、もともと別フレームワークで運用していた MySQL のテーブルから、**ent のスキーマを逆生成した**名残です。そのためのコマンドが Makefile に残っています。

```bash
make dbs tables=users,admins
# → ariga.io/entimport を実行
```

Prisma の `prisma db pull`（introspection）と同じ発想です。

第 4 章で触れた「`id` を `field.Uint64` で明示している理由」もここにあります。既存テーブルの ID が `bigint unsigned` だったからです。

逆生成したスキーマには、**DB の都合がそのまま残る**点に注意してください。

- テーブル名の単数形化がうまくいかず、不自然な型名になる（`addresses` → `Addresse` のような例）
- 真偽値を整数型の列（例: `bigint` の 0/1）で持っていると、`Int` のまま取り込まれる（12.11）

こうした型名や型は、後から直せます。型名だけを変えたいなら、`entsql.Annotation{Table: "..."}` で既存のテーブル名を固定しつつ ent の型名を変えられます。型名から決まる外部キー列名なども固定が要るので、差分が空になることを確かめてください（第 6 章）。GraphQL 上の名前だけを変えたいなら `entgql.Type("...")` を使います（第 13 章）。

ent のマイグレーション履歴が「全テーブルを作る巨大な初期ファイル」から始まっているのも、既存 DB を引き継いだ時点を起点にしたためと読めます。

---

## まとめ

- マイグレーションには **自動（`Schema.Create`）** と **バージョン管理** の 2 系統がある。Prisma の `db push` と `migrate` に相当
- 題材アプリは**本番ではバージョン管理だけ**。自動マイグレーションは `enttest` 経由でテストに使われる
- 差分計算と適用は **Atlas** の担当。ent は「あるべきスキーマ」を渡す
- **Replay モード**: SQL ファイルを使い捨ての dev database に流して「今」を求める。本番 DB に繋がなくていい
- **`atlas.sum`** はディレクトリの整合性を守る。手で編集せず `make db-hash`
- 流れは **`make gen` → `make migration` → `make db-apply` → `make db-lint`**
- 本番は**コンテナ起動時に `atlas migrate apply`**、CI では lint と Atlas Cloud への push
- **外部キー付き列の変更は 3 段階に書き換える**。ローカルでは気づけない
- **型を狭める前に、収まらない行が無いかデータを確かめる**
- `schema/snapshot` はマイグレーションとは無関係。`entimport` は既存 DB を引き継いだ名残で、DB 都合の型や名前が残る

---

[← 前: Hook と Interceptor](11-hooks-interceptors.md) | [目次](../README.md) | [次: entgql — GraphQL との連携 →](13-entgql.md)
