# Skill Assessment for Data Analyst
## Hailey Lee


#### Question 1:  Pull a list of the top 100 actors based on films they have been in all time within the movie rental database. Order the list descending by number of films

#### **Answer:**

```
/* Question 1 */

SELECT a.first_name as FirstName, a.last_name as LastName
              , COUNT( DISTINCT fa.film_id) as TotalFilms
FROM actor as a

LEFT JOIN film_actor as fa
	ON a.actor_id = fa.actor_id

GROUP BY a.first_name, a.last_name

ORDER BY 3 DESC	

LIMIT 100;

```


#### Question 2:  Pull a list of all films grouped by category.name excluding Sports and Games. Order the list descending by number of films.

#### **Answer:**

```
/* Question 2 */

SELECT c.name as CategoryName
, COUNT( DISTINCT fc.film_id) as Totalfilm
FROM category as c 
LEFT JOIN film_category as fc
	ON c.category_id = fc.category_id
WHERE c.name NOT IN ('Sports','Games')

GROUP BY 1;

```


#### Question 3: Write a query we can use to plot the total (1) number of rentals, (2) revenue from rentals, (3) number of individuals that rented movies in each of the last 12 weeks(assume the most recent week was the week with the most recent rental). Exclude your results to only include rentals from store_id 1.

#### _Comments_: 
- Lack of a table issue

"Store" table exists on the DB schema overview, however, when I run this query, it returns a error message saying, "permission denied for table store". Therefore, I had to use "Inventory" table to get store_id information

- Missing weeks on data


My query was designed to captured each of the last 12 _calendar_ weeks. However, the result is missing some weeks(including, 2020-08-10, 2020-07-20, 2020-06-08). I assumed that this is caused by either 1)This specific store(store_id = 1) had some bad weeks as business OR 2)The data is missing.

#### **Answer:**

```
WITH max_recent AS (
	SELECT MAX(rental_ts) as most_recent_rental
 	FROM rental 
)

SELECT 
DATE_TRUNC('week',r.rental_ts)::date as Week

,  COUNT(DISTINCT r.rental_id) as TotalRentals
,  SUM(p.amount) as TotalRevenue
, COUNT(DISTINCT r.customer_id) as TotalCustomers


FROM rental as r 
LEFT JOIN payment as p ON r.rental_id = p.rental_id 

JOIN inventory as i ON i.inventory_id = r.inventory_id 


-- join to get the last 12 weeks date
JOIN max_recent as mr ON r.rental_ts::date BETWEEN DATE_TRUNC('week',mr.most_recent_rental::date -84 )::date AND mr.most_recent_rental 

WHERE i.store_id = 1


GROUP BY 1
ORDER BY 1

```


#### Question 4: Write a query that returns the distinct number of active customers that rented PG-rated movies at least twice in the last 15 days of a month in both July 2020 and August 2020. Exclude any customers from Dallas.

#### **Answer:**

```
/* Question 4 */

WITH pgr_total AS (
	
	SELECT r.customer_id, COUNT( DISTINCT r.rental_id) AS number_of_rental 

	FROM rental as r 

	JOIN inventory as i ON i.inventory_id = r.inventory_id 
	JOIN film as f ON f.film_id = i.film_id 
	JOIN customer as c ON c.customer_id = r.customer_id
  	JOIN address as a ON a.address_id = c.address_id
  	JOIN city as ct ON ct.city_id = a.city_id

	WHERE f.rating = 'PG' 
		AND c.activebool = TRUE -- active customers only
  		AND ct.city NOT LIKE '%Dallas%'
		AND
( r.rental_ts::date BETWEEN '2020-07-16' AND '2020-07-31' 
OR r.rental_ts::date BETWEEN '2020-08-16' AND '2020-08-31')  -- to filter last 15 days of each month 
	
	GROUP BY r.customer_id
  )

SELECT COUNT(DISTINCT customer_id) as ActiveCustomers

FROM pgr_total 

WHERE number_of_rental >= 2

```


#### Question 5: Movie rentals are sometimes heavily concentrated on certain calendar days. This daily variance might make it more difficult to identify trends or gain any actionable insights. To counteract this, we might want to use a rolling sum/average instead. For each of the days in August 2020, write a query that shows the rolling average of rental rates over the prior seven days(assume we’re only looking at rentals that occur in August).

#### _Comments_:
- No updated time stamp for inventory

I attempted to calculate total number of inventory on the inventory table using the last update column and count the inventory id. However there was no updated time stamp within August 2020, therefore, last update is not a proper reference date. I’m unsure where else to find a proper date reference for the inventory

- Provided the Answer 2, the second solution

Knowing the limitation of inventory date reference, I tried to get the total rental with running 7-day sum




#### **Answer 1:**

```
WITH daily_rental_rate AS (
	
  SELECT i.last_update::date as Day
, COUNT(DISTINCT i.inventory_id) as total_inventory 
, COUNT ( DISTINCT r.inventory_id) as rental_inventory 

	FROM inventory i 
	LEFT JOIN rental r on r.inventory_id = i.inventory_id 
	
	WHERE i.last_update::date BETWEEN '2020-08-01' AND '2020-08-31'
	GROUP BY i.last_update::date
)

, moving_seven_days AS (
SELECT Day
, SUM(total_inventory) OVER (ORDER BY Day rows between 7 preceding and current row) as moving_sum_inventory_rental 

, SUM(rental_inventory) OVER (ORDER BY Day rows between 7 preceding and current row) as moving_sum_rental_inventory


FROM daily_rental_rate
GROUP BY Day,total_inventory, rental_inventory
) 

SELECT Day
 ,(1.00* moving_sum_rental_inventory /NULLIF(moving_sum_inventory_rental,0)) as moving_seven_day_rental_rate


FROM moving_seven_days


```

#### **Answer 2:**

```
WITH daily_rental_rate AS (
	
  SELECT r.rental_ts::date as RentalDate
, COUNT(DISTINCT i.inventory_id) as total_inventory 
, COUNT ( DISTINCT r.inventory_id) as rental_inventory 

	FROM inventory i 
	LEFT JOIN rental r on r.inventory_id = i.inventory_id 
	
	WHERE r.rental_ts::date BETWEEN '2020-08-01' AND '2020-08-31'
	GROUP BY r.rental_ts::date
)


SELECT RentalDate
, rental_inventory
, SUM(rental_inventory) OVER (ORDER BY RentalDate rows between 7 preceding and current row) as moving_sum_rental_inventory


FROM daily_rental_rate
GROUP BY RentalDate,total_inventory, rental_inventory


```


