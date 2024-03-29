---
date   : 2022-03-02
title  : 🔍 クエリ関数化
excerpt: ユーザー定義関数（UDF）
tags   : ["🔍", "Google BigQuery", "SQL", "UDF", "自作関数", "TEMPORARY FUNCTION"]
---

![BigQuery](https://cdn-ssl-devio-img.classmethod.jp/wp-content/uploads/2020/09/gcp-eyecatch-bigquery_1200x630.png)

## || ユーザー定義関数（UDF）
### | メリット
1. コーディングのコストを低減できる
2. コードを理解しやすくできる
3. バグ混入のリスクを低減できる

### | クエリ
```sql
------------------------------------ユーザー定義関数------------------------------------
-- visitStartTimeを日本時間の日付に変更する関数
CREATE TEMPORARY FUNCTION DATE_BY_VISITSTARTTIME(visitStartTime INT64)
    AS (
        DATE(TIMESTAMP_SECONDS(visitStartTime), "Asia/Tokyo"))
    );

-- ページに対応するIDを返す関数 --★ 複数回使っているUDFだが、この処理を変更すると全ての処理に変更が適応される
CREATE TEMPORARY FUNCTION PAGECATEGORY_ID_BY_PAGECATEGORY_NAME(pageCategoryName STRING)
    AS(
        CASE pageCategoryName
            WHEN "トップ" THEN 1
            WHEN "検索結果" THEN 2
            WHEN "商品詳細" THEN 3
            WHEN "その他" THEN 4
        END
    );

-- ページを判定し対応するページIDを返す関数
CREATE TEMPORARY FUNCTION CATEGORIZE_WEB_PAGECATEGORY_ID(pagePath STRING)
    AS (
        CASE
            WHEN REGEXP_CONTAINS(pagePath, r"[トップのページパスに該当する正規表現]")
                THEN PAGECATEGORY_ID_BY_PAGECATEGORY_NAME("トップ")
            WHEN REGEXP_CONTAINS(pagePath, r"[検索結果のページパスに該当する正規表現]")
                THEN PAGECATEGORY_ID_BY_PAGECATEGORY_NAME("検索結果")
            WHEN REGEXP_CONTAINS(pagePath, r"[商品詳細のページパスに該当する正規表現]")
                THEN PAGECATEGORY_ID_BY_PAGECATEGORY_NAME("商品詳細")
            ELSE
                PAGECATEGORY_ID_BY_PAGECATEGORY_NAME("その他")
        END
    );

--★ ここまではチーム内で共通して使っているUDFなので、レビュー不要
 
SELECT 
    DATE_BY_VISITSTARTTIME(visitStartTime) AS visitDate --★ 日時の変換処理が直感的でバグが混入しづらい
    , CATEGORIZE_WEB_PAGECATEGORY_ID(hits.page.pagePath) AS pageCategoryId --★ 記述すると長くなる分類処理が短く済む
    , COUNT(DISTINCT fullVisitorId) AS UU
FROM
    `[セッション情報からなるテーブル]`, UNNEST(hits) AS hits
WHERE
    pageCategoryId <> PAGECATEGORY_ID_BY_PAGECATEGORY_NAME("その他") --★ <> 4 だと4の意味がわからない、誤りがあっても気づけない
GROUP BY
    visitDate
    , pageCategoryId
```

> クエリ全体で見ると行数が多く感じるかもしれませんが、書く・読む作業が発生するのはほぼSELECT文のみで行数は少ないです。UDF部分はすでに用意してあるものを使っているため、書く・読む手間はほぼありません。
>
> 今回はUDFを一時的なUDFとして扱っていますが、永続的なUDFとして予め用意していると記述する必要もなくなります。
> 
> 参考：[標準 SQL ユーザー定義関数 | BigQuery | Google Cloud](https://cloud.google.com/bigquery/docs/reference/standard-sql/user-defined-functions?hl=ja)

（参考になりすぎるTT）


### | まねっこ（自作関数）

```sql
create temp function
    calc_elapsed_month(BASE_DATE datetime, VALUE_DATE datetime) as (
        /*
         * 経過月数を正しく規定する関数
         * eg. 集計期間2011/2/1~2012/1/31の1年間とする時、2011/1/31から翌日に予約をした会員も月差分で加えられてしまう。
         *     回避として、そういった1ケ月満たない会員に「-1」を渡す処理をする。
         */
        date_diff(BASE_DATE, VALUE_DATE, month) 
        + if(date_diff(BASE_DATE, date_add(VALUE_DATE, interval date_diff(BASE_DATE, VALUE_DATE, month) month), day) >= 0, 0, -1)
    );
```

## || 関数を永続的な UDF として作成 & 呼び出し
### | 作成
```sql
-- 作成した関数を永続化（データセットつけて実行する）
create function
    `prj.dataset`.calc_elapsed_month(BASE_DATE datetime, VALUE_DATE datetime) as (~);

```
上記ができると
prj > dataset >『routine』指定したデータセット直下にroutineなるものができている！

![無題](https://user-images.githubusercontent.com/28585421/173274249-bca84580-7877-407e-8f95-d3594310db83.png)

cf. [SQL UDF](https://cloud.google.com/bigquery/docs/reference/standard-sql/user-defined-functions?hl=ja#sql-udf-structure) - GoogleCloud


### | 呼び出し
```sql
-- データセット名をつければ『routine』からいつでも呼び込める！
select 
    `prj.dataset`.calc_elapsed_month(BASE_DATE, VALUE_DATE)
    , BASE_DATE
    , VALUE_DATE
from ~
```
cf. [永続的なユーザー定義関数（UDF）の呼び出し](https://cloud.google.com/bigquery/docs/reference/standard-sql/functions-reference?hl=ja#calling_persistent_user-defined_functions_udfs) -- GoogleCLoud

    ※プロジェクトを跨ぐ場合はも同様でプロジェクト名を付与して絶対パスにすると◎



## || JavaScript UDF

UDF内部にJSの処理を追加することもできるらしい！（すご）

```sql
# Model function
CREATE TEMPORARY FUNCTION lr_prob(x1 FLOAT64, x2 FLOAT64, w0 FLOAT64, w1 FLOAT64, w2 FLOAT64)
      RETURNS FLOAT64
      LANGUAGE js AS """
        return 1 / (1 + Math.exp(-(w0 + w1 * x1 + w2 * x2)));
      """;
```
cf.[JavaScript UDF](https://cloud.google.com/bigquery/docs/reference/standard-sql/user-defined-functions?hl=ja#javascript-udf-structure) - GoogleCLoud


機会学習とかで使えそう。(cf.[「BigQuery ML」：SQLで機械学習ってどういうこと？試しにSQLでロジスティック回帰を書いてみた。](https://www.wantedly.com/companies/wantedly/post_articles/129482))



## || REFERENCE
+ [BigQueryでユーザー定義関数（UDF）は武器になるという話](https://techblog.zozo.com/entry/bigquery-udf) - ZOZO
+ [Standard SQL user-defined functions](https://cloud.google.com/bigquery/docs/reference/standard-sql/user-defined-functions?hl=ja) - Google Cloud
