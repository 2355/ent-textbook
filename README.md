# TypeScript 開発者のための ent 入門

Node.js / TypeScript でバックエンドを書いてきた人が、**Go の ORM「ent」の「考え方」を掴む**ための教科書です。

## この本の前提

- 読者は ent も Go もほぼ未経験
- ただし TypeScript / Node.js の経験はあり、Prisma や TypeORM のような ORM を触ったことがある
- 「動くコードを写経する」ことより、**なぜそういう設計なのかを理解する**ことを重視

なので、コードは最小限です。スマホでスキマ時間に読めるよう、1 節はだいたい 3〜10 分で読み切れる長さにしてあります。

題材には、**架空の記事投稿サービスの GraphQL API**（以下「題材アプリ」）を使います。実務でよく使う ent の機能は、ひととおりどこかの章で扱っています。実務で踏みやすい落とし穴も「現場メモ」として載せています。

Go に慣れていない人のために、第 2 章で ent を読むのに必要な Go の基礎をまとめています。Go をすでに知っている人は飛ばして構いません。

## 読み方

上から順に読むことを想定しています。前の章を参照する箇所もありますが、気になる章から読んでも大筋はつかめます。

全部通して **5〜6 時間**くらいのボリュームです。1 日で読み切る必要はありません。「1 日 1 章」くらいのペースがちょうどいいはずです。第 3 部と第 4 部は長めの章が多いので、節の区切りで止めても構いません。

各章の冒頭に「この章でわかること」、末尾に「まとめ」があります。時間がないときはそこだけ拾ってください。

## 目次

### 第 1 部 — 世界観を掴む

1. [なぜ ent はこういう形なのか](docs/01-why-ent.md)
   ORM の系譜と ent の立ち位置。「スキーマをコードで書き、残りを生成する」という発想。Prisma との比較。
2. [TypeScript 使いのための Go](docs/02-go-for-ts.md)
   ent を読むのに必要な差分だけ。struct、ポインタと nil、エラー値、`context.Context`、interface。
3. [ent の全体像とコード生成](docs/03-ent-overview.md)
   スキーマ → entc → 生成コード。Client と Builder、題材アプリの構成と `make gen`。既存 DB からの取り込み。

### 第 2 部 — スキーマを書く

4. [フィールド — 列を定義する](docs/04-fields.md)
   型、`Optional` と `Nillable` の違い、デフォルト値、バリデーション、DB 型の上書き。
5. [エッジ — 関係を定義する](docs/05-edges.md)
   `To` / `From` / `Ref`、O2O・O2M・M2M、外部キーの明示、`Through`、自己参照、削除時の挙動。
6. [インデックス・アノテーション・Mixin](docs/06-indexes-annotations-mixin.md)
   複合ユニーク、テーブル設定、スキーマ部品の再利用。

### 第 3 部 — データを読み書きする

7. [クエリ — データを読む](docs/07-query.md)
   述語、取得メソッドの使い分け、並び替えとページング、eager loading とエッジ走査、集計。
8. [ミューテーション — データを書く](docs/08-mutation.md)
   Create / Update / Delete の Builder、一括作成、upsert、「Mutation」という中間表現。
9. [トランザクション・ロック・生 SQL](docs/09-transactions-and-sql.md)
   `Tx` と context 経由の受け渡し、`FOR UPDATE`、`Modify` と生 SQL という逃げ道。
10. [エラーの扱い](docs/10-errors.md)
    ent のエラー型と、それを Repository の外に漏らさない仕組み。

### 第 4 部 — ent を拡張する

11. [Hook と Interceptor](docs/11-hooks-interceptors.md)
    書き込みと読み込みに割り込むミドルウェア。論理削除 Mixin を分解して読む。privacy。
12. [マイグレーション](docs/12-migrations.md)
    自動マイグレーションとバージョン管理マイグレーション。Atlas を使った運用フロー。
13. [entgql — GraphQL との連携](docs/13-entgql.md)
    スキーマ生成、Relay ページネーション、並び替え・絞り込み入力、ディレクティブ、N+1 を避ける仕組み。

### 第 5 部 — 実務で使う

14. [アーキテクチャの中の ent](docs/14-architecture.md)
    ent を Repository 層に閉じ込める理由と方法。Entity・Usecase・Resolver との境界。
15. [テストの書き方](docs/15-testing.md)
    テスト用 DB、enttest、モックとの使い分け。
16. [ここから先へ](docs/16-next-steps.md)
    落とし穴チェックリスト、便利コマンド、公式ドキュメントの歩き方。

## 補足

コード例は説明のための最小限のものです。そのままコピーして動くことを保証していません。題材アプリのコードは説明のために省略・簡略化しています。

内容は ent v0.14.1 / entgql（entgo.io/contrib v0.6.0）時点の情報に基づいています。ent は活発に開発されているので、最新の挙動は公式ドキュメントを確認してください。

- [ent 公式ドキュメント（日本語）](https://entgo.io/ja/docs/getting-started/)
- [ent GitHub リポジトリ](https://github.com/ent/ent)
- [entgql（GraphQL 連携）ドキュメント](https://entgo.io/ja/docs/graphql)
- [Atlas ドキュメント](https://atlasgo.io/)
