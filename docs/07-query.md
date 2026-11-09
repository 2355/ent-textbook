# 7. クエリ — データを読む

> **この章でわかること**
> Query が「遅延評価のビルダー」であること。述語の組み立て方。`Get` / `Only` / `First` / `All` などの使い分け。並び替えとページング。eager loading と N+1。エッジの値が struct のどこに入るか。集計と、`Modify` という逃げ道。

長い章です。7.1〜7.5 が最重要です。7.8 以降は「こういう書き方を見かけたら読みに来る」くらいで構いません。

---

## 7.1 Query は「SQL の設計図」

ent でデータを読むときは、必ず `Query()` から始まります。

```go
articles, err := client.Article.Query().
	Where(article.StatusEQ(article.StatusPublished)).
	Order(ent.Desc(article.FieldPublishedAt)).
	Limit(10).
	All(ctx)
```

Prisma の `prisma.article.findMany({ where, orderBy, take })` に相当します。違うのは、**条件を 1 個のオブジェクトで渡すのではなく、メソッドを順につなげる**点です。

ここで大事なのは、**`All(ctx)` を呼ぶまで SQL は 1 本も走らない**ことです。

- `Query()` / `Where()` / `Order()` / `Limit()` … 設計図に書き足すだけ
- `All()` / `Only()` / `Count()` … 設計図を SQL にして実行する（**終端メソッド**）

Knex を使ったことがあれば同じ感覚です。`knex('users').where(...)` は await するまで実行されません。

### なぜ遅延評価なのか

設計図を変数に入れて、**条件によって書き足せる**からです。

```go
q := client.Wallet.Query()
if opt != nil && opt.Exclusive() {
	q = q.ForUpdate()
}
w, err := q.Where(wallet.UserID(uid)).Only(ctx)
```

Prisma だと `where` オブジェクトを条件分岐で組み立てることになりますが、ent ではビルダーそのものを育てます。

### 設計図を分岐させたいときは Clone

ビルダーは**ミュータブル**です。`q.Where(...)` は `q` 自身を書き換えて、同じ `q` を返します。

なので「件数と一覧で同じ条件を使い回したい」ときは `Clone()` でコピーを取ります。

```go
total, err := baseQuery.Clone().Count(ctx)
// baseQuery はまだ汚れていない
items, err := baseQuery.Limit(n).All(ctx)
```

`Clone()` しないと、`Count` 用に足した設定が一覧側に漏れることがあります。TS の配列を `[...arr]` でコピーしてから `sort` するのと同じ注意です。

---

## 7.2 述語 — WHERE を型で書く

`Where()` に渡すものを**述語**（predicate）と呼びます。ent はフィールドごとに述語関数を生成します。

| 述語 | SQL |
| --- | --- |
| `user.IDEQ(1)` / `user.ID(1)` | `id = 1` |
| `user.IDNEQ(1)` | `id <> 1` |
| `user.EmailNEQ("")` | `email <> ''` |
| `article.IDIn(ids...)` | `id IN (...)` |
| `article.PublishedAtLTE(now)` | `published_at <= ?` |
| `article.PublishedAtIsNil()` | `published_at IS NULL` |
| `article.CategoryIDNotNil()` | `category_id IS NOT NULL` |
| `article.TitleContains("go")` | `title LIKE '%go%'` |

`IsNil` / `NotNil` は **Optional なフィールドにだけ**生成されます（第 4 章）。

述語は**パッケージ名.関数名**の形です。`user` パッケージ、`article` パッケージは、エンティティごとに生成された小さなパッケージです（第 3 章）。

### 複数の述語

`Where()` に複数渡すと AND になります。`Where()` を 2 回呼んでも AND です。

```go
client.Like.Query().
	Where(like.UserIDEQ(userID)).
	Where(like.ArticleIDEQ(articleID)).
	Exist(ctx)
```

OR や NOT は、エンティティごとに生成された `Or` / `And` / `Not` を使います。

```go
article.Or(
	article.StatusEQ(article.StatusPublished),
	article.AuthorIDEQ(viewerID),
)
```

`article.Not(article.HasTags())` のように否定もできます。

### 述語をスライスで持ち回す

述語は `predicate.Tag` のような**型を持つ値**です。なので配列に入れて組み立てられます。

```go
preds := []predicate.Tag{
	tag.KindIn(kinds...),
	tag.NameNEQ("general"),
}
tags, err := client.Tag.Query().
	Where(preds...).
	All(ctx)
```

`preds...` は TS のスプレッド構文 `...preds` と同じ意味です（第 2 章）。

型があるので、**`User` の述語を `Tag` のクエリに渡すとコンパイルエラー**になります。Prisma の `where` 型と同じ安心感です。

### IDNEQ —「自分以外」の条件

`IDNEQ` は「ID Not EQual」で、`WHERE id <> ?` になります。「同じ名前のタグが**自分以外に**あるか」のような確認で使います。

```go
dup, err := client.Tag.Query().
	Where(
		tag.NameEQ(name),
		tag.IDNEQ(selfID),
	).
	Exist(ctx)
```

> **現場メモ**
> 「自分を除く」条件が要るかは、**自分の行がその時点で DB にあるか**で決まります。新規作成の保存前チェックなら、自分はまだ DB に無いので不要。更新（既存タグの改名など）の場合は、保存前でも自分の行が DB にあるので必要です。名前を変えずに保存すると、自分自身が重複としてヒットするからです。保存後に「他に同じサムネイルを使っている記事があるか」を確認する場合も、自分自身が必ずヒットするので必須です。レビューでは「この確認はいつ、どの操作で走るのか」を先に確かめると、`IDNEQ` の要否で迷いません。

### Contains の大文字小文字は DB が決める

`TitleContains("go")` は SQL の `LIKE '%go%'` になるだけです。**大文字と小文字を区別するかは ent ではなく、列の照合順序（collation）が決めます。**

- `utf8mb4_unicode_ci` など末尾が `_ci` → 区別しない（`Go` も `GO` もヒット）。照合規則によっては、かなの濁点・半濁点なども同一視されることがあるので、日本語の検索では MySQL のドキュメントで照合規則を確かめてください
- `utf8mb4_bin` → 区別する

第 6 章で見たとおり、照合順序はテーブル単位でアノテーションから指定できます。検索の挙動が想定と違ったら、まず照合順序を確認してください。区別しない検索をしたいなら `ContainsFold` という述語もあります。ただし MySQL では、`ContainsFold` は列の照合順序に関係なく `COLLATE utf8mb4_general_ci` を付けた `LIKE` を出します（ent の `dialect/sql/builder.go`）。テーブルの照合順序とは比較規則が違いうる点に注意してください。

---

## 7.3 関係をまたぐ述語 — Has と HasWith

「公開記事を含むコレクション」のように、関係先の条件で絞りたいことがあります。エッジ（第 5 章）ごとに次の述語が生成されます。

- `HasXxx()` … 関係先が 1 件以上ある
- `HasXxxWith(述語...)` … 条件を満たす関係先が 1 件以上ある

```go
client.Collection.Query().
	Where(collection.HasArticlesWith(
		article.PublishedAtNotNil(),
	)).
	All(ctx)
```

Prisma の `where: { articles: { some: { ... } } }` に当たります。SQL としては `IN (SELECT ...)` や `EXISTS (SELECT ...)` などのサブクエリになります（エッジの種類で形が変わります）。

`HasXxxWith` の中にさらに `HasXxxWith` を入れ子にできます。

### 落とし穴: 論理削除が効かない

題材アプリでは、一部のエンティティに論理削除（`deleted_at`）が入っています。普段のクエリでは自動で `deleted_at IS NULL` が付きます。

**ところが `HasXxxWith` のサブクエリには付きません。** なので明示的に書きます。

```go
collectionitem.HasArticleWith(
	article.DeletedAtIsNil(),
)
```

なぜそうなるのか、放置すると何が起きるのかは第 11 章で仕組みから説明します。今は「**HasXxxWith の先が論理削除対象なら `DeletedAtIsNil()` を足す**」とだけ覚えてください。

---

## 7.4 終端メソッドの使い分け

SQL を実行するメソッドはたくさんあります。よく使うものを並べます。

| メソッド | 返すもの | 0 件のとき |
| --- | --- | --- |
| `Get(ctx, id)` | 1 件 | NotFound エラー |
| `Only(ctx)` | 1 件 | NotFound エラー |
| `First(ctx)` | 先頭 1 件 | NotFound エラー |
| `All(ctx)` | スライス | 空スライス |
| `IDs(ctx)` | ID のスライス | 空スライス |
| `Count(ctx)` | 件数 `int` | `0` |
| `Exist(ctx)` | `bool` | `false` |

`Get` は `client.User.Get(ctx, id)` のように Query を経由せずに呼べる近道です。

### Only と First の違い

どちらも 1 件返しますが、意味が違います。

- **`Only`** は「**ちょうど 1 件のはず**」。2 件以上あると `NotSingularError` を返す
- **`First`** は「**先頭の 1 件でいい**」。何件あっても気にしない

生成コードを見ると、`Only` は `Limit(2)` で取って件数を数えています。

```go
switch len(nodes) {
case 1:
	return nodes[0], nil
case 0:
	return nil, &NotFoundError{...}
default:
	return nil, &NotSingularError{...}
}
```

ユニーク制約のある列で引くなら `Only` が意図をよく表します。`First` を使うなら `Order` を付けないと「どれが先頭か」は DB 任せです。

### 0 件はエラーという設計

Prisma の `findFirst` は 0 件で `null` を返します。ent の `Only` / `First` / `Get` は **`NotFoundError` を返します**。

Go には `null` の代わりに nil がありますが、「見つからない」を `nil, nil` で返すと呼び出し側がエラーチェックだけして nil 参照で落ちる事故が起きます。ent は「不在」をエラーとして返す側に倒している、と考えると納得しやすいでしょう。

```go
u, err := client.User.Query().
	Where(user.SubEQ(sub)).
	First(ctx)
if ent.IsNotFound(err) {
	// まだアカウントが紐付いていない（初回ログイン）
	return ctx, nil
}
```

エラーの扱いは第 10 章で詳しく見ます。

### X 付きメソッド

`AllX` / `SaveX` / `GetX` のように末尾に `X` が付いたものもあります。**エラーを返す代わりに panic します。**

テストのデータ準備では便利ですが、本番コードでは使わないのが無難です。

---

## 7.5 並び替えとページング

### Order の 3 つの書き方

**(1) 列名を渡す**

```go
Order(ent.Desc(article.FieldPublishedAt))
Order(ent.Asc(user.FieldID))
```

**(2) 生成された `ByXxx` を使う**

```go
Order(
	article.ByViewCount(sql.OrderDesc()),
	article.ByID(),
)
```

引数なしの `ByID()` は昇順です。新しめの書き方で、列名の文字列を扱わなくて済みます。

**(3) SQL 式で並べる**

```go
Order(func(s *sql.Selector) {
	s.OrderExpr(sql.Expr("FIELD(id, 3,1,2)"))
})
```

「レコメンドされた ID の順に並べたい」のように、列では表現できない順番に使います。`*sql.Selector` が出てきたら「SQL を直接いじっている」サインです（7.9）。

### 並びを安定させる

同点のレコードがあると、実行するたびに順番が変わることがあります。上の (2) の例で `ByID()` を第 2 キーに置いているのはそのためです。閲覧数が同じタグが、日によって入れ替わらないようにしています。

ページングするなら必須の配慮です。

### Limit と Offset

```go
query.Limit(perPage).Offset((page - 1) * perPage)
```

Prisma の `take` / `skip` です。**ページングは必ず DB 側で行う**のが原則です。全件取ってメモリで切るのは、件数が増えるほど重くなります。

GraphQL のカーソルページング（`Paginate`）もありますが、それは第 13 章で扱います。

---

## 7.6 エッジをたどる — QueryXxx

取得済みのエンティティから、関係先を引くことができます。

```go
count, err := a.QueryViews().Count(ctx)
```

`QueryXxx()` は**新しい Query ビルダー**を返します。なので `Where` も `Order` も `Paginate` もつなげられます。

```go
col.QueryCollectionItems().
	Where(...).
	Order(collectionitem.ByPosition()).
	All(ctx)
```

Query から Query へ渡ることもできます。

```go
client.User.Query().
	Where(user.ID(1)).
	QueryArticles().
	All(ctx)
```

公式ドキュメントでは、これを**トラバーサル**（グラフの走査）と呼んでいます。第 1 章で触れた「ent はデータをグラフとして見る」がいちばん表れる API です。

---

## 7.7 eager loading と N+1

### N+1 とは

記事 20 件を取って、それぞれの著者を `QueryAuthor()` で引くと、**1 + 20 回** SQL が走ります。

これが N+1 問題です。GraphQL のリゾルバーでは特に起きやすくなります。

### WithXxx で先読みする

```go
client.Article.Query().
	Where(...).
	WithAuthor().
	All(ctx)
```

`WithXxx()` を付けると、**関係先をまとめて先読み**します。Prisma の `include: { author: true }` です。

ent は先読みを JOIN では行いません。公式ドキュメントにこうあります。

> 複数のエッジを eager load できるため、1 回の JOIN ですべての関連を読み込むことはできない。そのため、関連ごとに追加のクエリを実行する。

（`doc/md/eager-load.mdx` より要約）

実際に流れる SQL はこうなります。

```sql
SELECT * FROM articles WHERE ... ;
SELECT * FROM users
  WHERE id IN (3, 8, 15, ...);
```

記事が 20 件でも 2000 件でも **2 本**です。`WithTags()` も足せば 3 本、と**関連の数だけ**増えます。件数には比例しません。

JOIN ではなく関連ごとに別クエリを投げる点は、Prisma の `include` の従来からの既定動作や、Rails の `preload` と同じ発想です。「JOIN していないから N+1 では？」と身構える必要はありません。

### 先読みに条件を付ける

`WithXxx` には関数を渡せます。関係先のクエリに条件や列の絞り込みを足せます。

```go
query.WithAuthor(func(q *ent.UserQuery) {
	q.Select(user.FieldName, user.FieldAvatarURL)
})
```

`WithAuthor().WithTags()` のように複数つなげることも、関数の中でさらに `WithXxx` してネストすることもできます。

### 先読みした値はどこに入るか

ここが TS の感覚と違うところです。

Prisma なら `article.author` に直接入ります。ent では **`Edges` という専用の struct** に入ります。

```go
type Article struct {
	ID    uint64
	// ...列のフィールド
	Edges ArticleEdges
}

type ArticleEdges struct {
	Author *User
	Tags   []*Tag
	loadedTypes [2]bool
}
```

（生成コードを簡略化）

`loadedTypes` は「先読みしたかどうか」の記録です。ここから、**「先読みしたが無かった」と「そもそも先読みしていない」を区別**できます。

その区別を返すのが `XxxOrErr()` です。

```go
author, err := obj.Edges.AuthorOrErr()
if ent.IsNotLoaded(err) {
	author, err = obj.QueryAuthor().Only(ctx)
}
return author, ent.MaskNotFound(err)
```

- 先読み済みで存在 → 値を返す
- 先読み済みで不在 → `NotFoundError`
- 先読みしていない → `NotLoadedError`

これは `Author` のような**単数のエッジ**の場合です。`TagsOrErr()` のような複数のエッジは、先読み済みなら空スライスを返し、NotFound にはなりません。

`IsNotLoaded` なら自分で引き直す、というのが GraphQL リゾルバーでの定番パターンです。最後の `ent.MaskNotFound(err)` は、**NotFound だけを nil に変える**ヘルパーです。「親が無ければ null を返す」GraphQL フィールドで使います。

### 列を絞る — Select

先読みに限らず、`Select` で取得する列を絞れます。

```go
client.Tag.Query().
	Where(tag.IDIn(ids...)).
	Select(tag.FieldID).
	All(ctx)
```

返ってくるのは `*ent.Tag` ですが、**選ばなかったフィールドはゼロ値**です。うっかり別の列を読むと空文字や 0 が返るので注意してください。

---

## 7.8 集計

### GroupBy

```go
ids, err := client.ArticleView.Query().
	Where(articleview.UserIDEQ(userID)).
	Select(articleview.FieldArticleID).
	GroupBy(articleview.FieldArticleID).
	Ints(ctx)
```

`GroupBy` のあとに `Ints` / `Strings` などで**型付きのスライス**として受け取ります。0 件なら空スライスで、エラーにはなりません（NotFound を返すのは単数形の `Int` などです）。

ent には `Aggregate(ent.Count())` のような集計 API もあります。ただ、JOIN を伴う集計は次の `Modify` で SQL を直接書くことが多くなります。

---

## 7.9 Modify — SQL に手を入れる

ent の Query で表現しきれないとき、**生成される SELECT 文を直接いじる**口が `Modify` です。`entc.go` の `sql/modifier` フィーチャーで有効になります。

記事ごとのユニーク閲覧者数を数える例です。

```go
client.Article.Query().
	Modify(func(sel *sql.Selector) {
		t := sql.Table(articleview.Table)
		sel.LeftJoin(t).On(
			sel.C(article.FieldID),
			t.C(articleview.FieldArticleID),
		)
		sel.AppendSelect(sql.As(
			sql.Count(sql.Distinct(
				t.C(articleview.FieldUserID),
			)),
			"viewer_count",
		))
		sel.GroupBy(sel.C(article.FieldID))
	}).
	All(ctx)
```

`*sql.Selector` は Knex のクエリビルダーとほぼ同じ感覚で使えます。

| Selector | Knex |
| --- | --- |
| `sel.Join(t).On(a, b)` | `.join(t, a, b)` |
| `sel.LeftJoin(t).On(...)` | `.leftJoin(...)` |
| `sel.AppendSelect(...)` | `.select(...)` を追加 |
| `sel.GroupBy(...)` | `.groupBy(...)` |
| `sel.OrderBy(sql.Desc(...))` | `.orderBy(..., 'desc')` |
| `sel.C("col")` | テーブル名付きの列名 |

`sql.Count` / `sql.Max` / `sql.Distinct` / `sql.As` などの関数で式を組み、どうしても書けない部分は `AppendSelectExpr(sql.ExprFunc(...))` で文字列を直接書き込みます。

### 追加した列を読む — Value

`Modify` で足した列（上の `viewer_count`）は、エンティティの struct にフィールドがありません。そこで `Value` で名前を指定して読みます。

```go
v, err := a.Value("viewer_count")
n, ok := v.(int64)
```

戻り値は `any` なので型アサーション（第 2 章）が必要です。TS の `as number` と違い、**実行時に本当に型を確認**します。`ok` が false なら型が違ったということです。

### Modify の副作用

`Modify` で SELECT を書き換えたクエリは、`Count` がうまく動かないことがあります。

`Count` は SELECT 列を `COUNT(...)` に差し替えます。しかし `Modify` で足した `GROUP BY` や、別名での `ORDER BY` はそのまま残ります。そのため壊れた SQL になります（ent の `sqlgraph/graph.go` の `count`）。

「じゃあ全件取って `len()` で数えよう」とするのは、件数が増えるほど重くなる回避策です。件数が要るなら、`Modify` を足す前のクエリを `Clone()` して数えるなど、別の手を考えてください。

`Modify` は強力ですが、ent の「型と生成コードに任せる」世界からは外れます。**使うのは ent の API で書けないときだけ**にしてください。

### Modify 以外の逃げ道

- **カスタム述語** … `Where` に `func(s *sql.Selector)` を渡す。`predicate.User(func(s *sql.Selector){ ... })` のように型を付けることもできる。`sql.ExprP("DATE(col) = DATE(?)", t)` で生 SQL の条件も書ける
- **サブクエリ** … `sql.Select(...).From(t).Where(...)` で組み、`sql.In` や `sql.NotExists` に渡す
- **生 SQL** … 第 9 章

---

## まとめ

- Query は**遅延評価のビルダー**。`All` / `Only` / `Count` などの**終端メソッド**で初めて SQL が走る
- ビルダーはミュータブル。使い回すなら **`Clone()`**
- 述語はフィールドごとに生成され、**型で守られる**。`Or` / `Not` もエンティティごとにある
- `IDNEQ` は `id <> ?`。要否は、確認の時点で自分の行が DB にあるか（更新時・保存後なら必要）で決まる。`Contains` の大小文字区別は **collation** 次第
- `HasXxxWith` は関係先の条件で絞る。**論理削除は効かないので `DeletedAtIsNil()` を明示**
- `Only` / `First` / `Get` は **0 件でエラー**。`Only` は 2 件以上でもエラー
- N+1 は **`WithXxx`** で防ぐ。JOIN ではなく関連ごとに `IN` クエリが 1 本増える方式
- 先読みした値は **`Edges`** に入る。`XxxOrErr` + `IsNotLoaded` で「未ロード」を区別できる
- ent の API で書けないときは **`Modify`** で SQL に手を入れる。追加列は `Value` で読む

---

[← 前: インデックス・アノテーション・Mixin](06-indexes-annotations-mixin.md) | [目次](../README.md) | [次: ミューテーション — データを書く →](08-mutation.md)
