# 16. ここから先へ

> **この章でわかること**
> ent を書くときの落とし穴チェックリスト。よく使うコマンド。新しいテーブルを追加するときの一連の流れ。公式ドキュメントの歩き方と、題材アプリでは使っていないが知っておくとよい機能。

最後の章です。ここまでで ent の考え方と、題材アプリでの使われ方は一通り見てきました。あとは実際に手を動かすときの装備を整えます。

---

## 16.1 落とし穴チェックリスト

本書に出てきたものを中心に、事故になりやすい点をまとめます。PR を出す前に、ここを引いてください。

### スキーマ

- [ ] `Optional` と `Nillable` を取り違えていないか。**NULL を許すのは `Optional`、Go でポインタになるのは `Nillable`**。「未設定」と「ゼロ値」を区別したいなら両方要る（第 4 章 4.4）
- [ ] **`Optional` だけの列が GraphQL に出ていないか**。DB が NULL でも Go ではゼロ値（`0` や `""`）になるので、GraphQL の型が null 可でも **null ではなく 0 が返る**（第 4 章 4.4）
- [ ] 既存の列の型を変えるとき（たとえば 0/1 の整数を `bool` に）、**変換できない値が DB に残っていないか**を先に確認したか。1 行でもあると、その行を含む**一覧の取得全体がエラーになる**（第 4 章、第 12 章）
- [ ] `ent/` 直下の**生成コードを手で直していないか**。直すのはスキーマ。生成物は次の `make gen` で消える（第 3 章）
- [ ] entgql が生成する GraphQL スキーマファイルを**手で直していないか**（第 13 章）
- [ ] スキーマを変えたあと **`make gen` を実行したか**（第 3 章）

### GraphQL に出るもの

- [ ] GraphQL に出したくないフィールド・エッジに `entgql.Skip()` を付けたか。付けないと**自動で公開される**（第 13 章 13.10）
- [ ] **Query のルートが無いから非公開、と思い込んでいないか**。型が GraphQL に出ていれば、**別の型のエッジからたどって取得できる**。意図しない公開面になっていないか、エッジの側も確認する（第 13 章 13.10）

### クエリ

- [ ] 論理削除のあるエンティティを **`HasXxxWith` で絞り込むとき、`DeletedAtIsNil()` を明示したか**。サブクエリには Interceptor が効かない一方、eager load には効く。この非対称で、**中間テーブルの行だけ残ってエッジが nil になる**（第 11 章 11.7）
- [ ] eager load していないエッジを `Edges.Xxx` で読んでいないか。**ロードしていなければ空**。`XxxOrErr()` と `ent.IsNotLoaded` で見分ける（第 7 章 7.7）
- [ ] `Modify` で独自の SELECT を足したクエリで `Count(ctx)` を呼んでいないか。**壊れた SQL になる**（第 7 章 7.9）
- [ ] 並び順のキーが重複しうるとき、**第 2 キー（ID など）を付けたか**。付けないと順序が DB 任せになる（第 7 章 7.5、第 15 章）
- [ ] 一覧の取得で**全件取ってメモリで切り出していないか**。ページネーションは DB 側で `Limit` / `Offset`（第 7 章 7.5）

### 論理削除

- [ ] **論理削除されたものを、null を許さないリレーションで参照していないか**。除外が効いたまま解決すると、**1 件でも該当があれば一覧全体がエラーになる**。退会ユーザーを参照する一覧などが典型。必要なら `SkipSoftDelete` を付ける（第 11 章 11.8）
- [ ] 「参照されているか」の確認に、**論理削除済みの行を含めるべきか**考えたか（第 11 章 11.8）
- [ ] `SkipSoftDelete` を **`context.Background()` から作っていないか**。必ず引数の ctx から作る。トランザクション・相関 ID・キャンセルが ctx に載っているから（第 11 章 11.8）
- [ ] 論理削除したとき、**それを参照している側をどう見せるか**を決めたか。決めていないと、上の 2 つの事故が起きる（第 11 章）

### 書き込みとトランザクション

- [ ] Repository で **`r.client` を直接使っていないか**。`Client(ctx, r.client)` を通さないと**トランザクションが効かない**（第 9 章 9.3、第 14 章）
- [ ] upsert で、**既存行を上書きしたい列に `UpdateXxx()` を付けたか**。`SetXxx` は新規挿入時の値にしかならない。忘れると insert-only になり、**既存行が静かに古い値のまま残る**（第 8 章 8.8）
- [ ] **トランザクションの中で作ったエンティティを、コミット後にそのまま返していないか**。そのエンティティはトランザクション用の client を抱えているので、あとでエッジをたどると `transaction has already been committed` で落ちる。**コミット後に `Unwrap()` するか、tx 内では ID だけ持ち出して完了後に取り直す**（第 9 章 9.4）
- [ ] `RunInTransaction` を**入れ子にしていないか**。素の client から毎回トランザクションを開く実装だと、内側は外側と**別のトランザクション**になる（第 9 章 9.3）
- [ ] バルクの `Update().Save(ctx)` の戻り値を**エンティティだと思っていないか**。返るのは**更新件数（`int`）**（第 8 章 8.2）
- [ ] ロックのラッパーを書いたとき、**用意したロックの種類がすべて ent のメソッドにつながっているか**。たとえば共有ロックの指定を受け取るのに `ForShare()` を呼んでいない、といった抜けが起きやすい（第 9 章 9.5）
- [ ] ロックを取るクエリが `RunInTransaction` の中で実行されているか（第 9 章 9.5）

### エラー

- [ ] Repository から **ent のエラーをそのまま返していないか**。必ず `wrapEntError` を通す（第 10 章）
- [ ] Usecase や Resolver で **`ent.IsNotFound` を書いていないか**。`database.IsNotFoundError` を使う（第 10 章）
- [ ] 不在を **`nil, nil` で返していないか**。NotFound もエラーで返す（第 10 章）
- [ ] `IsNotFound` のチェックが**本当に効く場所か**。`Ints` や `All` のような複数件の終端メソッドは 0 件でも NotFound を返さない（第 7 章 7.4）

### マイグレーション

- [ ] スキーマ変更と同じ PR に**マイグレーションファイルを含めたか**。`make gen` だけでは DB は変わらない（第 12 章）
- [ ] **外部キー制約のある列を変更**していないか。制約 DROP → 列変更 → 制約再作成の 3 段に書き直す。**ローカルでは失敗せず、デプロイ時に落ちることがある**（第 12 章 12.10）
- [ ] `atlas.sum` が衝突したとき、**手で直していないか**。`atlas migrate hash` で作り直す（第 12 章 12.7）
- [ ] 同じテーブルへのマイグレーションが PR 内で複数に割れていないか。ローカルで戻して削除し、1 つにまとめ直す（第 12 章）

### 層の境界

- [ ] Usecase で ent を使っていないか。`entity.Model()` も禁止（第 14 章）
- [ ] 新しい usecase パッケージの名前に **`usecase` を含めたか**。含めないと層の依存を検査する lint の対象から外れる（第 14 章）

---

## 16.2 よく使うコマンド

題材アプリでは、よく使うコマンドを Makefile にまとめています。中身は ent・Atlas の CLI なので、Makefile が無くても同じことができます。

### 生成

| コマンド | やること |
| --- | --- |
| `make gen` | ent・gqlgen・mock を生成（`go generate`） |

生成は重いので、入力や生成物のハッシュを比べて**変わったものだけ生成する**スクリプトを挟むと速くなります。その場合は「全部やり直す」コマンドも用意しておくと、検出が効かなかったときに困りません。

### スキーマ

| コマンド | やること |
| --- | --- |
| `go run -mod=mod entgo.io/ent/cmd/ent new Xxx` | スキーマの雛形を作る |
| `go run -mod=mod entgo.io/ent/cmd/ent describe ./ent/schema` | スキーマの一覧を表で出す |

`ent describe` は、フィールドの型・Optional・Unique やエッジの向きを一覧で見られます。大きなスキーマを読むときに便利です。

### マイグレーション

| コマンド | やること |
| --- | --- |
| `make migration name=xxx` | スキーマとの差分 SQL を生成（`ent/migrate/main.go`） |
| `atlas migrate new xxx` | 空の SQL を作る（データ移行用） |
| `atlas migrate apply` | DB に適用 |
| `atlas migrate status` | 適用状況を見る |
| `atlas migrate down` | 直前の適用を戻す |
| `atlas migrate rm xxx` | マイグレーションを削除 |
| `atlas migrate hash` | `atlas.sum` を作り直す |
| `atlas migrate lint` | マイグレーションを lint |
| `atlas migrate validate` | ディレクトリの整合性を検証 |

`--dir file://ent/migrate/migrations` と `--url`（DB の接続先）は、どのコマンドにも付けます。Makefile に包んでおくと楽です。

### 品質

| コマンド | やること |
| --- | --- |
| `make lint` | golangci-lint と自作 lint |
| `make test` | テスト |

---

## 16.3 新しいテーブルを追加する流れ

ここまでの章をつなげると、テーブルを 1 つ追加する作業はこうなります。例として、記事へのコメント `Comment` を足します。

### ① スキーマを書く（第 4〜6 章）

```bash
go run -mod=mod entgo.io/ent/cmd/ent \
  new Comment
```

`ent/schema/` に雛形ができます。題材アプリでは既存スキーマに合わせて、ID を明示します。

```go
// ent/schema/comment.go
func (Comment) Fields() []ent.Field {
  return []ent.Field{
    field.Uint64("id"),
    field.Text("body").
      Comment("本文"),
    field.Uint64("article_id"),
    // …
    field.Time("created_at").Optional().
      Default(time.Now),
    field.Time("updated_at").Optional().
      Default(time.Now).
      UpdateDefault(time.Now),
  }
}
```

ここで決めること:

- 各フィールドの `Optional` / `Nillable`
- エッジの向きと `OnDelete`
- インデックス・ユニーク制約
- 論理削除にするか。するなら**参照している側をどう見せるか**
- GraphQL に何を出すか（`entgql.Skip` / `QueryField` など）

### ② 生成する（第 3 章）

```bash
make gen
```

`ent/` と、entgql が生成する GraphQL スキーマが更新されます。差分を見て、**GraphQL に意図しない型やフィールドが出ていないか**確認します。エッジ経由でたどれるものも含めて見ます。

### ③ マイグレーションを作って適用する（第 12 章）

```bash
make migration name=create_comments
atlas migrate apply …
atlas migrate lint …
```

生成された SQL は必ず読みます。列の型・NULL 許容・外部キーが意図どおりかを見ます。

### ④ Entity を作る（第 14 章）

`domain/models/` に Entity を作ります。

- ent の構造体を非公開フィールドで持つ
- `NewXxxEntity(dto)`（DTO の `Validate()` 必須）
- `ReconstructXxxEntity(model)`
- nil チェック付きの Getter、検証を通す Setter

### ⑤ port と Repository を作る（第 14 章）

1. `domain/ports/` に Repository インターフェースを書く。各メソッドにコメント（NotFound 時の挙動も）を書く
2. port を大きな `DBRepository` インターフェースに埋め込む。埋め込まないと Usecase から `dbRepo` 経由で呼べず、モックにも入らない
3. `infrastructure/db/repository/` に実装を書く。`Client(ctx, r.client)` と `wrapEntError` を忘れない
4. 実装を `entRepository` に埋め込む
5. Repository のテストを書く（第 15 章）

### ⑥ Usecase と Resolver（第 13〜14 章）

1. Usecase を書き、gomock でテストする
2. GraphQL のスキーマに Query / Mutation を足し、`make gen`
3. Resolver で Usecase を呼び、`entity.Model()` を返す

ent が生成した型に、独自のフィールドを足したいときは `extend type` を使います（第 13 章）。

### ⑦ 仕上げ

```bash
make lint
make test
```

層の依存を検査する lint やエラー契約を検査する lint も、ここで走ります。

---

## 16.4 公式ドキュメントの歩き方

ent の公式ドキュメントは日本語版があります（ただし一部は英語のままです）。

- https://entgo.io/ja/docs/getting-started/

本書の章と、公式の対応ページです。URL は `https://entgo.io/ja/docs/` の後ろに下のパスを付けます。

| 本書 | 公式ページ |
| --- | --- |
| 第 3 章 | `code-gen`、`feature-flags` |
| 第 4 章 | `schema-fields` |
| 第 5 章 | `schema-edges` |
| 第 6 章 | `schema-indexes`、`schema-annotations`、`schema-mixin` |
| 第 7 章 | `crud`、`predicates`、`traversals`、`eager-load`、`paging`、`aggregate` |
| 第 8 章 | `crud` |
| 第 9 章 | `transactions`、`sql-integration` |
| 第 11 章 | `hooks`、`interceptors`、`privacy` |
| 第 12 章 | `versioned-migrations` |
| 第 13 章 | `graphql` |
| 第 15 章 | `testing` |

迷ったときは **`schema-edges`** と **`faq`** から読むのがおすすめです。エッジのページには O2O / O2M / M2M の全パターンが図付きで並んでいて、「こういう関係はどう書くんだっけ」の答えがだいたい見つかります。

### ソースを読む

ドキュメントで分からないことは、生成コードを読むのが一番確実です。ent の生成コードは素直なので、TS の型定義を読む感覚で読めます。

- 「このメソッドは何を返す？」→ `ent/xxx_update.go` などで定義を探す
- 「この述語はどんな SQL になる？」→ `ent/xxx/where.go`
- 「entgql は何をしている？」→ `ent/gql_*.go`

本書の記述も、多くはこうして生成コードで確かめています。

---

## 16.5 題材アプリでは使っていないが知っておくとよい機能

ent には、題材アプリでまだ使っていない機能もあります。名前だけでも知っておくと、「それ ent でできるよ」と言えるようになります。

### privacy

`entc.go` で有効になっていて `ent/privacy/` も生成されていますが、どのスキーマにも Policy がなく、**実際には使われていません**（第 11 章）。

スキーマに読み書きの許可ルールを書くと、**どこからクエリしても必ず評価される**仕組みです（公式 `privacy`）。題材アプリの認可は GraphQL ディレクティブと Resolver で行っているので（第 11 章）、出番がありません。

### Views

`ent.Schema` の代わりに `ent.View` を使うと、DB の VIEW をエンティティとして扱えます（公式 `schema-views`）。読み取り専用の集計テーブルなどに向きます。

### 双方向エッジ

同じ型同士の M2M で、`edge.To` だけを書き `edge.From` を書かないと、**向きのない関係**になります（公式 `schema-edges`「M2M Bidirectional」）。「A が B の友達なら B も A の友達」のような対称な関係です。

題材アプリのタグ階層（`Tag` の親子）は親子という向きがあるので、`To` と `From` の両方を書いています（第 5 章）。

### field.Immutable / Sensitive

- `Immutable()`: 作成時にしか設定できない。Update Builder に Setter が生成されない
- `Sensitive()`: 文字列フィールドを、`String()` やエンコード（JSON）に出さない。パスワードなどに

どちらも公式 `schema-fields` にあります。間違った更新や、ログへの機密情報の漏れを**型と生成コードで防げる**のが利点です。

### Aggregate

`Aggregate(ent.Count(), ent.Sum(…))` で集計を ent の API のまま書けます（公式 `aggregate`）。題材アプリでは集計を `GroupBy` か `Modify` で書いています（第 7 章）。

### 外部テンプレート

entc に自作の Go テンプレートを渡すと、生成コードにメソッドやファイルを追加できます（公式 `templates`）。題材アプリは entgql 以外の拡張を使っていません。

「全エンティティに同じ形のメソッドが欲しい」と思ったら、テーブルの数だけ手で書く前に思い出してください。

---

## 16.6 最後に

ent の考え方は、突き詰めると 2 つでした。

- **スキーマを Go のコードで書き、残りは生成する**
- **データをノード（エンティティ）とエッジ（関係）のグラフとして扱う**

最初は生成コードの量に圧倒されますが、どれもスキーマから機械的に作られたものです。`ent/` の巨大なファイル群を読み解く必要はありません。**スキーマを読めば、生成されるものは想像できる**ようになっているはずです。

あとは実際のコードで迷ったら、該当する章と、手元のプロジェクトの似たスキーマ・Repository を探してください。

お疲れさまでした。

---

[← 前: テストの書き方](15-testing.md) | [目次](../README.md)
