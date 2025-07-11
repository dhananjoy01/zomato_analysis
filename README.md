<p align="center">
  <img src="zomato_analysis/docs/zomato.jpg" alt="Zomato Analytics Dashboard";"/>
</p>

# Zomato Sales Data Analytics - Complete Solution

## 📌 Table of Contents
- Project Overview
- Database Schema
- Complete Task Solutions
- Customer Analysis
- Restaurant Performance
- Delivery Operations
- Business Intelligence
- Implementation Guide
- Author & Contact

## 📊 Project Overview

This comprehensive analysis of Zomato's food delivery data provides actionable insights across 20 key business questions. The project covers:

- **Customer behavior** patterns and segmentation
- **Restaurant performance** metrics by city
- **Delivery efficiency** and rider performance
- **Sales trends** and seasonal analysis

**Database**: PostgreSQL  
**Level**: Advanced Analytics  
**Key Skills**: SQL, Data Modeling, Business Intelligence

## 🏗️ Database Schema
<p align="center">
  <img src="zomato_analysis/docs/scema.jpg" alt="Zomato Analytics Dashboard";"/>
</p>
### Entity-Relationship Diagram


### Table Details
1. **customer** - 100+ records
   - `customer_id` (PK), `customer_name`, `reg_date`

2. **restaurant** - 100+ records
   - `restaurant_id` (PK), `restaurant_name`, `city`, `opening_hours`

3. **orders** - 500+ records
   - `order_id` (PK), foreign keys, `order_item`, `order_date`, `total_amount`

4. **rider** - 150+ records
   - `rider_id` (PK), `rider_name`, `sign_up_date`

5. **delivery** - 500+ records
   - `delivery_id` (PK), foreign keys, `delivery_status`, `delivery_time`

## 🔍 Complete Task Solutions

### Customer Analysis

#### 1. Top Ordered Dishes by Specific Customer
```sql
WITH customer_orders AS (
    SELECT 
        o.order_item,
        COUNT(*) AS order_count,
        DENSE_RANK() OVER(ORDER BY COUNT(*) DESC) AS rank
    FROM orders o
    JOIN customer c ON o.customer_id = c.customer_id
    WHERE c.customer_name = 'Rahul Verma'
    AND o.order_date >= CURRENT_DATE - INTERVAL '2 years'
    GROUP BY o.order_item
)
SELECT order_item, order_count
FROM customer_orders
WHERE rank <= 2;
```

#### 8. Customer Churn Analysis
```sql
SELECT
    c.customer_id,
    c.customer_name,
    MAX(o.order_date) AS last_order_date,
    CASE
        WHEN MAX(o.order_date) < '2024-01-01' THEN 'Churned'
        ELSE 'Active'
    END AS status
FROM customer c
LEFT JOIN orders o ON c.customer_id = o.customer_id
WHERE EXTRACT(YEAR FROM o.order_date) = 2023
GROUP BY c.customer_id, c.customer_name
HAVING MAX(o.order_date) < '2024-01-01'
ORDER BY last_order_date DESC;
```

#### 16. Customer Lifetime Value
```sql
SELECT
    c.customer_id,
    c.customer_name,
    COUNT(o.order_id) AS total_orders,
    SUM(o.total_amount) AS clv,
    ROUND(SUM(o.total_amount)/COUNT(o.order_id), 2) AS avg_order_value
FROM customer c
JOIN orders o ON c.customer_id = o.customer_id
GROUP BY c.customer_id, c.customer_name
ORDER BY clv DESC
LIMIT 20;
```

### Restaurant Performance

#### 6. Restaurant Revenue Ranking
```sql
WITH restaurant_revenue AS (
    SELECT
        r.restaurant_id,
        r.restaurant_name,
        r.city,
        SUM(o.total_amount) AS total_revenue,
        DENSE_RANK() OVER(PARTITION BY r.city ORDER BY SUM(o.total_amount) DESC) AS city_rank
    FROM restaurant r
    JOIN orders o ON r.restaurant_id = o.restaurant_id
    WHERE o.order_date >= CURRENT_DATE - INTERVAL '1 year'
    GROUP BY r.restaurant_id, r.restaurant_name, r.city
)
SELECT 
    restaurant_name,
    city,
    total_revenue,
    city_rank
FROM restaurant_revenue
WHERE city_rank <= 3
ORDER BY city, city_rank;
```

#### 7. Most Popular Dish by City
```sql
WITH dish_popularity AS (
    SELECT
        r.city,
        o.order_item,
        COUNT(*) AS order_count,
        DENSE_RANK() OVER(PARTITION BY r.city ORDER BY COUNT(*) DESC) AS rank
    FROM orders o
    JOIN restaurant r ON o.restaurant_id = r.restaurant_id
    GROUP BY r.city, o.order_item
)
SELECT
    city,
    order_item AS most_popular_dish,
    order_count
FROM dish_popularity
WHERE rank = 1
ORDER BY order_count DESC;
```

#### 9. Cancellation Rate Comparison
```sql
WITH cancellation_rates AS (
    SELECT
        r.restaurant_id,
        r.restaurant_name,
        SUM(CASE WHEN EXTRACT(YEAR FROM o.order_date) = 2023 THEN 1 ELSE 0 END) AS orders_2023,
        SUM(CASE WHEN EXTRACT(YEAR FROM o.order_date) = 2023 AND d.delivery_status = 'Cancelled' THEN 1 ELSE 0 END) AS cancelled_2023,
        SUM(CASE WHEN EXTRACT(YEAR FROM o.order_date) = 2024 THEN 1 ELSE 0 END) AS orders_2024,
        SUM(CASE WHEN EXTRACT(YEAR FROM o.order_date) = 2024 AND d.delivery_status = 'Cancelled' THEN 1 ELSE 0 END) AS cancelled_2024
    FROM restaurant r
    JOIN orders o ON r.restaurant_id = o.restaurant_id
    JOIN delivery d ON o.order_id = d.order_id
    GROUP BY r.restaurant_id, r.restaurant_name
)
SELECT
    restaurant_name,
    ROUND((cancelled_2023::NUMERIC/NULLIF(orders_2023,0))*100,2) AS cancellation_rate_2023,
    ROUND((cancelled_2024::NUMERIC/NULLIF(orders_2024,0))*100,2) AS cancellation_rate_2024,
    ROUND(((cancelled_2024::NUMERIC/NULLIF(orders_2024,0)) - (cancelled_2023::NUMERIC/NULLIF(orders_2023,0)))*100,2) AS change_pct
FROM cancellation_rates
ORDER BY ABS(change_pct) DESC;
```

### Delivery Operations

#### 10. Rider Average Delivery Time
```sql
SELECT
    r.rider_id,
    r.rider_name,
    ROUND(AVG(
        EXTRACT(EPOCH FROM (d.delivery_time - o.order_time))/60
    ),2) AS avg_delivery_minutes,
    COUNT(*) AS total_deliveries
FROM rider r
JOIN delivery d ON r.rider_id = d.rider_id
JOIN orders o ON d.order_id = o.order_id
WHERE d.delivery_status = 'Delivered'
GROUP BY r.rider_id, r.rider_name
ORDER BY avg_delivery_minutes;
```

#### 14. Rider Performance Ratings
```sql
WITH rider_performance AS (
    SELECT
        r.rider_id,
        r.rider_name,
        CASE
            WHEN EXTRACT(EPOCH FROM (d.delivery_time - o.order_time))/60 <= 30 THEN '5 Stars'
            WHEN EXTRACT(EPOCH FROM (d.delivery_time - o.order_time))/60 <= 45 THEN '4 Stars'
            ELSE '3 Stars'
        END AS rating,
        COUNT(*) AS delivery_count
    FROM rider r
    JOIN delivery d ON r.rider_id = d.rider_id
    JOIN orders o ON d.order_id = o.order_id
    WHERE d.delivery_status = 'Delivered'
    GROUP BY r.rider_id, r.rider_name, rating
)
SELECT
    rider_id,
    rider_name,
    rating,
    delivery_count,
    ROUND(delivery_count::NUMERIC/SUM(delivery_count) OVER(PARTITION BY rider_id)*100,2) AS pct_of_total
FROM rider_performance
ORDER BY rider_id, rating;
```

#### 18. Rider Efficiency Analysis
```sql
SELECT
    r.rider_id,
    r.rider_name,
    COUNT(*) AS total_deliveries,
    ROUND(AVG(EXTRACT(EPOCH FROM (d.delivery_time - o.order_time))/60),2) AS avg_delivery_minutes,
    ROUND(STDDEV(EXTRACT(EPOCH FROM (d.delivery_time - o.order_time))/60),2) AS std_deviation,
    MIN(EXTRACT(EPOCH FROM (d.delivery_time - o.order_time))/60) AS fastest_delivery,
    MAX(EXTRACT(EPOCH FROM (d.delivery_time - o.order_time))/60) AS slowest_delivery
FROM rider r
JOIN delivery d ON r.rider_id = d.rider_id
JOIN orders o ON d.order_id = o.order_id
WHERE d.delivery_status = 'Delivered'
GROUP BY r.rider_id, r.rider_name
ORDER BY avg_delivery_minutes;
```

### Business Intelligence

#### 15. Peak Order Days Analysis
```sql
WITH daily_orders AS (
    SELECT
        r.restaurant_name,
        TO_CHAR(o.order_date, 'Day') AS day_of_week,
        COUNT(*) AS order_count,
        DENSE_RANK() OVER(PARTITION BY r.restaurant_name ORDER BY COUNT(*) DESC) AS rank
    FROM orders o
    JOIN restaurant r ON o.restaurant_id = r.restaurant_id
    GROUP BY r.restaurant_name, day_of_week
)
SELECT
    restaurant_name,
    day_of_week,
    order_count
FROM daily_orders
WHERE rank = 1
ORDER BY order_count DESC;
```

#### 17. Monthly Sales Trends
```sql
WITH monthly_sales AS (
    SELECT
        EXTRACT(YEAR FROM order_date) AS year,
        EXTRACT(MONTH FROM order_date) AS month,
        TO_CHAR(order_date, 'Month') AS month_name,
        COUNT(*) AS order_count,
        SUM(total_amount) AS total_revenue,
        LAG(COUNT(*), 12) OVER(ORDER BY EXTRACT(YEAR FROM order_date), EXTRACT(MONTH FROM order_date)) AS prev_year_orders,
        LAG(SUM(total_amount), 12) OVER(ORDER BY EXTRACT(YEAR FROM order_date), EXTRACT(MONTH FROM order_date)) AS prev_year_revenue
    FROM orders
    GROUP BY year, month, month_name
)
SELECT
    year,
    month_name,
    order_count,
    total_revenue,
    prev_year_orders,
    prev_year_revenue,
    ROUND((order_count - prev_year_orders)/prev_year_orders::NUMERIC*100,2) AS order_growth_pct,
    ROUND((total_revenue - prev_year_revenue)/prev_year_revenue::NUMERIC*100,2) AS revenue_growth_pct
FROM monthly_sales
ORDER BY year, month;
```

#### 19. Seasonal Demand Analysis
```sql
SELECT
    CASE
        WHEN EXTRACT(MONTH FROM order_date) IN (12,1,2) THEN 'Winter'
        WHEN EXTRACT(MONTH FROM order_date) IN (3,4,5) THEN 'Spring'
        WHEN EXTRACT(MONTH FROM order_date) IN (6,7,8) THEN 'Summer'
        ELSE 'Fall'
    END AS season,
    order_item,
    COUNT(*) AS order_count,
    ROUND(AVG(total_amount),2) AS avg_order_value
FROM orders
GROUP BY season, order_item
ORDER BY season, order_count DESC;
```

#### 20. City Revenue Rankings
```sql
SELECT
    r.city,
    COUNT(*) AS total_orders,
    SUM(o.total_amount) AS total_revenue,
    ROUND(SUM(o.total_amount)/COUNT(*),2) AS avg_order_value,
    DENSE_RANK() OVER(ORDER BY SUM(o.total_amount) DESC) AS revenue_rank
FROM restaurant r
JOIN orders o ON r.restaurant_id = o.restaurant_id
WHERE EXTRACT(YEAR FROM o.order_date) = 2023
GROUP BY r.city
ORDER BY total_revenue DESC;
```
## 👨‍💻 Author & Contact

**Dhanan Joy Chandro Roy**  
Data Analyst | SQL Specialist | Business Intelligence Developer  

📌 Professional Profiles:
- [LinkedIn](https://www.linkedin.com/in/dhananjoy01)
- [Data Science Portfolio](https://www.datascienceportfol.io/dhananjoychandro01)
- [GitHub](https://github.com/dhananjoy01)

✉️ Contact: dhananjoychandro01@gmail.com

---

> "Data is the new oil, and analytics is the combustion engine." - This project transforms raw delivery data into actionable business intelligence for Zomato's operations.
