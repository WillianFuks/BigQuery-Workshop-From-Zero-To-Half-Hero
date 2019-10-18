# Somewhat Advanced Cookbook

First two examples are based on the official BigQuery [Cookbook](https://support.google.com/analytics/answer/4419694?hl=en), section "Advanced Query Examples".

- [Products purchased by customers who purchased Product A](#products-purchased-by-customers-who-purchased-product-a)
- [Average Number of Hits Before Purchase](#average-number-of-hits-before-purchase)
- [Cosine Similarity Between Products](#cosine-similarity-between-products)


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

This example is based on Dafiti's Google Analytics data. When customers run a search, an event is sent to GA; the challenge then is to find search performance in terms of sales and products interactions.

