# 10. エラーの扱い

> **この章でわかること**
> ent が返すエラーの種類と、それを判定する `IsXxx` 関数。Go の `errors.As` によるエラーチェーンの探索。ent のエラーを「DB のドメインエラー」に包み替えて、上の層に ent を漏らさない仕組み。「見つからない」をどう扱うかの規約。

---

## 10.1 Go のエラーをおさらい

第 2 章で見たとおり、Go には例外がありません。関数は**最後の戻り値で error を返し**、呼び出し側が毎回 `if err != nil` で確認します。

TS との大きな違いがもう 1 つあります。**エラーは「包んで」上に返す**のが普通です。

```go
return fmt.Errorf("commit failed: %w", err)
```

`%w` で包むと、元のエラーが**チェーンとして中に残ります**。JS の `new Error("...", { cause: err })` に当たります。

チェーンの中に特定の型のエラーがあるかを探すのが `errors.As` です。

```go
var nf *ent.NotFoundError
if errors.As(err, &nf) {
	// チェーンのどこかに NotFoundError がある
}
```

TS の `err instanceof NotFoundError` に近いですが、**包まれた内側まで探しに行く**点が違います。

---

## 10.2 ent のエラー型

ent は次のエラー型を生成します（`ent/ent.go`）。

| 型 | いつ起きるか |
| --- | --- |
| `NotFoundError` | `Get` / `Only` / `First` / `UpdateOneID` などで対象が無い |
| `NotSingularError` | `Only` で 2 件以上あった |
| `NotLoadedError` | 先読みしていないエッジを `Edges.XxxOrErr()` で読んだ |
| `ConstraintError` | ユニーク制約・外部キー制約の違反 |
| `ValidationError` | `Validate` / `MaxLen` などの違反、または Create で必須フィールドを未設定 |

それぞれに判定関数があります。中身はどれも `errors.As` です。

```go
func IsNotFound(err error) bool {
	if err == nil {
		return false
	}
	var e *NotFoundError
	return errors.As(err, &e)
}
```

なので**何重に包まれていても**判定できます。

### ValidationError は SQL の前に起きる

第 4 章の `MaxLen(100)` や `Validate(func...)` は、**DB に送る前に** ent が Go の中でチェックします。違反すると SQL は発行されず `ValidationError` が返ります。

一方 `ConstraintError` は、**DB が拒否した**結果です。ユニーク制約は DB にしか判定できないので、こちらは SQL を実行してから分かります。

### 制約違反の中身を見分ける

`ConstraintError` だけでは「ユニーク違反か外部キー違反か」が分かりません。そこで `sqlgraph` パッケージの関数を併用します。

```go
ent.IsConstraintError(err) &&
	sqlgraph.IsUniqueConstraintError(err)

ent.IsConstraintError(err) &&
	sqlgraph.IsForeignKeyConstraintError(err)
```

### MaskNotFound

```go
func MaskNotFound(err error) error {
	if IsNotFound(err) {
		return nil
	}
	return err
}
```

「見つからなければ nil で構わない」ときに使うヘルパーです。第 7 章の GraphQL リゾルバーの例で使われていました。

### ent の型ではないエラーも来る

ent を通して返ってくるのが、ent のエラー型だけとは限りません。ドライバーや Go 標準の `database/sql` のエラーがそのまま出てくることもあります。

代表例が第 9 章 9.4 で見た `sql: transaction has already been committed or rolled back` です。これは `database/sql` の `sql.ErrTxDone` で、`ent.IsXxx` のどれにも当たりません。

このエラーをログで見かけたら、「**tx の中で作ったエンティティを、コミット後に使っていないか**」を真っ先に疑ってください。

---

## 10.3 なぜ ent のエラーをそのまま返さないのか

ここからは題材アプリの設計の話です。

題材アプリでは、**Usecase 層は ent に依存してはいけない**ルールがあります。import だけでなく、ent 由来の型を扱うことも禁止です（第 14 章）。自作の lint が機械的に検査しています。

すると困ることがあります。Usecase が「見つからなかったら 404 を返したい」とき、`ent.IsNotFound(err)` を書けません。

TS でたとえると、サービス層で `err instanceof Prisma.PrismaClientKnownRequestError && err.code === 'P2025'` と書くようなものです。ORM を差し替えたら、サービス層の全部の分岐を書き直すことになります。

そこで、**Repository の出口で ent のエラーを「DB のドメインエラー」に包み替える**ことにしています。

---

## 10.4 database.Error — DB のドメインエラー

包み替え先の型は、ドメイン層の `database` パッケージ（Repository のインターフェースと同じ場所）に置きます。

```go
type ErrorKind int

const (
	ErrorKindOther ErrorKind = iota
	ErrorKindNotFound
	ErrorKindUniqueViolation
	ErrorKindForeignKeyViolation
)

type Error struct {
	Kind  ErrorKind
	cause error
}
```

- `Kind` … 上の層が分岐に使う**分類**
- `cause` … 元の ent のエラー。ログ・調査用に残す

`Unwrap()` で `cause` を返すので、チェーンは切れません。`Error()` も元のメッセージをそのまま返すので、ログやエラー監視サービスに送る情報量は変わりません。

判定用の関数も同じパッケージにあります。

```go
database.IsNotFoundError(err)
database.IsUniqueViolationError(err)
database.IsForeignKeyViolationError(err)
```

Usecase はこれだけを使います。ent の名前は出てきません。

`ErrorKindOther` を**ゼロ値**（`iota` の 0）にしているのは、分類し忘れたエラーが自動的に「その他」に落ちるようにするためです。

---

## 10.5 包み替えの実体 — Wrap

ent のエラーを `database.Error` に変換するのは、Repository 層の共通関数 `Wrap` です。

```go
func Wrap(err error) error {
	if err == nil {
		return nil
	}
	if _, ok := err.(*database.Error); ok {
		return err
	}
	var dErr *database.Error
	if errors.As(err, &dErr) {
		return database.NewError(dErr.Kind, err)
	}
	switch {
	case ent.IsNotFound(err):
		return database.NewError(
			database.ErrorKindNotFound, err)
	case ent.IsConstraintError(err) &&
		sqlgraph.IsUniqueConstraintError(err):
		return database.NewError(
			database.ErrorKindUniqueViolation, err)
	case ent.IsConstraintError(err) &&
		sqlgraph.IsForeignKeyConstraintError(err):
		return database.NewError(
			database.ErrorKindForeignKeyViolation, err)
	default:
		return database.NewError(
			database.ErrorKindOther, err)
	}
}
```

上から順に読むと、こうなります。

1. nil なら nil のまま
2. すでに `*database.Error` ならそのまま（**二重に包まない**）
3. チェーンの内側に `*database.Error` があれば、その分類を引き継いで包み直す
4. ent のエラーなら分類して包む
5. それ以外は「その他」

2 のおかげで、何度通しても**二重に包まれません**（冪等）。3 のおかげで、`fmt.Errorf` で包まれていても分類が引き継がれます。Repository が別の Repository を呼んでいても安全です。

10.2 の `sql.ErrTxDone` のような ent 以外のエラーは、5 の「その他」に落ちます。

### wrapEntError

Repository からは、パッケージごとの `wrapEntError` を通して呼びます。

```go
func wrapEntError(err error) error {
	return enterr.Wrap(err)
}
```

ただの委譲ですが、「返すエラーは同じパッケージのラッパーを通すこと」を自作 lint が検査するため、この形になっています。

Repository のメソッドは、ほぼすべてこう終わります。

```go
v, err := client.Xxx.Query()....Only(ctx)
if err != nil {
	return nil, wrapEntError(err)
}
```

`return wrapEntError(client.Xxx....Exec(ctx))` と 1 行で書くこともできます。`wrapEntError(nil)` は nil を返すからです。

### passThrough

第 9 章の `RunInTransaction` には `passThrough(err)` が出てきました。

```go
func passThrough(err error) error {
	return err
}
```

何もしない関数です。`RunInTransaction` のコールバックが返したエラーは**Usecase が作ったエラー**なので、DB エラーに包み替えてはいけません。「意図して素通ししている」ことを lint とレビュアーに示すためのマーカーです。使ってよいのは `RunInTransaction` の中だけ、と決めています。

---

## 10.6 「見つからない」の扱い

チームの規約は、NotFound について明確なルールを持っています。

### Repository: nil, nil を返さない

> NotFound（対象の不在）もエラーで返す。`nil, nil` 返却は禁止

Go では「エンティティ nil、error nil」も返せてしまいます。でもそうすると、呼び出し側は**エラーチェックを通過したのに nil を掴んで**落ちます。

ent が `Only` で 0 件をエラーにしているのと同じ理由です（第 7 章）。この「NotFound 潰し」も自作 lint で機械的に検出しています。

### Usecase: 不在が正常なら nil にしてよい

逆に、Usecase 層では不在を握りつぶしてよい場合があります。

> 不在を正常系とする Usecase は `database.IsNotFoundError(err)` で判定して握りつぶし、nil を返してよい

たとえば「有効なサブスクリプションが無い」は異常ではありません。

```go
sub, err := u.repo.GetActiveSubscription(
	ctx, userID,
)
if database.IsNotFoundError(err) {
	return nil, nil
}
```

**「不在かどうか」を判断する責任は、ビジネスの意味を知っている上の層にある**、という分担です。Repository は事実（無かった）を伝えるだけで、解釈しません。

### 削除は冪等に

削除系の Usecase は、対象が無くても成功を返すルールです。

```go
t, err := u.repo.GetTagByID(ctx, id)
if database.IsNotFoundError(err) {
	return nil
}
```

「もう消えている」なら、消したいという目的は達成済みだからです。

---

## 10.7 制約違反を業務の判断に使う

DB の制約違反を、**競合の検出**に使うこともできます。

```go
_, err = u.repo.CreateMembership(
	ctx, orgID, userID,
)
if database.IsUniqueViolationError(err) {
	// 並行リクエストが先に参加を完了させた。
	// 参加済みとして冪等に成功させる
	alreadyJoined = true
	return nil
}
```

事前に「まだ参加していないか」を確認していても、確認と作成の間に別のリクエストが割り込むことがあります。最後の砦はユニーク制約で、その違反を「先を越された」と解釈しています。

外部キー違反を「指定された記事が存在しない」と解釈して 404 にする、という使い方もあります。

どちらも、Usecase は ent も SQL も知らずに `database.IsXxxError` だけで書けています。10.3 の狙いどおりです。

---

## 10.8 ent のエラー判定が残る場所

すべての場所で `database.Error` を使うわけではありません。ent を直接扱う層では ent の判定関数が残ります。

- **Repository の内部** … ent のエラー型を見て処理を変えることがある（例: NotFound を正常系として扱う）
- **認証ミドルウェア** … 初回ログインのユーザーを `ent.IsNotFound` で判定（第 7 章の例）
- **ent の構造体を直接受け取る GraphQL リゾルバー** … `Edges.XxxOrErr()` と `ent.IsNotLoaded` / `ent.MaskNotFound`（第 7 章）

チームの規約は「`ent.IsNotFound` のようなベンダー型の判定を Resolver / Usecase に書かない」です。ent の構造体を直接受け取るリゾルバーのエッジ読み出しは例外的に残りやすいので、新しく書くときは規約に寄せてください。

---

## まとめ

- Go のエラーは `%w` で包んでチェーンにする。**`errors.As` で内側まで探せる**
- ent のエラーは `NotFound` / `NotSingular` / `NotLoaded` / `Constraint` / `Validation`。判定は `ent.IsXxx`
- **Validation は SQL の前、Constraint は DB が拒否した後**。制約の種類は `sqlgraph.IsXxxConstraintError` で見分ける
- ent 以外のエラーも来る。`sql.ErrTxDone` を見たら **tx の外でのエンティティ利用**を疑う
- Repository の出口で **`wrapEntError` → `Wrap`** を通し、`database.Error{Kind}` に包み替える
- 包み替えは冪等。元のエラーは `cause` としてチェーンに残る
- Usecase は **`database.IsNotFoundError` などだけ**で分岐する。ent を知らない
- Repository は **NotFound を nil, nil にしない**。不在を正常とみなすかは Usecase が決める

---

[← 前: トランザクション・ロック・生 SQL](09-transactions-and-sql.md) | [目次](../README.md) | [次: Hook と Interceptor →](11-hooks-interceptors.md)
