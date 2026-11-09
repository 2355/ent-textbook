# 15. テストの書き方

> **この章でわかること**
> ent を使うコードを、どの層でどうテストするか。テスト用 DB の作り方（SQLite と MySQL）、`enttest` が裏でやっていること、テストデータの factory、SQL を覗く方法。Usecase のテストで DB を使わない理由と、gomock でのモックのしかた。

第 14 章で「ent は Repository に閉じ込める」と書きました。テストもこの層構造に沿って分かれます。

---

## 15.1 層ごとの方針

| 層 | DB | 道具 |
| --- | --- | --- |
| Repository / QueryService | **使う**（テスト用 DB） | `enttest` + factory |
| Usecase / DomainService | **使わない** | gomock |
| Resolver | 場合による | 両方 |

Prisma で言えば、こう対応します。

- Repository のテスト ≒ テスト用 DB を立てて Prisma Client を本当に叩く統合テスト
- Usecase のテスト ≒ Repository を `jest.mock` で差し替える単体テスト

ent を使うコードは Repository に集まっているので、**「ent を本当に動かすテスト」も Repository のテストに集まります**。

---

## 15.2 テスト用 DB — enttest

Repository のテストは、毎回まっさらな DB を作って行います。そのための道具が ent の生成する `enttest` パッケージです。

題材アプリはこれを `store/` のヘルパで包んでいます。

```go
// store/
func GetForTestEntClient(
  t *testing.T,
) *ent.Client {
  t.Helper()
  client := enttest.Open(t, "sqlite3",
    "file:ent?mode=memory&_fk=1")
  return client
}
```

**SQLite のインメモリ DB** です。ファイルも DB サーバーも要らないので、速くて手軽です。`_fk=1` は SQLite で外部キー制約を有効にするオプションです。

### `enttest.Open` がやっていること

生成された `ent/enttest/enttest.go` を読むと、やっていることは 2 つだけです。

```go
// ent/enttest/enttest.go（生成コード）
func Open(t TestingT, driverName,
  dataSourceName string,
  opts ...Option) *ent.Client {
  o := newOptions(opts)
  c, err := ent.Open(driverName,
    dataSourceName, o.opts...)
  // …エラーなら t.FailNow()
  migrateSchema(t, c, o)
  return c
}
```

1. client を開く
2. **スキーマから自動マイグレーションでテーブルを作る**

2 が重要です。第 12 章で見たとおり、本番は Atlas のマイグレーションファイルで DB を作ります。**テストの DB は、マイグレーションファイルではなく ent スキーマから直接作られます**。

つまり、マイグレーション SQL の誤りはテストでは見つかりません。そこは `atlas migrate lint` や CI の Atlas チェックの担当です。

---

## 15.3 SQLite と MySQL の使い分け

本番は MySQL です。SQLite でテストすると、**MySQL とのちがいで「テストは通るのに本番で落ちる」「本番では動くのにテストで落ちる」**が起こります。

そこで題材アプリにはもう 1 つ、本物の MySQL を使うクライアントがあります。

```go
// store/
func GetForTestEntMySQLClient(
  t *testing.T,
) *ent.Client {
  t.Helper()
  testDBName := generateTestDBName()
  createTestDatabase(t, testDBName)
  client := enttest.Open(t,
    dialect.MySQL, /* DSN */)
  t.Cleanup(func() {
    client.Close()
    dropTestDatabase(t, testDBName)
  })
  return client
}
```

テストごとに `test_app_<時刻>_<乱数>` のような DB を作り、終わったら `DROP DATABASE` します。並列に走らせても DB 名が衝突しません。

### いつ MySQL を選ぶか

基本は SQLite、**MySQL 固有の挙動に依存するときだけ MySQL** です。選んだ理由はテストにコメントで残しておきます。

```go
// SQLite は RAND() をサポートしないため
// MySQL クライアントを使用する
client := store.GetForTestEntMySQLClient(t)
```

```go
// GROUP BY + MAX の受理を本番と同じ MySQL で
// 検証する（SQLite は非集約列の SELECT を
// 無条件に許す）
client := store.GetForTestEntMySQLClient(t)
```

パターンは 2 つです。

- **SQLite で落ちる**: `RAND()` のような MySQL 固有の関数を `Modify` や `sql.Expr` で書いた（第 7 章・第 9 章）
- **SQLite で通ってしまう**: MySQL なら弾かれる SQL を SQLite が許してしまう

後者が怖いところです。SQLite で緑でも、本番の MySQL でエラーになります。**`Modify` で手書きの SQL を足したら、MySQL でテストすることを検討してください**。

### 並び順にも差がある

もう 1 つ、見落としやすい差があります。

```go
// SQLite は同値の並びがたまたま id の降順に
// なるので、上のテストは第2キーが無くても通る。
// 並びを確定させているかは、発行された SQL で
// 見るしかない
```

`ORDER BY` のキーが同じ値の行の順序は、DB が自由に決めます。SQLite でたまたま期待どおりに並んでも、MySQL で同じになる保証はありません。これを確かめる方法が次の節です。

---

## 15.4 発行された SQL を覗く

ent が実際にどんな SQL を投げたかは、ログで確認できます。

```go
var logged []string
client := enttest.Open(t, "sqlite3",
  fmt.Sprintf(
    "file:%s?mode=memory&_fk=1", t.Name()),
  enttest.WithOptions(
    ent.Log(func(args ...any) {
      logged = append(logged,
        fmt.Sprint(args...))
    })))
defer client.Close()

r := &queryResolver{
  &Resolver{Client: client.Debug()},
}
```

2 つの道具を組み合わせています。

- `ent.Log(…)`: ログの出力先を差し替える。ここではスライスに溜める
- `client.Debug()`: SQL をログに出すモードの client を返す

あとは `logged` から `ORDER BY` を含む行を探し、第 2 キーが付いているかを検査します。Prisma の `log: ['query']` をテストの中で受け取るのと同じことです。

テスト以外でも、手元でクエリを確かめたいときに `client.Debug()` は便利です。ただし本番コードには残さないでください。

---

## 15.5 テストデータを作る — factory

テストデータは `test/factory/` の関数で作ります。どれも同じ形をしています。

```go
// test/factory/
func CreateArticle(
  t *testing.T,
  ctx context.Context,
  client *ent.Client,
  setter func(c *ent.ArticleCreate),
) *ent.Article {
  mutation := client.Article.Create().
    SetTitle(gofakeit.Sentence(5)).
    SetBody(gofakeit.Paragraph(1, 3, 10, " "))
  if setter != nil {
    setter(mutation)
  }
  a, err := mutation.Save(ctx)
  if err != nil {
    t.Fatal(err)
  }
  return a
}
```

ポイントは `setter` です。factory が `gofakeit` などでデフォルト値を埋めたあとに `setter` を呼ぶので、その上から**テストに必要な項目だけ上書き**できます。

```go
factory.CreateArticle(t, ctx, client,
  func(ac *ent.ArticleCreate) {
    ac.SetAuthorID(u.ID).
      SetDeletedAt(time.Now())
  })
```

第 8 章の Create Builder をそのまま渡しているので、型補完が効き、存在しない項目は書けません。TS の fishery や `@quramy/prisma-fabbrica` の `build({ … })` に近い使い心地です。

### 外部キーの親も用意する

テスト DB は空っぽから作られるので、外部キーの参照先がありません。factory の中には、親レコードを先に作るものがあります。

```go
// 著者（author_id）の参照先を先に作る。
// 存在しない ID を入れると
// 外部キー制約に引っかかる
func CreateArticle(t *testing.T,
  ctx context.Context, client *ent.Client,
  setter func(*ent.ArticleCreate),
) *ent.Article {
  author := CreateUser(t, ctx, client, nil)
  c := client.Article.Create().
    SetTitle(gofakeit.Sentence(3)).
    SetAuthorID(author.ID)
  if setter != nil {
    setter(c) // 著者を差し替えるならここで
  }
  a, err := c.Save(ctx)
  if err != nil {
    t.Fatal(err)
  }
  return a
}
```

`_fk=1` で外部キーを有効にしているからこそ必要になる処理です。

---

## 15.6 Repository のテストの形

Repository テストの形を見てみます。

```go
tests := []struct {
  name      string
  setupFunc func(ctx context.Context,
    client *ent.Client) uint64
  checkErr  func(t *testing.T, err error)
}{
  {
    name: "削除された記事は" +
      "NotFoundエラーになる",
    setupFunc: …,
    checkErr: func(t *testing.T, err error) {
      if !database.IsNotFoundError(err) {
        t.Errorf("…")
      }
    },
  },
}
for _, tt := range tests {
  t.Run(tt.name, func(t *testing.T) {
    t.Parallel()
    client := store.GetForTestEntClient(t)
    defer client.Close()
    // …
  })
}
```

題材アプリのテストの規約がそのまま現れています。

- **テーブル駆動**。ケースを構造体のスライスに並べ、`t.Run` で回す
- **アサーションライブラリを使わない**。testify などを使わず、`if` と `t.Errorf` で書く
- **エラーは `checkErr`** で検証する。文字列一致ではなく `database.IsNotFoundError` のような判定ヘルパを使う
- ケースごとに `GetForTestEntClient` で**新しい DB** を開く

Jest の `test.each` に近いですが、Go にはアサーション関数が標準にないので `if` を書きます。最初は冗長に感じますが、失敗メッセージを自分で書くぶん、何が起きたか分かりやすくなります。

### ent 構造体の比較

Repository のテストでは `google/go-cmp` の `cmp.Diff` で ent 構造体をまとめて比較します。`CreatedAt` のような毎回変わる値は `cmpopts.IgnoreFields` で外します。

### 論理削除のテスト

第 11 章の論理削除 Mixin も、テストで挙動を固定しておきます。`deleted_at` を入れた記事と入れない記事を作り、普通のクエリでもエッジ走査（`client.User.QueryArticles(u)` など）でも 1 件しか返らないことを確かめます。

上の「削除された記事は NotFound」も同じ仕組みに乗っています。`Article` スキーマが `SoftDeleteMixin` を持つからです。

---

## 15.7 Usecase のテスト — gomock

Usecase のテストでは DB を使いません。Repository の port をモックに差し替えます。

これは Usecase に限りません。**ent クライアントを直接使わず、Repository の port 経由でデータを取る**ように書いておけば、通知を送るアダプタのような周辺コードでも DB なしでテストできます。

> **現場メモ**
> 通知処理の中で ent クライアントを直接叩いていたため、テストに DB の準備が必要になっていた。port 経由に書き換えたら gomock だけで書けるようになり、テストのセットアップが大幅に減った。

### モックは生成する

port のインターフェースの上に、生成の指示を書いておきます。

```go
// domain/ports/
//go:generate mockgen -package $GOPACKAGE \
//  -source $GOFILE -destination mock_$GOFILE
```

`make gen` で `mock_xxx.go` が生成されます。ent と同じく**コード生成**で作るのが Go の流儀です。Jest のように実行時にモジュールを差し替える仕組みはありません。

### 大きなモック 1 つで足りる

第 14 章で見たとおり、個別の port は大きな `DBRepository` に埋め込まれています。Usecase はこの大きなインターフェースを受け取るので、**モックも `MockDBRepository` 1 つで足ります**。

個別のインターフェース（`ArticleRepository` など）にもモック生成の指示を書くことはできますが、必要になるのは次の場合だけです。

- テストしたい関数が、`DBRepository` ではなく**個別のインターフェースだけを引数に取る**とき

それ以外で個別モックを生成しても、使われないファイルが増えるだけです。生成の指示を足す前に、「この関数は何を引数に取っているか」を確認してください。

### 期待する呼び出しを書く

```go
mocks.DBRepository.EXPECT().
  GetUserByEmail(gomock.Any(),
    "test@example.com").
  Return(entity, nil)
```

Jest で書くと（`jest-mock-extended` を使った場合）こうです。

```ts
repo.getUserByEmail
  .calledWith(expect.anything(),
    "test@example.com")
  .mockResolvedValue(entity)
```

gomock は**期待していない呼び出しが来たらテストを落とします**。「呼ばれないはず」も検証できるわけです。

`gomock.Any()` は最小限にします。`context.Context` と検証不要な動的値だけに使い、ID や文字列は具体値を書くのがよいでしょう。

### Entity を用意する

Usecase は ent を触れないので、テストでも `ent.User` は作りません。Entity のテスト用ヘルパを使います。

```go
// domain/models/
entity, err := models.NewUserEntityMock(
  t, 1, func(dto *models.NewUserEntityDTO) {
    dto.Email = "test@example.com"
  })
```

比較も `entity.Model()` は使えないので、Getter で行います。

### トランザクションをモックする

`RunInTransaction` もモックにします。渡されたクロージャをそのまま実行するのが定番です。

```go
dbRepo.EXPECT().
  RunInTransaction(gomock.Any(),
    gomock.Any()).
  DoAndReturn(func(ctx context.Context,
    f func(ctx context.Context) error,
  ) error {
    return f(ctx)
  }).Times(1)
```

これで「トランザクションの中で何が呼ばれるか」を、本物の DB なしで検証できます。ロールバックそのものは Repository 側の責務なので、Usecase のテストでは扱いません。

---

## 15.8 何をどこでテストするか

迷ったときの目安です。

- **クエリの条件・並び順・eager load** → Repository のテスト（DB あり）
- **MySQL 固有の関数・集計** → Repository のテストを MySQL で
- **分岐・エラーの変換・呼び出し順** → Usecase のテスト（gomock）
- **entgql の `Paginate` に引数を渡すだけの Resolver** → テストしない（ent の動作確認にしかならないため）

最後の項目は、逆に言えば「並び順を固定するなど独自の処理を足した Resolver はテストする」ということです。15.4 の SQL を覗くテストがその例です。

---

## まとめ

- ent を本当に動かすテストは **Repository / QueryService** に集まる。Usecase は **gomock** で DB なし
- ent クライアントを直接使わず **port 経由**にしておけば、周辺コードも DB なしでテストできる
- テスト用 DB は **`enttest.Open`**。client を開き、**スキーマから自動マイグレーション**でテーブルを作る。マイグレーション SQL は検証されない
- 基本は **SQLite インメモリ**。`RAND()` などの固有関数や、**SQLite が甘く通してしまう SQL** は **MySQL** で
- **`ent.Log` + `client.Debug()`** で発行 SQL を捕まえられる
- テストデータは **`factory.CreateXxx(t, ctx, client, setter)`**。setter で必要な項目だけ上書き
- モックは大きな **`MockDBRepository` 1 つ**で足りる。個別モックは個別インターフェースを引数に取る関数のときだけ
- 規約は **テーブル駆動・アサーションライブラリなし・`checkErr`**

---

[← 前: アーキテクチャの中の ent](14-architecture.md) | [目次](../README.md) | [次: ここから先へ →](16-next-steps.md)
