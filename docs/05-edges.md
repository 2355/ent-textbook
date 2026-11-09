# 5. エッジ — 関係を定義する

> **この章でわかること**
> ent が「関係」をグラフの辺（エッジ）として扱う理由。`edge.To` と `edge.From` の役割分担。O2O・O2M・M2M の書き分けと、外部キー列がどちらのテーブルに作られるか。`Field` による外部キーの明示、`Through` による中間テーブルのエンティティ化、自己参照、削除時の挙動。

第 1 章で、ent は**データを「グラフ」として捉える**と書きました。テーブルが頂点（ノード）、テーブル同士の関係が辺（エッジ）です。

この章では、そのエッジの書き方を扱います。ここを押さえると、実務のスキーマはほぼ読めるようになります。

---

## 5.1 Prisma の relation と何が違うか

Prisma ではリレーションをこう書きます。

```prisma
model User {
  id       BigInt    @id
  articles Article[]
}

model Article {
  id        BigInt @id
  author_id BigInt?
  author    User?  @relation(
    fields: [author_id], references: [id])
}
```

両側にフィールドを書き、**外部キーを持つ側に `@relation`** を付けます。

ent でも両側に書く点は同じです。

```go
// ent/schema/user.go
edge.To("articles", Article.Type),

// ent/schema/article.go
edge.From("author", User.Type).
  Ref("articles").
  Field("author_id").
  Unique(),
```

違いは 2 つあります。

1. 両側の役割が **`To`（持ち主）と `From`（逆参照）** に分かれている
2. 「1 対多」か「多対多」かを、**`Unique()` の有無で表す**

順に見ていきます。

---

## 5.2 To と From

公式ドキュメントの説明はこうです。

> A schema that defines an edge using the `edge.To` builder owns the relation, unlike using the `edge.From` builder that gives only a back-reference for the relation (with a different name).

- **`edge.To`** — 関係の**持ち主**。関係の定義の本体
- **`edge.From`** — **逆参照**。反対側から辿れるようにするための別名

### Ref で 2 つを結ぶ

`From` 側の `.Ref("articles")` は、「相手のスキーマの `articles` という To エッジの逆向きです」という宣言です。

```text
User ──articles(To)──▶ Article
User ◀──author(From)── Article
        Ref("articles")
```

**1 本の関係に対して、名前が 2 つ付いている**と考えてください。

- User から見ると `articles`
- Article から見ると `author`

どちらの名前でも辿れます。`user.QueryArticles()` と `article.QueryAuthor()` の両方が生成されます（第 7 章）。

### From は省略できる

逆参照が不要なら、`From` は書かなくても構いません。`To` だけで関係は成立します。

逆に、**`From` だけを書くことはできません**。`Ref` で指す相手の `To` が必ず必要です。

### どちらを To にするか

迷ったら、**「〜が〜を持つ」と言える側を To** にします。

- ユーザーが記事を**持つ** → User に `To("articles")`
- 管理者がお知らせを**持つ** → Admin に `To("notices")`

---

## 5.3 関係の種類は Unique で決まる

ent は関係の種類（カーディナリティ）を、**エッジに `Unique()` が付いているかどうか**で判定します。

`Unique()` は「このエッジの先は 1 件だけ」という意味です。

| To 側 | From 側 | 関係 |
| --- | --- | --- |
| `Unique()` | `Unique()` | 1 対 1（O2O） |
| なし | `Unique()` | 1 対多（O2M） |
| `Unique()` | なし | 多対 1（M2O） |
| なし | なし | 多対多（M2M） |
| `Unique()` | From なし | 多対 1（M2O） |
| なし | From なし | 1 対多（O2M） |

M2O は O2M を逆から見たものです。To を書いた側が「多」になります。

TypeScript の型で言えば、`Unique()` 付きは `T | null`、なしは `T[]` です。生成される struct もそうなっています。

```go
// User 側（To, Unique なし）
Articles []*Article
// Article 側（From, Unique あり）
Author *User
```

### O2M — いちばん多い形

前節のユーザーと記事がこれです。1 人のユーザーが複数の記事を持ち、記事の著者は 1 人です。

実務のエッジの大半はこの形です。

### O2O — 両側が Unique

```go
// ent/schema/article.go
edge.To("setting", ArticleSetting.Type).
  Unique(),

// ent/schema/article_setting.go
edge.From("article", Article.Type).
  Ref("setting").
  Unique().
  Required().
  Field("article_id"),
```

記事 1 本に対して「記事ごとの設定」が最大 1 件。両側に `Unique()` が付いています。

### M2M — 両側とも Unique なし

1 本の記事は複数のコレクションに入り、1 つのコレクションは複数の記事を持ちます。これが多対多です。

題材アプリの M2M はすべて、5.6 節で扱う `Through` を使って書かれています。

---

## 5.4 外部キー列はどこに作られるか

**この章でいちばん大事な節です。**

To と From は「どちらが関係の持ち主か」という**ent 上の役割**です。一方、外部キー列がどちらのテーブルに置かれるかは、**DB の都合**で決まります。

この 2 つは**別の話**です。混同しやすいので整理します。

### O2M: 「多」の側に置かれる

1 対多の外部キーは、必ず「多」の側のテーブルに置かれます。これは ent に限らず RDB の原則です。

```text
users             articles
┌────┐            ┌────┬───────────┐
│ id │◀───────────│ id │ author_id │
└────┘            └────┴───────────┘
  To 側              From 側
```

User が To（持ち主）なのに、外部キー列 `author_id` は From 側の `articles` にあります。**O2M では、持ち主のテーブルに列が増えるわけではありません。**

逆に、To 側に `Unique()` を付けた M2O では、外部キーは **To を書いた側**のテーブルに置かれます。5.6 の `CollectionItem` の `edge.To("collection").Unique()` がこの形で、`collection_id` は `collection_items` にあります。

### O2O: From 側に置かれる

1 対 1 では、ent は**To の向かう先のテーブル**、つまり From 側に外部キーを置きます。

前節の例では、外部キー `article_id` は `article_settings` テーブルにあります。生成された `ent/migrate/schema.go` で確認できます。

```go
{Name: "article_id",
  Type: field.TypeUint64, Unique: true},
```

O2O なので、外部キー列にユニーク制約も付きます。

### M2M: 中間テーブルに置かれる

多対多では、どちらのテーブルにも列は増えません。両方の ID を持つ**中間テーブル**が作られます。

### 確かめる方法

迷ったら、`make gen` の後に `ent/migrate/schema.go` を見てください。各テーブルの `Columns` と `ForeignKeys` に、実際に作られる列と外部キーがすべて書かれています。

---

## 5.5 Field — 外部キーを普通の列として扱う

### Field を付けないと

実は `Field()` を書かなくても、ent は外部キー列を自動で作ります。ただしその列は**ent の内部に隠れ**、struct のフィールドとしては見えません。

外部キーの値だけが欲しいときにも、関連エンティティを取得する必要が出てきます。

### Field を付けると

```go
// ent/schema/article.go
func (Article) Fields() []ent.Field {
  return []ent.Field{
    // ...
    field.Uint64("author_id").Optional(),
  }
}

func (Article) Edges() []ent.Edge {
  return []ent.Edge{
    edge.From("author", User.Type).
      Ref("articles").
      Field("author_id").
      Unique(),
  }
}
```

1. `Fields()` に外部キー列を**普通のフィールドとして**宣言する
2. エッジの `.Field("author_id")` で、**その列がこのエッジの外部キーだ**と結びつける

これで `a.AuthorID` として値を直接読めます。`article.AuthorID(1)` のような述語で絞り込むこともできます。

Prisma の `@relation(fields: [author_id])` とほぼ同じ役割です。

題材アプリでは、**外部キーはほぼすべて `Field()` で明示する**流儀にしています。

### Field を書ける側

公式ドキュメントにあるとおり、`Field()` を書けるのは**外部キーを持つ側のエッジだけ**です。

O2M なら「多」の側（`From` の `author`）に書きます。「一」の側の `To("articles")` には書けません。

### Optional と Required

外部キー列が NULL になれるかどうかは、フィールドとエッジの両方で揃えます（揃っていないと `make gen` がエラーになります）。

```go
// NULL 許容の外部キー
field.Uint64("author_id").Optional(),
edge.From("author", User.Type).
  Ref("articles").
  Field("author_id").
  Unique(),

// NOT NULL の外部キー
field.Uint64("article_id"),
edge.From("article", Article.Type).
  Ref("comments").
  Field("article_id").
  Unique().
  Required(),
```

フィールドと違い、**エッジはデフォルトで任意**です（公式ドキュメント「Required」）。必須にするには `Required()` を付けます。

`Required()` を付けると、作成時にエッジの相手をセットしないと `Save` がエラーになります。ent v0.10 以降では、外部キー列も `NOT NULL` で作られます（自己参照のエッジを除く）。

---

## 5.6 Through — 中間テーブルをエンティティにする

### 素朴な M2M の弱点

ent では、`Through` を使わない M2M も書けます。その場合、中間テーブルは**ent が裏で勝手に作り、スキーマには現れません**。

これは Prisma の暗黙的な多対多（`@relation` なしの `Post[]` と `Tag[]`）と同じです。手軽ですが、中間テーブルに**列を足せません**。

「コレクションの中での並び順」や「お気に入りした日時」のように、**関係そのものに属する情報**を持たせたくなることはよくあります。

### Through の書き方

コレクションと記事の関係を見ます。3 つのスキーマが登場します。

```go
// ent/schema/collection.go
edge.To("articles", Article.Type).
  Through("collection_items",
    CollectionItem.Type),

// ent/schema/article.go
edge.From("collections", Collection.Type).
  Ref("articles").
  Through("collection_items",
    CollectionItem.Type),
```

`Through("collection_items", CollectionItem.Type)` は、「この M2M は `CollectionItem` というスキーマを中間テーブルとして使う」という指定です。

中間テーブルのスキーマはこうです。

```go
// ent/schema/collection_item.go
func (CollectionItem) Fields() []ent.Field {
  return []ent.Field{
    field.Uint64("id"),
    field.Uint64("collection_id"),
    field.Uint64("article_id"),
    field.Uint32("position").
      Comment("コレクション内の表示順"),
  }
}

func (CollectionItem) Edges() []ent.Edge {
  return []ent.Edge{
    edge.To("collection", Collection.Type).
      Unique().Required().
      Field("collection_id"),
    edge.To("article", Article.Type).
      Unique().Required().
      Field("article_id"),
  }
}
```

ポイントは 3 つです。

- 中間テーブルが**普通のスキーマ**として存在する
- 両側への外部キーを `Unique().Required().Field(...)` のエッジで持つ（公式ドキュメント「Edge Schema」で必須とされている形）
- **`position` のような追加の列を持てる**

Prisma で言えば、明示的な中間モデル（`PostTag` モデルを自分で書く方式）に相当します。

### 2 通りに辿れる

Through を使うと、関係を 2 通りに扱えます。

- `collection.QueryArticles()` — 中間テーブルを**飛ばして**記事を直接取る
- `collection.QueryCollectionItems()` — 中間テーブルの**行そのもの**を取る（並び順が欲しいとき）

公式ドキュメントは、Through を使う利点として「中間テーブルに追加の列を持てる」「CRUD 操作や Hook を中間テーブルに適用できる」ことを挙げています。

### M2M は Through で書くと決めておく

題材アプリでは、**スキーマを持たない隠れた中間テーブルを作らない**方針にしています。すべての M2M が Through です。

新しく M2M を作るときも Through で書いておくと、後から中間テーブルに列を足したくなっても、スキーマを書き換えずに済みます。

---

## 5.7 自己参照 — 同じ型どうしの関係

エッジの相手は、自分自身の型でも構いません。

題材アプリの `Tag` には、「親タグ」の下に複数の「子タグ」がぶら下がる階層があります。しかも子タグは複数の親に属せます。**同じ型の中での多対多**です。

```go
// ent/schema/tag.go
edge.To("children", Tag.Type).
  Through("tag_hierarchies",
    TagHierarchy.Type).
  StorageKey(edge.Columns(
    "parent_tag_id", "child_tag_id")),
edge.From("parents", Tag.Type).
  Ref("children"),
```

- `To("children")` — 親から子へ
- `From("parents").Ref("children")` — 子から親へ

To と From の仕組みは、相手が別の型のときと同じです。生成される struct は両方とも `[]*Tag` になります。

### StorageKey — 列名を指定する

Through を使う M2M では、エッジの 2 列を中間スキーマ（`TagHierarchy`）のどのフィールドに対応させるかを ent が判定します。名前で決まらなければ型で探します。

自己参照では 2 列が**どちらも `Tag` を指す**ので、型では判別できません。そのため `StorageKey(edge.Columns(...))` で中間スキーマのフィールド名を指定しています。指定しないと生成時に `missing edge-field` エラーになります（`entc/gen/graph.go`）。

- 1 つ目の `parent_tag_id` — To の起点側（親）
- 2 つ目の `child_tag_id` — To の行き先側（子）

公式ドキュメント「StorageKey」の M2M の例でも、`edge.Columns("user_id", "friend_id")` のように同じ順で指定しています。

`StorageKey` は、既存 DB の列名にエッジを合わせたいときにも使います。

---

## 5.8 削除時の挙動 — OnDelete

親の行を削除したとき、子の行をどうするか。これは**DB の外部キー制約**（`ON DELETE`）の話です。

ent では、エッジのアノテーションで指定します。

```go
// ent/schema/article.go
edge.To("revisions", ArticleRevision.Type).
  Annotations(
    entsql.Annotation{
      // 記事と一心同体のため
      // DeleteOnCascadeにする
      OnDelete: entsql.Cascade,
    },
  ),
```

よく使う値は 3 つです。

| 値 | 親を削除すると |
| --- | --- |
| `Cascade` | 子も削除される |
| `Restrict` | 子がいる限り削除できない |
| `SetNull` | 子の外部キーが NULL になる |

Prisma の `@relation(onDelete: Cascade)` と同じものです。実務では `Cascade` が大半で、`Restrict` と `SetNull` は要所で使う程度です。

### To 側に書く

**`OnDelete` は `To` のエッジに書きます。**

ent のコード生成（`entc/gen/graph.go`）は、外部キーを組み立てるときに `From`（逆参照）のエッジを読み飛ばし、`To` のエッジのアノテーションだけを見ます。From 側に書いても DDL には反映されません。

> **現場メモ: 効いていない OnDelete**
> 大きなスキーマを見直すと、`From` 側のエッジに `OnDelete: entsql.Cascade` が書かれているのに、DDL には何も反映されていない、という箇所が見つかることがあります。コンパイルも生成も通るので気づきにくい書き間違いです。削除時の挙動を確かめたいときは、スキーマではなく生成された `ent/migrate/schema.go` の `ForeignKeys` の `OnDelete` を見てください。

`Restrict` の使いどころは、コメントが分かりやすいです。

```go
// ent/schema/admin.go
edge.To("notices", Notice.Type).
  Annotations(
    entsql.Annotation{
      // お知らせを持ってるadminは削除できない
      OnDelete: entsql.Restrict,
    },
  ),
```

`SetNull` は、カテゴリを消しても記事は残したい、といった関係に使います。

```go
// ent/schema/category.go
edge.To("articles", Article.Type).
  Annotations(
    entsql.Annotation{
      OnDelete: entsql.SetNull,
    },
  ),
```

### 何も書かないと

指定しなかった場合の既定値も `graph.go` で決まっています。

- 外部キー列が NULL 許容なら `SET NULL`
- NOT NULL なら `NO ACTION`

### 2 つの注意点

1 つ目。**これは DB の機能です。** ent が Go のコードで子を消して回るわけではありません。ent 以外から DELETE しても同じく効きます。

2 つ目。**論理削除（第 6 章・第 11 章）とは連動しません。** 論理削除は実際には `UPDATE ... SET deleted_at = ...` なので、DB から見ると削除が起きていません。`Cascade` を付けていても子は残ります。

---

## 5.9 エッジのその他の設定

### Comment

フィールドと同じく、エッジにも `.Comment()` を付けられます。生成コードのコメントになります。

```go
edge.To("comments", Comment.Type).
  Comment("記事へのコメント"),
```

### Annotations

エッジの `Annotations()` には、`entsql`（DB 向け）の指示と並んで、**GraphQL 向けの指示**も入っています。

```go
edge.To("revisions", ArticleRevision.Type).
  Annotations(
    entsql.Annotation{OnDelete: entsql.Cascade},
    entgql.QueryField(),
    entgql.RelayConnection(),
  ),
```

`entgql.` で始まるものは、このエッジを GraphQL でどう公開するかの指定です。第 13 章で詳しく扱います。

エッジを GraphQL に公開すると、**Query のルートが無い型でも、エッジを辿れば取得できる**ようになる点には注意してください（第 13 章）。

---

## まとめ

- ent は関係を**グラフのエッジ**として書く。1 本の関係に、両側から辿るための名前が 2 つ付く
- **`To` が持ち主、`From` が逆参照**。`From` は `Ref` で相手の `To` を指す
- 関係の種類は **`Unique()` の有無**で決まる。両側 Unique なら O2O、片側なら O2M（または M2O）、なしなら M2M
- **外部キー列の位置は To/From とは別の話**。O2M・M2O なら「多」の側、O2O なら From 側、M2M なら中間テーブル。迷ったら `ent/migrate/schema.go` を見る
- `Field()` で外部キーを普通の列として公開する。明示しておくと値を直接読める
- エッジは**デフォルトで任意**。必須なら `Required()`
- **`Through` で中間テーブルをスキーマにする**と、追加の列を持てる
- 自己参照もできる。列名は `StorageKey` で指定する
- `OnDelete` は **To 側**に書く。DB の機能なので、論理削除とは連動しない

次章では、インデックスとテーブル単位の設定、そして Mixin によるスキーマ部品の再利用を扱います。

---

[← 前: フィールド — 列を定義する](04-fields.md) | [目次](../README.md) | [次: インデックス・アノテーション・Mixin →](06-indexes-annotations-mixin.md)
