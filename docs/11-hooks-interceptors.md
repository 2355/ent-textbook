# 11. Hook と Interceptor

> **この章でわかること**
> ent の「割り込み」の仕組み。書き込みに割り込む Hook と、読み込みに割り込む Interceptor。論理削除 Mixin を 1 行ずつ分解して読む。`ent/runtime` が必要な理由。論理削除で現場が踏みやすい落とし穴。privacy とは何か。

11.1〜11.5 が本題です。11.6 以降は「知っておくと事故を防げる」話です。

---

## 11.1 割り込みたい場面

アプリを作っていると、「すべての書き込みで」「すべての読み込みで」何かしたい場面が出てきます。

- 更新のたびにログを残したい
- 削除を、本当の削除ではなく「削除日時を入れる更新」にしたい
- 削除済みの行は、どのクエリでも自動で除外したい

各 Repository に毎回書くと、必ず書き忘れが起きます。

Node.js の世界では、この手の処理は**ミドルウェア**で書きます。

- Express / Koa: リクエストの前後に `(req, res, next) => ...`
- Prisma: `$extends` の `query` 拡張（旧 `$use` middleware）

ent にも同じ発想の仕組みが 2 つあります。

| 仕組み | 割り込む対象 |
| --- | --- |
| **Hook** | 書き込み（Create / Update / Delete） |
| **Interceptor** | 読み込み（Query） |

どちらも `next` の前後に処理を挟む点は Koa と同じです。ただし関数の形は「`next` を受け取って新しい関数を返す」高階関数で、Redux の middleware に近い形です。

---

## 11.2 Hook — 書き込みのミドルウェア

### 形

Hook の型は次のとおりです。

```go
type Hook func(Mutator) Mutator
```

`Mutator` は「Mutation を実行するもの」です。第 8 章で見たとおり、ent の書き込みはすべて **Mutation** という中間表現を経由します。Hook はその実行を包みます。

```go
func Logger() ent.Hook {
  return func(next ent.Mutator) ent.Mutator {
    return ent.MutateFunc(func(
      ctx context.Context, m ent.Mutation,
    ) (ent.Value, error) {
      // 前処理
      v, err := next.Mutate(ctx, m)
      // 後処理
      return v, err
    })
  }
}
```

Koa で書くとこうです。

```ts
const logger = async (ctx, next) => {
  // 前処理
  await next();
  // 後処理
};
```

`ent.MutateFunc` は「普通の関数を `Mutator` として扱うための変換」です。TypeScript なら関数をそのまま渡せますが、Go では interface を満たす型に変換する必要があるため、こういうアダプタが用意されています（第 2 章の interface の話）。

### Op — どの操作か

`m.Op()` で操作の種類がわかります。

| Op | 意味 |
| --- | --- |
| `ent.OpCreate` | 作成 |
| `ent.OpUpdate` | 条件指定の一括更新 |
| `ent.OpUpdateOne` | 1 件更新 |
| `ent.OpDelete` | 条件指定の一括削除 |
| `ent.OpDeleteOne` | 1 件削除 |

Op はビットフラグなので `|` で組み合わせられます。

### hook.On — 特定の操作だけに効かせる

生成される `ent/hook` パッケージに、Hook を絞り込むヘルパがあります。

```go
hook.On(Logger(), ent.OpUpdate|ent.OpUpdateOne)
```

「更新のときだけ `Logger` を通す」という意味です。ほかに `hook.Unless`（指定 Op 以外）などもあります。

題材アプリで使っているのは `hook.On` だけです。

---

## 11.3 Hook の置き場所は 2 つ

公式ドキュメント（`hooks.md`）は Hook を 2 種類に分けています。

**ランタイム Hook** — クライアントに後から登録する。

```go
client.Use(Logger())            // 全エンティティ
client.User.Use(Logger())       // User だけ
```

ログ・メトリクス・トレースのような横断的関心ごと向けです。

**スキーマ Hook** — スキーマの `Hooks()` メソッドに書く。

```go
func (User) Hooks() []ent.Hook {
  return []ent.Hook{ /* ... */ }
}
```

エンティティ固有のロジック向けです。

実行順は「ランタイム Hook → スキーマ Hook」です。

### 題材アプリではどちらを使っているか

- **ランタイム Hook（`client.Use`）は未使用**です。DB 接続でクライアントを作るときも登録していません
- スキーマ Hook も、各スキーマに直接 `Hooks()` を書いている箇所はありません
- 唯一の Hook は、**Mixin が持つ Hook**です（11.5 で読みます）

Mixin は第 6 章で見たとおり「スキーマ部品」なので、Mixin の Hook はスキーマ Hook として扱われます。

---

## 11.4 Interceptor — 読み込みのミドルウェア

Interceptor は Query 版の Hook です。

```go
ent.InterceptFunc(func(next ent.Querier) ent.Querier {
  return ent.QuerierFunc(func(
    ctx context.Context, q ent.Query,
  ) (ent.Value, error) {
    // 実行前
    v, err := next.Query(ctx, q)
    // 実行後（結果を加工できる）
    return v, err
  })
})
```

形は Hook とそっくりです。

### Traverser — もう一段手前で割り込む

Interceptor にはもう 1 つの形、**Traverser** があります。

```go
intercept.TraverseFunc(func(
  ctx context.Context, q intercept.Query,
) error {
  // q に条件を足す
  return nil
})
```

違いは割り込むタイミングです。公式ドキュメント（`interceptors.mdx`）の説明を要約すると:

- **Intercept**: SQL を実行する最終段で動く。結果の加工もできる。ログやキャッシュ向き
- **Traverse**: グラフ走査の**各ステップ**で動く。中間のクエリにも条件を足せる。**デフォルトの絞り込み条件**向き

「各ステップ」とは、第 7 章のエッジ走査のことです。

```go
client.User.Query().
  QueryGroups(). // User の Traverse
  QueryPosts().  // Group の Traverse
  All(ctx)       // Post の Traverse と
                 // Intercept
```

公式（`interceptors.mdx`）のとおり、各段の Traverse は「次の段へ進むとき」に適用され、最後の段は終端メソッドで適用されます。

途中のエンティティにも条件が入るので、「削除済みユーザーの投稿」が紛れ込みません。

論理削除のような「常にこの条件を付けたい」用途には Traverse が向いています。題材アプリの Mixin も `TraverseFunc` を使っています。

### 題材アプリでの利用

- `client.Intercept(...)` などのランタイム登録は**未使用**
- 唯一の Interceptor は論理削除 Mixin のもの

---

## 11.5 SoftDeleteMixin を分解して読む

題材アプリの Hook と Interceptor は、すべてこの 1 ファイルにあります。

`ent/common/softdelete_mixin.go`

`User` や `Article` など、論理削除したいエンティティの `Mixin()` に組み込まれています。

この Mixin は、ent 公式ドキュメント（`interceptors.mdx` の Soft Delete の例）とほぼ同じ形です。公式の例をそのまま採用している、と考えて読んでください。

### (1) 列を足す

```go
type SoftDeleteMixin struct {
  mixin.Schema
}

func (SoftDeleteMixin) Fields() []ent.Field {
  return []ent.Field{
    field.Time("deleted_at").
      Optional().
      Comment("論理削除日時"),
  }
}
```

`mixin.Schema` を埋め込むと、Mixin に必要なメソッドの空実装が手に入ります（TypeScript の抽象クラスを継承するイメージ）。必要なものだけ上書きします。

ここでは `deleted_at` 列を足しています。

### (2) スキップ用のフラグ

```go
type softDeleteKey struct{}

func SkipSoftDelete(
  parent context.Context,
) context.Context {
  return context.WithValue(
    parent, softDeleteKey{}, true,
  )
}
```

`context.Context` に「論理削除を無視せよ」という印を付ける関数です。

キーに専用の空 struct 型を使うのは Go の定石です。ほかのパッケージがたまたま同じキーを使って衝突する、という事故が型レベルで起きなくなります。

React で言えば、Context を「値を下流に渡す袋」として使うのと同じ発想です。呼び出しチェーンの奥にある Hook / Interceptor まで、引数を増やさずにフラグが届きます。

### (3) 読み込み: 自動で除外する

```go
func (d SoftDeleteMixin) Interceptors(
) []ent.Interceptor {
  return []ent.Interceptor{
    intercept.TraverseFunc(func(
      ctx context.Context, q intercept.Query,
    ) error {
      if skip, _ := ctx.Value(
        softDeleteKey{}).(bool); skip {
        return nil
      }
      d.P(q)
      return nil
    }),
  }
}
```

1. ctx にスキップの印があれば何もしない
2. なければ `d.P(q)` でクエリに条件を足す

`ctx.Value(...).(bool)` は第 2 章の型アサーションです。印が無いと `ok` が false になり、`skip` は `bool` のゼロ値 `false` になります。

### (4) P — 条件を足す部品

```go
func (d SoftDeleteMixin) P(
  w interface {
    WhereP(...func(*sql.Selector))
  },
) {
  w.WhereP(
    sql.FieldIsNull(
      d.Fields()[0].Descriptor().Name,
    ),
  )
}
```

- `d.Fields()[0]` は (1) で定義した `deleted_at`
- `sql.FieldIsNull(...)` は「その列 IS NULL」という述語
- `WhereP` は、生成された型付き述語ではなく**SQL レベルの述語**を直接足すメソッド

引数の型が interface になっているのがポイントです。

Mixin は特定のエンティティを知りません。`UserQuery` かもしれないし `ArticleQuery` かもしれない。そこで「`WhereP` を持っている何か」とだけ要求しています。TypeScript の構造的型付けで `{ whereP(...): void }` を受け取るのと同じです。

### (5) 書き込み: 削除を更新にすり替える

```go
func (d SoftDeleteMixin) Hooks() []ent.Hook {
  return []ent.Hook{
    hook.On(
      func(next ent.Mutator) ent.Mutator {
        return ent.MutateFunc(/* 下で読む */)
      },
      ent.OpDeleteOne|ent.OpDelete,
    ),
  }
}
```

`hook.On(..., OpDeleteOne|OpDelete)` なので、**削除のときだけ**動きます。中身はこうです。

```go
if skip, _ := ctx.Value(
  softDeleteKey{}).(bool); skip {
  return next.Mutate(ctx, m) // 物理削除
}
mx, ok := m.(interface {
  SetOp(ent.Op)
  Client() *gen.Client
  SetDeletedAt(time.Time)
  WhereP(...func(*sql.Selector))
})
if !ok {
  return nil, fmt.Errorf(
    "unexpected mutation type %T", m)
}
d.P(mx)
mx.SetOp(ent.OpUpdate)
mx.SetDeletedAt(time.Now())
return mx.Client().Mutate(ctx, m)
```

順に読みます。

1. スキップの印があれば、そのまま `next` に渡す → **本当に削除**される
2. `m` を「必要なメソッドを持つ interface」に型アサーションする。`SetDeletedAt` は `deleted_at` 列があるエンティティの Mutation にしか生成されないので、ここで確かめている
3. `d.P(mx)` で「`deleted_at IS NULL` の行だけ」に絞る（すでに消えた行の日時を上書きしないため）
4. **Op を Update に書き換える**
5. `deleted_at` に現在時刻を入れる
6. `next` ではなく `Client().Mutate` で**最初から実行し直す**

6 がこの Mixin の肝です。Op を書き換えた Mutation を、改めてクライアントに投げ直しています。今度は Op が Update なので、この Hook（削除専用）は素通りし、普通の UPDATE 文が発行されます。

### 呼び出し側から見ると

```go
// 論理削除（UPDATE ... SET deleted_at = now）
client.Article.DeleteOneID(id).Exec(ctx)

// 物理削除
client.Article.DeleteOneID(id).
  Exec(common.SkipSoftDelete(ctx))

// 削除済みも含めて読む
client.Article.Query().
  All(common.SkipSoftDelete(ctx))
```

**呼び出し側のコードは普通の削除と変わりません。** これが Hook で実装する利点であり、同時に怖さでもあります。コードを読んだだけでは論理削除だと気づけないからです。

題材アプリのチームの規約にも「SoftDeleteMixin を適用したエンティティでは `Delete()` / `DeleteOne()` は自動的に論理削除になる」と明記されています。暗黙の挙動は、規約やコメントで明文化しておくのが大切です。

---

## 11.6 ent/runtime — なぜ import が必要なのか

アプリのエントリーポイント（`main.go`）に、使っていないように見える import があります。

```go
import (
  _ "example.com/app/ent/runtime"
)
```

`_` は「名前は使わないが、パッケージの初期化処理だけ走らせたい」という Go の書き方です。TypeScript の `import "./polyfill";`（副作用だけの import）と同じです。

### 何をしているのか

生成されたエンティティごとのパッケージ（たとえば `ent/article/article.go`）を見ると、Hook の入れ物が空の変数として宣言されています。

```go
// Note that the variables below are
// initialized by the runtime package
var (
  Hooks        [1]ent.Hook
  Interceptors [1]ent.Interceptor
)
```

これを埋めるのが `ent/runtime/runtime.go` です。

```go
article.Hooks[0] = articleMixinHooks0[0]
article.Interceptors[0] = articleMixinInters0[0]
```

### なぜ別パッケージなのか

`import cycle`（循環 import）を避けるためです。

- スキーマ（`ent/schema`）は Hook を書くために `ent/hook` を import する
- `ent/hook` は生成された `ent` を import する
- 生成された `ent` が Hook を得るためにスキーマを import すると、**輪ができる**

Go は循環 import をコンパイルエラーにします（TypeScript は循環 import を許してしまい、実行時に `undefined` や初期化前参照のエラーを踏みます）。

そこで ent は、スキーマと生成コードの**両方を import する別パッケージ**（`ent/runtime`）を作り、起動時に結び付けます。公式ドキュメント（`hooks.md` の Hooks Registration）は、この import を「必須（MUST）」としています。

### import し忘れると

Hook / Interceptor の要素や、`Default(time.Now)` のような Default 関数が nil のまま残ります。生成コードはこれを検知して、次のようなエラーを返します。

```text
ent: uninitialized hook
(forgotten import ent/runtime?)

ent: uninitialized interceptor
(forgotten import ent/runtime?)

ent: uninitialized
article.DefaultCreatedAt
(forgotten import ent/runtime?)
```

書き込みだけでなく読み込み（生成された `prepareQuery`）でも出ます。論理削除を持たないエンティティでも、Default 関数の初期化漏れで `Create` が失敗します。

エラーメッセージがそのまま原因を教えてくれるので、見かけたらこの import を疑ってください。

エントリーポイントが複数ある（API サーバー、バッチ、CLI など）場合は、それぞれが直接または間接に `ent/runtime` を読み込んでいるかを確かめておくと安心です。`go list -deps ./cmd/xxx` の出力に含まれていれば読み込まれています。テスト用の `ent/enttest` は生成時に `ent/runtime` を import するので、テストでも Mixin は効きます。

---

## 11.7 落とし穴: HasXxxWith には効かない

論理削除でもっとも踏みやすい落とし穴です。

### 何が起きるか

第 7 章の `HasXxxWith` は、関連先の条件で絞り込む述語でした。

```go
client.Collection.Query().
  Where(collection.HasArticlesWith(
    article.StatusEQ(article.StatusPublished),
  )).
  All(ctx)
```

これは SQL では `IN (SELECT ...)` や `EXISTS (SELECT ...)` という**サブクエリ**になります（エッジの種類で形が変わります。この例は中間テーブル経由なので `IN` です。`dialect/sql/sqlgraph/graph.go` の `HasNeighborsWith`）。

サブクエリは「`Article` のクエリ」としては実行されません。述語として SQL 片が組み立てられるだけです。だから **Article の Interceptor は呼ばれず、`deleted_at IS NULL` が付きません**。論理削除済みの記事も条件を満たしてしまいます。

一方、`WithXxx`（eager load）や `QueryXxx`（走査）は本物のクエリとして実行されるので、Interceptor が効きます。

| 書き方 | 論理削除の除外 |
| --- | --- |
| `Query()` / `QueryXxx()` | 効く |
| `WithXxx()` | 効く |
| `QueryXxx().Only()` | 効く |
| `HasXxxWith(...)` | **効かない** |

### 対策: 明示する

```go
collectionitem.HasArticleWith(
  article.DeletedAtIsNil(),
)
```

`HasXxxWith` の中に `DeletedAtIsNil()` を**明示的に書きます**。なぜ書いているのかをコメントに残しておかないと、「Mixin があるのに冗長だ」と消されかねません。

### なぜ危ないのか — 非対称が nil を生む

中間テーブル経由で特に問題になります。

- 絞り込み（サブクエリ）は削除済みの記事を**通す**
- その後の eager load は削除済みの記事を**返さない**

結果、中間テーブル（`CollectionItem`）の行は返るのに、その先の記事が nil、という状態が生まれます。

> **現場メモ: 1 本の論理削除で一覧ごと null になった**
>
> 記事を論理削除したところ、コレクション画面の GraphQL レスポンスが**丸ごと null** になる事故がありました。
>
> 1. 記事を削除しても、中間テーブルの `CollectionItem` 行は残る（論理削除なので Cascade も効かない）
> 2. コレクション内の項目の一覧は、`CollectionItem` を起点に取得していた
> 3. 各項目から `item.article` を eager load すると、Interceptor が削除済みの記事を除外して **nil**
> 4. GraphQL スキーマでは `article: Article!`（non-null）。null を返せないので、GraphQL の仕様どおりエラーが親へ伝播した。このときは祖先のフィールドもすべて non-null だったため、**レスポンス全体が null** になった（null 可の祖先があれば、伝播はそこで止まる。entgql の Connection の `edges` / `node` は null 可）
>
> 対処は 2 段構えでした。
>
> - 項目の取得そのものを `HasArticleWith(article.DeletedAtIsNil())` で絞り、「中身が消えた項目」を最初から返さない
> - 同じ書き方をしている箇所を洗い出し、チームのナレッジに「`HasXxxWith` には論理削除が効かない」と注意書きを追加する
>
> 教訓は、**「論理削除したとき、関連する行はどう見えるべきか」を設計時に決めておく**ことです。物理削除なら外部キー制約（Cascade や Restrict）が決めてくれますが、論理削除では DB が何も助けてくれません。

---

## 11.8 SkipSoftDelete の使いどころと効く範囲

### ctx 全体に効く

`SkipSoftDelete` は ctx に印を付けるので、**その ctx を使うすべてのクエリ**に効きます。

意図的に使う分にはよいのですが、ctx はそのまま下流の関数に渡っていきます。**「どこまで効くか」を意識して範囲を狭く使う**のが安全です。

効くのは読み込みだけではありません。11.5 の Hook も同じ印を見るので、**同じ ctx で `Delete` すると物理削除**になります。「削除済みも含めて確認 → 問題なければ削除」と続く処理では、確認用の ctx を削除に流用しないでください。

逆に、**届かない範囲**もあります。Resolver の中で `ctx = SkipSoftDelete(ctx)` としても、Go の context は呼び出し元へ戻せません。GraphQL の子フィールドの Resolver は、gqlgen が持つ元の ctx で呼ばれます。その経路で関連を取り直すと、印が無いので削除済みは除外されます。

### 引数の ctx から作る

```go
// NG: 新しい空の ctx から作っている
ctx := common.SkipSoftDelete(
  context.Background())

// OK: 受け取った ctx から作る
ctx = common.SkipSoftDelete(ctx)
```

ctx には「論理削除をスキップする印」以外にも大事なものが載っています。

- **トランザクション**（第 9 章。tx 用クライアントは ctx で運ばれる）
- ログの相関 ID
- タイムアウトやキャンセル

`context.Background()` から作り直すと、これらが**全部消えます**。トランザクションの外でクエリが走り、ロールバックしても巻き戻らない、という事故につながります。`SkipSoftDelete` に限らず、ctx は**受け取ったものを包んで使う**のが Go の原則です。

> **現場メモ: 退会ユーザーが 1 人いるだけで一覧全体がエラーに**
>
> ユーザーは退会時に論理削除される設計でした。ところが、退会ユーザーが書いたコメントは残ります。
>
> 管理画面のコメント一覧で、GraphQL の `Comment.author` は `User!`（non-null）でした。論理削除の除外が効いたまま eager load すると、退会者のコメントでは `author` が nil になり、11.7 と同じ仕組みで**一覧全体がエラー**になりました。
>
> 対処は、一覧の Resolver で `SkipSoftDelete(ctx)` を使い、退会者も返すことでした。これが効いたのは、その Resolver の中で関連の eager load（CollectFields）まで済ませていたからです。上で見たとおり、子フィールドの Resolver が取り直す経路には印が届きません。そのうえで、次の点をコメントに明記しました。
>
> - 退会者の行も返る理由（non-null リレーションのため）
> - ctx 全体に効くので、eager load する**ほかの関連でも**論理削除済みの行が返ること
> - 退会者の個人情報（氏名など）は、退会時に匿名化されている前提であること。画面では空欄として扱う
>
> 「削除済みを見せない」は既定として正しくても、**親が残り続けるデータ**から参照される場合は、どう見せるかを決めておく必要があります。

> **現場メモ: 「使われているか」の確認には削除済みも数える**
>
> 記事のサムネイル画像を複数の記事で共有できる機能で、「ほかにこの画像を使っている記事があるか」を確認してからファイルを消す処理がありました。
>
> 調べると、**論理削除済みの記事とサムネイルを共有しているもの**が少なくない件数ありました。論理削除済みの記事は復元される可能性があるので、確認から外すと、復元したときに画像が消えている事故になります。そこで確認のクエリに `SkipSoftDelete(ctx)` を足しました。
>
> もう 1 つの学びは、**新しいデータの作り方を直しても、既存データに同じ状態が残っている限り防御コードは外せない**ことです。「今後は共有しない作りにした」だけでは、過去に作られた共有は消えません。

---

## 11.9 privacy — 生成されているが未使用

`ent/entc.go` の feature flag に `privacy` が入っているので、`ent/privacy` パッケージは生成されています。ただし題材アプリでは**どこからも使っていません**（スキーマに `Policy()` も無し）。

### 何をするものか

privacy は「誰が何を読み書きできるか」をスキーマに宣言する仕組みです（公式 `privacy.mdx`）。

```go
func (User) Policy() ent.Policy {
  return privacy.Policy{
    Mutation: privacy.MutationPolicy{
      privacy.AlwaysDenyRule(),
    },
    Query: privacy.QueryPolicy{
      privacy.AlwaysAllowRule(),
    },
  }
}
```

ルールを順に評価し、各ルールは次のいずれかを返します。

- `privacy.Allow` — 許可して評価終了
- `privacy.Deny` — 拒否して評価終了
- `privacy.Skip` — 次のルールへ

書き込み側は Hook（`Hooks[0]` に差し込まれる）、読み込み側は生成された `prepareQuery` の中で直接評価されます（`entc/gen/template/runtime.tmpl`、`builder/query.tmpl`）。Hook と同じ割り込みの発想で動く、認可専用の層です。

### 題材アプリでは認可をどこでやっているか

ent の外です。

- GraphQL のディレクティブ（`@isAuthenticated` など。第 13 章）
- Resolver で認可情報を取り出して判定する仕組み

DB 層ではなく API 層で認可する設計を取っているので、privacy を使う理由がない、という位置づけと読めます（チームの規約に明記はありません）。

---

## 11.10 使いどころの判断

Hook / Interceptor は強力ですが、**見えないところで挙動を変える**仕組みです。

向いているもの:

- 論理削除のような、**全員が例外なく守るべき**規則
- ログ・トレースのような、**ビジネスロジックに影響しない**処理

向いていないもの:

- 特定のユースケースだけで必要な処理 → Usecase に書く
- 外部 API 呼び出しのような重い副作用 → トランザクションとの整合が難しくなる

題材アプリが Hook / Interceptor を論理削除の 1 か所だけに絞っているのは、この判断の表れと読めます。

それでも 11.7・11.8 で見たとおり、論理削除 1 つでも落とし穴は多くあります。Hook / Interceptor を足すときは、**「効かない経路」と「効きすぎる経路」**を一緒に洗い出してください。

---

## まとめ

- **Hook は書き込み、Interceptor は読み込み**に割り込むミドルウェア。`next` の前後に処理を挟む考え方は Koa と同じ
- Hook は `hook.On(..., ent.OpDelete)` のように **Op で絞れる**
- Interceptor の **Traverse は走査の各段で効く**ので、デフォルトの絞り込みに向く
- 題材アプリの Hook / Interceptor は **SoftDeleteMixin だけ**。`client.Use` などのランタイム登録は未使用
- SoftDeleteMixin は **Delete の Op を Update に書き換えて投げ直す**ことで論理削除を実現している
- 物理削除・削除済み込みの取得は **`SkipSoftDelete(ctx)`**。ctx 全体に効く。**必ず受け取った ctx から作る**
- スキーマの Hook を有効にするには **`ent/runtime` の import が必須**。循環 import を避けるための分離
- **`HasXxxWith` のサブクエリには Interceptor が効かない**。`DeletedAtIsNil()` を明示する
- 論理削除では DB が関連を守ってくれない。**削除時に関連がどう見えるか**を設計時に決める
- privacy は生成されているが未使用。認可は GraphQL 層で行っている

---

[← 前: エラーの扱い](10-errors.md) | [目次](../README.md) | [次: マイグレーション →](12-migrations.md)
