# 2. TypeScript 使いのための Go

> **この章でわかること**
> ent のコードを読むのに必要な Go の知識だけ。パッケージ、struct とメソッド、ポインタと nil、エラー値、`context.Context`、interface、メソッドチェーン、クロージャ、ゼロ値、`go generate`。

Go の入門書ではありません。**ent と題材アプリを読むときにつまずく箇所だけ**を拾います。Go を知っている人は読み飛ばして構いません。

---

## 2.1 パッケージと import

Go のコードはすべて**パッケージ**に属します。ファイルの先頭で宣言します。

```go
package schema

import (
  "time"
  "entgo.io/ent"
  "entgo.io/ent/schema/field"
)
```

TS との対応はこうです。

| TS | Go |
| --- | --- |
| 1 ファイル = 1 モジュール | 1 ディレクトリ = 1 パッケージ |
| `export` を付ける | **名前を大文字で始める** |
| `import { x } from "y"` | `import "y"` して `y.X` |

2 行目が一番の違いです。Go に `export` キーワードはありません。

```go
func Client(...)   // 大文字 → 外から見える
func passThrough(...) // 小文字 → パッケージ内だけ
```

ent の生成コードを読むとき、`collection.TitleEQ` の `collection` は**パッケージ名**です。`ent/collection/` というディレクトリがあり、その中の関数を呼んでいます。

### import の別名

同じ名前のパッケージがぶつかるときは別名を付けます。

```go
import entcollection "example.com/app/ent/collection"
```

題材アプリの Repository では、ent のパッケージだと一目で分かるように `entcollection` のような別名を付けることがあります。

---

## 2.2 struct とメソッド

Go にクラスはありません。**struct（データ）と、それに紐づく関数（メソッド）**があるだけです。

```go
type Collection struct {
  ent.Schema
}

func (Collection) Fields() []ent.Field {
  return []ent.Field{ /* ... */ }
}
```

`func (Collection) Fields()` の括弧部分を**レシーバ**と呼びます。「`Collection` 型に `Fields` メソッドを生やす」という意味です。

TS で書くとこう考えればよいです。

```ts
class Collection extends Schema {
  fields(): Field[] { return [/* ... */]; }
}
```

### 埋め込み（embedding）

上の `ent.Schema` は、フィールド名なしで型だけ書いています。これは**埋め込み**です。

埋め込んだ型のメソッドは、外側の型のメソッドとしても使えます。継承に似ていますが、正確には「委譲を自動で書いてくれる」仕組みです。

ent のスキーマは必ず `ent.Schema` を埋め込みます。`ent.Schema` が `Fields()` や `Edges()` などの**空のデフォルト実装**を持っているので、必要なメソッドだけ書けば済みます。

### レシーバの変数名

レシーバに名前を付けることもあります。

```go
func (d SoftDeleteMixin) Hooks() []ent.Hook {
  // ここで d を使える（TS の this に相当）
}
```

`this` の代わりに、自分で好きな名前を付ける、と覚えてください。

---

## 2.3 ポインタと nil

ent を読むうえで**一番大事な節**です。

Go の変数は、基本的に**値そのもの**を持ちます。

```go
var title string    // 必ず何かの文字列が入る
var t *string       // 文字列への「ポインタ」
```

`*string` は「文字列がある場所を指す矢印」です。矢印は**どこも指していない（`nil`）**ことができます。

| TS | Go |
| --- | --- |
| `string` | `string` |
| `string \| null` | `*string` |
| `null` / `undefined` | `nil` |

ここが TS と大きく違うところです。

> **Go の `string` は null になれない。「値がないかもしれない」を表すにはポインタを使う。**

ent が生成する構造体を見てみます。

```go
// ent/collection.go（生成物。抜粋）
type Collection struct {
  Title       string
  PublishedAt *time.Time
}
```

`Title` は必ず値がある。`PublishedAt` は `nil` のことがある。この差は、スキーマで `Nillable()` を付けたかどうかで決まります（第 4 章）。

### & と *

```go
s := "hello"
p := &s     // & = 「s の場所」を取る → *string
fmt.Println(*p) // * = 矢印の先の値を読む
```

`nil` のポインタに `*` を付けると**実行時エラー（panic）**になります。TS で `undefined.foo` をしたときと同じです。

```go
if c.PublishedAt != nil {
  fmt.Println(*c.PublishedAt)
}
```

nil チェックしてから読む。TS の `?.` のような短縮記法は Go にはありません。

### もう 1 つのポインタの用途

ポインタは「nil になれる」以外に、**同じ実体を共有する**ためにも使います。

```go
client := ent.NewClient(...) // *ent.Client を返す
```

`*ent.Client` のように、大きな構造体や状態を持つオブジェクトは、ほぼ必ずポインタで受け渡しします。TS ではオブジェクトは常に参照渡しなので、**「Go でも普通のオブジェクトはポインタ」**と思っておけば大体合っています。

---

## 2.4 多値返却とエラー

Go に例外（`throw`）はありません。エラーは**戻り値**で返します。

```go
c, err := client.Collection.Get(ctx, id)
if err != nil {
  return nil, err
}
// ここから先は c を安全に使える
```

関数は複数の値を返せます。慣習として**最後の戻り値が `error`** です。

TS で言えば、全関数がこう書かれている世界です。

```ts
const [c, err] = await getCollection(id);
if (err) return [null, err];
```

`if err != nil { return ... }` は Go で最も多く書かれる 3 行です。最初は冗長に感じますが、**どこでエラーが起きうるかが全部見える**のが利点です。

### エラーを包む

```go
return fmt.Errorf("commit failed: %w", err)
```

`%w` で元のエラーを包みます。JS の `new Error(msg, { cause })` に相当します。

包まれたエラーは `errors.Is` / `errors.As` で中を調べられます。題材アプリではこれを使ってエラーを分類しています（第 10 章）。

### X で終わるメソッド

ent の生成コードには、`Get` と `GetX`、`All` と `AllX` のようにペアがあります。

```go
c, err := client.Collection.Get(ctx, id) // エラーを返す
c := client.Collection.GetX(ctx, id)     // エラーなら panic
```

`X` 付きは**エラーを返さず panic する**版です。テストや使い捨てスクリプト向けで、アプリのコードでは使いません。

---

## 2.5 context.Context

ent のほぼすべてのメソッドが、第 1 引数に `ctx` を取ります。

```go
client.Collection.Query().All(ctx)
```

`context.Context` は、**1 つのリクエストに付いて回る「荷物」**です。役割は 2 つあります。

1. **キャンセル** — リクエストが切れたら、DB クエリも止める
2. **値の受け渡し** — リクエストごとの値（ログインユーザー、トランザクションなど）を運ぶ

1 つ目は JS の `AbortSignal` に近いものです。

2 つ目は、Node.js の `AsyncLocalStorage` を**引数で明示的に回す**版、と考えると分かりやすいです。

```go
ctx = context.WithValue(ctx, key, value) // 荷物を足す
v := ctx.Value(key)                      // 荷物を取り出す
```

`ctx` は不変です。`WithValue` は**新しい ctx を返す**ので、元の ctx は変わりません。

### ent は ctx に何を載せるか

題材アプリの ent まわりでは、この仕組みを 2 か所で使っています（ほかに認証情報なども、ミドルウェアが ctx に載せています）。

- **トランザクション** — ctx にトランザクション用のクライアントを載せ、Repository で取り出す（第 9 章）
- **論理削除のスキップ** — ctx に「削除済みも含めて」という印を載せる（第 11 章）

```go
// infrastructure/db/repository/
//   transaction_manager.go（抜粋）
func Client(ctx context.Context, base *ent.Client) *ent.Client {
  c := ent.FromContext(ctx)
  if c != nil {
    return c
  }
  return base
}
```

「ctx にクライアントが載っていればそれを使い、なければ通常のクライアントを使う」。ctx が**暗黙の引数の通り道**になっていることが分かります。

---

## 2.6 interface と型アサーション

Go の interface は、TS の interface と同じく**構造的**です。

```go
type Stringer interface {
  String() string
}
```

`String() string` メソッドを持っている型は、宣言なしで自動的に `Stringer` を満たします。`implements` は書きません。TS の型チェックと同じ感覚です。

### 型アサーション

interface 型の値から、中身の具体的な型を取り出すのが**型アサーション**です。

```go
mx, ok := m.(interface {
  SetDeletedAt(time.Time)
})
if !ok {
  return nil, fmt.Errorf("unexpected type %T", m)
}
mx.SetDeletedAt(time.Now())
```

（題材アプリの `ent/common/softdelete_mixin.go` を簡略化）

`m.(T)` は「`m` が `T` として扱えるなら取り出す」です。`ok` が `false` なら取り出せなかったということです。

TS で言えば型ガードです。

```ts
if ("setDeletedAt" in m) { m.setDeletedAt(new Date()); }
```

上の例では、その場で無名の interface を書いて「`SetDeletedAt` を持っていれば」と判定しています。第 11 章で再登場します。

### any

`any` は `interface{}` の別名で、**何でも入る型**です。TS の `unknown` に近く、使う前に型アサーションが必要です。

---

## 2.7 メソッドチェーンと行末のドット

ent のコードはメソッドチェーンだらけです。

```go
field.Time("published_at").
  Optional().
  Nillable().
  Comment("公開日時")
```

**ドットが行頭ではなく行末にある**ことに注意してください。

TS ではこう書くのが普通です。

```ts
builder
  .optional()
  .nillable();
```

Go では行末に `)` があると、そこで文が終わったと解釈されます（セミコロンの自動挿入）。そのため**ドットを行末に置いて「まだ続く」と示す**必要があります。

grep で `.Optional(` を探しても見つからないことがあるのはこのためです。行頭の `Optional(` も探してください。

---

## 2.8 クロージャと関数型

Go の関数は値です。TS のアロー関数と同じように渡せます。

```go
WithCollectionItems(func(q *ent.CollectionItemQuery) {
  q.Order(collectionitem.ByPosition())
})
```

ent は**「設定を関数で受け取る」**スタイルを多用します。

- eager loading の絞り込み（第 7 章）
- Hook / Interceptor（第 11 章）
- トランザクションの中身（第 9 章）

```go
RunInTransaction(ctx, func(ctx context.Context) error {
  // この中が 1 つのトランザクション
  return nil
})
```

TS の `prisma.$transaction(async (tx) => { ... })` と同じ形です。

### 可変長引数

```go
func (q *CollectionQuery) Where(ps ...predicate.Collection)
```

`...T` は可変長引数です。TS の `...ps: T[]` と同じです。`Where(a, b, c)` のように複数渡せて、AND で結合されます。

---

## 2.9 ゼロ値

Go の変数は、宣言しただけで**必ず何かの値**を持ちます。

| 型 | ゼロ値 |
| --- | --- |
| `int`, `uint64` | `0` |
| `string` | `""` |
| `bool` | `false` |
| `time.Time` | `0001-01-01 00:00:00` |
| ポインタ, interface, slice, map | `nil` |

TS では未初期化は `undefined` ですが、Go では型ごとの「空っぽ」が入ります。

これが ent で重要になる場面があります。

```go
// Article の title は NULL 許容の列（Optional のみ）
type Article struct {
  Title string
}
```

DB の値が空文字なのか、そもそも NULL なのか。`string` 型だと**どちらも `""` になって区別できません**。区別したいならポインタ（`*string`）にする必要があります。

この「ゼロ値と NULL の区別」が、第 4 章の `Optional` と `Nillable` の話につながります。

---

## 2.10 go generate

Go には、コード生成のための仕組みが言語ツールに組み込まれています。

```go
// generate.go
package main

//go:generate go run -mod=mod ./ent/entc.go
//go:generate go run -mod=mod github.com/99designs/gqlgen
```

`//go:generate` で始まるコメントは、ただのコメントではありません。`go generate` コマンドを実行すると、**このコメントに書かれたコマンドが実行されます**。

`package.json` の `scripts` をソースコードの中に書いたもの、と考えてください。

```json
{ "scripts": { "generate": "prisma generate" } }
```

題材アプリでは `make gen` がこれを（差分判定付きで）呼び出します。詳しくは次章で扱います。

### build タグ

```go
//go:build ignore

package main
```

題材アプリの `ent/entc.go` の先頭にはこう書かれています。`//go:build ignore` は「通常のビルドに含めない」という指示です。

コード生成用のスクリプトは `go run` で単体実行するもので、アプリ本体に混ざっては困るからです。

---

## 2.11 早見表

最後に、この章の内容を 1 枚にまとめます。

| 概念 | TS | Go |
| --- | --- | --- |
| 公開 | `export` | 大文字始まり |
| クラス | `class` | struct ＋ メソッド |
| `this` | `this` | レシーバ（名前は自由） |
| null 許容 | `T \| null` | `*T`（ポインタ） |
| 例外 | `throw` | 戻り値の `error` |
| エラーの原因 | `{ cause }` | `fmt.Errorf("%w")` |
| リクエスト文脈 | `AsyncLocalStorage` | `context.Context` |
| 型ガード | `in` / `instanceof` | 型アサーション `x.(T)` |
| unknown | `unknown` | `any` |
| アロー関数 | `() => {}` | `func() {}` |
| 未初期化 | `undefined` | ゼロ値 |
| scripts | `package.json` | `//go:generate` |

---

## まとめ

- 公開・非公開は**名前の大文字・小文字**で決まる。`collection.TitleEQ` の `collection` はパッケージ名
- クラスはなく、**struct とメソッド**。ent のスキーマは `ent.Schema` を**埋め込んだ** struct
- **Go の `string` は null になれない**。値がないかもしれないなら `*string`（ポインタ）
- エラーは**戻り値**。`X` 付きメソッドはエラーの代わりに panic する
- `context.Context` は**リクエストに付いて回る荷物**。題材アプリではトランザクションや論理削除の制御に使う
- メソッドチェーンの**ドットは行末**
- Go の変数は必ず**ゼロ値**を持つ。ゼロ値と NULL を区別したいときにポインタが要る

---

[← 前: なぜ ent はこういう形なのか](01-why-ent.md) | [目次](../README.md) | [次: ent の全体像とコード生成 →](03-ent-overview.md)
