# 8. ミューテーション — データを書く

> **この章でわかること**
> Create / Update / Delete の Builder。`Save` と `Exec` の違い。`SetNillable` と `Clear` の使い分け。エッジの付け外し。一括作成。upsert と MySQL での注意点。そして、すべての書き込みの裏にある「Mutation」という中間表現。

8.6 の Mutation の話は、第 11 章（Hook）の土台になります。急いでいても 8.6 だけは読んでください。

---

## 8.1 書き込みも Builder

読み込みが `Query()` から始まったように、書き込みもエンティティごとの Builder から始まります。

```go
l, err := client.Like.Create().
	SetUserID(userID).
	SetArticleID(articleID).
	Save(ctx)
```

Prisma の `prisma.like.create({ data: {...} })` と比べると、**`data` オブジェクトの代わりに `SetXxx` を並べる**形です。

`SetXxx` はフィールドごとに生成されるので、存在しない列や型違いの値は**コンパイルエラー**になります。

公式ドキュメントでは、書き込みを 5 種類に分けています。

| 操作 | 入口 | 対象 |
| --- | --- | --- |
| Create | `Create()` | 1 件作る |
| UpdateOne | `UpdateOneID(id)` / `UpdateOne(e)` | 1 件更新 |
| Update | `Update().Where(...)` | 条件に合う全件更新 |
| DeleteOne | `DeleteOneID(id)` / `DeleteOne(e)` | 1 件削除 |
| Delete | `Delete().Where(...)` | 条件に合う全件削除 |

この 5 分類は 8.6 と第 11 章でまた出てきます。

---

## 8.2 Save と Exec

書き込みの終端メソッドは 2 つです。

- **`Save(ctx)`** … 結果を返す
- **`Exec(ctx)`** … 結果を捨てて error だけ返す

何が返るかは操作によって違います。

| 操作 | `Save` の戻り値 |
| --- | --- |
| Create | 作成したエンティティ |
| UpdateOne | 更新後のエンティティ |
| Update | 更新した**行数** `int` |

Delete には `Save` がなく `Exec` だけです。`Delete().Exec` は削除した行数と error、`DeleteOne().Exec` は error だけを返します。

結果が要らなければ `Exec` を使うのが素直です。

```go
return wrapEntError(
	client.Tag.DeleteOneID(id).Exec(ctx),
)
```

`wrapEntError` は第 10 章で扱うエラーの包み替えです。

`SaveX` / `ExecX` は第 7 章の `AllX` と同じく、エラーで panic する版です。テスト用と考えてください。

### Update の行数を使う

バルク更新の戻り値の行数は、**条件付き更新の成否判定**に使えます。

```go
affected, err := client.User.Update().
	Where(
		user.IDEQ(userID),
		user.CustomerIDIsNil(),
	).
	SetCustomerID(customerID).
	Save(ctx)
// ...
return affected > 0, nil
```

決済サービスの顧客 ID を「まだ空なら書く」処理です。1 本の UPDATE で行い、0 行なら「既に誰かが書いていた」と判断しています。SELECT してから UPDATE するより競合に強い書き方です。

---

## 8.3 1 件を更新する 3 つの入口

```go
// (1) ID 指定（いちばんよく使う）
client.Tag.UpdateOneID(id).
	SetName(name).
	Save(ctx)

// (2) 取得済みエンティティを渡す
tx.Article.UpdateOne(a).
	SetCategoryID(categoryID).
	Save(ctx)

// (3) エンティティ自身のメソッド
u.Update().
	SetLastLoginAt(now).
	Save(ctx)
```

どれも中身は同じ UpdateOne です。違いは「ID を持っているか、エンティティを持っているか」だけです。

`UpdateOneID` で対象が無ければ `NotFoundError` になります。

---

## 8.4 値をセットする・消す

### SetNillableXxx

`SetNillableXxx` は、Create では Optional か Default のあるフィールドに、Update ではほぼすべてのフィールドに生成されます。**ポインタを受け取り、nil なら何もしない**メソッドです。

生成コードはこれだけです。

```go
func (uc *UserCreate) SetNillableNickname(
	s *string,
) *UserCreate {
	if s != nil {
		uc.SetNickname(*s)
	}
	return uc
}
```

ドメインモデルのポインタをそのまま流し込むのに便利です。

```go
builder.SetNillablePublishedAt(
	entity.PublishedAt(),
)
```

### ClearXxx — NULL にする

ここで注意が必要です。**`SetNillable` に nil を渡しても NULL にはなりません。** 「何もしない」だけなので、更新では既存の値が残ります。

NULL にしたいなら `ClearXxx()` を呼びます。「画面の値で完全に上書きする」更新では、次のように書き分けます。

```go
if s := entity.Summary(); s != nil {
	update.SetSummary(*s)
} else {
	update.ClearSummary()
}
```

Prisma では `data: { x: undefined }` が「触らない」、`data: { x: null }` が「NULL にする」でした。ent ではこれが **`SetNillable(nil)` と `ClearXxx()`** に分かれています。

| やりたいこと | Prisma | ent |
| --- | --- | --- |
| 値を入れる | `x: v` | `SetX(v)` |
| 触らない | `x: undefined` | 何も呼ばない / `SetNillableX(nil)` |
| NULL にする | `x: null` | `ClearX()` |

`ClearCategoryID()` のように外部キー列にも使えます。

### デフォルト値と UpdateDefault

`Create` で `SetXxx` しなかったフィールドには、スキーマの `Default` が入ります（第 4 章）。

生成コードでは、`Save` の最初に `defaults()` が呼ばれ、**Mutation にまだ値が無いフィールドだけ**デフォルトを埋めています。

`UpdateDefault(time.Now)` を付けた `updated_at` は、更新のたびに自動で現在時刻になります。自分で `SetUpdatedAt` する必要はありません。

---

## 8.5 エッジを付け外しする

多対多のエッジには `AddXxxIDs` / `ClearXxx` が生成されます。

```go
// タグを完全に差し替える
builder.ClearTags()
if len(tagIDs) > 0 {
	builder.AddTagIDs(tagIDs...)
}
```

`ClearTags()` は**関係（中間テーブルの行）を外す**だけで、タグそのものは消しません。Prisma の `tags: { set: [] }` に近い操作です。

「全部外してから付け直す」は、差分を計算するより単純で、実務でもよく使われます。

外部キー列を持つエッジ（第 5 章の `Field` 指定）なら、`SetCategoryID(id)` のように列として直接セットするのが普通です。

---

## 8.6 Mutation という中間表現

ここまで Builder と呼んできたものの中身を覗いてみます。

```go
type UserCreate struct {
	config
	mutation *UserMutation
	hooks    []Hook
	conflict []sql.ConflictOption
}
```

（生成コードより）

**`SetXxx` は SQL を組み立てているのではなく、`UserMutation` に「この列をこの値にしたい」と書き込んでいます。**

`Save` を呼ぶと、こう流れます。

```go
func (uc *UserCreate) Save(
	ctx context.Context,
) (*User, error) {
	// ...defaults()
	return withHooks(
		ctx, uc.sqlSave, uc.mutation, uc.hooks,
	)
}
```

1. Builder が Mutation に変更内容を溜める
2. `Save` で Mutation が **Hook の列**を通る
3. 最後に `sqlSave` が Mutation を SQL に変換して実行する

つまり Mutation は「**DB に送る前の、変更内容の値オブジェクト**」です。

### なぜ中間表現を挟むのか

Hook（第 11 章）が「どの操作で、何を変えようとしているか」を**統一的に**読み書きできるようにするためです。

- 操作の種類（Create / Update / Delete…）は `Op()` で分かる
- どのフィールドを変えるかは Mutation から読める
- 操作そのものを書き換えることもできる

Express のミドルウェアが `req` オブジェクトを受け取って書き換えるのと似ています。`req` に当たるのが Mutation です。

UserCreate も UserUpdate も UserDelete も、**同じ `UserMutation` 型**を共有しています。そして全エンティティの Mutation は、共通の `ent.Mutation` インターフェースを満たします（`doc/md/hooks.md`）。

題材アプリの論理削除は、この仕組みを使って「Delete の Mutation を Update に書き換える」ことで実現しています。詳しくは第 11 章で見ます。

アプリのコードで Mutation を直接触るのは、その論理削除 Mixin くらいです。普段は意識しなくて構いませんが、**「書き込みは必ず Mutation と Hook を通る」**ことは覚えておいてください。

---

## 8.7 一括作成 — CreateBulk

複数件を 1 本の INSERT で作るには `CreateBulk` を使います。

```go
builders := make(
	[]*ent.CollectionItemCreate,
	0, len(articleIDs),
)
for i, aid := range articleIDs {
	builders = append(builders,
		client.CollectionItem.Create().
			SetCollectionID(colID).
			SetArticleID(aid).
			SetPosition(i),
	)
}
_, err := client.CollectionItem.
	CreateBulk(builders...).
	Save(ctx)
```

Create の Builder を**スライスに溜めてから**渡します。Prisma の `createMany({ data: [...] })` に当たります。ただし戻り値は件数ではなく作成したエンティティのスライスで、`createManyAndReturn` に近いです。

戻り値が要らなければ `.Exec(ctx)` で構いません。

---

## 8.8 upsert — あれば更新、なければ作成

`entc.go` の `sql/upsert` フィーチャーで、Create に `OnConflict` 系のメソッドが生えます。

「ユーザーが記事を最後に見た日時」を記録する例です。

```go
client.ArticleView.Create().
	SetUserID(userID).
	SetArticleID(articleID).
	SetLastViewedAt(now).
	OnConflictColumns(
		articleview.FieldUserID,
		articleview.FieldArticleID,
	).
	UpdateLastViewedAt().
	UpdateUpdatedAt().
	Exec(ctx)
```

- `OnConflictColumns(...)` … どの列の衝突を対象にするか
- `UpdateXxx()` … 衝突したら、その列を**今回 Create しようとした値で**上書きする
- `SetXxx(v)`（`OnConflict…` の**後**に書くもの）… 衝突したら、その列を**別の値 `v` で**上書きする

`UpdateXxx()` を書かなかった列は、衝突時には元の値のままです。上の例では「最終閲覧日時と更新日時だけ書き換え、作成日時は残す」ことになります。

基本は「create の値のうちどれを update にも使うか」を `UpdateXxx()` で選ぶ発想です。Prisma の `upsert({ where, create, update })` のように update 側だけ別の値にしたいときは、`OnConflict…` の後ろで `SetXxx(v)` を呼びます。

### 落とし穴: Set だけでは上書きされない

ここは強調しておきます。**`OnConflict…` より前（Create 側）の `SetXxx` で渡した値は、新規挿入のときにしか使われません。** MySQL で生成される SQL を見るとはっきりします。

```sql
INSERT INTO article_views
  (user_id, article_id, last_viewed_at, ...)
  VALUES (?, ?, ?, ...)
ON DUPLICATE KEY UPDATE
  last_viewed_at = VALUES(last_viewed_at),
  updated_at = VALUES(updated_at)
```

`ON DUPLICATE KEY UPDATE` 以下に出てくるのは、`OnConflict…` の後ろで `UpdateXxx()` か `SetXxx(v)` を呼んだ列だけです（ent の `dialect/sql/builder.go` の `SetExcluded`、生成テンプレート `upsert.tmpl`）。

更新する列を 1 つも指定しなかった場合の挙動は方言で違います。MySQL（数値 ID）では ent が `id = LAST_INSERT_ID(id)` を自動で付けるので、エラーにならず**何も更新されません**。SQLite / PostgreSQL では `DO UPDATE SET` の中身が空になり、エラーになります。

> **現場メモ**
> 「upsert なのに既存行が更新されない」不具合は、`UpdateXxx()` の書き忘れで起きます。エラーにはならず、既存行が**静かに古い値のまま**残るのが厄介です。新規ユーザーで動作確認すると挿入経路しか通らないので、テストでも気付きにくい。upsert を見かけたら「既存行で書き換えたい列が、すべて `OnConflict…` の後ろの `UpdateXxx()` か `SetXxx(v)` に並んでいるか」を確認してください。すべての列を上書きしたいなら `UpdateNewValues()` もあります。

### 衝突したら何もしない — Ignore

```go
client.Like.Create().
	SetUserID(userID).
	SetArticleID(articleID).
	OnConflict().
	Ignore().
	Exec(ctx)
```

「いいね」を連打されても 1 件だけ記録したい、といった用途に使います。

### MySQL での注意

MySQL では、upsert はこう書かれます（`dialect/sql/builder.go` の `writeConflict`）。

- `ON DUPLICATE KEY UPDATE` を出力する
- **`OnConflictColumns` で指定した列は SQL に出ない**
- `Ignore()` は `INSERT IGNORE` ではなく、`col = col`（自分自身を代入）という UPDATE になる。これは方言を問わず同じ
- `DoNothing()` は MySQL に無いため `Ignore()` に置き換えられる
- ent は既存行の ID を返すため、`id = LAST_INSERT_ID(id)` を自動で付け足す（`sqlgraph/graph.go`）

MySQL の `ON DUPLICATE KEY UPDATE` は、**どのユニーク制約に引っかかっても**発動します。`OnConflictColumns` は SQLite / PostgreSQL では意味を持ちますが、MySQL では「意図の表明」にしかなりません。

なので upsert するテーブルは、**（主キーを除いて）ユニーク制約が狙った 1 つだけ**になるように設計しておく必要があります（`index.Fields(...).Unique()`、第 6 章）。

テストは SQLite で動くことが多い（第 15 章）ので、この差はテストでは見えにくい点も覚えておいてください。

---

## 8.9 削除

```go
// 1 件
client.Like.DeleteOne(l).Exec(ctx)
client.Tag.DeleteOneID(id).Exec(ctx)

// 条件で一括
client.CollectionItem.Delete().
	Where(collectionitem.CollectionID(colID)).
	Exec(ctx)
```

1 つ大事な注意があります。**論理削除 Mixin が付いたエンティティでは、`Delete` は実際には UPDATE になります**（`deleted_at` に時刻を入れる）。

どのエンティティが対象か、物理削除したいときはどうするか、は第 11 章で扱います。

---

## まとめ

- 書き込みも Builder。`SetXxx` を並べて、**`Save`（結果を返す）か `Exec`（error だけ）**で実行する
- バルク `Update` の `Save` は**行数**を返す。条件付き更新の成否判定に使える
- **`SetNillable(nil)` は「触らない」、`ClearXxx()` が「NULL にする」**
- 多対多は `AddXxxIDs` / `ClearXxx` で関係を付け外しする
- Builder の中身は **Mutation**。`Save` で Mutation が Hook を通ってから SQL になる
- 一括作成は `CreateBulk`、upsert は `OnConflictColumns` + `UpdateXxx` / `Ignore`
- upsert で **Create 側の `SetXxx` は挿入時だけ**。既存行で更新する列は `OnConflict…` の後ろの `UpdateXxx()`（挿入値を流用）か `SetXxx(v)`（別の値）で明示する
- MySQL の upsert は**どのユニーク制約でも発動する**。対象列の指定は SQL に出ない

---

[← 前: クエリ — データを読む](07-query.md) | [目次](../README.md) | [次: トランザクション・ロック・生 SQL →](09-transactions-and-sql.md)
