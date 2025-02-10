## Overview

all following queries running on dataset consist of

- 10K Customers.
- 1K Categories.
- 15K Products.
- 5M Orders.
- 10M Order details.

You can find script to insert dummy data for each table on the following link [insert dummy data](./dummy.md)

### QUERY [5]: Retrieve the Most Recent Orders with Customer Information (1000 Orders)

**Initial Query**

```sql
SELECT
    c.category_name,
    COUNT(p.product_category_id) AS total_products
FROM category c
LEFT JOIN product p
    ON c.category_id = p.product_category_id
GROUP BY c.category_id
ORDER BY total_products DESC;
```

**Original Execution Time:** 9.019 ms

#### Optimization Technique

- Created an index on `product_category_id` in the `product` table to enhance the sequence scan, reducing it to an index-only scan
- the `product` table is aggregated using a subquery. This reduces the number of rows processed in the `JOIN` operation, improving performance.

**Optimization Query:**

```sql
-- create foreign key index on product table
CREATE INDEX idx_product_category_id ON product (product_category_id);


SELECT
    c.category_name,
    p.total_products
FROM category c
LEFT JOIN (
    -- pre aggregate product table to reduce joining rows
    SELECT
        p1.product_category_id,
        COUNT(p1.product_category_id) AS total_products
    FROM product p1
    GROUP BY p1.product_category_id
) AS p
ON c.category_id = p.product_category_id
GROUP BY c.category_id
ORDER BY total_products DESC;
```

**Execution Time After Optimization:** 2.744 ms

---

### QUERY [6]: Write SQL Query to Find the top customers by total spending

```sql
select
    c.category_name , count(p.product_id) as total_products
from
    category c
left join product p
    on c.category_id = p.product_category_id
group by c.category_id
order by total_products desc; -- Execution Time: 5.513 ms
```

**Original Execution Time:** 5.513 ms

#### Optimization Techniques

- create index for foriegn key on order_customer_id on order table to fast joining condition
- pre aggregate total_spending, sorting and limiting on order table
- use CTE for readability
- create covering index on customer table to cover customer name with customer id to convert sequential scan to index only scan

**Optimization Query:**

```sql
-- create index for foreign key on order table
create index idx_order_customer_id on orders(order_customer_id);

-- create covering index on customer table to cover name with id
create index idx_customer_id_with_name on customer(customer_id, customer_firstname, customer_lastname);

-- create sub query with cte
with TopSpenders as (
select
		o2.order_customer_id,
		sum(o2.order_total_amount) as total_spending
from
		orders o2
group by
		o2.order_customer_id
order by
		total_spending desc
limit 10)
-- use cte on join
select
	(c.customer_firstname || ' ' || c.customer_lastname) as fullname, t.total_spending
	from customer c
inner join TopSpenders t on
	t.order_customer_id = c.customer_id; --Execution Time: 867.979 ms
```

**Execution Time After Optimization:** 867.979 ms

---

### QUERY [7]: Retrieve the most recent orders with customer information with 1000 orders

**Initial Query**

```sql
select
	c.customer_firstname || ' ' || c.customer_lastname,
	o.order_id,
	o.order_date
from
	customer c
inner join orders o
	on
	o.order_customer_id = c.customer_id
order by
	o.order_date desc
limit 1000; -- Execution Time: 953.956 ms

```

**Original Execution Time:** 953.956 ms

#### Optimization Techniques

    - create covering index on customer table to cover name with id
    - as we order by order date, we create index on that column to speed up ordering operation.
    - create index for foreign key customer id on order table to speed up joining condition

**Optimization Query:**

```sql
-- create covering index on customer id with name
create index idx_customer_id_with_name on customer(customer_id, customer_firstname, customer_lastname);

-- create index on order date to speed ordering
create index idx_orders_order_date on orders (order_date);

-- create index for foreign key on order
create index idx_orders_customer on orders(order_customer_id);

```

**Execution Time After Optimization:** 3.620 ms

---

### QUERY [8]: List products that have low stock quantities of less than 10 quantities

**Initial Query**

```sql
select
	p.product_name,
	p.product_stock_quantity
from
	product p
where
	p.product_stock_quantity < 10;
-- Execution Time: 18.324 ms
```

**Original Execution Time:** 5.513 ms

#### Optimization Techniques

    - create index for product stock quantity

**Optimization Query:**

```sql
-- create index for product stock
create index idx_product_stock_quantity on product(product_stock_quantity);
```

**Execution Time After Optimization:** 0.987 ms

---

### QUERY [9]: Calculate the revenue generated from each product category

**Initial Query**

```sql
select
	c.category_name,
	sum(so.sales_quantity * so.sales_unit_price) as total_revenue
from
	category c
left join product p on
	c.category_id = p.product_category_id
left join sales_order so on
	so.sales_product_id = p.product_id
inner join orders o on
	so.sales_order_id = o.order_id
group by
	c.category_id;
--Execution Time: 5210.034 ms
```

**Original Execution Time:** 5210.034 ms

#### Optimization Techniques 1

- pre aggregate category revenue on sales order level

**Optimization Query:**

```sql
-- pre aggregate product category revenue
with products_revenue_cte as (
select
	product_category_id,
	sum(so.sales_quantity * so.sales_unit_price) as total_revenue
from
	sales_order so
inner join product p on
	so.sales_product_id = p.product_id
group by
	p.product_category_id
)
select
	c.category_name ,
	prc.total_revenue
from
	category c
left join products_revenue_cte prc on
	c.category_id = prc.product_category_id;
-- Execution Time: 1798.724 ms
```

**Execution Time After Optimization:** 1798.724 ms

#### Optimization Techniques 2

use one of the following approach

- Materialized View for Periodic reports, large datasets
- Denormalized Table + update triggers for realtime reports

```sql
-- create materialized view
create materialized view category_revenue_mv as(
select
	p.product_category_id as category_id,
	SUM(so.sales_quantity * so.sales_unit_price) as total_revenue
from
	sales_order so
inner join product p on
	so.sales_product_id = p.product_id
group by
	p.product_category_id
);


-- select data from materialized view
select
	c.category_name,
	cmv.total_revenue
from
	category c
inner join category_revenue_mv cmv
on
	c.category_id = cmv.category_id;
-- Execution Time: 0.379 ms

-- manually update materialized view
REFRESH MATERIALIZED VIEW category_revenue_mv;

-- we have to periodically update materialized view
CREATE EXTENSION IF NOT EXISTS pg_cron;
SELECT cron.schedule('0 * * * *', 'REFRESH MATERIALIZED VIEW category_revenue_mv');
```

**Execution Time After using MV:** 0.379 ms
