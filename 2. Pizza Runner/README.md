# 🍕 Case Study #2 - Pizza Runner

## ❓ Questions & Answers
## PART 0 - Data Transformation & Cleaning
`Customer_orders` table consists of indistinct and null values. Create a temporary table with modified data. Alternatively, the original table can also be modified.
```sql
CREATE TEMP TABLE customer_orders_temp AS
SELECT 
    order_id,
    customer_id,
    pizza_id,
    CASE 
        WHEN exclusions IS NULL OR exclusions LIKE 'null' THEN ''
        ELSE exclusions
    END AS exclusions,
    CASE 
        WHEN extras IS NULL OR extras LIKE 'null' THEN ''
        ELSE extras 
    END AS extras,
    order_time
FROM pizza_runner.customer_orders;
```

`runner orders` table also consists of indistinct and null values. Transform the data by creating a temporary table.
```sql
CREATE TEMP TABLE runner_orders_temp AS
SELECT 
  order_id, 
  runner_id,  
  CASE
	  WHEN pickup_time LIKE 'null' THEN ''
	  ELSE pickup_time
	  END AS pickup_time,
  CASE
	  WHEN distance LIKE 'null' THEN ''
	  WHEN distance LIKE '%km' THEN TRIM('km' from distance)
	  ELSE distance 
    END AS distance,
  CASE
	  WHEN duration LIKE 'null' THEN ''
	  WHEN duration LIKE '%mins' THEN TRIM('mins' from duration)
	  WHEN duration LIKE '%minute' THEN TRIM('minute' from duration)
	  WHEN duration LIKE '%minutes' THEN TRIM('minutes' from duration)
	  ELSE duration
	  END AS duration,
  CASE
	  WHEN cancellation IS NULL or cancellation LIKE 'null' THEN ''
	  ELSE cancellation
	  END AS cancellation
FROM pizza_runner.runner_orders;
```

Transforming the data changes the data type to varchar by default. Use `Alter` function to correct the data type.
```sql
ALTER TABLE runner_orders_temp
ALTER COLUMN pickup_time TYPE TIMESTAMP USING nullif(pickup_time, '')::timestamp,
ALTER COLUMN distance TYPE DOUBLE PRECISION USING nullif(distance, '')::DOUBLE PRECISION,
ALTER COLUMN duration TYPE INTEGER USING nullif(duration, '')::INTEGER,
ALTER COLUMN cancellation TYPE VARCHAR USING nullif(cancellation, '')::VARCHAR;
```

## PART 1 - PIZZA METRICS

1. How many pizzas were ordered?
```sql
SELECT COUNT(*) FROM customer_orders_temp;
```
| count |
| :-: |
| 14 |
***
2. How many unique customer orders were made?
```sql
SELECT COUNT(DISTINCT order_id) FROM customer_orders_temp;
```
| count |
| :-: |
| 10 |
***
3. How many successful orders were delivered by each runner?
```sql
SELECT 
  runner_id, 
  COUNT(order_id) AS successful_orders
FROM runner_orders_temp
WHERE cancellation IS NULL
GROUP BY runner_id
ORDER BY runner_id;
```
| runner_id | succesful_orders |
| :-: | :-: |
| 1 | 4 |
| 2 | 3 |
| 3 | 1 |
***
4. How many of each type of pizza was delivered?
```sql
SELECT 
    p.pizza_name,
    COUNT(c.pizza_id) AS total
FROM customer_orders_temp as c
JOIN runner_orders_temp as r
    ON c.order_id = r.order_id
JOIN pizza_names as p
    ON c.pizza_id = p.pizza_id
WHERE distance IS NOT NULL
GROUP BY p.pizza_name;
```
| pizza_name | total |
| :-: | :-: |
| Meatlovers | 9 |
| Vegetarian | 3 |
***
5. How many Vegetarian and Meatlovers were ordered by each customer?
```sql
SELECT
    c.customer_id,
    p.pizza_name,
    COUNT(c.pizza_id) as total
FROM customer_orders_temp as c
JOIN pizza_names as p
    ON c.pizza_id = p.pizza_id
GROUP BY c.customer_id, p.pizza_name
ORDER BY c.customer_id ASC;
```
| customer_id | pizza_name | total |
| :-: | :-: | :-: |
| 101 | Meatlovers | 2 |
| 101 | Vegetarian | 1 |
| 102 | Meatlovers | 2 |
| 102 | Vegetarian | 1 |
| 103 | Meatlovers | 3 |
| 103 | Vegetarian | 1 |
| 104 | Meatlovers | 3 |
| 105 | Vegetarian | 1 |
***
6. What was the maximum number of pizzas delivered in a single order?
```sql
WITH pizza_count_cte AS
(
  SELECT 
    c.order_id, 
    COUNT(c.pizza_id) AS pizza_per_order
  FROM customer_orders_temp AS c
  JOIN runner_orders_temp AS r
    ON c.order_id = r.order_id
  WHERE r.distance != 0
  GROUP BY c.order_id
)

SELECT 
  MAX(pizza_per_order) AS pizza_count
FROM pizza_count_cte;
```
| pizza_count |
| :-: |
| 3 |
***
7. For each customer, how many delivered pizzas had at least 1 change and how many had no changes?
```sql
SELECT 
  c.customer_id,
  SUM(
    CASE WHEN c.exclusions <> '' OR c.extras <> '' THEN 1
    ELSE 0
    END) AS at_least_1_change,
  SUM(
    CASE WHEN c.exclusions = '' AND c.extras = '' THEN 1 
    ELSE 0
    END) AS no_change
FROM customer_orders_temp AS c
JOIN runner_orders_temp AS r
  ON c.order_id = r.order_id
WHERE r.distance != 0
GROUP BY c.customer_id
ORDER BY c.customer_id;
```
| customer_id | at_leat_1_change | no_change |
| :-: | :-: | :-: |
| 101 | 0 | 2 |
| 102 | 0 | 3 |
| 103 | 3 | 0 |
| 104 | 2 | 1 |
| 105 | 1 | 0 |
***
8. How many pizzas were delivered that had both exclusions and extras?
```sql
SELECT  
  SUM(
    CASE WHEN exclusions IS NOT NULL AND extras IS NOT NULL THEN 1
    ELSE 0
    END) AS pizza_count_w_exclusions_extras
FROM customer_orders_temp AS c
JOIN runner_orders_temp AS r
  ON c.order_id = r.order_id
WHERE r.distance >= 1 
  AND exclusions <> '' 
  AND extras <> '';
```
| pizza_count_w_exclusions_extras |
| :-: |
| 1 |
***
9. What was the total volume of pizzas ordered for each hour of the day?
```sql
SELECT 
  date_part('hour', order_time) AS hour_of_day, 
  COUNT(order_id) AS pizza_count
FROM customer_orders_temp
GROUP BY hour_of_day
ORDER BY hour_of_day;
```
| hour_of_day | pizza_count |
| :-: | :-: |
| 11 | 1 |
| 13 | 3 |
| 18 | 3 |
| 19 | 1 |
| 21 | 3 |
| 23 | 3 |
***
10. What was the volume of orders for each day of the week?
```sql
SELECT
    to_char(order_time + INTERVAL '2D', 'Day') AS day_of_week,
    COUNT(order_id) AS pizza_count
FROM customer_orders_temp
GROUP BY day_of_week
ORDER BY day_of_week;
```
| day_of_week | pizza_count |
| :-: | :-: |
| Friday | 5 |
| Monday | 5 |
| Saturday | 3 |
| Sunday | 1 |
***
## PART 2 - Runner and Customer Experience

1. How many runners signed up for each 1 week period? (i.e. week starts 2021-01-01)
```sql
SELECT date_part('week', registration_date + interval '1W') as week_number,
COUNT(runner_id) as runner_signup
FROM runners
GROUP BY week_number
ORDER BY week_number;
```
| week_number | runner_signup |
| :-: | :-: |
| 1 | 2 |
| 2 | 1 |
| 3 | 1 |
***
2. What was the average time in minutes it took for each runner to arrive at the Pizza Runner HQ to pickup the order?
```sql
WITH time_taken_cte AS
(
  SELECT 
    r.runner_id,
    c.order_id, 
    c.order_time, 
    r.pickup_time, 
    EXTRACT(minute FROM r.pickup_time - c.order_time) AS pickup_minutes
  FROM customer_orders_temp AS c
  JOIN runner_orders_temp AS r
    ON c.order_id = r.order_id
  WHERE r.distance != 0
  GROUP BY c.order_id, c.order_time, r.pickup_time, r.runner_id
)

SELECT
  runner_id,
  AVG(pickup_minutes) AS avg_pickup_minutes
FROM time_taken_cte
GROUP BY runner_id
ORDER BY runner_id;
```
| runner_id | avg_pickup_minutes |
| :-: | :-: |
| 1 | 14.00 |
| 2 | 19.67 |
| 3 | 10.00 |
***
3. Is there any relationship between the number of pizzas and how long the order takes to prepare?
```sql
WITH prep_time_cte AS
(
  SELECT 
    c.order_id, 
    COUNT(c.order_id) AS pizza_order, 
    c.order_time, 
    r.pickup_time, 
    EXTRACT(minute FROM r.pickup_time - c.order_time) AS prep_time_minutes
  FROM customer_orders_temp AS c
  JOIN runner_orders_temp AS r
    ON c.order_id = r.order_id
  WHERE r.distance != 0
  GROUP BY c.order_id, c.order_time, r.pickup_time
)

SELECT 
  pizza_order, 
  AVG(prep_time_minutes) AS avg_prep_time_minutes
FROM prep_time_cte
WHERE prep_time_minutes > 1
GROUP BY pizza_order
ORDER BY pizza_order;
```
| pizza_order | avg_prep_time_minutes |
| :-: | :-: |
| 1 | 12 |
| 2 | 18 |
| 3 | 29 |
***
4. What was the average distance travelled for each customer?
```sql
SELECT 
  c.customer_id, 
  AVG(r.distance) AS avg_distance
FROM customer_orders_temp AS c
JOIN runner_orders_temp AS r
  ON c.order_id = r.order_id
WHERE r.duration != 0
GROUP BY c.customer_id
ORDER BY c.customer_id;
```
| customer_id | avg_distance |
| :-: | :-: |
| 101 | 20 |
| 102 | 16.73 |
| 103 | 23.39 |
| 104 | 10 |
| 105 | 25 |
***
5. What was the difference between the longest and shortest delivery times for all orders?
```sql
SELECT MAX(duration::NUMERIC) - MIN(duration::NUMERIC) AS delivery_time_difference
FROM runner_orders_temp
where duration IS NOT NULL;
```
| delivery_time_difference |
| :-: |
| 30 |
***
6. What was the average speed for each runner for each delivery and do you notice any trend for these values?
```sql
SELECT 
  r.runner_id, 
  c.customer_id, 
  c.order_id, 
  COUNT(c.order_id) AS pizza_count, 
  r.distance, (r.duration / 60) AS duration_hr , 
  ROUND((r.distance/r.duration * 60)::NUMERIC, 2) AS avg_speed
FROM runner_orders_temp AS r
JOIN customer_orders_temp AS c
  ON r.order_id = c.order_id
WHERE distance != 0
GROUP BY r.runner_id, c.customer_id, c.order_id, r.distance, r.duration
ORDER BY c.order_id;
```
| runner_id | customer_id | order_id | pizza_count | distance | duration_hr | avg_speed |
| :-: | :-: | :-: | :-: | :-: | :-: | :-: |
| 1 | 101 | 1 | 1 | 20 | 0 | 37.50 |
| 1 | 101 | 2 | 1 | 20 | 0 | 44.44 |
| 1 | 102 | 3 | 2 | 13.4 | 0 | 40.20 |
| 2 | 103 | 4 | 3 | 23.4 | 0 | 35.10 |
| 3 | 104 | 5 | 1 | 10 | 0 | 40.00 |
| 2 | 105 | 7 | 1 | 25 | 0 | 60.00 |
| 2 | 102 | 8 | 1 | 23.4 | 0 | 93.60 |
| 1 | 104 | 10 | 2 | 10 | 0 | 60.00 |
***
7. What is the successful delivery percentage for each runner?
```sql
SELECT 
  runner_id, 
  ROUND(100 * SUM(
    CASE WHEN distance IS NULL THEN 0
    ELSE 1 END) / COUNT(*), 0) AS success_perc
FROM runner_orders_temp
GROUP BY runner_id
ORDER BY runner_id;
```
| runner_id | successful_perc |
| :-: | :-: |
| 1 | 100 |
| 2 | 75 |
| 3 | 50 |
***
