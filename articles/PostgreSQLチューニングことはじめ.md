サポート終了が2024年で、まだまだ使用されていそうなPostgreSQL 12を対象にしました。
(参考)https://oss-db.jp/outline/silver#questionnaire_range_silver

インターンで初PosgreSQLチューニングをすることになったので、
チューニングに使えそうなの列挙していきます。

## 実行計画
(参考)https://www.postgresql.jp/document/12/html/sql-explain.html

SQLの基本的なチューニングに使う。
実行計画で時間がかかっていそうなところに注力したい。

### 取得方法

```sql
-- insert,update,deleteの場合は前後のbegin;rollback;を忘れずに実行する
-- BEGIN;
EXPLAIN ANALYZE /* SQL文 */;
-- ROLLBACK;
```

### 見方

まず、階層を見る。
上の階層→下の階層で、rowsに対しループして実行しているイメージ。
各行の処理の内容と、実際のSQLを照合しておく。

- cost: 処理の重さ
- rows: 選択されている件数

costが低くてもrowsが多いと繰り返し件数がかさむ。
rowsが低くてもcostが高いと1回あたりの処理に時間がかかる可能性が高い。
それぞれ影響しあって累乗的に増えていく。

パフォーマンスチューニングは無限にできる。集中と選択が大事。
難易度や要件にもよるが、基本は「重いところから」

各処理の意味は都度ググる感じで。
結合処理のNested Loop,Hash,Mergeの特性がわかっていると触りやすいかも。

## ANALYZE,VACUUM
(参考)https://www.postgresql.jp/document/12/html/sql-analyze.html
(参考)https://www.postgresql.jp/document/12/html/sql-vacuum.html

ANALYZEは、適切な実行計画のために必要な、実データの格納状況について統計を取得する。
VACUUMは、deleteやupdateで不要になった行を、物理的に削除する。

通常実行で満足できなければ、各オプションで強めの実行(語彙力)にできる。
例えばVACUUMのFULLオプションなら、テーブルを完全にコピーし直すので、新品で整列されたテーブルになる。(ただし、排他ロックされ、長時間かかる。)

### 実行方法
```sql
-- VACUUM,ANALYZEはそれぞれ単品実行できる
VACUUM ANALYZE /* table_name */ /* column_name */;
```

## システムカタログ
(参考)https://www.postgresql.jp/document/12/html/catalogs-overview.html
(参考)https://oss-db.jp/outline/gold/v2#questionnaire_range_gold

公式ドキュメントより
> システムカタログとは、リレーショナルデータベース管理システムがテーブルや列の情報などのスキーマメタデータと内部的な情報を格納する場所です。

以下、チューニングに使えそうなもの書いていく。

### pg_statistics
(参考)https://www.postgresql.jp/document/12/html/catalog-pg-statistic.html
(参考)https://www.postgresql.jp/document/12/html/view-pg-stats.html
(参考)https://www.slideshare.net/nttdata-tech/postgresql-monitoring-features-ntt-data

ANALYZEを実行した結果が格納される。

pg_statisticsは元表、pg_statsは一般ユーザーが使うビュー。
基本的にpg_statsで問題なさそう。

statistics(統計情報)の名の通り、一般に言う統計をテーブルに対して取得しているイメージ。
列の値の分布を記録するヒストグラムなど。

勝手にオプティマイザが使っているのでそこまで見る機会はないと思う。
全力でチューニングしたい時は、中のデータの分布を知っておかないと適切な索引や実行計画が選べない。

### pg_locks
(参考)https://www.postgresql.jp/docs/12/view-pg-locks.html

現在発生中のロック状況が見れる。
表レベル、行レベルなどロックの種類を知っておくと幸せそう。
大きすぎるロックは事故の原因。

### 統計情報コレクタ
(参考)https://www.postgresql.jp/document/12/html/monitoring-stats.html

テーブルの統計情報と違う性質の統計情報。
例えばpg_stat_databaseでは、データベース上のトランザクションの実行回数、キャッシュヒットブロック数などが見れる。

定期的に確認しておくと、曜日や時間帯などの傾向が確認できる。
データベース全体のチューニングには必要。SQLチューニングに利用できるかは不明。

以下、OSS-DB Goldの試験範囲に記載あるもの。見ておくといいことありそう。

> pg_stat_activity、pg_stat_database
> pg_stat_all_tables 等、行レベル統計情報
> pg_statio_all_tables 等、ブロックレベル統計情報

> pg_stat_archiver
> pg_stat_bgwriter
> 待機イベント(pg_stat_activity.wait_event)
> pg_stat_progress_vacuum

## ヒント句
(参考)https://pghintplan.osdn.jp/pg_hint_plan-ja.html

オプティマイザが言う事聞かないとき、祈りながらつけるやつ。
公式ではなく、外部モジュールな点に注意。

## insert,updateのオーバーヘッド
(参考)https://www.postgresql.jp/document/12/html/populate.html

データ移行など、insert,updateが頻繁に走るときは、オーバーヘッドを減らすべし。
オーバーヘッドは具体的には、

- インデックスにもinsert(updateも内部的にdelete→insertしている、はず)
- 制約の条件を満たしているか照合(一意制約、外部キー制約)

など。あとは参考URLで(丸投げ)

## その他

### SQL関連書籍の目次
目次レベルでも思い出すといいことあるかも(お祈り)
時間あれば書籍読み返したい。

https://www.shoeisha.co.jp/book/detail/9784798157825#contents
https://www.shoeisha.co.jp/book/detail/9784798128931#contents
https://gihyo.jp/book/2019/978-4-297-10408-5/#toc
https://www.oreilly.co.jp/books/9784873115894/#toc

### スライド
色々あると思いますがとりあえず道中で見つけたやつ。

https://www.slideshare.net/nttdata-tech/postgresql-monitoring-features-ntt-data 
