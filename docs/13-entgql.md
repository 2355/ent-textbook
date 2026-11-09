# 13. entgql — GraphQL との連携

> **この章でわかること**
> ent のスキーマから GraphQL スキーマを生成する仕組み。Relay 形式のページネーション、並び替えと絞り込みの入力型、表に出さない設定、認可ディレクティブ。N+1 を避ける CollectFields。Resolver が ent の型を直接返している理由。意図せず公開される型と、型を変えても API を保つ方法。

長い章です。13.1〜13.5 が土台、13.6 以降は個別機能のカタログです。

---

## 13.1 entgql とは何か

題材アプリは GraphQL サーバーです。GraphQL の部分は **gqlgen** というライブラリが担当しています。

gqlgen は「スキーマファースト」のツールです。

1. `.graphqls` ファイルに GraphQL スキーマを書く
2. gqlgen がそれに合う Go のインターフェースを生成する
3. 人間が Resolver を実装する

Apollo Server で `typeDefs` を書いて `resolvers` を実装し、GraphQL Code Generator で型を生成するのと同じ流れです。

ここで問題が起きます。**ent のスキーマと GraphQL のスキーマを二重に書くことになる**のです。

- ent: `User` には `name` と `email` がある
- GraphQL: `type User { name: String! email: String! }`

**entgql** はこれをつなぐ ent の拡張です（`entgo.io/contrib/entgql`）。

- ent のスキーマから **GraphQL スキーマ（`ent.graphqls`）を生成**する
- ページネーション・並び替え・絞り込みの**入力型と実装を生成**する
- gqlgen が **ent の型をそのまま GraphQL の型として使える**ようにする

Prisma で言えば、Prisma スキーマから GraphQL 型と CRUD を生やす Pothos の Prisma プラグインや TypeGraphQL-Prisma に近い位置づけです。

---

## 13.2 生成の流れ

`make gen` の中で次の順に動きます（`generate.go`）。

```go
//go:generate go run -mod=mod ./ent/entc.go
//go:generate go run -mod=mod github.com/99designs/gqlgen
```

```text
ent/schema/*.go
   │ entc + entgql 拡張
   ├→ ent/*.go（ORM のコード）
   ├→ ent/gql_*.go（GraphQL 用の補助コード）
   └→ graphqls/ent.graphqls
          │
graphqls/*.graphqls（手書き）
          │ gqlgen
          ├→ internal/generated.go（実行エンジン）
          ├→ graph/model/models_gen.go
          └→ graph/*.resolvers.go（Resolver の雛形）
```

ポイントは 2 段構えであることです。

1. **entgql が `ent.graphqls` を書き出す**
2. **gqlgen が `ent.graphqls` と手書きの `.graphqls` をまとめて読む**

新しいエンティティを ent に足して `make gen` すると、それだけで GraphQL の型が生えます。

`ent.graphqls` は数万行規模になることもある巨大な生成物です。**手で編集してはいけません**。変えたいときは ent スキーマを直して `make gen` です。

### extend type で足す

生成された型に、独自のフィールドを足したいことがあります。

- 表示用に計算した値（コレクション内の記事数など）
- 独自ロジックの Query や Mutation

そういうものは手書きの `.graphqls` に、**`extend type`** で書きます。

```graphql
# graphqls/collections.graphqls（手書き）
extend type Collection {
  articleCount: Int!
}
```

gqlgen は `ent.graphqls` の `type Collection` とこれを 1 つの型にまとめ、`articleCount` の分だけ Resolver の雛形を作ります。

TypeScript の **declaration merging**（同名の `interface` を複数書くと合体する）に近い感覚です。生成物には手を触れず、別ファイルで「足し算」だけするので、`make gen` で上書きされる心配がありません。

---

## 13.3 entc.go の設定

拡張の設定は `ent/entc.go` にあります。

```go
ex, err := entgql.NewExtension(
  entgql.WithConfigPath("./gqlgen.yml"),
  entgql.WithSchemaPath(
    "./graphqls/ent.graphqls"),
  entgql.WithSchemaGenerator(),
  entgql.WithWhereInputs(true),
  entgql.WithSchemaHook(/* 後述 */),
)
```

| オプション | 意味 |
| --- | --- |
| `WithConfigPath` | gqlgen の設定を読み、型の対応を合わせる |
| `WithSchemaPath` | 生成する GraphQL スキーマの出力先 |
| `WithSchemaGenerator` | GraphQL スキーマを**生成する**（無いと補助コードだけ） |
| `WithWhereInputs(true)` | 絞り込み用の `XxxWhereInput` を生成する |
| `WithSchemaHook` | 生成したスキーマを書き出す前に加工する |

### WithSchemaHook — node / nodes を消している

```go
entgql.WithSchemaHook(func(
  _ *gen.Graph, s *ast.Schema,
) error {
  if q := s.Types["Query"]; q != nil {
    q.Fields = slices.DeleteFunc(q.Fields,
      func(f *ast.FieldDefinition) bool {
        return f.Name == "node" ||
          f.Name == "nodes"
      })
  }
  return nil
}),
```

entgql は Relay の仕様に従い、`Query.node(id: ID!)` と `Query.nodes(ids: [ID!]!)` を自動で生やします。「ID さえわかれば、どんな型のオブジェクトでも取れる」入口です。

題材アプリはこれを生成直後に削除しています。理由は `entc.go` のコメントに書かれています。

- ID から型を逆引きするための `ent_types` テーブルを作っていないので、**そもそも動かない**
- 動かないだけでなく、**匿名で全 ent 型に ID で到達できる口**になってしまう

Relay の Node インターフェース自体（`type Admin implements Node`）は残っています。`gqlgen.yml` でも `Node` は `ent.Noder` に対応付けられています。ただし Resolver から `Noder` を使う箇所はなく、実質未使用です。

---

## 13.4 gqlgen.yml — ent の型をそのまま使う

gqlgen 側の設定（`gqlgen.yml`）にも ent との連携があります。

### autobind

```yaml
autobind:
  - example.com/app/ent
```

「GraphQL の型名と同じ名前の Go 型が `ent` パッケージにあれば、それを使え」という指示です。

これにより GraphQL の `User` 型は、gqlgen が新しく struct を作るのではなく、**ent が生成した `*ent.User` に直接対応**します。

### ID を uint64 に

```yaml
models:
  ID:
    model:
      - github.com/99designs/gqlgen/graphql.Uint64
```

GraphQL の `ID` は本来文字列ですが、題材アプリの ent の ID は `uint64` です（第 4 章）。この設定で、GraphQL の `ID` を Go 側では `uint64` として扱います。

ほかにも `UUID` を自前の UUID 型に、`Int` に都道府県コードの `Prefecture` 型も結び付けられるようにする、などの対応がここに書かれています。

---

## 13.5 Resolver が ent の型を返す

autobind のおかげで、Resolver はこう書けます。

```go
func (r *queryResolver) Admins(
  ctx context.Context,
  after *entgql.Cursor[uint64], first *int,
  before *entgql.Cursor[uint64], last *int,
  orderBy []*ent.AdminOrder,
  where *ent.AdminWhereInput,
) (*ent.AdminConnection, error) {
  return r.Client.Admin.Query().
    Paginate(ctx, after, first, before, last,
      ent.WithAdminFilter(where.Filter),
      ent.WithAdminOrder(orderBy),
    )
}
```

**ent のクエリ結果を、詰め替えずにそのまま返しています。**

`Resolver` 構造体は `Client *ent.Client` を持っていて、Resolver から直接 ent を叩けます。

### エッジは Resolver を書かなくても解ける

`Article.category` のようなエッジのフィールドは、多くの場合 Resolver を書く必要すらありません。entgql が `ent/gql_edge.go` にメソッドを生成するからです。

```go
func (a *Article) Category(
  ctx context.Context,
) (*Category, error) {
  result, err := a.Edges.CategoryOrErr()
  if IsNotLoaded(err) {
    result, err = a.QueryCategory().Only(ctx)
  }
  return result, MaskNotFound(err)
}
```

1. eager load 済みならそれを返す
2. まだなら、その場でクエリする

gqlgen は autobind した型のメソッドを見つけると、それを Resolver として使います。

### なぜ詰め替えないのか

題材アプリのアーキテクチャ規約では、ビジネスロジックで ent を使うことは禁止されています（第 14 章）。それでも Resolver 層だけは、ドメインモデルの `Model()` で ent の構造体を返すことが**推奨**されています。

理由は「gqlgen と ent のクエリ最適化を効かせるため」です。13.12 で見る CollectFields は、Resolver が ent の型を扱っていることが前提だからです。

---

## 13.6 QueryField — Query にフィールドを生やす

ここからは、スキーマの `Annotations()` に書く entgql のアノテーションを順に見ます。

`entgql.QueryField()` は、**`Query` 型にその型の一覧取得フィールドを生やす**指示です。

```go
func (Article) Annotations(
) []schema.Annotation {
  return []schema.Annotation{
    entgql.QueryField(),
    entgql.RelayConnection(),
    // ...
  }
}
```

これで `Query.articles(...)` が生成されます。

### 効くのは型レベルだけ

題材アプリでは `QueryField` がフィールドやエッジの `Annotations(...)` にも大量に書かれています。しかし実際に効いているのは、型レベル（`Annotations()` メソッド）に書いたものだけです。

entgql v0.6.0 のスキーマ生成（`entgql/schema.go`）を読むと、`QueryField` を参照しているのは型のアノテーションだけです。**フィールドやエッジに付けた `QueryField` は生成結果に影響しません**。エッジは `QueryField` の有無にかかわらず GraphQL のフィールドになります。

新しく書くときは、型レベル以外の `QueryField` は真似しなくて構いません。

---

## 13.7 RelayConnection と Paginate

### Relay のカーソルページネーション

`entgql.RelayConnection()` を付けると、その型は **Relay Cursor Connections** 形式でページングできるようになります。Apollo Client や Relay を使ったことがあれば見覚えのある形です。

```graphql
type ArticleConnection {
  edges: [ArticleEdge]
  pageInfo: PageInfo!
  totalCount: Int!
}
type ArticleEdge {
  node: Article
  cursor: Cursor!
}
```

Connection / Edge / PageInfo / Cursor の型と、それを埋める Resolver の実装を手で書くと、かなりの量になります。entgql はこの**ボイラープレートを丸ごと生成**します。gqlgen と ent を組み合わせる大きな動機の 1 つです。

引数は 4 つです。

| 引数 | 意味 |
| --- | --- |
| `first` / `after` | 前から n 件、このカーソルの後ろから |
| `last` / `before` | 後ろから n 件、このカーソルの前から |

オフセット（`page=3`）ではなく**カーソル（「この行の次から」）**で位置を表すので、ページを送っている間に行が増減してもずれません。

### 型レベルとエッジ

- **型レベル**に付けると、`XxxConnection` 型が作られ、`QueryField` と組み合わせて `Query.xxx` が Connection を返すようになる
- **エッジ**に付けると、そのエッジのフィールドが Connection になる（`articles(first:, after:, ...)`）

エッジに付けるには、**行き先の型にも型レベルの `RelayConnection` が必要**です（無いと生成時にエラー）。また、1 件しかない（Unique）エッジには付けられません。

### Paginate — 実装は 1 行

ページングの SQL は entgql が生成した `Paginate` メソッド（`ent/gql_pagination.go`）が組み立てます。Resolver は引数を渡すだけです。

```go
return obj.QueryArticles().
  Paginate(ctx, after, first, before, last,
    opts...)
```

エッジの Connection も、`QueryXxx()` で走査した先に `Paginate` を呼ぶだけです。

カーソルの型は `*entgql.Cursor[uint64]` です。ID が `uint64` なのでこうなります。

### COUNT はいつ流れるか

`Paginate` は、クエリが **`totalCount` か `pageInfo` を要求しているときだけ**、件数を数える `COUNT` クエリを追加で発行します（生成テンプレートの `collection.tmpl` / `pagination.tmpl`）。

```graphql
# COUNT が流れる
articles(first: 20) {
  totalCount
  edges { node { title } }
}

# COUNT は流れない
articles(first: 20) {
  edges { node { title } }
}
```

大きなテーブルの `COUNT` は重くなりがちです。フロントエンドが件数やページ情報を表示しない画面なら、クエリから外すだけで SQL が 1 本減ります。SQL ログを見て「なぜ COUNT が流れているのか」と思ったら、クエリの選択フィールドを確認してください。

### ページ番号方式との併存

題材アプリには、`page` / `perPage` で `Offset` / `Limit` する Resolver も多くあります（第 7 章）。管理画面の「ページ番号付き一覧」のように、カーソルが合わない画面ではそちらを使います。

---

## 13.8 並び替え — OrderField と MultiOrder

### OrderField

フィールドに `entgql.OrderField("名前")` を付けると、そのフィールドで並び替えられるようになります。

```go
field.Time("created_at").
  Annotations(
    entgql.OrderField("CREATED_AT"),
  ),
```

生成される GraphQL:

```graphql
enum ArticleOrderField {
  CREATED_AT
  PUBLISHED_AT
}
input ArticleOrder {
  direction: OrderDirection! = ASC
  field: ArticleOrderField!
}
```

### エッジの件数で並べる

エッジに `エッジ名の大文字_COUNT` という名前で付けると、**そのエッジの件数順**になります（`エッジ名_フィールド名` でエッジ先のフィールド順も指定できます。それ以外の名前は生成時エラー。`entgql/template.go`）。

```go
edge.To("views", ArticleView.Type).
  Annotations(
    entgql.OrderField("VIEWS_COUNT"),
  )
```

これで「閲覧数の多い順」が GraphQL から指定できます。名前は `エッジ名の大文字_COUNT` にする決まりです（公式 `tutorial-todo-gql-paginate.md`）。

### MultiOrder

型レベルに `entgql.MultiOrder()` を付けると、`orderBy` が**配列**になります。

```graphql
articles(orderBy: [ArticleOrder!]): ...
```

「公開日の新しい順、同じなら ID 順」のような複数キーの並び替えができます。付けていない型は `orderBy: CollectionOrder` のように単数です。

### Resolver 側

Resolver では `ent.WithXxxOrder(orderBy)` を `Paginate` に渡すだけです。独自の並び替え入力を作る Resolver では、方向に `entgql.OrderDirectionAsc` / `OrderDirectionDesc` を使っている例もあります。

---

## 13.9 絞り込み — WhereInput

`WithWhereInputs(true)` を有効にしているので、型ごとに `XxxWhereInput` が生成されます。

```graphql
input ArticleWhereInput {
  not: ArticleWhereInput
  and: [ArticleWhereInput!]
  or: [ArticleWhereInput!]
  id: ID
  idIn: [ID!]
  idGT: ID
  title: String
  titleContains: String
  hasTags: Boolean
  hasTagsWith: [TagWhereInput!]
  # ...
}
```

第 7 章の述語（`TitleContains`、`HasTagsWith` など）が、**そのまま GraphQL の入力**になっています。Hasura や Prisma の `where` 引数とよく似た使い心地です。

### Resolver 側

```go
ent.WithArticleFilter(where.Filter)
```

`Filter` は、`WhereInput` を述語に変換してクエリに足すメソッドです（`ent/gql_where_input.go`）。

`where` が `nil` のとき、`where.Filter` を呼んで大丈夫か気になるかもしれません。生成コードはレシーバが `nil` なら何もせずに返す作りになっています。

```go
func (i *ArticleWhereInput) Filter(
  q *ArticleQuery,
) (*ArticleQuery, error) {
  if i == nil {
    return q, nil
  }
  // ...
}
```

そのため無条件に渡しても動きます。`if where != nil` で囲む書き方も混在しています。

### 強力すぎることに注意

WhereInput は**全フィールド・全エッジで絞り込める**入力です。

たとえば `hasXxxWith` を使えば、本来見せたくないエッジの中身を「条件に合うかどうか」で探れてしまいます。そこで次節の `Skip` で、出したくないものを削ります。

---

## 13.10 出さない設定 — Skip

`entgql.Skip(...)` は、「この型・フィールド・エッジを GraphQL に出さない」指示です。

何を出さないかを**ビットフラグ**で選べます（`entgql/annotation.go`）。

| モード | 出さないもの |
| --- | --- |
| `SkipType` | 型・フィールドそのもの |
| `SkipEnumField` | enum フィールドの GraphQL enum |
| `SkipOrderField` | 並び替え用の型 |
| `SkipWhereInput` | WhereInput（フィールドに付けると、そのフィールドだけ除外） |
| `SkipMutationCreateInput` | Create 用入力型 |
| `SkipMutationUpdateInput` | Update 用入力型 |
| `SkipAll` | 上のすべて |

**引数なしの `Skip()` は `SkipAll` と同じ**です。

### 既定は「全部公開」

entgql の既定は、**ent にある型・フィールド・エッジをすべて GraphQL に出す**ことです。Skip は「出さないものを選ぶ」ための仕組みで、出すものを選ぶ仕組みではありません。

つまり、新しいエンティティやエッジを足すと、何もしなくても GraphQL から見える可能性があります。

> **現場メモ: Query の入口が無くても、型は届く**
>
> 管理画面だけで使うつもりのテーブルを ent に足したことがありました。`Query` にルートのフィールドも Resolver も作っていないので、外からは触れないと思っていました。
>
> ところが、そのテーブルは別の公開型（カテゴリ）とエッジでつながっていました。entgql はエッジを GraphQL のフィールドとして生成するので、**`categories { edges { node { 管理用の型 { ... } } } }` とたどれば取得できてしまう**状態でした。
>
> 「Query に入口が無い = 非公開」ではありません。GraphQL では**グラフのどこか 1 か所からたどれれば届く**ので、公開面は「スキーマ全体の到達可能性」で考える必要があります。
>
> 対策は、非公開にしたいエッジや型に `Skip` を付けて**スキーマから消す**か、ディレクティブで守ることです。新しい型を足したら、生成された `ent.graphqls` の差分で「どこから到達できるようになったか」を確認する習慣をつけると安心です。

### 使い分け

**`Skip()` / `SkipAll` — 完全に隠す**

```go
// 公開記事のクエリから逆向きにたどると
// 非公開のコレクションまで見えてしまうため
// GraphQL へ公開しない
edge.From("collections", Collection.Type).
  // ...
  Annotations(entgql.Skip()),
```

記事から逆向きにコレクションを辿れると、非公開のコレクションまで見えてしまうため、エッジごと消しています。中間テーブル（`CollectionItem` など）やログ系の型にも、型レベルで `SkipAll` を付けることがあります。

**`SkipType` — フィールドを隠す**

```go
field.Text("thumbnail_key").
  Comment("ストレージ上のパス").
  Annotations(entgql.Skip(entgql.SkipType)),
```

ストレージのパスは生のまま出さず、手書きの `.graphqls` で URL に変換したフィールドを別に用意する、という使い方です。

**`SkipWhereInput` — 表示はするが絞り込みには使わせない**

```go
edge.To("payments", Payment.Type).
  Annotations(
    entgql.Directives(/* ADMIN 限定 */),
    entgql.Skip(entgql.SkipWhereInput),
  )
```

エッジ自体は管理者だけが見られるようディレクティブで守っています（次節）。しかし WhereInput に `hasPaymentsWith` が残っていると、**ディレクティブを通らずに条件で探れて**しまいます。そこで WhereInput からは外しています。

### Skip して手書きで出し直す — 型を変えても API を保つ

ent の型を変えたいが、GraphQL の型は変えたくない、という場面があります。

> **現場メモ: 数値を bool にしたいが、配布済みアプリは数値を待っている**
>
> 第 12 章 12.11 の「0/1 の数値列を bool に変えた」話の続きです。
>
> DB と ent のフィールドは `bool` にしたい。ところが GraphQL では、そのフィールドは `Int!` として公開済みでした。しかも、**配布済みのクライアント**が `isFeatured === 1` のように数値として比較していました。GraphQL の型を `Boolean!` に変えると、古いクライアントが壊れます。

entgql はフィールドの Go 型から GraphQL の型を決めます。ent 側を `field.Bool` にすると、生成される GraphQL も `Boolean!` になってしまいます。

そこで次の 3 段で対応します。

1. ent のフィールドに **`entgql.Skip()`** を付け、自動生成の対象から外す
2. 手書きの `.graphqls` で **`extend type`** し、同じ名前のフィールドを `Int!` で定義し直す
3. `gqlgen.yml` でそのフィールドに **`resolver: true`** を指定し、`bool` → `0/1` に変換する Resolver を手で書く

```yaml
# gqlgen.yml
models:
  Article:
    fields:
      isFeatured:
        resolver: true
```

`resolver: true` は「ent の struct のフィールドに直接バインドせず、Resolver を生成せよ」という gqlgen の指定です。これで DB と ent は正しい型に揃いつつ、API の形は変わりません。

厳密には、Go の型（`bool`）と GraphQL の型（`Int!`）が合わなければ、gqlgen は自動で Resolver に切り替えます（ログに出るだけでエラーにはなりません）。`resolver: true` は必須ではなく、**意図を明示してそのログを消すため**の指定と考えてください。

注意点が 1 つあります。**`Skip()`（= `SkipAll`）を付けると、WhereInput の検索条件からも消えます**。`isFeatured: Int` で絞り込んでいたクライアントがいれば、絞り込み用の入力も手書きで足す必要があります。Skip のどのフラグを付けるかで、何が消えるかを確認してから使ってください。

型名だけを変えたいなら、もっと軽い方法があります。`entgql.Type("NewName")` を型レベルに付けると、ent の型名はそのままで GraphQL 上の型名だけを変えられます（`entgql/annotation.go`）。

---

## 13.11 認可 — Directives

### ディレクティブとは

GraphQL のディレクティブは `@` で始まる注釈です。

```graphql
type Query {
  admins(...): AdminConnection!
    @isAuthenticated(requires: [ADMIN])
}
```

gqlgen では、ディレクティブは**フィールド解決を包むミドルウェア**として実装します。NestJS の Guard や `graphql-shield` に近いものです。

### 宣言・付与・実装

題材アプリでは 3 か所に分かれています。

**宣言**（手書きの `.graphqls`）

```graphql
enum Guard { ADMIN USER }
directive @isAuthenticated(requires: [Guard!])
  on FIELD_DEFINITION | OBJECT
directive @isOwnByUser on FIELD_DEFINITION
```

**ent スキーマからの付与**（`ent/common/directive.go` のヘルパ）

```go
entgql.Directives(
  common.GetAuthDirective(
    []string{common.GuardAdmin}),
)
```

`GetAuthDirective` は `entgql.Directive` を組み立てるヘルパです。`GetIsOwnByUser()` などもあります。

**実装**（Resolver と同じパッケージ）

```go
func IsAuthenticated(ctx context.Context,
  obj interface{}, next graphql.Resolver,
  requires []model.Guard,
) (interface{}, error) {
  for _, v := range requires {
    switch v {
    case model.GuardAdmin:
      u, ok := auth.GetPrincipal(ctx)
      if ok && u.IsAdmin() {
        return next(ctx)
      }
    // ...
    }
  }
  return nil, apperror.NewUnauthorizedError(...)
}
```

Express のミドルウェアと同じく、**通すなら `next(ctx)`、通さないならエラー**です。

| ディレクティブ | 通す条件 |
| --- | --- |
| `@isAuthenticated` | 指定ロール（ADMIN / USER）でログインしている |
| `@isOwnByUser` | 管理者、または対象の `User` が本人 |

`@isOwnByUser` は `obj.(*ent.User).ID` で「今解決している User」を取り出しています。ディレクティブが `User` のフィールドに付くからこそできる判定で、Resolver が ent の型を返していることがここでも効いています。

### スキーマファイルに DB と API が同居する

ディレクティブを ent スキーマ側で付けられるのは便利です。生成された `Query` に、ガードを付け忘れることがありません。

その代わり、`ent/schema/` のファイルには **DB の話（列・インデックス・外部キー）と API の話（公開・並び替え・認可）が混ざり**ます。最初は読みにくく感じるはずです。

これは entgql の設計上そうなっているもので、割り切って読むのが現実的です。読むときは次のように頭の中で分けると整理しやすくなります。

| 書かれているもの | 関心 |
| --- | --- |
| `field.*` / `edge.*` / `index.*` / `entsql.*` | DB |
| `entgql.*` | GraphQL |

### どこに付けるかで効く範囲が変わる

`Directives` を付ける場所は 3 つあり、効く範囲が違います。

| 付け方 | 生成先 |
| --- | --- |
| 型レベル `entgql.Directives(...)` | `type Admin @isAuthenticated(...)` |
| `entgql.QueryField().Directives(...)` | `Query.admins` フィールドだけ |
| フィールド / エッジの `entgql.Directives(...)` | そのフィールドだけ |

型レベルのディレクティブは、**その型を返すあらゆるフィールド**で発動します。gqlgen の生成コード（`internal/generated.go`）を見ると、手書きの型（ログイン結果を返す型など）のフィールドにも `IsAuthenticated` が差し込まれているのがわかります。

これが落とし穴になることがあります。

```go
// 型レベルの Directives にすると Collection 型を返す
// 一般ユーザー向けフィールドにも波及してしまうため、
// この QueryField（Query.collections）に限定して
// 管理者限定にする
entgql.QueryField().Directives(
  common.GetAuthDirective(
    []string{common.GuardAdmin})),
```

「一覧（`Query.collections`）は管理者だけ。でも個別のコレクションは一般ユーザーにも見せたい」という要件なので、型ではなく `QueryField` にだけ付けています。

---

## 13.12 N+1 を避ける — CollectFields

### GraphQL の N+1

GraphQL ではこういうクエリが普通に来ます。

```graphql
{
  articles(first: 20) {
    edges { node { title category { name } } }
  }
}
```

素朴に実装すると、記事 20 件を取ったあと、**記事ごとにカテゴリを 1 回ずつ**取りに行きます。1 + 20 回の SQL です。

Node.js では DataLoader でまとめるのが定番です。entgql は別のアプローチを取ります。

### 先にクエリを見てしまう

entgql は `CollectFields` というメソッドを生成します（`ent/gql_collection.go`）。

- GraphQL のクエリが**どのフィールドを要求しているか**を読む
- 要求されているエッジに対して、**自動で `WithXxx()`（eager load）を足す**

上の例なら、`category` が要求されているのを見て `WithCategory()` を足し、2 回の SQL で済ませます。13.5 の `gql_edge.go` のメソッドが「eager load 済みならそれを返す」作りになっていたのは、このためです。

ent の eager load は JOIN ではなく、関係ごとに `WHERE id IN (...)` のクエリを 1 本ずつ投げます（第 7 章）。記事が 20 件でも 200 件でも、カテゴリを取る SQL は 1 本です。

### Paginate が中で呼んでいる

実は `Paginate` は内部で `CollectFields` 相当の処理を呼んでいます（`ent/gql_pagination.go` の `collectField`）。Connection を返す Resolver は、何もしなくても N+1 対策が効いています。

Connection を使わない取得では、明示的に呼びます。題材アプリでは Repository で呼んでいる例があります。

```go
query, err := client.Article.Query().
  Where(/* ... */).
  Order(ent.Desc(article.FieldPublishedAt)).
  CollectFields(ctx)
// ...
articles, err := query.All(ctx)
```

`CollectFields(ctx)` は ctx から GraphQL のリクエスト情報を取り出すので、GraphQL のリクエスト中でしか意味を持ちません。

### 手動で判定する場合

`CollectFields` が使えない複雑なクエリでは、gqlgen の `graphql.CollectFields` で要求フィールドを自分で調べることもあります。

```go
for _, f := range graphql.CollectFields(
  opCtx, fc.Field.Selections, nil) {
  // latestView を選んでいるか調べる
}
```

選ばれていたときだけ、eager load を足します。

### LimitPerRow — 親ごとに n 件

「記事ごとに**最新の閲覧履歴 1 件だけ**」が欲しい、という要件を考えます。

普通に `Limit(1)` すると、**全体で 1 件**になってしまいます。そこで entgql の `LimitPerRow` を `Modify`（第 7 章）に渡します。

```go
q.WithViews(
  func(q *ent.ArticleViewQuery) {
    q.Where(articleview.UserID(userID)).
      Modify(entgql.LimitPerRow(
        article.ViewsColumn,
        1, /* last_viewed_at DESC */))
  })
```

`LimitPerRow` はウィンドウ関数 `ROW_NUMBER() OVER (PARTITION BY ...)` を使ったサブクエリに書き換え、**親（記事）ごとに n 件**に絞ります（`entgql/pagination.go`）。本来は Connection のエッジで「各ノードにつき first 件」を実現するための部品です。

---

## 13.13 gql_transaction — OpenTx だけ使っている

`ent/gql_transaction.go` には `OpenTx` が生成されます。

```go
func (c *Client) OpenTx(ctx context.Context,
) (context.Context, driver.Tx, error) {
  tx, err := c.Tx(ctx)
  // ...
  ctx = NewTxContext(ctx, tx)
  ctx = NewContext(ctx, tx.Client())
  return ctx, tx, nil
}
```

本来は `entgql.Transactioner` と組み合わせ、**GraphQL の mutation を自動でトランザクションに包む**ための部品です（公式 `tutorial-todo-gql-tx-mutation.md`）。

題材アプリは `Transactioner` を使っていません。代わりに、第 9 章で見た `RunInTransaction` が `OpenTx` を呼んでいます。トランザクションの範囲を Usecase が明示的に決める設計だからです。

tx の中で作った ent のエンティティを、そのまま Resolver から返すときは注意が必要です。エンティティは tx 用のクライアントを抱えているので、コミット後に `gql_edge.go` のフォールバック（`QueryXxx().Only(ctx)`）が走ると、閉じた tx でクエリしようとしてエラーになります（第 9 章）。

---

## 13.14 使っていない機能

entgql には題材アプリが使っていない機能もあります。読むときに混乱しないよう挙げておきます。

- **Mutation 入力の生成**（`entgql.Mutations()`）— `CreateXxxInput` を自動生成する。題材アプリの mutation の入力型は手書き
- **`Transactioner`** — 13.13 のとおり
- **`node` / `nodes`** — 13.3 のとおり削除済み

---

## まとめ

- entgql は **ent スキーマから GraphQL スキーマ（`ent.graphqls`）を生成**し、gqlgen と橋渡しする
- `ent.graphqls` は**手編集禁止**。独自のフィールドは手書きの `.graphqls` で `extend type` して足す（declaration merging に近い）
- `autobind` で **Resolver は ent の型をそのまま返せる**。エッジは `gql_edge.go` が自動で解く
- `QueryField` は **型レベルでだけ効く**。`RelayConnection` + `Paginate` で Relay 形式のページングが 1 行
- `Paginate` は **`totalCount` / `pageInfo` を要求されたときだけ COUNT を流す**
- 並び替えは `OrderField` / `MultiOrder`、絞り込みは `WhereInput`。どちらも Resolver は `WithXxx...` を渡すだけ
- entgql の既定は**全部公開**。Query の入口が無くても**エッジ経由で届く**。出さないものは `Skip` で消す
- **ディレクティブで守ったエッジは `SkipWhereInput` も付ける**
- ent の型を変えても API を保つには **`Skip` + `extend type` + `resolver: true`**。Skip で WhereInput も消える点に注意
- ディレクティブは**型レベルだと、その型を返す全フィールドに波及**する。範囲を絞るなら `QueryField().Directives`
- **N+1 は CollectFields が eager load を足して防ぐ**。`Paginate` は自動、それ以外は明示的に呼ぶ
- node / nodes、Transactioner、Mutation 入力生成は使っていない

---

[← 前: マイグレーション](12-migrations.md) | [目次](../README.md) | [次: アーキテクチャの中の ent →](14-architecture.md)
