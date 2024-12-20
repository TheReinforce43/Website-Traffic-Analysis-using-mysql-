# Maven Fuzzy Factory SQL Queries

This README provides an overview of SQL queries used to analyze traffic, conversion rates, trends, and landing page performance for the Maven Fuzzy Factory dataset. The queries cover diverse scenarios, from basic traffic breakdowns to advanced trend analyses and landing page tests.

---

## 1. Order Item Refunds

```sql
SELECT * FROM mavenfuzzyfactory.order_item_refunds;
```

This query retrieves all data from the `order_item_refunds` table.

---

## 2. Site Traffic Breakdown

### General Traffic Breakdown

```sql
SELECT
    utm_source,
    utm_campaign,
    http_referer,
    COUNT(DISTINCT website_session_id) AS sessions
FROM
    mavenfuzzyfactory.website_sessions
WHERE
    created_at < '2012-04-12'
        AND utm_source IS NOT NULL
GROUP BY 1, 2, 3
ORDER BY 4 DESC;
```
This query counts unique website sessions based on UTM parameters and HTTP referers.

### Traffic Breakdown for GSearch (Nonbrand Campaign)

```sql
SELECT
    utm_source,
    utm_campaign,
    http_referer,
    COUNT(DISTINCT website_session_id) AS sessions
FROM
    mavenfuzzyfactory.website_sessions
WHERE
    created_at < '2012-04-12'
        AND utm_source IS NOT NULL
        AND utm_campaign = 'nonbrand'
GROUP BY 1, 2, 3
ORDER BY 4 DESC;
```
Filters traffic data specifically for "GSearch" under the "nonbrand" campaign.

---

## 3. GSearch Conversion Rate (CVR)

```sql
WITH cte AS (
    SELECT
        COUNT(DISTINCT ws.website_session_id) AS sessions,
        COUNT(DISTINCT order_id) AS orders
    FROM
        mavenfuzzyfactory.website_sessions AS ws
        LEFT JOIN
        mavenfuzzyfactory.orders USING (website_session_id)
    WHERE
        ws.utm_source = 'gsearch'
        AND ws.utm_campaign = 'nonbrand'
        AND ws.created_at < '2012-04-14'
)
SELECT *, CONCAT(ROUND((orders / sessions) * 100, 2), '%') AS CVR
FROM cte;
```
Calculates the conversion rate for GSearch under the "nonbrand" campaign, filtering for a CVR of at least 4%.

---

## 4. Trended Summaries with Sessions

```sql
SELECT
    YEAR(created_at) AS cr_year,
    WEEK(created_at) AS cr_week,
    COUNT(DISTINCT website_session_id) AS sessions
FROM
    mavenfuzzyfactory.website_sessions
WHERE
    website_session_id BETWEEN 100000 AND 115000
GROUP BY 1, 2
ORDER BY 3 DESC;
```
Summarizes session trends by year and week.

---

## 5. Pivot Table for Orders

```sql
SELECT
    primary_product_id,
    COUNT(DISTINCT CASE
        WHEN items_purchased = 1 THEN order_id
        ELSE NULL
    END) AS order_1_purchased,
    COUNT(DISTINCT CASE
        WHEN items_purchased = 2 THEN order_id
        ELSE NULL
    END) AS order_2_purchased
FROM
    mavenfuzzyfactory.orders
WHERE
    order_id BETWEEN 31000 AND 32000
GROUP BY 1
ORDER BY 1, 2 DESC;
```
Generates a pivot table for orders based on the number of items purchased.

---

## 6. GSearch Volume Trends

```sql
SELECT
    MIN(DATE(created_at)) AS week_start,
    MAX(DATE(created_at)) AS week_end,
    COUNT(DISTINCT website_session_id) AS sessions
FROM
    mavenfuzzyfactory.website_sessions
WHERE
    created_at < '2012-05-10'
        AND utm_source = 'gsearch'
        AND utm_campaign = 'nonbrand'
GROUP BY YEARWEEK(created_at);
```
Analyzes weekly trends for GSearch traffic under the "nonbrand" campaign.

---

## 7. GSearch Device-Level Performance

```sql
WITH cte AS (
    SELECT
        ws.device_type,
        COUNT(DISTINCT website_session_id) AS sessions,
        COUNT(DISTINCT order_id) AS orders
    FROM
        mavenfuzzyfactory.website_sessions AS ws
        LEFT JOIN
        mavenfuzzyfactory.orders AS o USING (website_session_id)
    WHERE
        LOWER(ws.utm_source) = 'gsearch'
        AND ws.created_at < '2012-05-11'
        AND LOWER(ws.utm_campaign) = 'nonbrand'
    GROUP BY ws.device_type
)
SELECT *, CONCAT(ROUND((orders / sessions) * 100, 2), ' %') AS conversion_rate FROM cte;
```
Tracks device-level performance for GSearch traffic under the "nonbrand" campaign.

---

## 8. GSearch Device-Level Trends

```sql
SELECT
    MIN(DATE(created_at)) AS week_start,
    COUNT(CASE
        WHEN device_type = 'desktop' THEN website_session_id
        ELSE NULL
    END) AS dtop_sessions,
    COUNT(CASE
        WHEN device_type = 'mobile' THEN website_session_id
        ELSE NULL
    END) AS mob_sessions
FROM
    mavenfuzzyfactory.website_sessions
WHERE
    utm_campaign = 'nonbrand'
        AND utm_source = 'gsearch'
        AND created_at BETWEEN '2012-04-16' AND '2012-06-08'
GROUP BY YEARWEEK(created_at);
```
Provides weekly device-level session trends.

---

## 9. Top 5 Website Pages by Sessions (Before 2012-06-09)

```sql
SELECT
    pageview_url, COUNT(DISTINCT website_session_id) AS sessions
FROM
    mavenfuzzyfactory.website_pageviews AS wp
        LEFT JOIN
    mavenfuzzyfactory.website_sessions AS ws USING (website_session_id)
WHERE
    ws.created_at < '2012-06-09'
GROUP BY pageview_url
ORDER BY sessions DESC
LIMIT 5;
```
Identifies the top 5 website pages based on sessions.

---

## 10. Top 5 Entry Pages

### Step A: Identify First Page Views

```sql
CREATE TEMPORARY TABLE first_page_views
SELECT
    website_session_id,
    MIN(website_pageview_id) AS min_page_views
FROM mavenfuzzyfactory.website_pageviews
WHERE created_at < '2012-06-12'
GROUP BY website_session_id;
```

### Step B: Find Landing Pages

```sql
SELECT
    wp.pageview_url AS landing_page,
    COUNT(DISTINCT fpv.website_session_id) AS first_hitting_page
FROM
    first_page_views AS fpv
        LEFT JOIN
    website_pageviews AS wp ON fpv.min_page_views = wp.website_pageview_id
GROUP BY 1;
```

---

## 11. Landing Page Performance (January 2014)

### Steps to Analyze:
1. Find the first page views for relevant sessions.
2. Identify landing pages.
3. Count bounces (sessions with only one pageview).
4. Summarize total sessions and bounced sessions.

---

## 12. Landing Page Test Analysis

### Steps:
1. Identify the launch date of the new page.
2. Analyze session data for "/lander-1" and compare to other pages.
3. Summarize sessions and bounces by page.

---

### Notes:
- Temporary tables are used extensively to organize intermediate results.
- Filters are applied to focus on specific campaigns, devices, or timeframes.
- Advanced metrics such as conversion rates and bounce rates are calculated for better insights.

