# Simple E-Commerce Database Schema

### Index

- [Simple E-Commerce Database Schema](#simple-e-commerce-database-schema)
    - [Index](#index)
    - [Overview](#overview)
    - [Database Schema](#database-schema)
    - [Database Schema](#database-schema-1)
    - [ERD (Class Diagram)](#erd-class-diagram)
    - [SQL Queries](#sql-queries)
  - [Daily Revenue Report](#daily-revenue-report)
  - [Monthly Top-Selling Products](#monthly-top-selling-products)
  - [Customers with Orders Over $500](#customers-with-orders-over-500)

### Overview

This repository contains the schema for a simple e-commerce application. It includes the structure for categories, products, customers, orders, and order details. This schema can be used to manage product listings, customer accounts, orders, and sales for a basic e-commerce platform.

---

### Database Schema

First you need to create database

        create database "simple-ecommerce";

The following tables are defined:

1.  **Category**

    This table holds product categories for the e-commerce store.

    - **_id_**: Unique identifier for the category.
    - **_name_**: The name of the category (e.g., Electronics, Clothing).
    - **_created_at_**: Timestamp of when the category was created.
    - **_updated_at_**: Timestamp of when the category was last updated.

             create table category (
                 id serial NOT NULL,
                 "name" varchar(100) NOT null unique,
                 created_at timestamptz DEFAULT now() NOT NULL,
                 updated_at timestamptz DEFAULT now() NOT NULL,
                 constraint category_pk primary key (id)
            );

2.  **Product**

    This table stores details of the products available for purchase.

    - **_id_**: Unique identifier for the product.
    - **_category_id_**: The category to which the product belongs (foreign key)
    - **_name_**: The name of the product.
    - **_description_**: A description of the product.
    - **_price_**: The price of the product.
    - **_stock_quantity_**: The number of items available in stock.
    - **_created_at_**: Timestamp of when the product was created.
    - **_updated_at_**: Timestamp of when the product was last updated.

            create table product (
                id serial not null,
                category_id int4 not null,
                "name" varchar(150) not null, -- check char
                description text,
                price numeric(10,2) not null check (price >= 0),
                stock_quantity int not null check (stock_quantity >= 0),
                created_at timestamptz DEFAULT now() NOT NULL,
                updated_at timestamptz DEFAULT now() NOT NULL,
                constraint product_pk primary key(id),
                constraint product_category_fk foreign key (category_id) references category(id) on delete cascade
            );

3.  **Customer**

    This table stores customer information.

    - **_id_**: Unique identifier for the customer.
    - **_firstname_**: Customer's first name.
    - **_lastname_**: Customer's last name.
    - **_email_**: Customer's email address (unique).
    - **_password_**: Customer's hashed password.
    - **_created_at_**: Timestamp of when the customer was created.
    - **_updated_at_**: Timestamp of when the customer was last updated.

            create table customer (
                id serial not null,
                firstname varchar(50) not null,
                lastname varchar(50) not null,
                email varchar(320) not null unique,
                password varchar not null, -- assuming bcrypt hash
                created_at timestamptz DEFAULT now() NOT NULL,
                updated_at timestamptz DEFAULT now() NOT NULL,
                constraint customer_pk primary key(id)
            );

4.  **Order**

    This table stores customer orders.

    - **_id_**: Unique identifier for the order.
    - **_customer_id_**: The customer who placed the order (foreign key).
    - **_order_date_**: The date the order was placed.
    - **_total_amount_**: The total amount of the order.
    - **_created_at_**: Timestamp of when the order was created.
    - **_updated_at_**: Timestamp of when the order was last updated.

            create table "order" (
                id serial not null,
                customer_id int4 not null,
                order_date timestamp not null,
                total_amount numeric(10,2) not null check (total_amount >= 0),
                created_at timestamptz DEFAULT now() NOT NULL,
                updated_at timestamptz DEFAULT now() NOT NULL,
                constraint order_pk primary key(id),
                constraint order_customer_fk foreign key (customer_id) references customer(id) on delete cascade
            );

5.  **Order Details**

    This table stores the details of the products ordered within each order.

    - **_id_**: Unique identifier for the order detail.
    - **_order_id_**: The order to which the detail belongs (foreign key).
    - **_product_id_**: The product being ordered (foreign key).
    - **_quantity_**: The number of units of the product in the order.
    - **_unit_price_**: The price per unit of the product at the time of order.
    - **_created_at_**: Timestamp of when the order detail was created.
    - **_updated_at_**: Timestamp of when the order detail was last updated.

            create table order_details (
                id serial not null,
                order_id int4 not null,
                product_id int4 not null,
                quantity int not null check (quantity > 0),
                unit_price decimal(12,2) not null check (unit_price >= 0),
                created_at timestamptz DEFAULT now() NOT NULL,
                updated_at timestamptz DEFAULT now() NOT NULL,
                constraint order_details_pk primary key(id),
                constraint order_details_order_fk foreign key (order_id) references "order"(id) on delete cascade,
                constraint order_details_product_fk foreign key (product_id) references product(id) on delete cascade
            );

---

### Database Schema

The schema includes SQL scripts to insert dummy data into the database tables. This data can be used for testing and development purposes.

[Dummy data scripts](./dummy.md)

### ERD (Class Diagram)

image here

---

### SQL Queries

## Daily Revenue Report

The following query generates the total revenue for a specific day:

            select
                count(id) as orders,
                sum(o.total_amount) as revenue
            from
                "order" o
            where
                o.order_date between '2024-01-01 00:00:00' and '2024-01-01 23:59:59';

## Monthly Top-Selling Products

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

## Customers with Orders Over $500

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
