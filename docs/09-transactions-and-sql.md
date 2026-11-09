# 9. トランザクション・ロック・生 SQL

> **この章でわかること**
> ent の `Tx` の基本。「トランザクションを context に入れて運ぶ」設計の理由と仕組み。トランザクションの外にエンティティを持ち出すときの落とし穴。`FOR UPDATE` による行ロック。生 SQL を投げる方法と、その代償。DB 接続を組み立てる場所と、ドライバーをラップして計測を差し込む方法。

---

## 9.1 Tx の基本

ent のトランザクションは `client.Tx(ctx)` で始めます。

```go
tx, err := client.Tx(ctx)
if err != nil {
	return err
}
defer func(tx *ent.Tx) {
	_ = tx.Rollback()
}(tx)

_, err = tx.Article.Update().
	SetViewCount(0).
	Save(ctx)
// ...
return tx.Commit()
```

（閲覧数を集計し直すバッチの冒頭を簡略化）

ポイントは 2 つです。

**(1) `tx.Article` は `client.Article` と同じ API**

`ent.Tx` は、Client と同じエンティティごとのフィールドを持っています。`tx.` で始めればトランザクション内、`client.` で始めれば外、です。

**(2) `defer tx.Rollback()` で後始末を予約する**

`defer` は「この関数を抜けるときに実行する」予約です（第 2 章）。TS の `try { ... } finally { rollback() }` に相当します。

Commit 済みのトランザクションに対する Rollback は失敗するだけで害がありません。なので「とりあえず Rollback を予約しておき、成功したら Commit する」のが Go の定番です。

バッチのように 1 つの関数で完結する処理なら、この素朴な書き方で十分です。

---

## 9.2 アプリ本体の問題: tx をどう渡すか

アプリ本体はそうはいきません。

題材アプリでは、トランザクションの範囲を決めるのは **Usecase 層**、DB を触るのは **Repository 層**です（第 14 章）。ポイントを使う処理なら、こうなります。

```
Usecase
 ├─ repo.GetWallet(...)            ← 同じ tx で
 ├─ repo.CreatePointTransaction(...) ← 実行したい
 └─ repo.UpdateWalletBalance(...)
```

しかも Usecase 層は **ent に依存してはいけない**ルールです（import だけでなく、ent 由来の型を扱うことも禁止。第 14 章）。`*ent.Tx` を引数で回すことはできません。

TS / Node でも同じ悩みがあります。TypeORM なら `EntityManager` を全関数に引数で渡すか、`AsyncLocalStorage` を使うライブラリ（typeorm-transactional など）で暗黙に引き回すか。Prisma なら `$transaction(async (tx) => ...)` の `tx` を下に渡していくか。

題材アプリは **`context.Context` に tx を入れて運ぶ**方式を選んでいます。Node の `AsyncLocalStorage` とほぼ同じ発想です。

---

## 9.3 context で tx を運ぶ仕組み

### 生成されるヘルパー

ent は、context に Client や Tx を出し入れするヘルパーを生成します（`ent/ent.go`）。

```go
func NewContext(
	parent context.Context, c *Client,
) context.Context {
	return context.WithValue(
		parent, clientCtxKey{}, c,
	)
}

func FromContext(ctx context.Context) *Client {
	c, _ := ctx.Value(clientCtxKey{}).(*Client)
	return c
}
```

`NewTxContext` / `TxFromContext` も同じ形です。中身はただの `context.WithValue` です。

さらに entgql 拡張が `OpenTx` を生成します（`ent/gql_transaction.go`）。

```go
func (c *Client) OpenTx(ctx context.Context) (
	context.Context, driver.Tx, error,
) {
	tx, err := c.Tx(ctx)
	// ...
	ctx = NewTxContext(ctx, tx)
	ctx = NewContext(ctx, tx.Client())
	return ctx, tx, nil
}
```

`tx.Client()` は「このトランザクションの中で動く Client」を返します。**その Client を context に入れた新しい context** を返すのが `OpenTx` です。

### RunInTransaction

題材アプリはこれを使って `RunInTransaction` を実装しています。

```go
func (r *transactionManager) RunInTransaction(
	ctx context.Context,
	fn func(ctx context.Context) error,
) error {
	tCtx, tx, err := r.client.OpenTx(ctx)
	if err != nil {
		return wrapEntError(err)
	}
	err = fn(tCtx)
	if err != nil {
		// Rollback（失敗時の処理は省略）
		_ = tx.Rollback()
		return passThrough(err)
	}
	if err := tx.Commit(); err != nil {
		return wrapEntError(...)
	}
	return nil
}
```

Prisma の `$transaction(async (tx) => { ... })` とそっくりです。違いは、コールバックが受け取るのが `tx` ではなく **tx 入りの context（`tCtx`）**であることです。

- `fn` が error を返す → Rollback
- `fn` が nil を返す → Commit

Usecase 側はこう使います。

```go
err := u.repo.RunInTransaction(ctx,
	func(ctx context.Context) error {
		w, err := u.repo.GetWallet(
			ctx, userID, lock.Exclusive(),
		)
		// ...残高チェック
		return u.repo.UpdateWalletBalance(
			ctx, w.ID(), w.Balance()-cost,
		)
	},
)
```

Usecase は ent を一切知りません。**context を渡しているだけ**です。

### Repository 側: Client(ctx, r.client)

最後のピースが、Repository の全メソッドの冒頭にある 1 行です。

```go
client := Client(ctx, r.client)
```

中身はこうです。

```go
func Client(
	ctx context.Context, baseClient *ent.Client,
) *ent.Client {
	c := ent.FromContext(ctx)
	if c != nil {
		return c
	}
	return baseClient
}
```

- context に tx 版 Client があれば → それを使う（トランザクション内）
- なければ → 普通の Client を使う

これで **Repository は「自分がトランザクションの中にいるか」を気にせず書ける**ようになります。

### やってはいけないこと

チームの規約では、ent の Client は必ず `Client(ctx, r.client)` を通して使うことになっています。

`r.client.User.Query()` と書くと、context に tx があっても**無視して別の接続で**実行されます。ロックやロールバックの範囲から外れてしまいます。

もう 1 つ、上の `RunInTransaction` は `r.client`（素の Client）から**毎回新しいトランザクションを開きます**。`RunInTransaction` の中でさらに `RunInTransaction` を呼ぶと、入れ子ではなく**外側とは独立したトランザクション**になります（ent の `Client.Tx` が入れ子を検出して `ErrTxStarted` を返すのは、tx 版の Client から呼んだときだけです）。

ここから先は実装から導いた推論で、実行しての確認はしていません。

- 外側が `FOR UPDATE` で握った行に内側が触れると、別の接続なのでロック待ちになる。外側は内側の終了を待つので、タイムアウトするまで止まる
- 外側がロールバックしても、内側で Commit 済みの分は残る

`RunInTransaction` を入れ子で呼べる設計にしたいなら、「context に tx があればそれを使い回す」分岐を入れる必要があります。

### なぜ context なのか

context 方式には「どこから tx が来たのか見えにくい」という欠点があります。`AsyncLocalStorage` と同じ欠点です。

それでも採用している理由は次のとおりです。

- Usecase 層から ent の型を完全に追い出せる
- Repository のシグネチャにトランザクションが現れない。同じメソッドを tx の中でも外でも使える
- Go では context を第 1 引数で回すのがもともとの慣習（第 2 章）なので、引数が増えない

`NewTxContext` / `TxFromContext` / `NewContext` / `FromContext` は ent が標準で生成するヘルパーです。context で運ぶ方式は、ent 自身が想定している使い方の 1 つと言えます。

### context は「引数から」派生させる

context 方式には 1 つ大事な作法があります。**新しい context を作るときは、必ず引数で受け取った `ctx` から派生させる**ことです。

```go
// ✗ tx・相関 ID・キャンセルが消える
ctx2 := common.SkipSoftDelete(
	context.Background(),
)

// ✓ 引数の ctx に値を足す
ctx2 := common.SkipSoftDelete(ctx)
```

`SkipSoftDelete` は第 11 章で扱う「論理削除済みも含めて読む」ためのフラグです。こういうフラグを立てたいとき、手近な `context.Background()` から作ってしまうと、**それまで ctx に載っていたものが全部消えます**。

- tx 版の Client → トランザクションの外で実行される
- リクエストの相関 ID → ログが追えなくなる
- キャンセル・タイムアウト → クライアントが切断しても処理が止まらない

`context.WithValue` は「元の context に値を 1 つ足した子」を作る関数です。React の Context を Provider でネストするのと同じで、**親を引き継いでこそ意味がある**と覚えてください。

---

## 9.4 トランザクションの外にエンティティを持ち出す

tx の中で作ったエンティティを、そのまま上の層に返したくなることがあります。

```go
var created *ent.Collection
err := RunInTransaction(ctx,
	func(ctx context.Context) error {
		c, err := Client(ctx, base).Collection.
			Create().
			SetTitle(title).
			Save(ctx)
		created = c
		// ...items を作る
		return err
	},
)
return created, err // ← 危ない
```

これは**後から壊れます。**

### なぜ壊れるのか

ent のエンティティは、自分を取得した Client の設定（`config`）を持っています。生成コードを見ると分かります。

```go
// Create の保存処理（生成コード）
_node = &Collection{config: cc.config}

// エッジをたどるメソッド（生成コード）
func (c *Collection) QueryCollectionItems() *CollectionItemQuery {
	return NewCollectionClient(c.config).
		QueryCollectionItems(c)
}
```

tx 版の Client で作ったエンティティは、**tx のドライバーを握ったまま**です。コミットした後にこのエンティティから `QueryCollectionItems()` などでエッジをたどると、終わったトランザクションで SQL を投げることになり、次のエラーになります。

```
sql: transaction has already been committed or rolled back
```

これは ent ではなく Go 標準の `database/sql` のエラー（`sql.ErrTxDone`）です。

GraphQL では特にはまりやすい罠です。ミューテーションの戻り値として返したエンティティに対して、クライアントが関連フィールドを要求すると、リゾルバーが**コミット後に**エッジをたどるからです（第 7 章の `IsNotLoaded` なら引き直すパターン）。

> **現場メモ**
> フロントエンドが当初 `id` しか要求していなかったため、このバグは長いあいだ表に出ませんでした。画面改修で関連フィールドを足した瞬間にエラーになります。「動いているから大丈夫」ではなく、**tx の中で作ったエンティティを外に返していないか**をコードで確認するのが確実です。

### 対処 1: Unwrap する（公式の方法）

ent は、この場面のために `Unwrap()` を生成しています。tx 由来のエンティティが握っているドライバーを、tx を開く前の元のドライバーに戻すメソッドです。

```go
// コミット後に呼ぶ
c = c.Unwrap()
items, err := c.QueryCollectionItems().
	All(ctx)
```

公式ドキュメント（`doc/md/transactions.md`）も「トランザクション完了後に、作成したエンティティからエッジをたどるなら `Unwrap()` を呼ぶ必要がある」としています。生成された `Update()` のコメントにも同じ注意があります。

なお、tx 由来**でない**エンティティに `Unwrap()` を呼ぶと panic します（生成コードが `*txDriver` かどうかを確認しているため）。

### 対処 2: ID だけ持ち出して取り直す

題材アプリでは、tx の中では ID だけを持ち出し、完了後にトランザクションの外で取り直す方法も使っています。取り直すときに、返したい関連をまとめて eager load できるのが利点です。

```go
var id uint64
err := RunInTransaction(ctx,
	func(ctx context.Context) error {
		c, err := ...Save(ctx)
		if err != nil {
			return err
		}
		id = c.ID
		return nil
	},
)
if err != nil {
	return nil, err
}
return repo.GetCollectionByID(ctx, id)
```

クエリが少し増えますが、管理画面のような低頻度の操作なら気にならない程度です。

---

## 9.5 行ロック — ForUpdate

「読んだ値をもとに更新する」処理では、読んでから書くまでの間に他の処理が割り込むと壊れます。ポイント残高から引く処理がまさにそれです。

`SELECT ... FOR UPDATE` で、**読んだ行に排他ロック**をかけて防ぎます。`entc.go` の `sql/lock` フィーチャーで Query に `ForUpdate()` が生えます。

題材アプリでは、ロックの有無を **ent に依存しない値**で受け取る作りにしています。

```go
type LockType int

const (
	LockTypeNone LockType = iota
	LockTypeShared    // FOR SHARE
	LockTypeExclusive // FOR UPDATE
)

func Exclusive() *Option { ... }
```

Repository はこれを見て `ForUpdate()` を付けます。

```go
q := client.Wallet.Query()
if opt != nil &&
	opt.LockType == lock.LockTypeExclusive {
	q = q.ForUpdate()
}
w, err := q.Where(wallet.UserID(uid)).Only(ctx)
```

Usecase 側は `lock.Exclusive()` を渡すだけです（9.3 の例）。

### ロックはトランザクションの中で

行ロックは**トランザクションが終わるまで**保持されます。トランザクションの外で `FOR UPDATE` しても、その SELECT が終わった瞬間に解放されるので意味がありません。

なので `lock.Exclusive()` を渡す呼び出しは、必ず `RunInTransaction` の中に置きます。そして Repository が `Client(ctx, r.client)` を使っているからこそ、ロックが**同じトランザクションで**かかります。9.3 の仕組みはここでも効いています。

### 独自の型で包むときの注意

ロック種別を独自の型で受け取る設計では、**型に定義した値と、実際に ent を呼ぶ分岐がずれる**ことがあります。上の例でも `LockTypeShared` を定義しているのに、Repository は `Exclusive` しか見ていません。`Shared()` を渡しても `ForShare()` は呼ばれず、**黙ってロックなしで実行**されます。

抽象化した値を ent の呼び出しに翻訳する箇所では、すべての値を扱っているかを確認してください。

---

## 9.6 生 SQL を投げる

ent の API でも `Modify`（第 7 章）でも書けないクエリがあります。たとえばウィンドウ関数です。

`entc.go` の `sql/execquery` フィーチャーで、Client に `QueryContext` / `ExecContext` が生えます。

```go
rows, err := client.QueryContext(
	ctx, query, args...,
)
if err != nil {
	return nil, wrapEntError(err)
}
defer rows.Close()

var ids []uint64
for rows.Next() {
	var id uint64
	if err := rows.Scan(&id); err != nil {
		return nil, wrapEntError(err)
	}
	ids = append(ids, id)
}
```

（「ユーザーごとに直近に見た記事」を `ROW_NUMBER() OVER (PARTITION BY ...)` で求める処理を簡略化）

戻り値は Go 標準の `*sql.Rows` です。`rows.Next()` で 1 行ずつ進めて `Scan` で変数に読み込みます。Node の `mysql2` で `query()` の結果を自分でマッピングするのに近い、低レベルな API です。

ここでは**ID だけを取り**、続けて ent で取り直しています。

```go
client.ArticleView.Query().
	Where(articleview.IDIn(ids...)).
	WithArticle().
	All(ctx)
```

「複雑な絞り込みだけ生 SQL、エンティティの組み立ては ent」という分担です。エンティティへのマッピングや eager loading を自前で書かずに済みます。

### 生 SQL の代償

公式ドキュメント（`doc/md/features.md`）は次のように警告しています。

> `ExecContext` / `QueryContext` で実行した文は ent を通らないので、Hook・privacy・バリデーターといったアプリの基本的な層を飛ばすことがある

具体的に効くのは**論理削除**です。論理削除は Interceptor（第 11 章）で `deleted_at IS NULL` を足しているので、**生 SQL には付きません**。必要なら自分で書いてください。

---

## 9.7 Client はどこで作られるか

最後に、ここまで使ってきた `client` の出どころを見ておきます。

```go
db, err := sql.Open("mysql", dsn)
// コネクションプール設定
db.SetMaxOpenConns(30)
db.SetMaxIdleConns(30)
db.SetConnMaxLifetime(5 * time.Minute)

drv := newTracingDriver(
	entsql.OpenDB(dialect.MySQL, db),
)
client := ent.NewClient(ent.Driver(drv))
```

3 段になっています。

1. `sql.Open` … Go 標準の DB 接続プール（`*sql.DB`）を作る
2. `entsql.OpenDB` … それを ent のドライバーで包む
3. `ent.NewClient` … ドライバーから Client を作る

ent 公式の入門では `ent.Open("mysql", dsn)` の 1 行で済ませますが、題材アプリは**プール設定とドライバーの差し替え**をしたいので分解しています。

DSN の `parseTime=True&loc=Asia%2FTokyo` は、DATETIME の読み書きを日本時間で行うための設定です。`parseTime` で `time.Time` に変換し、`loc` で読み込み時の解釈と書き込み時の変換の両方を Asia/Tokyo にします。

Client を作る関数は Client と一緒に**後始末用の関数**を返し、各エントリーポイント（API サーバー、ワーカー、バッチ）で `defer closeFunc()` しています。

### ドライバーをラップして計測する

`newTracingDriver` は、ent のドライバーに **APM（Sentry など）のトレース span を差し込むラッパー**です。

```go
type tracingDriver struct {
	*entsql.Driver
}

func (d *tracingDriver) Query(
	ctx context.Context,
	query string, args, v any,
) error {
	span := startDBSpan(ctx, "db.sql.query", query)
	defer span.Finish()
	return d.Driver.Query(
		span.Context(), query, args, v,
	)
}
```

（エラー処理を省略）

`*entsql.Driver` を**埋め込み**（第 2 章）、`Exec` / `Query` / `ExecContext` / `QueryContext` / `Tx` / `BeginTx` だけを上書きします。トランザクションも同じように包めば、tx 内のクエリにも span が付きます。

ent の Client は**ドライバーというインターフェースにしか依存していない**ので、こういう差し込みが外から自由にできます。Express のミドルウェアを DB 層に挟むようなイメージです。

---

## まとめ

- `client.Tx(ctx)` で tx を作り、`tx.Xxx` で操作、`defer Rollback` + 最後に `Commit` が基本形
- アプリ本体では **tx 入りの Client を context に入れて運ぶ**。`AsyncLocalStorage` と同じ発想
- Usecase は `RunInTransaction(ctx, func(tCtx) error)`。**error を返せば Rollback、nil なら Commit**
- Repository は必ず **`Client(ctx, r.client)`** で Client を取り出す。`r.client` 直書きは tx を素通りする
- 新しい context は**引数の ctx から派生**させる。`Background()` から作ると tx も相関 ID も消える
- tx の中で作ったエンティティは **tx を握ったまま**。外に返すならコミット後に `Unwrap()` するか、ID だけ持ち出して取り直す
- 行ロックは `ForUpdate()`。**トランザクションの中で**、同じ tx の Client から使う
- 生 SQL は `client.QueryContext`。**Hook・論理削除を通らない**ので、絞り込みだけに使って ent で取り直す
- Client は `sql.DB` → `entsql` ドライバー → `ent.NewClient` の 3 段。ドライバーをラップして計測を差し込める

---

[← 前: ミューテーション — データを書く](08-mutation.md) | [目次](../README.md) | [次: エラーの扱い →](10-errors.md)
