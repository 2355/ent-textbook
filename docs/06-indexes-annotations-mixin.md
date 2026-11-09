# 6. インデックス・アノテーション・Mixin

> **この章でわかること**
> 複合インデックスと複合ユニークの書き方。スキーマに「おまけ情報」を付けるアノテーションの仕組みと、全テーブルに付けておきたい設定。型名を変えるときの名前の固定。Mixin でスキーマの部品を使い回す方法と、論理削除 Mixin の外形。

前の 2 章で、列（フィールド）と関係（エッジ）を見ました。スキーマ定義のメソッドは、あと 3 つ残っています。

```go
func (T) Indexes() []ent.Index
func (T) Annotations() []schema.Annotation
func (T) Mixin() []ent.Mixin
```

`Fields` / `Edges` / `Annotations` はほぼ全スキーマに書かれ、`Indexes` と `Mixin` は必要なスキーマにだけ書かれる、というのが典型的な姿です。この章でこれらを片付けます。

---

## 6.1 インデックス

### 書き方

```go
// ent/schema/collection.go
func (Collection) Indexes() []ent.Index {
  return []ent.Index{
    // 公開一覧の取得はpublished_atでの
    // 絞り込みと並べ替えを行うため
    index.Fields("published_at"),
  }
}
```

`index.Fields(列名...)` で 1 つのインデックスです。Prisma の `@@index([published_at])` に相当します。

### 複合インデックス

列名を複数並べると、複合インデックスになります。**並べた順が、インデックスの列順**です。

```go
// ent/schema/article.go
// 論理削除を除いた記事一覧を
// 公開日時順に取得するため
index.Fields("deleted_at", "published_at"),
// 論理削除を除いた記事一覧を
// 閲覧数順に取得するため
index.Fields("deleted_at", "view_count"),
```

先頭が `deleted_at` なのは、論理削除（6.6 節）によって**ほぼすべてのクエリに `deleted_at IS NULL` が付く**からです。

インデックスには、このように**どのクエリのためのものか**をコメントで書いておくのがおすすめです。後から「消してよいか」を判断できます。

### 複合ユニーク

`.Unique()` を付けると、ユニークインデックスになります。

```go
// ent/schema/collection_item.go
index.Fields("collection_id", "article_id").
  Unique(),
```

「同じコレクションに同じ記事は 1 回だけ」という制約です。Prisma の `@@unique([collection_id, article_id])` と同じです。

第 4 章で見たフィールドの `.Unique()` は**単一列**のユニークでした。**複数列の組み合わせでユニークにしたいときは、こちらを使います。**

実務の複合ユニークのほとんどは、次のどちらかです。

- `user_id` × 何か（同じユーザーが同じものを 2 回登録しない）
- 中間テーブルの 2 つの外部キー（同じ組み合わせを 2 回持たない）

### 制約の理由を書く

複合ユニークにも、なぜその組み合わせなのかを書いておくと、後から読む人が助かります。

```go
// ent/schema/tag.go
// 種類(kind)が違えば同じ名前・スラッグのタグが
// 存在しうるため、name/slug単体ではなく
// kindとの複合でユニークにする
func (Tag) Indexes() []ent.Index {
  return []ent.Index{
    index.Fields("name", "kind").Unique(),
    index.Fields("slug", "kind").Unique(),
  }
}
```

逆に、**あえて張らなかった制約**の理由を残すのも有効です。

```go
// ent/schema/collection_item.go
// (collection_id, position) にはユニーク制約を
// 張らない。保存が全行入れ替えのため重複は発生
// せず、将来部分更新に変更した際の一時的な重複で
// 保存が止まる事故を避けるため。
```

### 外部キー列にもインデックスを張れる

`collection_id` のような外部キー列は、第 5 章で見たとおり `Field()` で普通のフィールドとして宣言できます。だから `index.Fields` にそのまま書けます。

公式ドキュメント（schema-indexes「Index On Edge Fields」）でも、エッジの列を使ったインデックスはこの方法が案内されています。外部キーを `Field()` で明示しておけば、困ることはありません。

### インデックス名

インデックス名は ent が自動で付けます。生成された `ent/migrate/schema.go` を見ると、`型名の小文字_列名_列名` という形です。

```go
Name: "collectionitem_collection_id_article_id",
```

---

## 6.2 アノテーションとは

アノテーションは、**スキーマに付ける「おまけ情報」**です。

公式ドキュメント（schema-annotations）の説明では、アノテーションはコード生成時にスキーマに添付される任意のメタデータで、**拡張機能やテンプレートがそれを読んで使う**ものです。

ent 本体のフィールドやエッジの定義は、「どの DB でも通じる」ように作られています。MySQL の照合順序や GraphQL の公開方法のような、**特定の拡張だけが気にする情報**は、アノテーションに入れて渡します。

TypeScript で言えば、デコレータで付けるメタデータに近いものです。

### 付ける場所は 3 つ

アノテーションは 3 つの場所に付けられます。

```go
// 1. スキーマ（テーブル）全体
func (T) Annotations() []schema.Annotation

// 2. フィールド
field.String("x").Annotations(...)

// 3. エッジ
edge.To("y", Y.Type).Annotations(...)
```

### 誰が読むのかで 2 種類

この本で扱うアノテーションは、パッケージで 2 種類に分かれます。

| パッケージ | 読む人 | 用途 |
| --- | --- | --- |
| `entsql` | SQL 方言・マイグレーション | DB の設定 |
| `entgql` | entgql 拡張 | GraphQL の公開方法 |

```go
// ent/schema/article.go
func (Article) Annotations() []schema.Annotation {
  return []schema.Annotation{
    entgql.QueryField(),
    entgql.RelayConnection(),
    entgql.MultiOrder(),
    entsql.Annotation{
      Collation: "utf8mb4_unicode_ci",
    },
  }
}
```

**`entgql.` で始まるものは、すべて第 13 章で扱います。** スキーマを読むときは、ひとまず「GraphQL 用の指示」として読み飛ばして構いません。

ただし、ここで 1 つ覚えておいてください。entgql を使うと、**`ent/schema` の 1 ファイルに DB の話と GraphQL の話が同居します**。「この型を GraphQL でどう公開するか」「誰に見せるか」まで、スキーマ側に書くことになります。違和感があるかもしれませんが、entgql の設計としてそういうものだと割り切って読むのが近道です。

この章では `entsql` のほうを見ていきます。

---

## 6.3 entsql — DB 向けのアノテーション

よく使う `entsql` の設定です。

| 設定 | 付ける場所 | 内容 |
| --- | --- | --- |
| `Collation` | スキーマ | 照合順序 |
| `WithComments(true)` | スキーマ | コメントを DB にも書く |
| `Table(...)` | スキーマ | テーブル名の上書き |
| `Default(...)` | フィールド | DB 側のデフォルト値（第 4 章） |
| `OnDelete` | エッジ | 削除時の挙動（第 5 章） |

`Default` は第 4 章、`OnDelete` は第 5 章で扱いました。残りの 3 つを見ます。

### Collation — 照合順序

```go
entsql.Annotation{
  Collation: "utf8mb4_unicode_ci",
},
```

照合順序は、MySQL が**文字列を比較・並べ替えるときのルール**です。

ent の MySQL 方言は、何も指定しないとテーブルを `utf8mb4_bin` で作ります（`dialect/sql/schema/mysql.go`）。`_bin` はバイト列としての比較で、大文字と小文字を区別します。

題材アプリは全テーブルに `utf8mb4_unicode_ci` を指定しています。`_ci` は case-insensitive、つまり**大文字と小文字を区別しない比較**です。

これは検索やユニーク制約にも影響します。

- `WHERE email = 'A@example.com'` が `a@example.com` の行にもヒットする
- ユニーク列で `abc` と `ABC` が**重複扱い**になる
- ent の `TitleContains("go")` のような部分一致検索も、`Go` や `GO` にヒットする

最後の点は見落としがちです。ent の述語は SQL の `LIKE` を組み立てるだけで、**大文字小文字を区別するかどうかは DB の照合順序が決めます**。「検索で大文字小文字を区別したい／したくない」という要件が来たら、まずテーブルの照合順序を確認してください。

**新しいスキーマを作るときも、既存と同じ指定を入れてください。** 1 つだけ照合順序が違うテーブルがあると、JOIN や比較でエラーや予期しない結果の原因になります。

### WithComments — コメントを DB にも書く

第 4 章で、`.Comment()` は既定では Go のコードにしか反映されないと書きました。

```go
entsql.WithComments(true),
```

をスキーマのアノテーションに入れると、**フィールドのコメントが DB の列コメントにも書き込まれます**。`SHOW CREATE TABLE` や DB クライアントからも説明が見えるようになります。

新しく作るスキーマには付けておくと、DB だけを見る人（データ分析など）にも親切です。

### Table — テーブル名の上書き

ent のテーブル名は、型名を**複数形の snake_case** にしたものが既定です（`entc/gen/type.go` の `Table()`）。`CollectionItem` なら `collection_items` です。

既存のテーブル名がこの規則に合わない場合、`entsql.Table` で上書きします。

```go
// ent/schema/article_stat.go
entsql.Table("article_stats"),
```

`entsql.Annotation{Table: "article_stats"}` と書いても同じです。

### 複数の entsql.Annotation は合成される

1 つのスキーマに `entsql.Annotation` を 2 つ書いても構いません。

```go
// ent/schema/collection_item.go
entsql.WithComments(true),
entsql.Annotation{
  Collation: "utf8mb4_unicode_ci",
},
```

`entsql.WithComments(true)` は中身が `&entsql.Annotation{WithComments: ...}` を返すだけの関数です。同じ種類のアノテーションは `Merge` メソッドで**1 つに合成されます**（`dialect/entsql/annotation.go`）。

---

## 6.4 型名を変えたいとき

entimport で逆生成した型には、不自然な名前が残ることがあります（第 4 章のコラム）。あとから名前を直したくなったとき、**どこまで変えたいか**で手段が分かれます。

### GraphQL の型名だけ変える: entgql.Type

Go のコードはそのままで、GraphQL 上の型名だけを直したいなら、スキーマのアノテーションに `entgql.Type` を足します。

```go
func (Addresse) Annotations() []schema.Annotation {
  return []schema.Annotation{
    entgql.Type("Address"),
  }
}
```

entgql はスキーマの型名を GraphQL の型名に使いますが、`entgql.Type` があればそちらを優先します（entgql の `template.go` の `gqlTypeFromNode`）。Go 側の `ent.Addresse` は変わりません。

ただし変わるのは型名 1 つではありません。entgql は Connection・Edge・Order・WhereInput・Create/Update 入力型の名前も GraphQL の型名から作るので（entgql の `schema.go` の `paginationNames` など）、`AddresseConnection` や `AddresseWhereInput` もまとめて改名されます。変数の型としてこれらを書いている既存のクエリは検証エラーになるので、**クライアントにとっては破壊的変更**です。

### ent の型名ごと変える: Table で既存テーブルを固定

Go の型名も直したいなら、スキーマの型名を変えます。このときの落とし穴が**テーブル名**です。

前節のとおり、テーブル名は型名から自動で決まります。型名を変えると、**既定のテーブル名も変わる**ことがあります。

- `ArticleStat` → 既定のテーブル名は `article_stats`
- `ArticleStatistic` に改名 → 既定は `article_statistics`

このまま `make gen` してマイグレーションを作ると、ent は「`article_stats` を消して `article_statistics` を作る」差分を出しかねません。

そこで、改名と同時に**既存のテーブル名を `entsql.Table` で固定**します。

```go
func (ArticleStatistic) Annotations() []schema.Annotation {
  return []schema.Annotation{
    entsql.Annotation{Table: "article_stats"},
  }
}
```

これでテーブル名は変わりません。ただし `entsql.Table` が固定するのは**テーブル名だけ**です。外部キー列を `edge.Field` で明示していないエッジや、`Through` を使わない多対多の中間テーブルは、名前が**型名から自動で決まります**（ent の `entc/gen/graph.go`）。こうした列・テーブルがあるなら、`edge.Field` / `edge.StorageKey` で既存の名前に固定する必要があります。GraphQL の型名も Go の型名に追従して変わるので、クライアントへの影響は別途確認してください。

| やりたいこと | 手段 | 変わるもの |
| --- | --- | --- |
| GraphQL の名前だけ | `entgql.Type` | GraphQL |
| ent の型名ごと | 型名の変更 + `entsql.Table` | Go と GraphQL |

どちらの場合も、変更後に生成されたマイグレーションが**空になること**を必ず確認してください（第 12 章）。空でなければ、どこかの名前が型名に引きずられて変わっています。

---

## 6.5 Mixin — スキーマの部品

### 何のためにあるか

`created_at` と `updated_at` のような、**多くのスキーマに共通する定義**があるとします。

各スキーマに毎回コピーするのは面倒ですし、直すときに漏れます。そこで、**共通部分を別の型にまとめて、各スキーマに混ぜ込む**仕組みが Mixin です。

公式ドキュメントの説明はこうです。

> A `Mixin` allows you to create reusable pieces of `ent.Schema` code that can be injected into other schemas using composition.

Prisma には同等の機能がなく、共通の列は各モデルに書くしかありません。TypeScript で言えば、複数のオブジェクト型に共通のプロパティをまとめた型を `&` で合成するイメージです。

### Mixin が持てるもの

`ent.Mixin` は interface で、スキーマと同じ名前のメソッドを持ちます。

```go
type Mixin interface {
  Fields() []Field
  Edges() []Edge
  Indexes() []Index
  Hooks() []Hook
  Interceptors() []Interceptor
  Policy() Policy
  Annotations() []schema.Annotation
}
```

**フィールドやインデックスだけでなく、Hook や Interceptor（振る舞い）も混ぜ込めます。** ここが単なる「共通カラムのコピー」と違うところです。

### mixin.Schema を埋め込む

7 つ全部を実装するのは大変です。そこで ent は、全メソッドが空の実装を返す `mixin.Schema` を用意しています（`schema/mixin/mixin.go`）。

```go
type MyMixin struct {
  mixin.Schema
}
```

第 2 章で見た**構造体の埋め込み**です。`mixin.Schema` の空実装が引き継がれるので、**必要なメソッドだけを上書き**すれば済みます。

TypeScript で言えば、全メソッドが空の基底クラスを `extends` して、必要なものだけ override する感覚です。

### スキーマに混ぜる

スキーマ側では、`Mixin()` メソッドで使う Mixin を並べます。

```go
// ent/schema/article.go
func (Article) Mixin() []ent.Mixin {
  return []ent.Mixin{
    common.SoftDeleteMixin{},
  }
}
```

これだけで、Mixin の `Fields()` が返す列が `Article` に追加されます。

生成された `ent/migrate/schema.go` を見ると、**Mixin の列は `id` のすぐ後、スキーマ自身の列より前**に並びます。

```go
{Name: "id", Type: field.TypeUint64, ...},
{Name: "deleted_at", Type: field.TypeTime,
  Nullable: true},
{Name: "uuid", Type: field.TypeUUID, ...},
```

---

## 6.6 論理削除の Mixin: SoftDeleteMixin

題材アプリで使っている Mixin は 1 つだけです。論理削除のための `SoftDeleteMixin` で、`ent/common/softdelete_mixin.go` に置いています。

ユーザー・記事・コメントなど、「消しても履歴として残したい」スキーマに付けています。

### 外形

中身の詳しい動きは第 11 章で扱います。ここでは**何を混ぜ込んでいるか**だけを見ます。

```go
type SoftDeleteMixin struct {
  mixin.Schema
}

// 1. 列を足す
func (SoftDeleteMixin) Fields() []ent.Field {
  return []ent.Field{
    field.Time("deleted_at").
      Optional().
      Comment("論理削除日時"),
  }
}

// 2. 読み込みに割り込む
func (d SoftDeleteMixin) Interceptors()
  []ent.Interceptor { ... }

// 3. 書き込みに割り込む
func (d SoftDeleteMixin) Hooks() []ent.Hook {
  ... }
```

3 つの部品が 1 つの Mixin にまとまっています。

1. **`Fields`** — `deleted_at` 列を足す
2. **`Interceptors`** — クエリに `deleted_at IS NULL` を自動で足す
3. **`Hooks`** — `Delete` を「`deleted_at` に現在時刻をセットする `Update`」に書き換える

この結果、Mixin を付けたスキーマでは、

- 普通にクエリすると**削除済みの行は見えない**
- `Delete` / `DeleteOne` を呼ぶと**論理削除になる**

という振る舞いになります。**列の定義と、その列を使う振る舞いが、1 つの部品として配られる**わけです。これが Mixin の真価です。

削除済みも含めて取得したいときや、物理削除したいときは、`common.SkipSoftDelete(ctx)` で作った context を渡します。これも第 11 章で扱います。

### 実行時の登録

Mixin の Hook や Interceptor は、`ent/runtime` パッケージを import したときに登録されます。

```go
// cmd/server/main.go
import _ "example.com/app/ent/runtime"
```

公式ドキュメント（hooks「Hooks Registration」）は、スキーマの Hook を有効にするにはこの import が**必須**だとしています。Go の `import _` は「パッケージの初期化処理だけを実行する」という意味です。

この import がないと、Mixin の Hook や Interceptor が登録されません。ent のクライアントを使うエントリーポイント（API サーバー、バッチ、CLI など）を新しく作るときは、この import が直接か間接に含まれているかを確認してください。`go list -deps ./cmd/xxx` の出力に `ent/runtime` があれば取り込まれています。

### Mixin の列にもインデックスを張れる

Mixin から来た `deleted_at` も、スキーマの `Indexes()` に普通に書けます。6.1 節の `index.Fields("deleted_at", "published_at")` がその例です。

---

## まとめ

- `index.Fields(...)` でインデックス。**並べた順が列順**
- 複数列のユニークは `index.Fields(...).Unique()`。単一列ならフィールドの `.Unique()`
- インデックスには**どのクエリのためか**をコメントで残す
- アノテーションは**拡張が読むためのメタデータ**。`entsql`（DB）と `entgql`（GraphQL）の 2 種類。entgql を使うとスキーマに DB と GraphQL の話が同居する
- 照合順序は DB の設定。ent の既定は `utf8mb4_bin`。`_ci` にすると `Contains` などの検索も**大文字小文字を区別しなくなる**
- `WithComments(true)` でコメントが DB の列コメントにもなる
- 型名を変えるなら、GraphQL だけは `entgql.Type`、ent ごとなら **`entsql.Table` で既存のテーブル名を固定**する
- **Mixin は列だけでなく Hook / Interceptor も混ぜ込める**部品。`mixin.Schema` を埋め込んで必要なメソッドだけ書く
- 論理削除 Mixin を付けると、**Delete が論理削除になり、クエリから削除済みが消える**
- Mixin の Hook は **`ent/runtime` の import** で登録される

ここまででスキーマの書き方は一通り終わりです。次章からは、生成されたコードを使ってデータを読み書きします。

---

[← 前: エッジ — 関係を定義する](05-edges.md) | [目次](../README.md) | [次: クエリ — データを読む →](07-query.md)
