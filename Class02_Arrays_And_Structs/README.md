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

Notice the response can be represented as a JSON as well where each row represents each user:

```json
[
  {
    "user_id": "1",
    "total_pageviews": "3",
    "total_clicks": "0"
  },
  {
    "user_id": "2",
    "total_pageviews": "5",
    "total_clicks": "1"
  },
  {
    "user_id": "3",
    "total_pageviews": "1",
    "total_clicks": "0"
  }
]
```

We consider this data of type "unnested"; it's a flatten representation divided by each record row.

In this example, we have a simulation of customers visiting a web page; first column is the user identification, second is how many pages were visited and, finally, how many clicks were tracked.

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

This is the result:

<table pan-table="" class="p6n-bq-results-table-pb p6n-table" role="grid" jslog="47391;track:generic_click"> <thead pan-sort-agent="sortCtrl"> <tr><!----> <th>Row</th> <!----><th ng-repeat="header in ctrl.schema.fields track by (ctrl.tableId + $index)"> user_id </th><!----><th ng-repeat="header in ctrl.schema.fields track by (ctrl.tableId + $index)"> total_pageviews </th><!----><th ng-repeat="header in ctrl.schema.fields track by (ctrl.tableId + $index)"> total_clicks </th><!----><th ng-repeat="header in ctrl.schema.fields track by (ctrl.tableId + $index)"> hit_number </th><!----><th ng-repeat="header in ctrl.schema.fields track by (ctrl.tableId + $index)"> page </th><!----><th ng-repeat="header in ctrl.schema.fields track by (ctrl.tableId + $index)"> sku </th><!----> <th class="p6n-bq-empty-last-column"></th> </tr> </thead> <tbody> <!----><tr pan-table-row="" ng-repeat="row in ctrl.rows | panSortBy:(sortCtrl&amp;&amp;sortCtrl.getActiveKey()):&quot;normal&quot;:sortCache:paginateCtrl  track by (ctrl.tableId + ':' + row.rowTrackBy + ':row' + $index)" class="p6n-bq-last-row-of-record" ng-bind-html="row.htmlRow" ng-init="$last &amp;&amp; panTableCtrl.onRowRepeatEnd()" pan-table-row-after-repeat="row"><td class="p6n-bq-row-number">1</td><td class="p6n-bq-number-cell"><div>1</div></td><td class="p6n-bq-number-cell"><div>3</div></td><td class="p6n-bq-number-cell"><div>0</div></td><td class="p6n-bq-number-cell"><div>0</div></td><td><div>bq_workshop_page_1.html</div></td><td><div class="p6n-bq-null-cell">null</div></td><td class="p6n-bq-empty-last-column"></td></tr><!----><tr pan-table-row="" ng-repeat="row in ctrl.rows | panSortBy:(sortCtrl&amp;&amp;sortCtrl.getActiveKey()):&quot;normal&quot;:sortCache:paginateCtrl  track by (ctrl.tableId + ':' + row.rowTrackBy + ':row' + $index)" class="p6n-bq-last-row-of-record" ng-bind-html="row.htmlRow" ng-init="$last &amp;&amp; panTableCtrl.onRowRepeatEnd()" pan-table-row-after-repeat="row"><td class="p6n-bq-row-number">2</td><td class="p6n-bq-number-cell"><div>1</div></td><td class="p6n-bq-number-cell"><div>3</div></td><td class="p6n-bq-number-cell"><div>0</div></td><td class="p6n-bq-number-cell"><div>1</div></td><td><div>bq_workshop_page_2.html</div></td><td><div class="p6n-bq-null-cell">null</div></td><td class="p6n-bq-empty-last-column"></td></tr><!----><tr pan-table-row="" ng-repeat="row in ctrl.rows | panSortBy:(sortCtrl&amp;&amp;sortCtrl.getActiveKey()):&quot;normal&quot;:sortCache:paginateCtrl  track by (ctrl.tableId + ':' + row.rowTrackBy + ':row' + $index)" class="p6n-bq-last-row-of-record" ng-bind-html="row.htmlRow" ng-init="$last &amp;&amp; panTableCtrl.onRowRepeatEnd()" pan-table-row-after-repeat="row"><td class="p6n-bq-row-number">3</td><td class="p6n-bq-number-cell"><div>1</div></td><td class="p6n-bq-number-cell"><div>3</div></td><td class="p6n-bq-number-cell"><div>0</div></td><td class="p6n-bq-number-cell"><div>2</div></td><td><div>bq_workshop_page_3.html</div></td><td><div class="p6n-bq-null-cell">null</div></td><td class="p6n-bq-empty-last-column"></td></tr><!----><tr pan-table-row="" ng-repeat="row in ctrl.rows | panSortBy:(sortCtrl&amp;&amp;sortCtrl.getActiveKey()):&quot;normal&quot;:sortCache:paginateCtrl  track by (ctrl.tableId + ':' + row.rowTrackBy + ':row' + $index)" class="p6n-bq-last-row-of-record" ng-bind-html="row.htmlRow" ng-init="$last &amp;&amp; panTableCtrl.onRowRepeatEnd()" pan-table-row-after-repeat="row"><td class="p6n-bq-row-number">4</td><td class="p6n-bq-number-cell"><div>2</div></td><td class="p6n-bq-number-cell"><div>5</div></td><td class="p6n-bq-number-cell"><div>1</div></td><td class="p6n-bq-number-cell"><div>0</div></td><td><div>bq_workshop_page_1.html</div></td><td><div>sku0</div></td><td class="p6n-bq-empty-last-column"></td></tr><!----><tr pan-table-row="" ng-repeat="row in ctrl.rows | panSortBy:(sortCtrl&amp;&amp;sortCtrl.getActiveKey()):&quot;normal&quot;:sortCache:paginateCtrl  track by (ctrl.tableId + ':' + row.rowTrackBy + ':row' + $index)" class="p6n-bq-last-row-of-record" ng-bind-html="row.htmlRow" ng-init="$last &amp;&amp; panTableCtrl.onRowRepeatEnd()" pan-table-row-after-repeat="row"><td class="p6n-bq-row-number">5</td><td class="p6n-bq-number-cell"><div>2</div></td><td class="p6n-bq-number-cell"><div>5</div></td><td class="p6n-bq-number-cell"><div>1</div></td><td class="p6n-bq-number-cell"><div>1</div></td><td><div>bq_workshop_page_2.html</div></td><td><div class="p6n-bq-null-cell">null</div></td><td class="p6n-bq-empty-last-column"></td></tr><!----><tr pan-table-row="" ng-repeat="row in ctrl.rows | panSortBy:(sortCtrl&amp;&amp;sortCtrl.getActiveKey()):&quot;normal&quot;:sortCache:paginateCtrl  track by (ctrl.tableId + ':' + row.rowTrackBy + ':row' + $index)" class="p6n-bq-last-row-of-record" ng-bind-html="row.htmlRow" ng-init="$last &amp;&amp; panTableCtrl.onRowRepeatEnd()" pan-table-row-after-repeat="row"><td class="p6n-bq-row-number">6</td><td class="p6n-bq-number-cell"><div>2</div></td><td class="p6n-bq-number-cell"><div>5</div></td><td class="p6n-bq-number-cell"><div>1</div></td><td class="p6n-bq-number-cell"><div>2</div></td><td><div>bq_workshop_page_3.html</div></td><td><div class="p6n-bq-null-cell">null</div></td><td class="p6n-bq-empty-last-column"></td></tr><!----><tr pan-table-row="" ng-repeat="row in ctrl.rows | panSortBy:(sortCtrl&amp;&amp;sortCtrl.getActiveKey()):&quot;normal&quot;:sortCache:paginateCtrl  track by (ctrl.tableId + ':' + row.rowTrackBy + ':row' + $index)" class="p6n-bq-last-row-of-record" ng-bind-html="row.htmlRow" ng-init="$last &amp;&amp; panTableCtrl.onRowRepeatEnd()" pan-table-row-after-repeat="row"><td class="p6n-bq-row-number">7</td><td class="p6n-bq-number-cell"><div>2</div></td><td class="p6n-bq-number-cell"><div>5</div></td><td class="p6n-bq-number-cell"><div>1</div></td><td class="p6n-bq-number-cell"><div>3</div></td><td><div>bq_workshop_page_4.html</div></td><td><div class="p6n-bq-null-cell">null</div></td><td class="p6n-bq-empty-last-column"></td></tr><!----><tr pan-table-row="" ng-repeat="row in ctrl.rows | panSortBy:(sortCtrl&amp;&amp;sortCtrl.getActiveKey()):&quot;normal&quot;:sortCache:paginateCtrl  track by (ctrl.tableId + ':' + row.rowTrackBy + ':row' + $index)" class="p6n-bq-last-row-of-record" ng-bind-html="row.htmlRow" ng-init="$last &amp;&amp; panTableCtrl.onRowRepeatEnd()" pan-table-row-after-repeat="row"><td class="p6n-bq-row-number">8</td><td class="p6n-bq-number-cell"><div>3</div></td><td class="p6n-bq-number-cell"><div>1</div></td><td class="p6n-bq-number-cell"><div>0</div></td><td class="p6n-bq-number-cell"><div>0</div></td><td><div>bq_workshop_page_1.html</div></td><td><div class="p6n-bq-null-cell">null</div></td><td class="p6n-bq-empty-last-column"></td></tr><!----> </tbody> </table>


Noticed what happened here: as the data is flatten, we need to keep repeating the columns "user_id", "total_pageviews" and so on through each row.

When working with small data this is fine; big data is another thing though; reading through terabytes of the same repeated field is not a very effective approach after all.

### Arrays

That's where Arrays and Structs comes into play. [Arrays](https://cloud.google.com/bigquery/docs/reference/standard-sql/arrays) basically let us repeate a given field; here's one way of building it in BigQuery:

```sql
WITH `data` AS (
  SELECT 1 AS user_id, 3 AS total_pageviews, 0 AS total_clicks, ['page1', 'page2', 'page3'] AS pages UNION ALL
  SELECT 2, 5, 1, ['page1', 'page2', 'page3', 'page4', 'page5'] UNION ALL
  SELECT 3, 1, 0, ['page1']
)


SELECT
  *
FROM `data`
```

Results:

<table pan-table="" class="p6n-bq-results-table-pb p6n-table" role="grid" jslog="47391;track:generic_click"> <thead pan-sort-agent="sortCtrl"> <tr><!----> <th>Row</th> <!----><th ng-repeat="header in ctrl.schema.fields track by (ctrl.tableId + $index)"> user_id </th><!----><th ng-repeat="header in ctrl.schema.fields track by (ctrl.tableId + $index)"> total_pageviews </th><!----><th ng-repeat="header in ctrl.schema.fields track by (ctrl.tableId + $index)"> total_clicks </th><!----><th ng-repeat="header in ctrl.schema.fields track by (ctrl.tableId + $index)"> pages </th><!----> <th class="p6n-bq-empty-last-column"></th> </tr> </thead> <tbody> <!----><tr pan-table-row="" ng-repeat="row in ctrl.rows | panSortBy:(sortCtrl&amp;&amp;sortCtrl.getActiveKey()):&quot;normal&quot;:sortCache:paginateCtrl  track by (ctrl.tableId + ':' + row.rowTrackBy + ':row' + $index)" class="" ng-bind-html="row.htmlRow" ng-init="$last &amp;&amp; panTableCtrl.onRowRepeatEnd()" pan-table-row-after-repeat="row"><td class="p6n-bq-row-number">1</td><td class="p6n-bq-number-cell"><div>1</div></td><td class="p6n-bq-number-cell"><div>3</div></td><td class="p6n-bq-number-cell"><div>0</div></td><td><div>page1</div></td><td class="p6n-bq-empty-last-column"></td></tr><!----><tr pan-table-row="" ng-repeat="row in ctrl.rows | panSortBy:(sortCtrl&amp;&amp;sortCtrl.getActiveKey()):&quot;normal&quot;:sortCache:paginateCtrl  track by (ctrl.tableId + ':' + row.rowTrackBy + ':row' + $index)" class="" ng-bind-html="row.htmlRow" ng-init="$last &amp;&amp; panTableCtrl.onRowRepeatEnd()" pan-table-row-after-repeat="row"><td class="p6n-bq-row-number"></td><td></td><td></td><td></td><td><div>page2</div></td><td class="p6n-bq-empty-last-column"></td></tr><!----><tr pan-table-row="" ng-repeat="row in ctrl.rows | panSortBy:(sortCtrl&amp;&amp;sortCtrl.getActiveKey()):&quot;normal&quot;:sortCache:paginateCtrl  track by (ctrl.tableId + ':' + row.rowTrackBy + ':row' + $index)" class="p6n-bq-last-row-of-record" ng-bind-html="row.htmlRow" ng-init="$last &amp;&amp; panTableCtrl.onRowRepeatEnd()" pan-table-row-after-repeat="row"><td class="p6n-bq-row-number"></td><td></td><td></td><td></td><td><div>page3</div></td><td class="p6n-bq-empty-last-column"></td></tr><!----><tr pan-table-row="" ng-repeat="row in ctrl.rows | panSortBy:(sortCtrl&amp;&amp;sortCtrl.getActiveKey()):&quot;normal&quot;:sortCache:paginateCtrl  track by (ctrl.tableId + ':' + row.rowTrackBy + ':row' + $index)" class="" ng-bind-html="row.htmlRow" ng-init="$last &amp;&amp; panTableCtrl.onRowRepeatEnd()" pan-table-row-after-repeat="row"><td class="p6n-bq-row-number">2</td><td class="p6n-bq-number-cell"><div>2</div></td><td class="p6n-bq-number-cell"><div>5</div></td><td class="p6n-bq-number-cell"><div>1</div></td><td><div>page1</div></td><td class="p6n-bq-empty-last-column"></td></tr><!----><tr pan-table-row="" ng-repeat="row in ctrl.rows | panSortBy:(sortCtrl&amp;&amp;sortCtrl.getActiveKey()):&quot;normal&quot;:sortCache:paginateCtrl  track by (ctrl.tableId + ':' + row.rowTrackBy + ':row' + $index)" class="" ng-bind-html="row.htmlRow" ng-init="$last &amp;&amp; panTableCtrl.onRowRepeatEnd()" pan-table-row-after-repeat="row"><td class="p6n-bq-row-number"></td><td></td><td></td><td></td><td><div>page2</div></td><td class="p6n-bq-empty-last-column"></td></tr><!----><tr pan-table-row="" ng-repeat="row in ctrl.rows | panSortBy:(sortCtrl&amp;&amp;sortCtrl.getActiveKey()):&quot;normal&quot;:sortCache:paginateCtrl  track by (ctrl.tableId + ':' + row.rowTrackBy + ':row' + $index)" class="" ng-bind-html="row.htmlRow" ng-init="$last &amp;&amp; panTableCtrl.onRowRepeatEnd()" pan-table-row-after-repeat="row"><td class="p6n-bq-row-number"></td><td></td><td></td><td></td><td><div>page3</div></td><td class="p6n-bq-empty-last-column"></td></tr><!----><tr pan-table-row="" ng-repeat="row in ctrl.rows | panSortBy:(sortCtrl&amp;&amp;sortCtrl.getActiveKey()):&quot;normal&quot;:sortCache:paginateCtrl  track by (ctrl.tableId + ':' + row.rowTrackBy + ':row' + $index)" class="" ng-bind-html="row.htmlRow" ng-init="$last &amp;&amp; panTableCtrl.onRowRepeatEnd()" pan-table-row-after-repeat="row"><td class="p6n-bq-row-number"></td><td></td><td></td><td></td><td><div>page4</div></td><td class="p6n-bq-empty-last-column"></td></tr><!----><tr pan-table-row="" ng-repeat="row in ctrl.rows | panSortBy:(sortCtrl&amp;&amp;sortCtrl.getActiveKey()):&quot;normal&quot;:sortCache:paginateCtrl  track by (ctrl.tableId + ':' + row.rowTrackBy + ':row' + $index)" class="p6n-bq-last-row-of-record" ng-bind-html="row.htmlRow" ng-init="$last &amp;&amp; panTableCtrl.onRowRepeatEnd()" pan-table-row-after-repeat="row"><td class="p6n-bq-row-number"></td><td></td><td></td><td></td><td><div>page5</div></td><td class="p6n-bq-empty-last-column"></td></tr><!----><tr pan-table-row="" ng-repeat="row in ctrl.rows | panSortBy:(sortCtrl&amp;&amp;sortCtrl.getActiveKey()):&quot;normal&quot;:sortCache:paginateCtrl  track by (ctrl.tableId + ':' + row.rowTrackBy + ':row' + $index)" class="p6n-bq-last-row-of-record" ng-bind-html="row.htmlRow" ng-init="$last &amp;&amp; panTableCtrl.onRowRepeatEnd()" pan-table-row-after-repeat="row"><td class="p6n-bq-row-number">3</td><td class="p6n-bq-number-cell"><div>3</div></td><td class="p6n-bq-number-cell"><div>1</div></td><td class="p6n-bq-number-cell"><div>0</div></td><td><div>page1</div></td><td class="p6n-bq-empty-last-column"></td></tr><!----> </tbody> </table>

Or in JSON format:

```json
[
  {
    "user_id": "1",
    "total_pageviews": "3",
    "total_clicks": "0",
    "pages": [
      "page1",
      "page2",
      "page3"
    ]
  },
  {
    "user_id": "2",
    "total_pageviews": "5",
    "total_clicks": "1",
    "pages": [
      "page1",
      "page2",
      "page3",
      "page4",
      "page5"
    ]
  },
  {
    "user_id": "3",
    "total_pageviews": "1",
    "total_clicks": "0",
    "pages": [
      "page1"
    ]
  }
]
```

Notice the very cool consequence of that: we no longer have repetitions happening. In just one row, BigQuery already has access to all data it can get about each user.

For those who know about [norm forms](https://en.wikipedia.org/wiki/Database_normalization) applied to databases, BigQuery goes to the complete opposite direction: the more denormalized, the better! Reason being that costs are not as bad as processing resources; if in just on read BigQuery already has access to everything it needs, then we can build some pretty neat strategies to avoid leveraging computing in exchange for a bit more of storage costs (which, specially in these days, is quite cheap and not source of much concern).

### Structs

Besides arrays, we also have the type [Struct](https://cloud.google.com/bigquery/docs/reference/standard-sql/data-types#struct-type).

Think of struct as if we are opening a new JSON, or in other words, starting path of a new branch in a tree.

Here's an example using structs with our previous example:

```sql
WITH `data` AS (
  SELECT 1 AS user_id, STRUCT(3 AS pageviews, 0 AS clicks) AS totals, ARRAY<STRUCT<number INT64, page STRING, sku STRING>> [
      STRUCT(0 AS number, 'bq_workshop_page_1.html' AS page, NULL AS sku),
      STRUCT(1 AS number, 'bq_workshop_page_2.html' AS page, NULL AS sku),
      STRUCT(2 AS number, 'bq_workshop_page_3.html' AS page, NULL AS sku)
  ] AS hits UNION ALL
  SELECT 2 AS user_id, STRUCT(5 AS pageviews, 1 AS clicks) AS totals, ARRAY<STRUCT<number INT64, page STRING, sku STRING>> [
      STRUCT(0 AS number, 'bq_workshop_page_1.html' AS page, 'sku0' AS sku),
      STRUCT(1 AS number, 'bq_workshop_page_2.html' AS page, NULL AS sku),
      STRUCT(2 AS number, 'bq_workshop_page_3.html' AS page, NULL AS sku),
      STRUCT(3 AS number, 'bq_workshop_page_4.html' AS page, NULL AS sku)
  ] AS hits UNION ALL
  SELECT 1 AS user_id, STRUCT(3 AS pageviews, 0 AS clicks) AS totals, ARRAY<STRUCT<number INT64, page STRING, sku STRING>> [
      STRUCT(0 AS number, 'bq_workshop_page_1.html' AS page, NULL AS sku),
      STRUCT(1 AS number, 'bq_workshop_page_2.html' AS page, NULL AS sku),
      STRUCT(2 AS number, 'bq_workshop_page_3.html' AS page, NULL AS sku)
  ] AS hits
)

SELECT
  *
FROM `data`
```

