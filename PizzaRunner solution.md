## STEP 1 DATA PREPARATION

In Customers_orders Tables

The exclusions and extras columns will need to be cleaned up before using them in your queries as it contains some NULL values and even some blank spaces.


The solution to remove null values in the table above:-

— — — -Updating customer_orders table to remove nulls
````sql
Select *
from 
Pizza_runner.customer_orders
Update Pizza_runner.customer_orders
SET exclusions =
CASE
 WHEN exclusions IS null OR exclusions LIKE ‘null’ THEN ‘ ‘
 ELSE exclusions
 END 
 
Update Pizza_runner.customer_orders
SET extras = 
CASE
 WHEN extras IS NULL or extras LIKE ‘null’ THEN ‘ ‘
 ELSE extras
 END
````


Let's take a look at the runner_orders table now
There are some known data issues with this table, I need to make sure to check the data types for each column in the schema SQL!

In pickup_time column, remove nulls and replace with blank space ' '.
In distance column, remove "km" and nulls and replace them with blank spaces.
In duration column, remove "minutes", "minute" and nulls and replace them with blank spaces.
In cancellation column, remove NULL and null, and replace with blank space ' '.


````sql
Update Pizza_runner.runner_orders
SET pickup_time =  
 CASE
 WHEN pickup_time LIKE 'null' THEN ' '
 ELSE pickup_time
 END,
distance= 
 CASE
 WHEN distance LIKE 'null' THEN ' '
 WHEN distance LIKE '%km' THEN TRIM('km' from distance)
 ELSE distance 
 END , 
duration= 
 CASE
 WHEN duration LIKE 'null' THEN ' '
 WHEN duration LIKE '%mins' THEN TRIM('mins' from duration)
 WHEN duration LIKE '%minute' THEN TRIM('minute' from duration)
 WHEN duration LIKE '%minutes' THEN TRIM('minutes' from duration)
 ELSE duration
 END ,
cancellation=
 CASE
 WHEN cancellation IS NULL or cancellation LIKE 'null' THEN ' '
 ELSE cancellation
 END
 `````
Changing the datatypes of the required columns
````sql
ALTER TABLE runner_orders_temp
ALTER COLUMN pickup_time type DATE,
ALTER COLUMN distance type FLOAT,
ALTER COLUMN duration type INT
````
FINALLY, OUR DATA PREPARATION IS DONE, LETS'S GET TO SOME REAL WORK NOW………….

# A. Pizza Metrics

1.How many pizzas were ordered?
````sql
Select count(order_id)
from Pizza_runner.Customer_orders
````
2. How many unique customer orders were made?
````sql
Select count(Distinct order_id)
from Pizza_runner.Customer_orders
````
3. How many successful orders were delivered by each runner?
````sql
SELECT 
 runner_id, 
 COUNT(order_id) AS successful_orders
FROM Pizza_runner.runner_orders
WHERE distance != 0
GROUP BY runner_id;
````
Runner 1 has 4 successful delivered orders.
Runner 2 has 3 successful delivered orders.
Runner 3 has 1 successful delivered order.

4. How many of each type of pizza was delivered?
````sql
SELECT 
  p.pizza_name, 
  COUNT(c.pizza_id) AS delivered_count
FROM pizza_runner.customer_orders AS c
JOIN pizza_runner.runner_orders AS r
  ON c.order_id = r.order_id
JOIN pizza_runner.pizza_names AS p
  ON c.pizza_id = p.pizza_id
WHERE r.distance!= 0
GROUP BY p.pizza_name
````
5. How many Vegetarian and Meatlovers were ordered by each customer?
````sql
SELECT 
 c.customer_id, 
 p.pizza_name, 
 COUNT(p.pizza_name) AS order_count
FROM pizza_runner.customer_orders AS c
JOIN pizza_runner.pizza_names AS p
 ON c.pizza_id= p.pizza_id
GROUP BY c.customer_id, p.pizza_name
ORDER BY c.customer_id;
````
6. What was the maximum number of pizzas delivered in a single order?
````sql
SELECT  
 c.order_id, 
 COUNT(c.pizza_id) AS pizza_per_order
 FROM pizza_runner.customer_orders AS c
 JOIN pizza_runner.runner_orders AS r
 ON c.order_id = r.order_id
 WHERE r.distance != 0
 GROUP BY c.order_id
 Order by pizza_per_order desc
 LIMIT 1
````
7. For each customer, how many delivered pizzas had at least 1 change, and how many had no changes?
````sql
SELECT 
 c.customer_id,
 SUM(
 CASE WHEN c.exclusions <> ‘ ‘ OR c.extras <> ‘ ‘ THEN 1
 ELSE 0
 END) AS at_least_1_change,
 SUM(
 CASE WHEN c.exclusions = ‘ ‘ AND c.extras = ‘ ‘ THEN 1 
 ELSE 0
 END) AS no_change
FROM Pizza_runner.customer_orders AS c
JOIN Pizza_runner.runner_orders AS r
 ON c.order_id = r.order_id
WHERE r.distance != 0
GROUP BY c.customer_id
ORDER BY c.customer_id;
````
9. What was the total volume of pizzas ordered for each hour of the day?
````sql
 SELECT 
 extract(HOUR from order_time) AS hour_of_day, 
 COUNT(order_id) AS pizza_count
FROM Pizza_runner.customer_orders
GROUP BY extract(HOUR from order_time);
````
10. What was the volume of orders for each day of the week?
````sql
SELECT
to_char( (DATEADD(DAY, 2, order_time),’day’), 
 COUNT(order_id) 
from pizza_runner.customer_orders
GROUP BY to_char( (DATEADD(DAY, 2, order_time),’day’)
````
DONE WITH EXPLORING THE PIZZA METRICS. LET US MOVE ON TO THE NEXT GRILL……..

# B. Runner and Customer Experience

1.How many runners signed up for each 1 week period? (i.e. week starts 2021-01-01)

Solution
````sql
SELECT 
 Extract(‘week’ from registration_date+3) AS registration_week,
 COUNT(runner_id) AS runner_signup
FROM Pizza_runner.runners
GROUP BY Extract(‘week’ from registration_date+3)
order by registration_week
````
Note:- Added 3 to registration_date to manage with ISO week numbers, since the actual week starts from 4thJan 2021(Monday) but we need to consider the week start as 1st Jan 2021.



2. What was the average time in minutes it took for each runner to arrive at the Pizza Runner HQ to pick up the order?

Solution
````sql
WITH time_taken AS
(
 SELECT 
 c.order_id, 
 c.order_time, 
 r.pickup_time,
 Date_part( ‘Minute’ , r.pickup_time- c.order_time) as pick_mins
 FROM pizza_runner.customer_orders AS c
 JOIN Pizza_runner.runner_orders AS r
 ON c.order_id = r.order_id
 WHERE r.distance != 0
 GROUP BY c.order_id, c.order_time, r.pickup_time
)
SELECT 
 AVG(pick_mins) AS avg_pickup_minutes
FROM time_taken
WHERE pick_mins > 1;
````

3. Is there any relationship between the number of pizzas and how long the order takes to prepare?

Solution
````sql
With cte as 
 (SELECT 
 c.order_id, 
 COUNT(c.pizza_id) AS pizza_count, 
 c.order_time, 
 r.pickup_time, 
 
 DATE_PART(‘minute’, r.pickup_time-c.order_time) as prep_time 

 FROM pizza_runner.customer_orders AS c
 JOIN pizza_runner.runner_orders AS r
 ON c.order_id = r.order_id
 WHERE r.distance != 0
 GROUP BY c.order_id, c.order_time, r.pickup_time )
 
 SELECT 
 pizza_count, 
 AVG(prep_time) AS avg_prep_time
FROM cte
GROUP BY pizza_count 
 Order by 1

````

Noticeable Trends

. The average Preparation time for making one pizza is 12 minutes.

. Two Pizza can be made in an average of 18 minutes.

. Making 3 pizzas will require you to have approx of 30 minutes.

4. What was the average distance traveled for each customer?

Solution
````sql
Select 
 co.customer_id ,
 AVG(ro.distance)
 
from pizza_runner.runner_orders ro
JOIN pizza_runner.customer_orders co
ON ro.order_id = co.order_id 
WHERE ro.duration<>0 
GROUP BY 1 
 Order by 1
````
5. What was the difference between the longest and shortest delivery times for all orders?

Solution
````sql
SELECT MAX(duration::NUMERIC) — MIN(duration::NUMERIC) AS delivery_time_difference
FROM Pizza_runner.runner_orders
where duration::text not like ‘ ‘;
````

6. What was the average speed for each runner for each delivery and do you notice any trend in these values?

Solution

** Average Speed = Total Distance traveled (km)/ Total Time taken(hr)
````sql
SELECT 
 ro.runner_id, 
 co.order_id, 
 ro.distance,
 co.customer_id, 
 Round((ro.distance::numeric/ro.duration * 60),2) AS AVG_SPEED
FROM pizza_runner.runner_orders AS ro
JOIN Pizza_runner.customer_orders AS co
 ON ro.order_id = co.order_id
WHERE distance != 0
GROUP BY ro.runner_id, co.order_id, ro.distance,ro.duration,co.customer_id
ORDER BY co.order_id;
````

Noticeable Trends

Runner 1 avg speed fluctuates between 37.5 km/h to 40 km/h.

Runner 3 avg speed is 40 km/h.

Runner 2 avg speed shows some disrupting fluctuation from 35kmph to 93kmph even for the same distance i.e 23.4km.

Runner 1 covers 10 km with an average speed of 60kmph and the same distance is covered by Runner 3 with an average speed of 40kmph for the same customer 104.

7. What is the successful delivery percentage for each runner?

Solution
````sql
SELECT 
 runner_id, 
 ROUND(100 * SUM(
 CASE WHEN distance != 0 THEN 1
 ELSE 0 END) / COUNT(order_id), 0) AS success_perc
FROM pizza_runner.runner_orders
GROUP BY runner_id;
````


Learning is never-ending. I look forward to learning more and enhancing my skills. Please feel free to drop suggestions with respect to my learning journey or even frame my medium stories to make them better.

PS: Since I am learning too. Please bare with me if there are any deviations from the exact solutions.

The highest in me recognizes the highest in you✊

