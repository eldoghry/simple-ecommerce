## SQL Queries

### Index

- [SQL Queries](#sql-queries)
  - [Index](#index)
  - [Daily Revenue Report](#daily-revenue-report)
  - [Monthly Top-Selling Products](#monthly-top-selling-products)
  - [Customers with Orders Over $500](#customers-with-orders-over-500)
  - [suggest popular products](#suggest-popular-products)
    - [Solution 1](#solution-1)
    - [Solution 2](#solution-2)
    - [Solution 3](#solution-3)
  - [filtering product based on word](#filtering-product-based-on-word)
  - [create sales history denormalization table](#create-sales-history-denormalization-table)

### Daily Revenue Report

The following query generates the total revenue for a specific day:

            select
                count(id) as orders,
                sum(o.total_amount) as revenue
            from
                "order" o
            where
                o.order_date between '2024-01-01 00:00:00' and '2024-01-01 23:59:59';

### Monthly Top-Selling Products

This query retrieves the top ten selling products for a given month:

        select
            od.product_id,
            p."name",
            count(od.product_id) as total
        from
            order_details od
        inner join "order" o on
            o.id = od.order_id
        inner join product p on
            od.product_id = p.id
        where
            DATE_TRUNC('month', o.order_date) = '2024-01-01'::DATE
        group by
            od.product_id, p."name"
        order by
            total desc
        limit 10;

### Customers with Orders Over $500

This query retrieves customers who have placed orders totaling more than $500 in the past month:

        select
            c.id as customer_id,
            (c.firstname || ' ' || c.lastname) as fullname,
            sum(o.total_amount) as total_amount
        from
            "order" o
        inner join customer c on
            o.customer_id = c.id
        where
            o.total_amount > 500
        and
            order_date >= date_trunc('month',current_date) - interval '1 month' -- start of last month
        and
            order_date < date_trunc('month', current_date) -- start of current month
        group by
            c.id
        order by
            total_amount desc;

### suggest popular products

Can you design a query to suggest popular products in the same category for the same author, excluding the Purchsed product from the recommendations?

#### Solution 1

        select
            p.id,
            p."name",
            p.category_id
        from
            product p
        where
            p.category_id = 2
            and p.id not in (
            select
                p.id
            from
                "order" o
            inner join order_details od on
                o.id = od.order_id
            inner join product p on
                od.product_id = p.id
            where
                o.customer_id = 1001
                and p.category_id = 2);

#### Solution 2

        select
            s1.id,
            s1."name",
            s1.category_id
        from
            (
            select
                p.id,
                p."name",
                p.category_id
            from
                product p
            where
                p.category_id = 2) as s1
        left join (
            select
                p.id
            from
                "order" o
            inner join order_details od on
                o.id = od.order_id
            inner join product p on
                od.product_id = p.id
            where
                o.customer_id = 1001
                and p.category_id = 2
        ) as s2
        on
            s1.id = s2.id
        where
            s2.id is null;

#### Solution 3

using denormalization idea

first create denormalization table which contain customer id, order_id, product_id and category_id

        create table sales_history_products (
            order_id int not null,
            customer_id int not null,
            product_id int not null,
            category_id int not null,
            constraint shp_order_fk foreign key (order_id) references "order"(id) on delete cascade,
            constraint shp_customer_fk foreign key (customer_id) references customer(id) on delete cascade,
            constraint shp_product_fk foreign key (product_id) references product(id) on delete cascade,
            constraint shp_category_fk foreign key (category_id) references category(id) on delete cascade,
            constraint shp_pk primary key (order_id, customer_id, product_id, category_id)
        );

then inserting data into denormalization table

        insert
            into
            sales_history_products (order_id,
            customer_id,
            product_id,
            category_id)
        (
            select
                o.id,
                o.customer_id,
                p.id,
                c.id
            from
                "order" o
            inner join order_details od on
                o.id = od.order_id
            inner join product p on
                od.product_id = p.id
            inner join category c on
                c.id = p.category_id);

then query on the new table

        select
            *
        from
            product p
        where
            p.category_id = 2
            and p.id not in (
            select
                shp.product_id
            from
                sales_history_products shp
            where
                shp.customer_id = 1001
                and shp.category_id = 2);

### filtering product based on word

Write a SQL query to search for all products with the word "camera" in either the product name or description.

        select
            *
        from
            product p
        where
            p."name" ilike '%camera%'
            or p.description ilike '%camera%';

### create sales history denormalization table

create table fist with important data that need.

        create table sales_history (
            sale_id int not null primary key,
            customer_id int not null,
            firstname varchar(50) not null,
            lastname varchar(50) not null,
            order_date timestamp not null,
            total_amount numeric(10,
        2) not null check (total_amount >= 0),
            constraint sales_history_customer_fk foreign key (customer_id) references customer(id) on
        delete
            cascade
        );

insert data from required tables

        insert
            into
            sales_history(sale_id,
            customer_id,
            firstname,
            lastname,
            order_date,
            total_amount)
        (
            select
                "order".id,
                "order".customer_id,
                customer.firstname ,
                customer.lastname ,
                "order".order_date,
                "order".total_amount
            from
                "order",
                customer
            where
                "order".customer_id = customer.id);

create trigger to update denormalization where issue update or insert happen on the original tables

        soon ...
