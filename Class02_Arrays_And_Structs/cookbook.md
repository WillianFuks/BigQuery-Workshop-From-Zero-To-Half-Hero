# Somewhat Advanced Cookbook

First two examples are based on the official BigQuery [Cookbook](https://support.google.com/analytics/answer/4419694?hl=en), section "Advanced Query Examples".

- [Products purchased by customers who purchased Product A](#products-purchased-by-customers-who-purchased-product-a)
- [Average Number of Hits Before Purchase](#average-number-of-hits-before-purchase)
- [Cosine Similarity Between Products](#cosine-similarity-between-products)
- [Conversion Path](#conversion-path)
- [Price Performance Variation ](#price-performance-variation)
- [Querying From Multiple Datasets](#querying-from-multiple-datasets)

### Products purchased by customers who purchased Product A

Let's say we want to find top products purchased along with another given product for [google analytics data](https://support.google.com/analytics/answer/3437719?hl=en). Here's a possible solution:

```sql
WITH `data` AS (
  SELECT ARRAY<STRUCT<product ARRAY<STRUCT<v2ProductName STRING> >, eCommerceAction STRUCT<action_type STRING> >> [STRUCT([STRUCT('name0' AS v2ProductName), STRUCT('name1' AS v2ProductName) ], STRUCT('1' AS action_type)), STRUCT([STRUCT('name0' AS v2ProductName), STRUCT('name1' AS v2ProductName) ], STRUCT('2' AS action_type)), STRUCT([STRUCT('name0' AS v2ProductName), STRUCT('name1' AS v2ProductName) ], STRUCT('6' AS action_type))] AS hits UNION ALL
  SELECT ARRAY<STRUCT<product ARRAY<STRUCT<v2ProductName STRING> >, eCommerceAction STRUCT<action_type STRING> >> [STRUCT([STRUCT('name0' AS v2ProductName), STRUCT('name1' AS v2ProductName) ], STRUCT('6' AS action_type)), STRUCT([STRUCT('name0' AS v2ProductName), STRUCT('name2' AS v2ProductName) ], STRUCT('6' AS action_type)), STRUCT([STRUCT('name1' AS v2ProductName), STRUCT('name2' AS v2ProductName) ], STRUCT('6' AS action_type))] AS hits 
)


SELECT
  v2ProductName,
  COUNT(v2ProductName) freq
FROM(
  SELECT 
    ARRAY(
      SELECT AS STRUCT
        v2ProductName,
        ecommerceaction.action_type 
      FROM UNNEST(hits), UNNEST(product)
      WHERE v2ProductName != 'name0' AND v2ProductName IS NOT NULL AND ecommerceaction.action_type = '6' AND EXISTS(SELECT 1 FROM UNNEST(product) WHERE v2ProductName = 'name0')
    ) AS hits
  FROM `data`
  WHERE EXISTS(SELECT 1 FROM UNNEST(hits), UNNEST(product) WHERE v2ProductName = 'name0' AND ecommerceAction.action_type = '6')
), UNNEST(hits)
GROUP BY v2ProductName
ORDER BY freq DESC
```

We simulated some GA data but we can use a public dataset as well:

```sql
DECLARE productName STRING;
SET productName = "Android Men's Long Sleeve Badge Crew Tee Heather";


SELECT
  v2ProductName,
  COUNT(v2ProductName) freq
FROM(
  SELECT 
    ARRAY(
      SELECT AS STRUCT
        v2ProductName,
        ecommerceaction.action_type 
      FROM UNNEST(hits), UNNEST(product)
      WHERE v2ProductName != productName AND v2ProductName IS NOT NULL AND ecommerceaction.action_type = '6' AND EXISTS(SELECT 1 FROM UNNEST(product) WHERE v2ProductName = productName)
    ) AS hits
  FROM `bigquery-public-data.google_analytics_sample.ga_sessions_20170801`
  WHERE EXISTS(SELECT 1 FROM UNNEST(hits), UNNEST(product) WHERE v2ProductName = productName AND ecommerceAction.action_type = '6')
), UNNEST(hits)
GROUP BY v2ProductName
ORDER BY freq DESC
```

* scripting in BigQuery is still Beta as of the writing of this document.
* Notice that by simulating data we run sort of a query test validation, just as unit testing does for software development.

### Average Number of Hits Before Purchase

Given GA data with each action of customers having an ID number starting from 1, find on average how many actions customers make before purchasing something.

Here's a possible solution:

```sql
WITH `data` AS (
  SELECT ARRAY<STRUCT<product ARRAY<STRUCT<v2productName STRING> >, eCommerceAction STRUCT<action_type STRING>, hitNumber INT64 >> [STRUCT([STRUCT('name0' AS productName), STRUCT('name1' AS productName) ], STRUCT('1' AS action_type), 1 AS hitNumber), STRUCT([STRUCT('name0' AS productName), STRUCT('name1' AS productName) ], STRUCT('2' AS action_type), 2 AS hitNumber), STRUCT([STRUCT('name0' AS productName), STRUCT('name1' AS productName) ], STRUCT('6' AS action_type), 3 AS hitNumber)] AS hits UNION ALL
    SELECT ARRAY<STRUCT<product ARRAY<STRUCT<productName STRING> >, eCommerceAction STRUCT<action_type STRING>, hitNumber INT64 >> [STRUCT([STRUCT('name0' AS productName), STRUCT('name1' AS productName) ], STRUCT('6' AS action_type), 1), STRUCT([STRUCT('name0' AS productName), STRUCT('name2' AS productName) ], STRUCT('6' AS action_type), 2), STRUCT([STRUCT('name1' AS productName), STRUCT('name2' AS productName) ], STRUCT('6' AS action_type), 3)] AS hits UNION ALL
    SELECT ARRAY<STRUCT<product ARRAY<STRUCT<productName STRING> >, eCommerceAction STRUCT<action_type STRING>, hitNumber INT64 >> [] 
)

SELECT
  SUM(hits) / SUM(freq) avg_hits
FROM(
  SELECT 
    (SELECT SUM(hitNumber) FROM UNNEST(hits) WHERE ecommerceAction.action_type = '6') hits,
    (SELECT 1 FROM UNNEST(hits) WHERE ecommerceAction.action_type = '6' LIMIT 1) freq
  FROM `data`
)
```

On real data:

```sql
SELECT
  SUM(hits) / SUM(freq) avg_hits
FROM(
  SELECT 
    (SELECT SUM(hitNumber) FROM UNNEST(hits) WHERE ecommerceAction.action_type = '6') hits,
    (SELECT 1 FROM UNNEST(hits) WHERE ecommerceAction.action_type = '6' LIMIT 1) freq
  FROM `bigquery-public-data.google_analytics_sample.ga_sessions_20170801`
)
```

### Cosine Similarity Between Products

Using GA data, let's find the cosine similarity between products using customers interactions with items.

Cosine is defined as:

<p align="center">
  <img src="http://www.sciweavers.org/tex2img.php?eq=cos%3CA%2C%20B%3E%20%3D%20%5Cfrac%7B%5Csum_%7Bi%3D0%7D%5En%20a_i%20%2A%20b_i%7D%7B%5Cleft%5CVert%20A%20%5Cright%5CVert%20%2A%20%5Cleft%5CVert%20B%20%5Cright%5CVert%7D%0A&bc=White&fc=Black&im=jpg&fs=12&ff=arev&edit=0" align="center" border="0" alt="cos<A, B> = \frac{\sum_{i=0}^n a_i * b_i}{\left\Vert A \right\Vert * \left\Vert B \right\Vert}" width="218" height="50" />
</p>

Where vectors A and B are the customers interactions.

We'll consider that browsing an item adds 0.5 point, adding it to cart adds 2 and purchasing, 6.

Here's a possible solution:


```sql
CREATE TEMP FUNCTION normalize_scores(productA STRING, products ARRAY<STRUCT<productB STRING, score FLOAT64> >, apply_sqrt BOOL) RETURNS STRUCT<productA STRING, products ARRAY<STRUCT<productB STRING, score FLOAT64> > > AS ((
    SELECT AS STRUCT
      productA,
      ARRAY(SELECT AS STRUCT productB, score / (SELECT IF(apply_sqrt, SQRT(score), score) FROM UNNEST(products) a WHERE a.productB = productA) AS score FROM UNNEST(products)) AS products      
));


WITH `data` AS (
  SELECT ARRAY<STRUCT<product ARRAY<STRUCT<v2productName STRING> >, eCommerceAction STRUCT<action_type STRING> >> [STRUCT([STRUCT('name0' AS v2productName), STRUCT('name1' AS v2productName) ], STRUCT('1' AS action_type)), STRUCT([STRUCT('name0' AS v2productName)], STRUCT('2' AS action_type)), STRUCT([STRUCT('name0' AS v2productName), STRUCT('name1' AS v2productName) ], STRUCT('3' AS action_type)), STRUCT([STRUCT('name1' AS v2productName), STRUCT('name2' AS v2productName)], STRUCT('6' AS action_type))] AS hits
  UNION ALL
  SELECT ARRAY<STRUCT<product ARRAY<STRUCT<v2productName STRING> >, eCommerceAction STRUCT<action_type STRING> >> [STRUCT([STRUCT('name0' AS v2productName), STRUCT('name2' AS v2productName) ], STRUCT('3' AS action_type)), STRUCT([STRUCT('name1' AS v2productName)], STRUCT('2' AS action_type)), STRUCT([STRUCT('name0' AS v2productName), STRUCT('name2' AS v2productName) ], STRUCT('3' AS action_type)), STRUCT([STRUCT('name1' AS v2productName), STRUCT('name2' AS v2productName)], STRUCT('6' AS action_type))] AS hits
)


SELECT AS VALUE
  normalize_scores(productB, ARRAY_AGG(STRUCT(productA, score)), FALSE)
FROM(
  SELECT AS VALUE
    normalize_scores(productA, ARRAY_AGG(STRUCT(productB, score)), TRUE)
  FROM(
    SELECT
      a.productName AS productA,
      b.productName AS productB,
      SUM(a.score * b.score) score
    FROM(
      SELECT
        ARRAY(
          SELECT AS STRUCT
            v2ProductName AS productName,
            SUM(CASE WHEN ecommerceAction.action_type = '2' THEN 0.5 WHEN ecommerceAction.action_type = '3' THEN 2 ELSE 6 END)  AS score FROM UNNEST(hits), UNNEST(product) WHERE ecommerceAction.action_type IN ('2', '3', '6') GROUP BY v2productName) AS products
      FROM `bigquery-public-data.google_analytics_sample.ga_sessions_20170801`
    ), UNNEST(products) AS a, UNNEST(products) AS b
    GROUP BY productA, productB
  )
  GROUP BY productA
), UNNEST(products)
GROUP BY productB
```

This is an entire recommender system algorithm represented in 20 lines capable or processing terabytes of data in a few seconds.

We recommend you running parts of this query to see what's going on. Briefly, first there's the most internal query `SELECT ARRAY (...)` where a mapping between actions and points is done.
Then, those scores are summed up which represents the numerator part of the cosine equation. Finally, the normalization works by dividing the whole thing by the norm of each vector (we do this twice as we have two products, each has to be divided by its own norm).

### Conversion Path

An important analysis done by ecommerces in general is trying to understand what impact each marketing channel has on sales.

One way to do so is to track all visits from each customer until a purchase is done and then somehow assign that transaction to all channels observed so far. This can be done using some rule, for instance, a common approach is to assign the very first and last observed channels the weight `20%` and everything in between receives `60%`.

Here's a query that replicates transactions among all channels observed before:

```sql
CREATE TEMP FUNCTION process_transactions(hits ARRAY<STRUCT<transactionId STRING, transactionRevenue INT64> > ) RETURNS STRING AS ((
    SELECT ARRAY_TO_STRING(ARRAY(SELECT CONCAT(transactionId, ';', CAST(transactionRevenue/1e6 AS STRING)) FROM UNNEST(hits) WHERE transactionId IS NOT NULL AND transactionRevenue IS NOT NULL), '|')
));


# Retrieves customers, channels used to get into our website and transactions within the session.
WITH aggregated_channels AS(
    SELECT
      fullvisitorId user_id,
      ARRAY_AGG((
          SELECT AS STRUCT
            visitStartTime AS visit_time,
            CONCAT(trafficSource.source, "|", trafficSource.medium) AS channel,
            process_transactions(ARRAY(SELECT AS STRUCT transaction.transactionId, transaction.transactionRevenue FROM UNNEST(hits))) AS transactions,
            IF(totals.transactions IS NOT NULL, 1 , NULL) AS transaction_flag
      )) channels
    FROM `bigquery-public-data.google_analytics_sample.ga_sessions_20170801`
    GROUP BY user_id
),
# Assigns each transaction to all channels observed in sessions before
channels_with_consolidated_transactions AS(
  SELECT
    user_id,
    ARRAY(
      SELECT AS STRUCT
        * EXCEPT(transactions),
        ARRAY(
          SELECT AS STRUCT
            SPLIT(value, ';')[SAFE_OFFSET(0)] id,
            SPLIT(value, ';')[SAFE_OFFSET(1)] revenue
          FROM UNNEST(SPLIT(transactions, '|')) AS value
        ) AS transactions
      FROM UNNEST(channels)
    ) AS channels
  FROM(
    SELECT
      user_id,
      ARRAY(
        SELECT AS STRUCT
          * EXCEPT(channels_transactions_flag, transactions),
          MAX(transactions) OVER(PARTITION BY channels_transactions_flag) transactions
        FROM UNNEST(channels)
      ) AS channels
    FROM(
      SELECT
        user_id,
        ARRAY(SELECT AS STRUCT * EXCEPT(transaction_flag), SUM(transaction_flag) OVER(ORDER BY visit_time DESC) channels_transactions_flag FROM UNNEST(channels)) channels
      FROM aggregated_channels
      WHERE ((SELECT 1 FROM UNNEST(channels) WHERE transactions != '' LIMIT 1)) > 0
    )
  )
)


SELECT
  user_id,
  channels.*
FROM(
  SELECT
    user_id,
       ARRAY(
        SELECT AS STRUCT
          * EXCEPT(transactions, id, revenue),
          transactions.id AS transaction_id,
          transactions.revenue AS transaction_revenue
        FROM UNNEST(channels), UNNEST(transactions) AS transactions
      ) AS channels
  FROM channels_with_consolidated_transactions
), UNNEST(channels) AS channels
WHERE TRUE
  AND transaction_id != ''
```

Notice here the technique `SUM(flag) OVER(ORDER BY ...)`. This can be quite helpful to track down a list of rows tied to a specific event; in our case, the transaction id.

Also, we used `WITH` clauses to help in reading the query, only remember that results there are not consolidated in any form which means each time we query data inside this block, BigQuery will process everything again. Use it just for readability purposes!

### Search Performance

Let's find the performance of a search engine working for a given ecommerce with the following rules:
  - When customers make a search, an event is registered. For a given catalog page, the only evidence of that being search related is through the URL that should contain the string `q=text search`
  - After a search, customer can browse any other page, not related to a search result. 
  - More than one search can be performed, sales for each has to be distributed accordingly.
  - For each search term, we want to find its total ocurrencies, clicks on products from the catalog, how many bounces we had (clients that leave the page right after seeing the search result) and revenue.

We've simulated `data` on this one. Here's a possible solution:

```sql
CREATE TEMP FUNCTION removeAccents(text STRING) RETURNS STRING AS ((
  SELECT REGEXP_REPLACE(NORMALIZE(text, NFD), r'\pM', '')
));

CREATE TEMP FUNCTION slugify(phrase STRING) RETURNS STRING AS ((
  SELECT REGEXP_REPLACE(REGEXP_REPLACE(REGEXP_REPLACE(REGEXP_REPLACE(REGEXP_REPLACE(REGEXP_REPLACE(removeAccents(LOWER(phrase)), r'\s+', '-'), r'&', '-e-'), r'[^\w-]+', ''), r'--+', '-'), r'^-+', ''), r'-+$', '')
));

# Extracts revenue from list of products purchased from the search page
CREATE TEMP FUNCTION processSearchRevenue(skus_clicked ARRAY<STRING>, purchased_skus ARRAY<STRUCT<sku STRING, revenue FLOAT64> >) RETURNS FLOAT64 AS ((
  SELECT SUM(revenue) FROM UNNEST(purchased_skus) WHERE EXISTS(SELECT 1 FROM UNNEST(skus_clicked) sku_clicked WHERE sku_clicked = sku)
));

# Compares the search term to the string in the page URL; returns True if there's a match
CREATE TEMP FUNCTION isSearchPage(search_term STRING, page_url STRING) RETURNS BOOL AS ((
  SELECT LOGICAL_AND(STRPOS(page_url, x) > 0 OR STRPOS(LOWER(page_url), removeAccents(LOWER(x))) > 0) FROM UNNEST(SPLIT(search_term, ' ')) x
));


WITH `data` AS(
  SELECT "1" AS fullvisitorid, 1 AS visitid,  "20171220" AS date, STRUCT<totalTransactionRevenue FLOAT64>  (100000000.0) AS totals, ARRAY<STRUCT<hitNumber INT64, page STRUCT<pagePath STRING>, ecommerceAction STRUCT<action_type STRING>, eventInfo STRUCT<eventCategory STRING, eventLabel STRING>, product ARRAY<STRUCT<productSku STRING, isClick BOOL, productQuantity INT64, productPrice FLOAT64> >>> 
    [STRUCT(1 AS hitNumber, STRUCT("/" AS pagePath) AS page, STRUCT("0" AS action_type) AS ecommerceAction, STRUCT(NULL AS eventCategory, NULL AS eventLabel) AS eventInfo, [STRUCT("" AS productSku, False AS isClick, 0 AS productQuantity, 0.0 AS productPrice)] AS product),
     STRUCT(2 AS hitNumber, STRUCT("/?q=fake+search" AS pagePath) AS page, STRUCT("0" AS action_type) AS ecommerceAction, STRUCT(NULL AS eventCategory, NULL AS eventLabel) AS eventInfo, [STRUCT("sku0" AS productSku, False AS isClick, 0 AS productQuantity, 0.0 AS productPrice), STRUCT("sku1" AS productSku, False AS isClick, 0 AS productQuantity, 0.0 AS productPrice)] AS product),
     STRUCT(3 AS hitNumber, STRUCT("/?q=fake+search" AS pagePath) AS page, STRUCT("0" AS action_type) AS ecommerceAction, STRUCT(NULL AS eventCategory, NULL AS eventLabel) AS eventInfo, [STRUCT("sku0" AS productSku, True AS isClick, 0 AS productQuantity, 0.0 AS productPrice)] AS product),
     STRUCT(4 AS hitNumber, STRUCT("/checkout" AS pagePath) AS page, STRUCT("6" AS action_type) AS ecommerceAction, STRUCT(NULL AS eventCategory, NULL AS eventLabel) AS eventInfo, [STRUCT("sku0-000" AS productSku, False AS isClick, 1 AS productQuantity, 100000000.0 AS productPrice)] AS product)] hits

  UNION ALL
  
  SELECT "2" AS fullvisitorid, 1 as visitid, "20171220" AS date, STRUCT<totalTransactionRevenue FLOAT64>  (NULL) AS totals, ARRAY<STRUCT<hitNumber INT64, page STRUCT<pagePath STRING>, ecommerceAction STRUCT<action_type STRING>, eventInfo STRUCT<eventCategory STRING, eventLabel STRING>, product ARRAY<STRUCT<productSku STRING, isClick BOOL, productQuantity INT64, productPrice FLOAT64> >>> 
    [STRUCT(1 AS hitNumber, STRUCT("/" AS pagePath) AS page, STRUCT("0" AS action_type) AS ecommerceAction, STRUCT(NULL AS eventCategory, NULL AS eventLabel) AS eventInfo, [STRUCT("" AS productSku, False AS isClick, 0 AS productQuantity, 0.0 AS productPrice)] AS product),
     STRUCT(2 AS hitNumber, STRUCT("/" AS pagePath) AS page, STRUCT("0" AS action_type) AS ecommerceAction, STRUCT("search" AS eventCategory, "search string" AS eventLabel) AS eventInfo, NULL AS product), STRUCT(3 AS hitNumber, STRUCT("/?q=search+string" AS pagePath) AS page, STRUCT("0" AS action_type) AS ecommerceAction, STRUCT(NULL AS eventCategory, NULL AS eventLabel) AS eventInfo, [STRUCT("" AS productSku, False AS isClick, 0 AS productQuantity, 0.0 AS productPrice)] AS product)] hits

  UNION ALL     
     
  SELECT "2" AS fullvisitorid, 2 as visitid, "20171220" AS date, STRUCT<totalTransactionRevenue FLOAT64>  (NULL) AS totals, ARRAY<STRUCT<hitNumber INT64, page STRUCT<pagePath STRING>, ecommerceAction STRUCT<action_type STRING>, eventInfo STRUCT<eventCategory STRING, eventLabel STRING>, product ARRAY<STRUCT<productSku STRING, isClick BOOL, productQuantity INT64, productPrice FLOAT64> >>> 
    [STRUCT(1 AS hitNumber, STRUCT("/" AS pagePath) AS page, STRUCT("0" AS action_type) AS ecommerceAction, STRUCT(NULL AS eventCategory, NULL AS eventLabel) AS eventInfo, [STRUCT("" AS productSku, False AS isClick, 0 AS productQuantity, 0.0 AS productPrice)] AS product),
     STRUCT(2 AS hitNumber, STRUCT("/" AS pagePath) AS page, STRUCT("0" AS action_type) AS ecommerceAction, STRUCT("search" AS eventCategory, "search string" AS eventLabel) AS eventInfo, NULL AS product),
     STRUCT(3 AS hitNumber, STRUCT("/?q=search+string" AS pagePath) AS page, STRUCT("0" AS action_type) AS ecommerceAction, STRUCT(NULL AS eventCategory, NULL AS eventLabel) AS eventInfo, [STRUCT("" AS productSku, False AS isClick, 0 AS productQuantity, 0.0 AS productPrice)] AS product),     
     STRUCT(4 AS hitNumber, STRUCT("/?q=search+string" AS pagePath) AS page, STRUCT("0" AS action_type) AS ecommerceAction, STRUCT(NULL AS eventCategory, NULL AS eventLabel) AS eventInfo, [STRUCT("sku0" AS productSku, True AS isClick, 0 AS productQuantity, 0.0 AS productPrice)] AS product),
     STRUCT(5 AS hitNumber, STRUCT("/" AS pagePath) AS page, STRUCT("0" AS action_type) AS ecommerceAction, STRUCT("search" AS eventCategory, "search another string" AS eventLabel) AS eventInfo, NULL AS product),
     STRUCT(6 AS hitNumber, STRUCT("/?q=search+another+string" AS pagePath) AS page, STRUCT("0" AS action_type) AS ecommerceAction, STRUCT(NULL AS eventCategory, NULL AS eventLabel) AS eventInfo, [STRUCT("sku0" AS productSku, True AS isClick, 0 AS productQuantity, 0.0 AS productPrice)] AS product)] hits
     
  UNION ALL
 
  SELECT "3" AS fullvisitorid, 1 as visitid, "20171220" AS date, STRUCT<totalTransactionRevenue FLOAT64>  (200000000.0) AS totals, ARRAY<STRUCT<hitNumber INT64, page STRUCT<pagePath STRING>, ecommerceAction STRUCT<action_type STRING>, eventInfo STRUCT<eventCategory STRING, eventLabel STRING>, product ARRAY<STRUCT<productSku STRING, isClick BOOL, productQuantity INT64, productPrice FLOAT64> >>> 
    [STRUCT(1 AS hitNumber, STRUCT("/" AS pagePath) AS page, STRUCT("0" AS action_type) AS ecommerceAction, STRUCT(NULL AS eventCategory, NULL AS eventLabel) AS eventInfo, [STRUCT("" AS productSku, False AS isClick, 0 AS productQuantity, 0.0 AS productPrice)] AS product), 
     STRUCT(2 AS hitNumber, STRUCT("/" AS pagePath) AS page, STRUCT("0" AS action_type) AS ecommerceAction, STRUCT("search" AS eventCategory, "search string" AS eventLabel) AS eventInfo, [STRUCT("" AS productSku, False AS isClick, 0 AS productQuantity, 0.0 AS productPrice)] AS product),
     STRUCT(3 AS hitNumber, STRUCT("/?q=search+string" AS pagePath) AS page, STRUCT("0" AS action_type) AS ecommerceAction, STRUCT(NULL AS eventCategory, NULL AS eventLabel) AS eventInfo, [STRUCT("sku0" AS productSku, True AS isClick, 0 AS productQuantity, 0.0 AS productPrice)] AS product),
     STRUCT(4 AS hitNumber, STRUCT("/checkout" AS pagePath) AS page, STRUCT("6" AS action_type) AS ecommerceAction, STRUCT(NULL AS eventCategory, NULL AS eventLabel) AS eventInfo, [STRUCT("sku0" AS productSku, False AS isClick, 2 AS productQuantity, 100000000.0 AS productPrice)] AS product)] hits
     
  UNION ALL
 
  SELECT "4" AS fullvisitorid, 1 as visitid, "20171220" AS date, STRUCT<totalTransactionRevenue FLOAT64>  (100000000.0) AS totals, ARRAY<STRUCT<hitNumber INT64, page STRUCT<pagePath STRING>, ecommerceAction STRUCT<action_type STRING>, eventInfo STRUCT<eventCategory STRING, eventLabel STRING>, product ARRAY<STRUCT<productSku STRING, isClick BOOL, productQuantity INT64, productPrice FLOAT64> >>> 
    [STRUCT(1 AS hitNumber, STRUCT("/" AS pagePath) AS page, STRUCT("0" AS action_type) AS ecommerceAction, STRUCT(NULL AS eventCategory, NULL AS eventLabel) AS eventInfo, [STRUCT("" AS productSku, False AS isClick, 0 AS productQuantity, 0.0 AS productPrice)] AS product),
     STRUCT(2 AS hitNumber, STRUCT("/" AS pagePath) AS page, STRUCT("0" AS action_type) AS ecommerceAction, STRUCT("search" AS eventCategory, "Sêãrchí CrÃzĩ Éstrìng" AS eventLabel) AS eventInfo, NULL AS product),
     STRUCT(3 AS hitNumber, STRUCT("/?q=Sêãrchí CrÃzĩ Éstrìng" AS pagePath) AS page, STRUCT("0" AS action_type) AS ecommerceAction, STRUCT(NULL AS eventCategory, NULL AS eventLabel) AS eventInfo, [STRUCT("sku0" AS productSku, True AS isClick, 0 AS productQuantity, 0.0 AS productPrice)] AS product),
     STRUCT(4 AS hitNumber, STRUCT("/?q=Searchi%20Crazi%20Estring&sort=discount" AS pagePath) AS page, STRUCT("0" AS action_type) AS ecommerceAction, STRUCT(NULL AS eventCategory, NULL AS eventLabel) AS eventInfo,
     [STRUCT("sku0" AS productSku, True AS isClick, 0 AS productQuantity, 0.0 AS productPrice)] AS product),
     STRUCT(5 AS hitNumber, STRUCT("/?q=Searchi%20Crazi%20Estring&sort=discount" AS pagePath) AS page, STRUCT("0" AS action_type) AS ecommerceAction, STRUCT("search" AS eventCategory, "Searchi crÃzĩ Éstrìng" AS eventLabel) AS eventInfo, [STRUCT("" AS productSku, False AS isClick, 0 AS productQuantity, 0.0 AS productPrice)] AS product),
     STRUCT(6 AS hitNumber, STRUCT("/?q=Searchi%20crazi%20Estring&sort=discount" AS pagePath) AS page, STRUCT("0" AS action_type) AS ecommerceAction, STRUCT(NULL AS eventCategory, NULL AS eventLabel) AS eventInfo, [STRUCT("sku0" AS productSku, True AS isClick, 0 AS productQuantity, 0.0 AS productPrice)] AS product),    
     STRUCT(7 AS hitNumber, STRUCT("/checkout" AS pagePath) AS page, STRUCT("6" AS action_type) AS ecommerceAction, STRUCT(NULL AS eventCategory, NULL AS eventLabel) AS eventInfo, [STRUCT("sku0" AS productSku, False AS isClick, 1 AS productQuantity, 100000000.0 AS productPrice)] AS product)] hits
     
  UNION ALL
 
  SELECT "4" AS fullvisitorid, 2 as visitid, "20171220" AS date, STRUCT<totalTransactionRevenue FLOAT64>  (100000000.0) AS totals, ARRAY<STRUCT<hitNumber INT64, page STRUCT<pagePath STRING>, ecommerceAction STRUCT<action_type STRING>, eventInfo STRUCT<eventCategory STRING, eventLabel STRING>, product ARRAY<STRUCT<productSku STRING, isClick BOOL, productQuantity INT64, productPrice FLOAT64> >>> 
    [STRUCT(1 AS hitNumber, STRUCT("/" AS pagePath) AS page, STRUCT("0" AS action_type) AS ecommerceAction, STRUCT(NULL AS eventCategory, NULL AS eventLabel) AS eventInfo, [STRUCT("" AS productSku, False AS isClick, 0 AS productQuantity, 0.0 AS productPrice)] AS product),
     STRUCT(2 AS hitNumber, STRUCT("/" AS pagePath) AS page, STRUCT("0" AS action_type) AS ecommerceAction, STRUCT("search" AS eventCategory, "Seãrchí Crazĩ estrìng" AS eventLabel) AS eventInfo, [STRUCT("" AS productSku, False AS isClick, 0 AS productQuantity, 0.0 AS productPrice)] AS product),
     STRUCT(3 AS hitNumber, STRUCT("/?q=Seãrchí Crazĩ estrìng" AS pagePath) AS page, STRUCT("0" AS action_type) AS ecommerceAction, STRUCT(NULL AS eventCategory, NULL AS eventLabel) AS eventInfo, NULL AS product),
     STRUCT(4 AS hitNumber, STRUCT("/?q=Searchi%20Crazi%20estring&sort=discount" AS pagePath) AS page, STRUCT("0" AS action_type) AS ecommerceAction, STRUCT(NULL AS eventCategory, NULL AS eventLabel) AS eventInfo, [STRUCT("sku1" AS productSku, True AS isClick, 0 AS productQuantity, 0.0 AS productPrice)] AS product),
     STRUCT(5 AS hitNumber, STRUCT("/?q=Searchi%20Crazi%20estring&sort=discount" AS pagePath) AS page, STRUCT("0" AS action_type) AS ecommerceAction, STRUCT("search" AS eventCategory, "search other string" AS eventLabel) AS eventInfo,
     NULL AS product),
     STRUCT(6 AS hitNumber, STRUCT("/?q=search%20other%20string&sort=discount" AS pagePath) AS page, STRUCT("0" AS action_type) AS ecommerceAction, STRUCT(NULL AS eventCategory, NULL AS eventLabel) AS eventInfo, [STRUCT("sku0" AS productSku, True AS isClick, 0 AS productQuantity, 0.0 AS productPrice)] AS product),
     STRUCT(7 AS hitNumber, STRUCT("/checkout" AS pagePath) AS page, STRUCT("6" AS action_type) AS ecommerceAction, STRUCT(NULL AS eventCategory, NULL AS eventLabel) AS eventInfo, [STRUCT("sku0" AS productSku, False AS isClick, 1 AS productQuantity, 100000000.0 AS productPrice), STRUCT("sku1" AS productSku, False AS isClick, 1 AS productQuantity, 150000000.0 AS productPrice)] AS product)] hits
     
  UNION ALL
 
  SELECT "4" AS fullvisitorid, 1 as visitid, "20171221" AS date, STRUCT<totalTransactionRevenue FLOAT64>  (100000000.0) AS totals, ARRAY<STRUCT<hitNumber INT64, page STRUCT<pagePath STRING>, ecommerceAction STRUCT<action_type STRING>, eventInfo STRUCT<eventCategory STRING, eventLabel STRING>, product ARRAY<STRUCT<productSku STRING, isClick BOOL, productQuantity INT64, productPrice FLOAT64> >>> 
    [STRUCT(1 AS hitNumber, STRUCT("/" AS pagePath) AS page, STRUCT("0" AS action_type) AS ecommerceAction, STRUCT(NULL AS eventCategory, NULL AS eventLabel) AS eventInfo, [STRUCT("" AS productSku, False AS isClick, 0 AS productQuantity, 0.0 AS productPrice)] AS product),
     STRUCT(2 AS hitNumber, STRUCT("/" AS pagePath) AS page, STRUCT("0" AS action_type) AS ecommerceAction, STRUCT("search" AS eventCategory, "Seãrchí Crazĩ estrìng" AS eventLabel) AS eventInfo, NULL AS product),
     STRUCT(3 AS hitNumber, STRUCT("/?q=Seãrchí Crazĩ estrìng" AS pagePath) AS page, STRUCT("0" AS action_type) AS ecommerceAction, STRUCT(NULL AS eventCategory, NULL AS eventLabel) AS eventInfo, [STRUCT("sku0" AS productSku, True AS isClick, 0 AS productQuantity, 0.0 AS productPrice)] AS product)] hits
)

SELECT
  date,
  search_term,
  COUNT(search_term) AS ocurrencies,
  SUM(search_click) AS clicks,
  SUM(search_revenue) AS revenue,
  SUM(search_page_bounce) AS search_page_bounces
FROM(
  SELECT
    date,
    ARRAY(SELECT AS STRUCT
      search_term,
      MAX(IF(sku_clicked IS NOT NULL, 1, 0)) AS search_click,
      processSearchRevenue(ARRAY_AGG(DISTINCT sku_clicked IGNORE NULLS), products_purchased) AS search_revenue,
      MAX(bounce) AS search_page_bounce
    FROM UNNEST(hits) WHERE search_term IS NOT NULL GROUP BY search_term) AS hits
  FROM(
    SELECT
       date,
       ARRAY(SELECT AS STRUCT
         slugify(FIRST_VALUE(search_term) OVER (PARTITION BY search_flag ORDER BY hit_number)) AS search_term,
         IF(isSearchPage(FIRST_VALUE(search_term) OVER (PARTITION BY search_flag ORDER BY hit_number), page_url) AND EXISTS(SELECT 1 FROM UNNEST(products) WHERE isClick), ARRAY(SELECT productSku FROM UNNEST(products))[SAFE_OFFSET(0)], NULL) AS sku_clicked,
         IF(isSearchPage(FIRST_VALUE(search_term) OVER (PARTITION BY search_flag ORDER BY hit_number), page_url) AND hit_number = MAX(hit_number) OVER(), 1, NULL) AS bounce
       FROM UNNEST(hits)) hits,
       ARRAY(SELECT AS STRUCT productSku AS sku, SUM(revenue) AS revenue FROM UNNEST(hits), UNNEST(products) WHERE action_type = '6' GROUP BY 1) AS products_purchased
    FROM(
      SELECT
        date,
        ARRAY_CONCAT_AGG(ARRAY(SELECT AS STRUCT
          hitNumber AS hit_number,
          page.pagePath AS page_url,
          IF(eventInfo.eventCategory = 'search', eventInfo.eventLabel, NULL) AS search_term,
          SUM(IF(eventInfo.eventCategory = 'search', 1, 0)) OVER(ORDER BY hitNumber) AS search_flag,
          ecommerceAction.action_type AS action_type,
          ARRAY(SELECT AS STRUCT productSku, isClick, (productQuantity * productPrice / 1e6) AS revenue FROM UNNEST(product)) AS products
        FROM UNNEST(hits))) AS hits
      FROM `data`
      GROUP BY date, fullvisitorid
    )
  )
), UNNEST(hits)
GROUP BY date, search_term
```

The query is a bit complex. Let's understand its constituent parts, first we have:

```sql
SELECT
  date,
  ARRAY_CONCAT_AGG(ARRAY(SELECT AS STRUCT
    hitNumber AS hit_number,
    page.pagePath AS page_url,
    IF(eventInfo.eventCategory = 'search', eventInfo.eventLabel, NULL) AS search_term,
    SUM(IF(eventInfo.eventCategory = 'search', 1, 0)) OVER(ORDER BY hitNumber) AS search_flag,
    ecommerceAction.action_type AS action_type,
    ARRAY(SELECT AS STRUCT productSku, isClick, (productQuantity * productPrice / 1e6) AS revenue FROM UNNEST(product)) AS products
  FROM UNNEST(hits))) AS hits
FROM `data`
GROUP BY date, fullvisitorid
```

First we take each `hits` array from customer and aggregate it (`ARRAY_CONCAT_AGG`).

For each `hit`, we extract the search term and prepare sales from products.

Then we have upper the query:

```sql
    SELECT
       date,
       ARRAY(SELECT AS STRUCT
         slugify(FIRST_VALUE(search_term) OVER (PARTITION BY search_flag ORDER BY hit_number)) AS search_term,
         IF(isSearchPage(FIRST_VALUE(search_term) OVER (PARTITION BY search_flag ORDER BY hit_number), page_url) AND EXISTS(SELECT 1 FROM UNNEST(products) WHERE isClick), ARRAY(SELECT productSku FROM UNNEST(products))[SAFE_OFFSET(0)], NULL) AS sku_clicked,
         IF(isSearchPage(FIRST_VALUE(search_term) OVER (PARTITION BY search_flag ORDER BY hit_number), page_url) AND hit_number = MAX(hit_number) OVER(), 1, NULL) AS bounce
       FROM UNNEST(hits)) hits,
       ARRAY(SELECT AS STRUCT productSku AS sku, SUM(revenue) AS revenue FROM UNNEST(hits), UNNEST(products) WHERE action_type = '6' GROUP BY 1) AS products_purchased
    FROM (...)
```

Here we repeate the value of the search term using as partition the search flags.

What happens next is straightfoward: for each search term we track clicks, bounces, sales and general performances.

Finally last query is just a consolidation of everything that happened for each date and search term.

### Price Performance Variation 

For each sku, we'll find how many impressions and product views it had for each selling price. A view is considered to be valid if and only if it happens after a catalog impression event has been observed.

One possible solution would be:

```sql
SELECT
  date,
  sku,
  ARRAY_AGG(STRUCT(price, impressions, views)) AS performance
FROM(
  SELECT
    date,
    sku,
    price,
    SUM(impression) AS impressions,
    SUM(view) AS views
  FROM(
    SELECT
     date,
     ARRAY(SELECT AS STRUCT
       productSKU AS sku,
       productPrice / 1e6 AS price,
       MAX(CAST(isImpression AS INT64)) AS impression,
       IF(MIN(IF(ecommerceAction.action_type = '2', time, NULL)) > MIN(IF(isImpression, time, NULL)), 1, NULL) AS view
     FROM UNNEST(hits) LEFT JOIN UNNEST(product) WHERE (productSKU != '(not set)' AND productSKU IS NOT NULL)  GROUP BY productSKU, price HAVING impression IS NOT NULL) AS products
      FROM `bigquery-public-data.google_analytics_sample.ga_sessions_20170801`
    WHERE EXISTS(SELECT 1 FROM UNNEST(hits), UNNEST(product) WHERE isImpression)
  ), UNNEST(products)
  GROUP BY date, sku, price
)
GROUP BY date, sku
```

### Querying From Multiple Datasets

Sometimes your data might come from distinct datasets with same schema. Here's one example for handling that:

```sql
SELECT
  date,
  device_ AS device_category,
FROM(
  SELECT 
    *
  FROM (
    SELECT
      *,
      device.deviceCategory AS device_
    FROM `{projectId}{datasetId}{first_table}`
    WHERE _TABLE_SUFFIX  BETWEEN FORMAT_DATE("%Y%m%d", DATE_SUB(CURRENT_DATE(), INTERVAL 1 DAY)) AND FORMAT_DATE("%Y%m%d", DATE_SUB(CURRENT_DATE(), INTERVAL 1 DAY))
  ) UNION ALL (
  SELECT
    *,
    'app' AS device_
  FROM `{projectId}{datasetId}{second_table}`
  WHERE _TABLE_SUFFIX  BETWEEN FORMAT_DATE("%Y%m%d", DATE_SUB(CURRENT_DATE(), INTERVAL 1 DAY)) AND FORMAT_DATE("%Y%m%d", DATE_SUB(CURRENT_DATE(), INTERVAL 1 DAY))
  )
)
GROUP BY date, device_
```

We simply encapsulate everything in a `SELECT * FROM (...)` followed by a `UNION ALL`.
