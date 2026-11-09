# 1. なぜ ent はこういう形なのか

> **この章でわかること**
> ORM の大まかな系譜と、その中での ent の立ち位置。「スキーマを Go のコードで書き、残りを生成する」という発想。データを「グラフ」として見るということ。Prisma / TypeORM との対応関係。

コードはほとんど出てきません。ent を触る前の「地図」を手に入れる章です。

---

## 1.1 ORM が解こうとしている問題

アプリケーションのコードはオブジェクトで考えます。DB はテーブルと行で考えます。

この 2 つの世界の間には、どうしても翻訳が要ります。

- 行をオブジェクトに詰め替える
- オブジェクトの変更を `UPDATE` 文にする
- 「このユーザーの記事一覧」を `JOIN` やサブクエリにする

これを毎回手で書くと、退屈で、間違えやすく、型も付きません。

```ts
// 手書き SQL の典型的な辛さ（TS の例）
const rows = await db.query(
  "SELECT * FROM collections WHERE id = ?", [id]
);
const p = rows[0] as Collection; // ← 祈り
```

`as Collection` は**ただの祈り**です。列名を typo しても、列を足し忘れても、コンパイラは何も言いません。

ORM（Object-Relational Mapper）は、この翻訳を肩代わりする道具です。

---

## 1.2 ORM の 3 つの系統

TypeScript の世界で見かける ORM は、大きく 3 系統に分けられます。

| 系統 | 代表例 | 型の出どころ |
| --- | --- | --- |
| クエリビルダ | Knex, Kysely | 手書きの型定義 |
| 実行時マッピング | TypeORM, Sequelize | デコレータ＋実行時の反射 |
| コード生成 | Prisma | スキーマから生成 |

### クエリビルダ

SQL をメソッドチェーンで組み立てるだけの薄い層です。自由度は高いですが、「テーブルの形」は自分で型として書く必要があります。

### 実行時マッピング

TypeORM はクラスにデコレータを付けてテーブルを表します。

```ts
@Entity()
class Collection {
  @PrimaryGeneratedColumn() id: number;
  @Column() title: string;
}
```

手軽ですが、リレーションの読み込み漏れなどは**実行時まで分からない**ことが多い系統です。

### コード生成

Prisma は `schema.prisma` という専用 DSL からクライアントを生成します。

```prisma
model Collection {
  id    Int    @id
  title String
}
```

`prisma generate` を実行すると、`prisma.collection.findMany()` のような**完全に型付けされた API** ができあがります。

**ent はこの 3 番目、コード生成の系統です。**

---

## 1.3 ent の発想 — Schema As Code

ent の README は、自分の特徴をこう書いています（`entgo.io/ent` の README より）。

- **Schema As Code** — DB スキーマを Go のオブジェクトとして書く
- **Easily Traverse Any Graph** — グラフ構造を簡単に辿れる
- **Statically Typed And Explicit API** — コード生成による 100% 静的型付けの明示的な API
- **Extendable** — Go テンプレートで拡張できる

Prisma との最大の違いは 1 つ目です。

> **スキーマを専用 DSL ではなく、普通の Go のコードで書く。**

```go
// ent/schema/collection.go（題材アプリ。抜粋）
func (Collection) Fields() []ent.Field {
  return []ent.Field{
    field.Uint64("id"),
    field.String("hash").Unique(),
    field.String("title"),
  }
}
```

これは「設定ファイル」に見えますが、**ただの Go の関数**です。関数を呼んで値を返しているだけです。

普通のコードなので、次のことができます。

- 共通部品を関数や構造体にまとめて使い回せる（第 6 章の Mixin）
- コメント、定数、ヘルパー関数を自由に使える
- IDE の補完・ジャンプがそのまま効く

Prisma で「この 3 モデルに同じ列を足したい」と思ったら、コピペするしかありません。ent ならそれを関数にできます。

---

## 1.4 なぜコード生成なのか

「Go のコードで書けるなら、生成なんてせず実行時に解釈すればいいのでは？」と思うかもしれません。TypeORM がそうしているように。

これは Go という言語の事情が大きく関わっています。

TypeScript には、型を自在に組み立てる強力な仕組みがあります。

```ts
// Prisma の型はこういう芸当で成り立っている
type Select<T> = { [K in keyof T]?: boolean };
```

Mapped types、conditional types、テンプレートリテラル型…。TS の型システムは「型で計算する」ことができます。

**Go にはこれがありません。** ジェネリクスはありますが、「構造体のフィールド名から関数を作る」ような芸当はできません。

すると、`TitleEQ("foo")` のような**フィールドごとの関数**を型安全に用意する手段は、事実上 1 つしかありません。

> **あらかじめ全部書いておく。つまり、生成する。**

ent はスキーマを読み、エンティティごと・フィールドごとに大量の Go コードを書き出します。テーブルが 100 を超えるような API なら、`ent/` 配下に同じ数だけのパッケージが生成されます。

人間が書くのはスキーマだけ。残りは機械が書く。これが ent の基本姿勢です。

---

## 1.5 データを「グラフ」として見る

ent の公式ドキュメントは、最初の原則としてこう掲げています（`doc/md/getting-started.mdx`）。

> Easily model database schema as a graph structure.
> （DB スキーマをグラフ構造として簡単にモデリングできる）

ここでいう「グラフ」は、折れ線グラフではなく**ノードと辺**のグラフです。

| グラフの言葉 | ent の言葉 | DB の言葉 |
| --- | --- | --- |
| ノード | エンティティ | テーブルの行 |
| ノードの属性 | フィールド | 列 |
| 辺 | エッジ | 外部キー・中間テーブル |

ent のスキーマは、`Fields()`（属性）と `Edges()`（辺）の 2 本立てで書きます。

```go
func (Collection) Edges() []ent.Edge {
  return []ent.Edge{
    edge.To("articles", Article.Type).
      Through("collection_items", CollectionItem.Type),
  }
}
```

この視点が効いてくるのはクエリを書くときです。

```go
// 「このコレクションに入っている記事」を辿る
client.Collection.Query().
  Where(collection.ID(id)).
  QueryArticles().
  All(ctx)
```

`JOIN` を意識せず、**ノードからノードへ辺を辿る**ように書けます。これを ent は「トラバーサル」と呼びます（第 7 章）。

GraphQL を使ったことがあれば、この感覚は馴染みがあるはずです。GraphQL もデータをグラフとして扱います。ent が GraphQL と相性がいい（第 13 章）のは偶然ではありません。

---

## 1.6 Prisma と並べてみる

TS 使いにとって一番近いのは Prisma です。対応関係を表にしておきます。

| やりたいこと | Prisma | ent |
| --- | --- | --- |
| スキーマを書く | `schema.prisma` | `ent/schema/*.go` |
| コード生成 | `prisma generate` | `go generate`（題材アプリは `make gen`） |
| クライアント | `PrismaClient` | `*ent.Client` |
| 取得 | `findMany({ where })` | `Query().Where(...).All(ctx)` |
| リレーション読み込み | `include` | `WithXxx()` |
| マイグレーション | `prisma migrate` | Atlas（第 12 章） |
| ミドルウェア | `$extends` | Hook / Interceptor（第 11 章） |

書き味の違いも見ておきましょう。

```ts
// Prisma: オブジェクトで条件を渡す
prisma.collection.findMany({
  where: { title: "foo" },
  include: { articles: true },
});
```

```go
// ent: メソッドチェーンと関数で条件を渡す
client.Collection.Query().
  Where(collection.TitleEQ("foo")).
  WithArticles().
  All(ctx)
```

Prisma はオブジェクトリテラル、ent は関数呼び出しです。表現の仕方は違いますが、**「生成された型付き API で書く」という点は同じ**です。

`collection.TitleEQ` は ent が生成した関数で、`ent/collection/where.go` に生成されます。引数は `string` 型なので、数値を渡せばコンパイルエラーになります。

---

## 1.7 ent が「しない」こと

ent の性格を知るには、何をしないかを知るのも大事です。

### 暗黙のリレーション読み込みをしない

TypeORM の lazy relation のように、プロパティに触った瞬間に裏でクエリが飛ぶ、ということはありません。

**読み込みたいエッジは `WithXxx()` で明示します。** 明示しなければ、エッジは空のままです。

これは不便に見えますが、「どこで何本クエリが飛ぶか」がコードから読み取れるという利点があります。N+1 問題を見つけやすくなります。

### 変更追跡をしない

TypeORM の `save()` のように「オブジェクトを書き換えて保存すれば差分を検出してくれる」仕組みはありません。

```go
// 更新は「何を変えるか」を明示する
client.Collection.UpdateOneID(id).
  SetTitle("新しいタイトル").
  Save(ctx)
```

取得したエンティティ（`*ent.Collection`）は**ただのデータの入れ物**です。そのフィールドを書き換えても DB には何も起きません。

### SQL を完全には隠さない

ent は SQL を抽象化しますが、逃げ道もちゃんと用意しています。

- `sql.Selector` を直接いじる述語やモディファイア（第 9 章）
- 生 SQL の実行（第 9 章）

「ORM で書けないものは SQL で書けばいい」という現実的な設計です。実運用の API でも、集計や複雑な JOIN でこの逃げ道を使う場面は珍しくありません。

---

## 1.8 題材アプリにおける ent

最後に、この本の題材アプリで ent がどう位置付けられているかを押さえておきます。

題材アプリは、架空の記事投稿サービスの GraphQL API サーバーです。DB は MySQL です。

ent はここで 3 つの役割を担っています。

1. **DB アクセス層** — Repository 層から MySQL を読み書きする
2. **マイグレーションの源** — スキーマの差分から SQL を生成する（第 12 章）
3. **GraphQL スキーマの源** — entgql 拡張で `ent.graphqls` を生成する（第 13 章）

3 つ目は特に重要です。題材アプリでは、ent のスキーマを変えると **GraphQL の API も変わります**。ent のスキーマは DB と API の両方の「正」になっています。

一方で、ent に依存してよい場所は厳しく制限されています。

```
Resolver 層   ← ent の構造体を返してよい
Usecase 層    ← ent 禁止
Repository 層 ← ent はここに閉じ込める
```

（題材アプリのチームの規約より）

なぜこう分けるのか、どう守らせているのかは第 14 章で扱います。今は「ent はどこでも使ってよいわけではない」とだけ覚えておいてください。

---

## まとめ

- ORM は「オブジェクトの世界」と「テーブルの世界」の翻訳係
- ent は Prisma と同じ**コード生成**系の ORM。ただしスキーマは専用 DSL ではなく**普通の Go のコード**で書く
- コード生成を選ぶのは、Go の型システムでは TS のような型計算ができないから。**フィールドごとの関数を全部生成する**
- ent はデータを**ノード（エンティティ）とエッジ**のグラフとして扱う。クエリは「辺を辿る」ように書ける
- 暗黙のリレーション読み込みも変更追跡もしない。**何をするかは常に明示**
- 題材アプリでは ent のスキーマが **DB・マイグレーション・GraphQL の 3 つの正**になっている

---

[目次](../README.md) | [次: TypeScript 使いのための Go →](02-go-for-ts.md)
