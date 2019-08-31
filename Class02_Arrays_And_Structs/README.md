# Arrays And Structs

## Introduction

As we've seen in Class 01, one of the biggest powers of BigQuery comes from operation nested data directly. The better we use this paradigm the less slots will be allocated to our queries which in turn makes BigQuery more effective, cheaper and faster.

But also, as we've discussed there, it gets some practice before getting used to that. So with no further ado, let's get some practice.

## Practice Time

Head over to your BigQuery WebUI (if you haven't already please follow through [Class00 Setup](../Class00_Setup/README.md) for setting up the required environment) and run the following query:

```sql
WITH `data` AS (
  SELECT 1 AS user_id, 3 AS total_pageviews, 0 AS total_clicks UNION ALL
  SELECT 2, 5, 1 UNION ALL
  SELECT 3, 1, 0
)


SELECT
  *
FROM `data`
```

You'll get something like this:

<p align="center">
  <img src="./images/ex1.png">
</p>

Notice the response can be represented as a JSON:

<p align="center">
  <img src="./images/ex1_json.png">
</p>

We consider this data of type "unnested"; it's a flatten representation divided by each row of what happened.

In this example, we have a simulation of customers visiting some sort of web page; first column is the user identification, second is how many pages were visited and finally how many clicks were tracked.

Let's improve on this data now: suppose we want to track which were the pages visited and which products were clicked for those customers visiting our web site; we'll track each "hit" on our web site, its number identification and what happened there.

Run the following in your BigQuery:

```sql
WITH `data` AS (
  SELECT 1 AS user_id, 3 AS total_pageviews, 0 AS total_clicks, 0 AS hit_number, 'bq_workshop_page_1.html' AS page, NULL AS sku UNION ALL
  SELECT 1 AS user_id, 3 AS total_pageviews, 0 AS total_clicks, 1 AS hit_number, 'bq_workshop_page_2.html' AS page, NULL AS sku UNION ALL
  SELECT 1 AS user_id, 3 AS total_pageviews, 0 AS total_clicks, 2 AS hit_number, 'bq_workshop_page_3.html' AS page, NULL AS sku UNION ALL
  
  SELECT 2 AS user_id, 5 AS total_pageviews, 1 AS total_clicks, 0 AS hit_number, 'bq_workshop_page_1.html' AS page, 'sku0' AS sku UNION ALL
  SELECT 2 AS user_id, 5 AS total_pageviews, 1 AS total_clicks, 1 AS hit_number, 'bq_workshop_page_2.html' AS page, NULL AS sku UNION ALL
  SELECT 2 AS user_id, 5 AS total_pageviews, 1 AS total_clicks, 2 AS hit_number, 'bq_workshop_page_3.html' AS page, NULL AS sku UNION ALL
  SELECT 2 AS user_id, 5 AS total_pageviews, 1 AS total_clicks, 3 AS hit_number, 'bq_workshop_page_4.html' AS page, NULL AS sku UNION ALL
  
  SELECT 3 AS user_id, 1 AS total_pageviews, 0 AS total_clicks, 0 AS hit_number, 'bq_workshop_page_1.html' AS page, NULL AS sku
)

SELECT
  *
FROM `data`
```

