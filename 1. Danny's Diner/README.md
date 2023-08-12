# 👨‍🍳 Case Study #1 - Danny's Diner

## ❓ Questions & Answers

1. What is the total amount each customer spent at the restaurant?
```sql
SELECT 
	sales.customer_id,
	SUM(menu.price) as total_amount
FROM dannys_diner.sales
JOIN dannys_diner.menu
	ON sales.product_id = menu.product_id
GROUP BY sales.customer_id
ORDER BY sales.customer_id ASC;
```

| customer_id | total_amount |
| ----------- | ----------- |
| A           | 76          |
| B           | 74          |
| C           | 36          |
***
2. How many days has each customer visited the restaurant?
```sql
SELECT
	customer_id,
	COUNT(DISTINCT order_date) as visit_count
FROM dannys_diner.sales
GROUP BY customer_id;
```

| customer_id | visit_count |
| ----------- | ----------- |
| A           | 4          |
| B           | 6          |
| C           | 2          |
***
3. What was the first item from the menu purchased by each customer?
```sql
WITH cte AS (
SELECT
	sales.customer_id,
	sales.order_date,
	menu.product_name,
	DENSE_RANK() OVER(
		PARTITION BY sales.customer_id
		ORDER BY sales.order_date) AS rank
FROM dannys_diner.sales
JOIN dannys_diner.menu
	ON sales.product_id = menu.product_id
)

SELECT
	customer_id,
	product_name
FROM cte
WHERE rank = 1
GROUP BY customer_id, product_name;
```

| customer_id | product_name | 
| ----------- | ----------- |
| A           | curry        | 
| A           | sushi        | 
| B           | curry        | 
| C           | ramen        |
***
4. What is the most purchased item on the menu and how many times was it purchased by all customers?
```sql
SELECT 
	menu.product_name,
	COUNT(sales.product_id) AS total
FROM dannys_diner.sales
JOIN dannys_diner.menu
	ON sales.product_id = menu.product_id
GROUP BY menu.product_name 
ORDER BY total DESC
LIMIT 1;
```

| product_name | total | 
| ----------- | ----------- |
| ramen       | 8 |
***
5. Which item was the most popular for each customer?
```sql
WITH cte AS (
	SELECT 
		sales.customer_id, 
		menu.product_name, 
		COUNT(menu.product_id) AS order_count,
		DENSE_RANK() OVER (
  			PARTITION BY sales.customer_id 
  			ORDER BY COUNT(sales.customer_id) DESC) AS rank
	FROM dannys_diner.menu
	INNER JOIN dannys_diner.sales
	ON menu.product_id = sales.product_id
	GROUP BY sales.customer_id, menu.product_name
)

SELECT 
	customer_id, 
	product_name, 
	order_count
FROM cte
WHERE rank = 1;
```

| customer_id | product_name | order_count |
| ----------- | ---------- |------------  |
| A           | ramen        |  3   |
| B           | sushi        |  2   |
| B           | curry        |  2   |
| B           | ramen        |  2   |
| C           | ramen        |  3   |
***
6. Which item was purchased first by the customer after they became a member?
```sql
WITH cte AS (
SELECT
	members.customer_id,
	sales.product_id,
	ROW_NUMBER() OVER(
		PARTITION BY members.customer_id
		ORDER BY sales.order_date) AS row_num
FROM dannys_diner.members
INNER JOIN dannys_diner.sales
	ON members.customer_id = sales.customer_id
	AND sales.order_date > members.join_date
)

SELECT 
  customer_id, 
  product_name 
FROM cte
INNER JOIN dannys_diner.menu
  ON cte.product_id = menu.product_id
WHERE row_num = 1
ORDER BY customer_id ASC;
```

| customer_id | product_name |
| ----------- | ---------- |
| A           | ramen        |
| B           | sushi        |
***
7. Which item was purchased just before the customer became a member?
```sql
WITH purchased_prior_member AS (
  SELECT 
    members.customer_id, 
    sales.product_id,
    ROW_NUMBER() OVER (
      PARTITION BY members.customer_id
      ORDER BY sales.order_date DESC) AS rank
  FROM dannys_diner.members
  INNER JOIN dannys_diner.sales
    ON members.customer_id = sales.customer_id
    AND sales.order_date < members.join_date
)

SELECT 
  p_member.customer_id, 
  menu.product_name 
FROM purchased_prior_member AS p_member
INNER JOIN dannys_diner.menu
  ON p_member.product_id = menu.product_id
WHERE rank = 1
ORDER BY p_member.customer_id ASC;
```

| customer_id | product_name |
| ----------- | ---------- |
| A           | sushi        |
| B           | sushi        |
***
8. What is the total items and amount spent for each member before they became a member?
```sql
SELECT 
	sales.customer_id,
	COUNT(sales.product_id) AS total_items,
	SUM(menu.price) AS total_amount
FROM dannys_diner.sales
INNER JOIN dannys_diner.members
	ON sales.customer_id = members.customer_id
	AND sales.order_date < members.join_date
INNER JOIN dannys_diner.menu
	ON sales.product_id = menu.product_id
GROUP BY sales.customer_id
ORDER BY sales.customer_id ASC;
```

| customer_id | total_items | total_amount |
| ----------- | ---------- |----------  |
| A           | 2 |  25       |
| B           | 3 |  40       |
***
9. If each $1 spent equates to 10 points and sushi has a 2x points multiplier — how many points would each customer have?
```sql
WITH cte AS
(
	SELECT
	menu.product_id,
	CASE
		WHEN product_id = 1 THEN price * 20
		ELSE price * 10
		END AS points
	FROM dannys_diner.menu
)
SELECT 
  sales.customer_id, 
  SUM(cte.points) AS total_points
FROM dannys_diner.sales
INNER JOIN cte
  ON sales.product_id = cte.product_id
GROUP BY sales.customer_id
ORDER BY sales.customer_id;
```

| customer_id | total_points | 
| ----------- | ---------- |
| A           | 860 |
| B           | 940 |
| C           | 360 |
***
10. In the first week after a customer joins the program (including their join date) they earn 2x points on all items, not just sushi — how many points do customer A and B have at the end of January?
```sql
WITH dates_cte AS (
  SELECT 
    customer_id, 
      join_date, 
      join_date + 6 AS valid_date, 
      DATE_TRUNC(
        'month', '2021-01-31'::DATE)
        + interval '1 month' 
        - interval '1 day' AS last_date
  FROM dannys_diner.members
)

SELECT 
  sales.customer_id, 
  SUM(CASE
    WHEN menu.product_name = 'sushi' THEN 2 * 10 * menu.price
    WHEN sales.order_date BETWEEN dates.join_date AND dates.valid_date THEN 2 * 10 * menu.price
    ELSE 10 * menu.price END) AS points
FROM dannys_diner.sales
INNER JOIN dates_cte AS dates
  ON sales.customer_id = dates.customer_id
  AND dates.join_date <= sales.order_date
  AND sales.order_date <= dates.last_date
INNER JOIN dannys_diner.menu
  ON sales.product_id = menu.product_id
GROUP BY sales.customer_id
ORDER BY sales.customer_id;
```

| customer_id | points | 
| ----------- | ---------- |
| A           | 1020 |
| B           | 320 |
