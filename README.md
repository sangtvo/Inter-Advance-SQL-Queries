# Intermediate-Advance SQL Queries
> A collection of SQL queries that I created.

Table of Contents
---
1. [General Information](#general-information)
2. [Orders vs. Logs Table](#)
    * [A query to output all unique set of order_id and merchant_name that were created in January 2021, and whether the order has any history of having been declined.](#write-a-query-to-output-all-unique-set-of-order_id-and-merchant_name-that-were-created-in-january-2021-and-whether-the-order-has-any-history-of-having-been-declined)
    * [A query that provides a weekly report of January 2021, summarizing per merchant, total number of orders, count of orders where the order total is under $100, count of orders where order total is greater than or equal to $100, approval rate (%) by order count, and approval rate (%) by order amount.](#write-a-query-that-provides-a-weekly-report-of-january-2021-summarizing-per-merchant-total-number-of-orders-count-of-orders-where-the-order-total-is-under-100-count-of-orders-where-order-total-is-greater-than-or-equal-to-100-approval-rate--by-order-count-and-approval-rate--by-order-amount)
    * [A query that shows the top 5 days in January 2021 in ascending order with the highest order volume in count.](#write-a-query-that-shows-the-top-5-days-in-january-2021-in-ascending-order-with-the-highest-order-volume-in-count)
3. [Customers vs. Loans Table](#)
    * [A query to list customers that have more than one loan.](#)
    * [A query that returns each customer and the loan_id associated with the first loan each customer had created in 2019](#)


General Information
---
A collection of SQL queries that I created over-time and using PostgreSQL syntax.


Orders vs. Logs Table
---
Orders table that includes purchases and logs table includes all metadata associated with an order. 

#### A query to output all unique set of order_id and merchant_name that were created in January 2021, and whether the order has any history of having been declined.

Desired output: order_id | merchant_name | has_declined_log

Approach:
* CTE with only January 2021 data and filter by using EXTRACT year and month from order_created_at
* SELECT DISTINCT order_id, merchant_name, and has_declined_log 
* Filter has_declined_log as TRUE 

```sql
WITH t1 AS ( 
SELECT * 
FROM orders 
WHERE EXTRACT('year' FROM order_created_at)::int = 2021 AND 
 EXTRACT('month' FROM order_created_at)::int = 1 
) 

SELECT DISTINCT order_id, merchant_name, has_declined_log 
FROM t1 
WHERE has_declined_log IS TRUE 
; 
```

#### A query that provides a weekly report of January 2021, summarizing per merchant, total number of orders, count of orders where the order total is under $100, count of orders where order total is greater than or equal to $100, approval rate (%) by order count, and approval rate (%) by order amount.

Desired output: order_week | merchant_name | total_orders_count | orders_under100_count | orders_over100_count | approval_rate_by_count | approval_rate_by_sum

Approach:
* CTE with only January 2021 data and JOIN orders and logs table 
* EXTRACT year and month from order_created_at as filter and EXTRACT week 
* COUNT(*) for order count 
* SUM CASE WHEN for orders under/over 100 and approval rates 
* GROUP and ORDER BY order_week and merchant_name

```sql
WITH t2 AS ( 
SELECT *, EXTRACT('week' FROM order_created_at)::int AS order_week 
FROM orders o 
LEFT JOIN logs l 
 ON o.order_id = l.order_id 
WHERE EXTRACT('year' FROM order_created_at)::int = 2021 AND EXTRACT('month' FROM 
order_created_at)::int = 1 
) 

SELECT order_week, 
 merchant_name, 
 COUNT(*) AS total_orders_count, 
 SUM(CASE WHEN total_order_price < 100 
 THEN 1 ELSE 0 END) AS orders_under100_count, 
 SUM(CASE WHEN total_order_price >= 100 
 THEN 1 ELSE 0 END) AS orders_over100_count, 
 (SUM(CASE WHEN raw_log = 'order approved' 
 THEN 1 ELSE 0 END)::float / COUNT(*))*100 AS approval_rate_by_count, 
 (SUM(CASE WHEN raw_log = 'order approved' 
 THEN total_order_price ELSE 0 END))::float / 
 SUM(total_order_price))*100 AS approval_rate_by_sum 
FROM t2 
GROUP BY order_week, merchant_name 
ORDER BY order_week, merchant_name 
;
```

#### A query that shows the top 5 days in January 2021 in ascending order with the highest order volume in count.

Desired output: order_created_at | orders_count

Approach:
* CTE with only January 2021 data and create new column using DATE_TRUNC to pull day and ignore timestamp in order_created_at
* COUNT(*) for order volume 
* ROW_NUMBER() in DESC order by order volume 
* GROUP BY day and ORDER BY order volume in ASC 
* SELECT and filter row number <= 5 for top 5 days in ascending order with highest order volume in count

```sql
WITH t3 AS ( 
SELECT DATE_TRUNC('day', order_created_at) AS new_order_created_at, 
 COUNT(*) AS orders_count, 
 ROW_NUMBER() OVER (ORDER BY COUNT(*) DESC) AS row_num 
FROM orders 
WHERE order_created_at BETWEEN '2021-01-01' AND '2021-02-01' 
GROUP BY 1 
ORDER BY 2 ASC 
) 

SELECT new_order_created_at, orders_count 
FROM t3 
WHERE row_num <= 5 
;
```

Customers vs. Loans Table
---
Loans table with loan_id, created_at, and customer_id as a foreign key. Each customer can have more than one loan.

#### A query to list customers that have more than one loan.

Desired output: customer_id | num_of_loans

Approach:
* JOIN the two tables
* GROUP BY the id's
* COUNT that is > 1

```sql
SELECT c.customer_id
FROM customers c
LEFT JOIN loans l
	ON c.customer_id = l.customer_id
GROUP BY 1
HAVING COUNT(*) > 1
```

#### A query that returns each customer and the loan_id associated with the first loan each customer had created in 2019.

Desired output: customer_id | loan_id

Approach:
* CTE with only 2019 data
* EXTRACT year from created_at and cast as integer
* RANK() OVER to rank customer ID ordered by created_at
* Select the first rank as the earliest/first loan per customer

```sql
WITH new_data AS (
SELECT *, EXTRACT(year FROM created_at)::int AS year
FROM loans
WHERE EXTRACT(year FROM created_at)::int = 2019
)

SELECT customer_id, loan_id
FROM (
		SELECT *, RANK() OVER (PARTITION BY customer_id ORDER BY created_at ASC) AS rank
		FROM new_data
) AS a
WHERE rank = 1
```