# 14. アーキテクチャの中の ent

> **この章でわかること**
> 題材アプリのレイヤ構造と、ent をどの層に閉じ込めているか。なぜ閉じ込めるのか。Entity が ent の構造体を包んでいる理由。それを機械的に守らせる lint。残りがちな「例外」と、ent を剥がすときのチェックリスト。

ここまでは ent そのものの話でした。この章では視点を一段上げて、**アプリの中で ent がどこに置かれているか**を見ます。

実務で一番よく問われるのは「この ent のコード、どのファイルに書けばいいの？」です。その答えがこの章です。

---

## 14.1 4 つの層

題材アプリは次の 4 層で組まれています。

```
Resolver 層   graph/
   ↓
Usecase 層    usecase/
   ↓
Repository 層 infrastructure/db/
   ↓
Ent 層        ent/
```

NestJS で言えば、だいたいこう対応します。

| 題材アプリ | NestJS |
| --- | --- |
| Resolver | `@Resolver()` クラス |
| Usecase | `@Injectable()` の Service |
| Repository | Repository クラス |
| Ent | Prisma Client / TypeORM |

違いは 1 点。**ent を触ってよいのは Repository 層まで**、というルールが厳しく決まっていることです。

> Usecase 層での ent 参照禁止。
> （チームの規約）

NestJS の Service で `this.prisma.user.findMany()` を直接呼ぶのは、よくある書き方です。題材アプリではそれを禁止しています。

---

## 14.2 なぜ閉じ込めるのか

理由は 3 つあります。

### (1) ビジネスロジックをテストしやすくする

Usecase が ent を直接呼ぶと、Usecase のテストに DB が要ります。

Repository の**インターフェース**越しに呼んでいれば、テストではそれをモックに差し替えられます（第 15 章）。Jest で `jest.mock('./userRepository')` するのと同じ発想です。

### (2) ent の都合をビジネスロジックに漏らさない

ent の型には ent の事情が詰まっています。

- `Nillable` ならポインタ、そうでなければ値（第 4 章）
- エッジは `Edges` に入っていたり、いなかったり（第 7 章）
- エラーは `*ent.NotFoundError` など ent 固有の型（第 10 章）

これが Usecase に入り込むと、**「ent をやめたい」「列を変えたい」がビジネスロジックの書き換えになります**。

### (3) DB アクセスの場所を 1 か所に集める

N+1 やインデックスの効かないクエリは、DB アクセスが散らばっていると見つけにくくなります。Repository 層に集めておけば、そこだけ見ればよくなります。

---

## 14.3 Repository — ent が「住む」場所

ent のクエリは `infrastructure/db/repository/` に書きます。

まず port（インターフェース）があります。Usecase はこちらにだけ依存します。

```go
// domain/ports/
type ArticleRepository interface {
  // 記事を取得
  GetArticleByID(ctx context.Context,
    id uint64) (*models.ArticleEntity, error)
  // …
}
```

**引数にも戻り値にも ent の型が出てきません。** 出てくるのは `models.ArticleEntity`（後述）だけです。

実装はこうです。

```go
// infrastructure/db/repository/
type articleRepository struct {
  client *ent.Client
}

func (r *articleRepository) GetArticleByID(
  ctx context.Context, id uint64,
) (*models.ArticleEntity, error) {
  client := Client(ctx, r.client)
  a, err := client.Article.Query().
    Where(article.ID(id)).First(ctx)
  if err != nil {
    return nil, wrapEntError(err)
  }
  return models.ReconstructArticleEntity(a), nil
}
```

ent の結果を受け取ったら、その場で 2 つの変換をして外に出します。

- **値**: `*ent.Article` → `*models.ArticleEntity`
- **エラー**: ent のエラー → `*database.Error`（`wrapEntError`。第 10 章）

ent の型は、この関数の外に一切出ていきません。

### `Client(ctx, r.client)` を忘れない

冒頭の `Client(ctx, r.client)` は第 9 章で見たトランザクションの仕掛けです。

```go
func Client(ctx context.Context,
  baseClient *ent.Client) *ent.Client {
  c := ent.FromContext(ctx)
  if c != nil {
    return c
  }
  return baseClient
}
```

ctx にトランザクション版の client が入っていればそちらを、なければ素の client を使います。

**`r.client` を直接使うと、Usecase が `RunInTransaction` で囲んでいてもトランザクションの外で実行されます。** エラーにならないので気づきにくい、一番静かな事故です。

### 1 つの大きな Repository

port は細かく分かれていますが、実装は 1 つの構造体に埋め込みでまとめています。

```go
type entRepository struct {
  *transactionManager
  *userRepository
  *articleRepository
  // … テーブルの数だけ続く
}
```

Go の埋め込み（第 2 章）で、各 Repository のメソッドがすべて `entRepository` のメソッドになります。これで `RunInTransaction` も各 Repository メソッドも、1 つの値 `dbRepo` から呼べます。

port 側も同じ形です。個別の port を 1 つの大きなインターフェースに埋め込んでおき、Usecase はそれを受け取ります。

```go
// domain/ports/
type DBRepository interface {
  TransactionManager
  UserRepository
  ArticleRepository
  // …
}
```

新しい Repository を足したら、**この大きなインターフェースへの埋め込みも忘れずに**。忘れると `dbRepo` 経由で呼べず、モック（第 15 章）にも現れません。

---

## 14.4 QueryService — 読み取り専用の窓口

ent を使ってよい場所はもう 1 つあります。**QueryService** です。

```go
// domain/ports/
type QueryService interface {
  ArticleQueryService
  FeedQueryService
  // …
}
```

Repository が「1 つのエンティティを読み書きする」窓口なのに対し、QueryService は**画面の都合に合わせた複雑な読み取り**の窓口です。

たとえば `FeedQueryService` の戻り値は、記事・タグ・既読かどうかをまとめた `Feed` 型です。1 つのエンティティには収まらないので、Repository ではなく QueryService に置きます。

CQRS の「Query 側」を軽く切り出したもの、と考えてください。こちらも port の戻り値に ent の型は出しません。

---

## 14.5 Entity — ent の構造体を包む

`models.ArticleEntity` の中身を見てみます。

```go
// domain/models/
type ArticleEntity struct {
  model *ent.Article   // 小文字 = 非公開
  v     *ArticleValidation
}

func (e *ArticleEntity) Title() string {
  if e.model == nil {
    return ""
  }
  return e.model.Title
}
```

**Entity は ent の構造体を 1 つだけ「非公開フィールド」として抱えています。** 外からは Getter でしか値を読めません。

TS で書くとこうです。Prisma が生成した型を private に持つクラスです。

```ts
class ArticleEntity {
  #model: PrismaArticle  // private
  get title() {
    return this.#model?.title ?? ""
  }
}
```

### 書き換えは Setter だけ

フィールドが非公開なので、外から `e.model.Title = "…"` とは書けません。値を変えたいときは Entity の Setter を通します。

```go
func (e *ArticleEntity) SetTitle(
  title string,
) error {
  if err := e.v.Title(title); err != nil {
    return err
  }
  e.model.Title = title
  return nil
}
```

**書き換えの入口を絞ることで、検証を必ず通す**ようにしています。TS の `private` + setter と同じ考え方です。

### 作り方は 2 通り

```go
// 新しく作る（入力を検証する）
func NewArticleEntity(
  dto NewArticleEntityDTO,
) (*ArticleEntity, error)

// DB から読んだものを包み直す（検証しない）
func ReconstructArticleEntity(
  m *ent.Article,
) *ArticleEntity
```

`New` は DTO の `Validate()` を通します。ユーザー入力から作るからです。

`Reconstruct` は検証しません。DB に入っている時点で正しいはずだからです。Repository が使うのはこちらです。

### なぜ ent の構造体を「捨てない」のか

普通のクリーンアーキテクチャなら、ent の構造体とは別にドメインの構造体を定義して、詰め替えます。題材アプリはそうせず、**ent の構造体を中に持ったまま**にしています。

理由は Resolver 層にあります。

---

## 14.6 Resolver — ent の型を返す例外

Resolver は Usecase から Entity を受け取り、**`Model()` で ent の構造体を取り出して返します**。

```go
// graph/
func (r *queryResolver) Collection(
  ctx context.Context, id uint64,
) (*ent.Collection, error) {
  entity, err :=
    r.Usecase.GetCollection(ctx, id)
  if err != nil {
    return nil, err
  }
  return entity.Model(), nil
}
```

GraphQL の `Collection` 型は、gqlgen の autobind で ent の `Collection` 構造体に結び付いています（第 13 章）。

ent の構造体を返すと、**その先のフィールド解決を entgql の生成コードに任せられます**。たとえばエッジは生成された `gql_edge.go` のメソッドで解決され、eager load 済みなら追加クエリを投げません。

```go
// ent/gql_edge.go（生成コード）
func (c *Comment) Author(
  ctx context.Context,
) (*User, error) {
  result, err := c.Edges.AuthorOrErr()
  if IsNotLoaded(err) {
    result, err = c.QueryAuthor().Only(ctx)
  }
  return result, err
}
```

自前のドメイン構造体に詰め替えてしまうと、この仕組みが使えません。**ent の構造体を Entity の中に持ち続けるのは、最後に Resolver で取り出すため**です。

ただし規約には条件が付いています。

> レスポンス用途に限定し、ビジネスロジックでは `models` を使う
> （チームの規約）

`Model()` は「GraphQL に返すための出口」です。Resolver の中で `Model().Title` を読んで分岐を書く、といった使い方はしません。

### 依存の全体像

```
Resolver ── ent 型を返してよい
   │  Entity
Usecase ─── ent 禁止（models のみ）
   │  port（interface）
Repository ─ ent を使う。Entity と
   │          database.Error に変換
ent
```

---

## 14.7 境目の変換は 1 か所に置く

DB の表現とドメインの表現が食い違うことがあります。

たとえば、既存 DB から取り込んだ列が整数型の 0/1 で、ドメインでは `bool` として扱いたい場合です（第 4 章）。

このとき、変換をあちこちに書くと「0 以外は true？ 1 だけ true？」の解釈が散らばります。

```go
// 0/1 と bool の対応はここが正本。
// 他の箇所はこの定数を参照する。
const flagOn = 1

func flagToBool(v int) bool {
  return v == flagOn
}
```

- **変換関数は 1 か所**に置き、Repository の境目で使う
- 「なぜこの値なのか」の説明は**定数のコメントを正本**にし、他のコメントからはそこを参照する

小さなことですが、DB とドメインの境目を Repository に閉じ込める設計と同じ考え方です。

---

## 14.8 lint で強制する

「Usecase で ent を使わない」は、ルールとして書いておくだけでは守られにくいものです。

そこで題材アプリでは、**層の依存を検査する自作 lint** で機械的に止めています。

```bash
go run ./tools/<lint 名> ./...
```

`make lint` と CI に組み込んでおけば、違反があればマージできません。

### 対象

検査対象はパッケージパスの規則で決めます。

- パスの要素に `usecase` を含むパッケージ（サブパッケージ含む）
- ドメインサービスのパッケージとその配下

この方式だと、新しい usecase パッケージを作るときに**名前に `usecase` を含めないと検査の網から外れます**。命名規則とセットで運用します。

### import を見るだけでは足りない

この lint は import 文**だけでなく**、**式の型**も見ます。import しなくても ent の型は入り込めるからです。

```go
// ent を import していなくても違反
a := entity.Model()   // *ent.Article 型
t := a.Title
```

検出するのは次の 3 種類です。

- (a) ent パッケージの import
- (b) 型に ent 由来の型を含む式
- (c) ent で宣言されたフィールド・メソッドの選択

「ent」とみなすのは、生成コード（`example.com/app/ent` 配下）と ent 本体（`entgo.io/ent` 配下）の両方です。`entgo.io/ent/dialect/sql` で述語を組むのも ent 依存だからです。

TS で言えば、ESLint の `no-restricted-imports` を型情報付きで強化したもの、というイメージです。

Go では `golang.org/x/tools/go/analysis` を使うと、こうした型情報付きの lint を比較的簡単に書けます。

---

## 14.9 DI — client はどこから来るのか

`*ent.Client` は起動時に 1 つだけ作ります（`store/` の DB 接続処理）。それを依存を組み立てる factory が各部品に配ります。

```go
func (f *DependencyFactory) CreateDependencies(
  ctx context.Context,
) Dependencies {
  dbRepo := repository.New(f.client)
  queryService :=
    queryservice.New(f.client)
  // …
}
```

Repository と QueryService は client を受け取りますが、Usecase が受け取るのはそれらの **port** です。

```go
// usecase/
type articleUsecase struct {
  dbRepo ports.DBRepository
}
```

NestJS のように DI コンテナが自動で注入するのではなく、**コンストラクタ（`New` 関数）に手で渡していく**のが Go の流儀です。

---

## 14.10 残りがちな例外

理想は「ent は Repository と QueryService だけ」ですが、長く運用しているアプリではそうなりきらないことがあります。題材アプリにも次のような例外が残っています。

### Resolver が client を直接持つ

```go
// graph/
type Resolver struct {
  Client  *ent.Client
  Usecase usecase.Usecase
  // …
}
```

`r.Client` で直接クエリを組む Resolver の典型は、entgql の `Paginate` に引数をそのまま渡すだけのものです（第 13 章）。

```go
func (r *queryResolver) Articles(…) (
  *ent.ArticleConnection, error,
) {
  return r.Client.Article.Query().
    Paginate(ctx, after, first,
      before, last,
      ent.WithArticleFilter(where.Filter),
      ent.WithArticleOrder(orderBy))
}
```

これ自体はロジックを持たないので、テストの対象外にしても困りません。

一方、Resolver で `Query().Where(…).Only(ctx)` を組んで分岐するような、ロジックを含むものが残っていることもあります。**新しく書くコードでは真似しないでください**。

### port 自体が ent を露出している

14.3 では「port の引数にも戻り値にも ent の型が出てこない」と書きましたが、例外になりがちな形があります。

- port のメソッドが `*ent.XxxQuery` を返し、呼び出し側で条件を足す
- port のメソッドが `func(*ent.XxxSelect)` を引数に取り、列の選び方を呼び出し側に任せる

どちらも「便利だから」と入りやすい形です。新しい port では ent の型を出さないでください。

---

## 14.11 ent を剥がすときのチェックリスト

古い Usecase には ent を直接使っているものがあり、少しずつ剥がしていくことになります。そのときの手順はこうです。

- [ ] Usecase から `ent` の import・`s.client` 直接アクセス・`entity.Model()` 呼び出しを消す
- [ ] DB アクセスとページネーション計算を Repository 層へ移す
- [ ] テストの DB 依存も剥がす。`enttest` やテスト用 DB のセットアップを消し、Repository を gomock でモックにする

3 つ目を忘れがちです。コードだけ移しても、テストが DB を使ったままなら「Usecase を DB なしでテストできる」という利点が手に入りません。

### 新しいエンティティの雛形

Entity・バリデーション・Repository の雛形を生成するコマンドを用意しておくと、層構造を崩さずに済みます。

ただし雛形のテンプレートは、ディレクトリ構成を変えたときに置いていかれがちです。**生成されたファイルの置き場所が現在の構成と合っているかは、生成後に確認してください**。

---

## まとめ

- 題材アプリは **Resolver → Usecase → Repository → ent** の 4 層。ent を触ってよいのは **Repository と QueryService だけ**
- 閉じ込める理由は、**Usecase を DB なしでテストできる**ことと、**ent の都合をビジネスロジックに漏らさない**こと
- Repository は ent の結果を **Entity** に、ent のエラーを **`database.Error`** に変換して返す。client は必ず **`Client(ctx, r.client)`** で取る
- Entity は ent の構造体を**非公開で抱え**、書き換えは Setter に絞って検証を通す。作り方は `New`（検証あり）と `Reconstruct`（検証なし）の 2 通り
- ent の構造体を捨てないのは、Resolver で **`Model()` で取り出して返し、entgql の最適化を効かせる**ため
- DB とドメインの表現の変換は **1 か所**に置く
- ルールは**自作 lint** が式の型レベルで強制する
- Resolver の `r.Client` 直使いは**過去の例外**。新しいコードでは書かない

---

[← 前: entgql — GraphQL との連携](13-entgql.md) | [目次](../README.md) | [次: テストの書き方 →](15-testing.md)
