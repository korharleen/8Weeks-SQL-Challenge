1. What is the total amount each customer spent at the restaurant?

Answer:-

````sql
SELECT s.customer_id,
 sum(m.price) total_amt_spent
FROM dannys_diner.sales s
JOIN dannys_diner.menu m
ON s.product_id= m.product_id
GROUP BY 1
````
Outcome:-
Customer A spends $ 76.
Customer B spends $ 74.
Customer C spends $36.

2. How many days has each customer visited the restaurant?

Answer:-

Logic:- I have counted the distinct order dates for each customer to get the desired result.

````sql
SELECT s.customer_id,
 count(distinct s.order_date) no_of_days_visited
FROM dannys_diner.sales s
JOIN dannys_diner.menu m
ON s.product_id= m.product_id
GROUP BY 1
````
Customer A visited 4 times.

Customer B visited 6 times.

Customer C visited 2 times.

3. What was the first item from the menu purchased by each customer?

Answer:-

````sql
WITH RANK AS
(SELECT 
s.customer_id,
m.product_name,
DENSE_RANK() OVER(PARTITION BY s.customer_id ORDER BY s.order_date) as ranks
FROM dannys_diner.sales s
JOIN dannys_diner .menu m
ON s.product_id = m.product_id)
SELECT r.customer_id,r.product_name
FROM RANK r
where r.ranks=1
group by 1,2
````
4. What is the most purchased item on the menu and how many times was it purchased by all customers?

Answer:-

````sql
SELECT
m.product_name,
COUNT( s.product_id) AS purchase_count
FROM dannys_diner.sales s
JOIN dannys_diner.menu m
ON s.product_id = m.product_id
GROUP BY 1
Order by purchase_count DESC 
LIMIT 1
````
5. Which item was the most popular for each customer?

Answer:-

Logic:-
I made a group by customer_id and the items and then count the purchase of each item by each customer to know their most popular dishes.

````sql
SELECT
s.customer_id,
m.product_name,
COUNT( s.product_id) AS purchase_count
FROM dannys_diner.sales s
JOIN dannys_diner.menu m
ON s.product_id = m.product_id
GROUP BY 1,2
Order by customer_id
````
We can conclude:-
Customer A is a fan of Ramen, He ordered ramen maximum times out of all his orders.
Customer B likes everything equally and ordered all items equally.
Customer C is a dedicated fan of Ramen and does not like to experiment much.

6. Which item was purchased first by the customer after they became a member?

Answer:-

Logic:-

STEP 1 -created a CTE that helped me get all required columns, and ranking order_dates by making different partitions for each customer.
````sql
WITH CTE AS
(SELECT 
s.customer_id,
m.join_date,
s.order_date,
s.product_id,
DENSE_RANK() OVER(PARTITION BY s.customer_id ORDER BY order_date) as Order_Rank
FROM dannys_diner.sales s
JOIN dannys_diner.members m
ON s.customer_id = m.customer_id 
WHERE s.order_date>=m.join_date)
````
STEP 2: created a join with menu table to get product name, to get the first order only used where clause.
````sql
SELECT c.customer_id,m.product_name
FROM CTE c
JOIN dannys_diner.menu m
ON c.product_id = m.product_id
where c.Order_rank=1
````
We concluded that Customer C is not a member yet, Customer A ordered curry after he became the member, and Customer B’s first order after becoming the member was SUSHI.

7. Which item was purchased just before the customer became a member?

Solution:-
````sql
WITH CTE AS
(SELECT
s.customer_id,m.join_date,s.order_date,s.product_id,
DENSE_RANK() OVER(PARTITION BY s.customer_id ORDER BY order_date DESC) as order_rank
FROM dannys_diner.sales s
JOIN dannys_diner.members m
ON s.customer_id=m.customer_id
WHERE s.order_date<m.join_date)
SELECT 
c.customer_id,m.product_name
From dannys_diner.menu m 
JOIN cte c
ON m.product_id = c.product_id
where c.order_rank = 1
````
8. What are the total items and amount spent for each member before they became a member?

Solution:-
````sql
WITH CTE AS
(SELECT
s.customer_id,m.join_date,s.order_date,s.product_id,
DENSE_RANK() OVER(PARTITION BY s.customer_id ORDER BY order_date DESC) as order_rank
FROM dannys_diner.sales s
JOIN dannys_diner.members m
ON s.customer_id=m.customer_id
WHERE s.order_date<m.join_date)
SELECT 
c.customer_id,
count(DISTINCT c.product_id)as total_items,
SUM(m.price) as amount_spent
From dannys_diner.menu m 
JOIN cte c
ON m.product_id = c.product_id
GROUP BY 1
````
9. If each $1 spent equates to 10 points and sushi has a 2x points multiplier — how many points would each customer have?

Solution:-
````sql
WITH POINTS_EARNED AS
(SELECT *,
CASE
WHEN product_id =1 Then price*20
Else price*10
END as points
from Dannys_diner.Menu)
SELECT 
s.customer_id,
sum(p.points) as total_points
FROM dannys_diner.Sales s
JOIN points_earned p
ON s.product_id = p.product_id
Group by 1
````
10. In the first week after a customer joins the program (including their join date) they earn 2x points on all items, not just sushi — how many points do customers A and B have at the end of January?

Solution

Step 1 Find out the customer’s valid timeline.
````sql
with required_dates as
(
 SELECT *,
 DATEADD(DAY, 6,join_date) AS valid_date,
 EOMONTH(‘2021–01–31’) AS last_date
 FROM dannys_diner.members m
)
````
Step 2:-Then, use CASE WHEN to allocate points by dates and product_name.
````sql
SELECT d.customer_id, s.order_date, d.join_date, 
 d.valid_date, d.last_date, m.product_name, m.price,
 SUM(CASE
 WHEN m.product_name = ‘sushi’ THEN 2 * 10 * m.price
 WHEN s.order_date BETWEEN d.join_date AND d.valid_date THEN 2 * 10 * m.price
 ELSE 10 * m.price
 END) AS points
FROM dates_cte AS d
JOIN dannys_diner.sales AS s
 ON d.customer_id = s.customer_id
JOIN dannys_diner.menu AS m
 ON s.product_id = m.product_id
WHERE s.order_date < d.last_date
GROUP BY d.customer_id, s.order_date, d.join_date, d.valid_date, d.last_date, m.product_name, m.price
````
Conclusion

Customer A has 1,370points, Customer B has 820 points.

# Insights from the Analysis

The most popular item is ramen, followed by curry and sushi. So doing some experiments by introducing more ramen-inclusive dishes can help in attracting more customers and make happy existing customers.

Customer C is not a member yet out of all 3 and is a ramen fan. Maybe the first trick is to help in making him a member.

Customer B is the most dedicated he ordered the maximum times.

Customer A has spent the most out of all three.

