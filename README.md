# Data-Bank-Case-Study
This case study was created by Danny Ma and can be found [here](https://8weeksqlchallenge.com/case-study-4/). Below is a description of the case study and my answers to the provided questions. PostgreSQL (pgAdmin 4) was used for this project.
## Introduction
There is a new innovation in the financial industry called Neo-Banks: new aged digital only banks without physical branches.

Danny thought that there should be some sort of intersection between these new age banks, cryptocurrency and the data world…so he decides to launch a new initiative - Data Bank!

Data Bank runs just like any other digital bank - but it isn’t only for banking activities, they also have the world’s most secure distributed data storage platform!

Customers are allocated cloud data storage limits which are directly linked to how much money they have in their accounts. There are a few interesting caveats that go with this business model, and this is where the Data Bank team need your help!

The management team at Data Bank want to increase their total customer base - but also need some help tracking just how much data storage their customers will need.

This case study is all about calculating metrics, growth and helping the business analyse their data in a smart way to better forecast and plan for their future developments!

## Available Data
The Data Bank team have prepared a data model for this case study as well as a few example rows from the complete dataset below to get you familiar with their tables.

![Screenshot of Entity Relationship Diagram](https://8weeksqlchallenge.com/images/case-study-4-erd.png)

### Table 1: Regions
Just like popular cryptocurrency platforms - Data Bank is also run off a network of nodes where both money and data is stored across the globe. In a traditional banking sense - you can think of these nodes as bank branches or stores that exist around the world.

This regions table contains the region_id and their respective region_name values

![regions table summary](https://user-images.githubusercontent.com/129814364/229970861-a5f64012-9f4b-4b1a-9b2b-d71b7886bef6.JPG)

### Table 2: Customer Nodes
Customers are randomly distributed across the nodes according to their region - this also specifies exactly which node contains both their cash and data.

This random distribution changes frequently to reduce the risk of hackers getting into Data Bank’s system and stealing customer’s money and data!

Below is a sample of the top 10 rows of the data_bank.customer_nodes

![customer nodes table summary](https://user-images.githubusercontent.com/129814364/229971006-f4ff8239-e5b7-4c46-b427-7ed6041ce90e.JPG)

### Table 3: Customer Transactions
This table stores all customer deposits, withdrawals and purchases made using their Data Bank debit card.

![customer transactions table summary](https://user-images.githubusercontent.com/129814364/229971128-756f660c-c481-44a8-90a2-6b0cdfc26029.JPG)

## Case Study Questions and Answers
### A. Customer Nodes Exploration
1. How many unique nodes are there on the Data Bank system?
```sql
SELECT DISTINCT node_id AS unique_nodes
FROM customer_nodes;
```
#### Answer:
![A1 Answer](https://user-images.githubusercontent.com/129814364/230783766-abfbf86f-bf6d-40f7-b90b-10e980b71174.JPG)

2. What is the number of distinct nodes and customers per region?
```sql
SELECT region_name, COUNT(DISTINCT node_id) AS node_count, COUNT(DISTINCT customer_id) AS customer_count
FROM customer_nodes
INNER JOIN regions
USING(region_id)
GROUP BY region_name;
```
#### Answer:
![A2 Answer](https://user-images.githubusercontent.com/129814364/230783948-da7b7322-84af-417f-a11d-f19f13b721c8.JPG)

3. How many days on average are customers reallocated to a different node?
Before answering this question, I validated the date columns in the customer_nodes table. The end_date column had 500 rows with a date of 9999-12-31, which is not correct. I excluded rows with this date when answering the question.
```sql
SELECT MIN(start_date), MAX(start_date), MIN(end_date), MAX(end_date)
FROM customer_nodes;
```
![A3 Validation (1)](https://user-images.githubusercontent.com/129814364/230784847-2c26b39e-63f8-4a89-83fe-fccf8e5dfd1c.JPG)
```sql
SELECT COUNT(*)
FROM customer_nodes
WHERE end_date = CAST('9999-12-31' AS date);
```
![A3 Validation (2)](https://user-images.githubusercontent.com/129814364/230784885-b3daa26b-2419-4567-8d1b-e109755a2a05.JPG)
```sql
SELECT ROUND(AVG(end_date - start_date), 2) AS avg_days
FROM customer_nodes
WHERE end_date <> CAST('9999-12-31' AS date);
```
#### Answer:
![A3 Answer](https://user-images.githubusercontent.com/129814364/230784295-8e86d535-b492-408f-9725-f40bee3900a3.JPG)

4. What is the median, 80th, and 95th percentile for this same reallocation days metric for each region?
```sql
SELECT 	region_name,
		percentile_cont(0.5) WITHIN GROUP (ORDER BY day_range ASC) AS median,
		percentile_cont(0.8) WITHIN GROUP (ORDER BY day_range ASC) AS percentile_80th,
		percentile_cont(0.95) WITHIN GROUP (ORDER BY day_range ASC) AS percentile_95th
FROM (SELECT end_date - start_date AS day_range, region_name
	  FROM customer_nodes
	  INNER JOIN regions
	  USING(region_id)
	  WHERE end_date <> CAST('9999-12-31' AS date)) AS subquery
GROUP BY region_name;
```
#### Answer:
![A4 Answer](https://user-images.githubusercontent.com/129814364/230784392-989d67cf-14a6-41e6-bfcb-3dfe57632d5b.JPG)

### B. Customer Transactions
1. What is the unique count and total amount for each transaction type?
```sql
SELECT txn_type AS transaction_type, COUNT(*) AS total_count, SUM(txn_amount) AS total_amount
FROM customer_transactions
GROUP BY txn_type;
```
#### Answer:
![B1 Answer](https://user-images.githubusercontent.com/129814364/230789097-f4a4a76d-d7b0-4437-b2d6-6d218185655b.JPG)
2. What is the average total historical deposit counts and amounts for all customers?
```sql
SELECT ROUND(AVG(deposit_count), 2) AS avg_total_deposit_count, ROUND(AVG(deposit_amt), 2) AS avg_total_deposits
FROM (SELECT customer_id, COUNT(txn_type) AS deposit_count, SUM(txn_amount) AS deposit_amt
FROM customer_transactions
WHERE txn_type = 'deposit'
GROUP BY customer_id) AS subquery;
```
#### Answer:
![B2 Answer](https://user-images.githubusercontent.com/129814364/230789791-825b88d3-5224-49ce-a3ca-e8a477619b5a.JPG)
3. For each month - how many Data Bank customers make more than 1 deposit and either 1 purchase or 1 withdrawal in a single month?
```sql
WITH counts AS (SELECT EXTRACT(MONTH FROM txn_date) AS month, customer_id,
				SUM(CASE WHEN txn_type = 'deposit' THEN 1 END) AS deposit_count,
				SUM(CASE WHEN txn_type = 'purchase' THEN 1 END) AS purchase_count,
				SUM(CASE WHEN txn_type = 'withdrawal' THEN 1 END) AS withdrawal_count
				FROM customer_transactions
				GROUP BY EXTRACT(MONTH FROM txn_date), customer_id)
SELECT month, COUNT(DISTINCT customer_id) AS customer_count
FROM counts
WHERE deposit_count > 1 AND (purchase_count = 1 OR withdrawal_count = 1)
GROUP BY month;
```
#### Answer:
![B3 Answer](https://user-images.githubusercontent.com/129814364/230789894-1cf20abc-cc56-4e20-b7d2-56b80d0cc63c.JPG)
4. What is the closing balance for each customer at the end of the month?
```sql
WITH CTE AS (SELECT EXTRACT(MONTH FROM txn_date) AS month, customer_id,
				SUM(CASE WHEN txn_type = 'deposit' THEN txn_amount ELSE -txn_amount END) AS monthly_amt
				FROM customer_transactions
				GROUP BY EXTRACT(MONTH FROM txn_date), customer_id
				ORDER BY customer_id, month)

SELECT month, customer_id, monthly_amt, SUM(monthly_amt) OVER(PARTITION BY customer_id ORDER BY customer_id ASC, month ASC) AS acct_running_total
FROM CTE
ORDER BY customer_id ASC, month ASC;
```
#### Answer:
![B4 Answer](https://user-images.githubusercontent.com/129814364/230789933-382e569c-1191-4984-94e7-f75f9ec248cd.JPG)
5. What is the percentage of customers who increase their closing balance by more than 5%?
```sql
WITH CTE AS (SELECT EXTRACT(MONTH FROM txn_date) AS month, customer_id,
				SUM(CASE WHEN txn_type = 'deposit' THEN txn_amount ELSE -txn_amount END) AS monthly_amt
				FROM customer_transactions
				GROUP BY EXTRACT(MONTH FROM txn_date), customer_id
				ORDER BY customer_id, month),

CTE2 AS (SELECT month, customer_id, monthly_amt, SUM(monthly_amt) OVER(PARTITION BY customer_id ORDER BY customer_id ASC, month ASC) AS acct_running_total 
FROM CTE
ORDER BY customer_id ASC, month ASC),

CTE3 AS (SELECT month, customer_id, FIRST_VALUE(acct_running_total) OVER (PARTITION BY customer_id ORDER BY customer_id ASC, month ASC) AS first_value,
LAST_VALUE(acct_running_total) OVER (PARTITION BY customer_id ORDER BY customer_id ASC, month ASC ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING) AS last_value
FROM CTE
INNER JOIN CTE2
USING(customer_id, month)
ORDER BY customer_id ASC, month ASC)

SELECT customer_id, first_value, last_value
FROM CTE
INNER JOIN CTE2
USING(customer_id, month)
INNER JOIN CTE3
USING(customer_id, month)
GROUP BY customer_id, first_value, last_value
HAVING last_value > 1.05 * first_value
ORDER BY customer_id;

SELECT DISTINCT customer_id
FROM customer_transactions
```
#### Answer:
![B5 Answer](https://user-images.githubusercontent.com/129814364/230789985-efeb2fca-8d70-4796-81b5-39236edc83d2.JPG)
