## Dummy Data Insertion

### Index

- [Dummy Data Insertion](#dummy-data-insertion)
  - [Index](#index)
  - [Overview](#overview)
  - [Insert dummy data into the category table](#insert-dummy-data-into-the-category-table)
  - [Insert dummy data into the product table](#insert-dummy-data-into-the-product-table)
  - [Insert dummy data into the customer table](#insert-dummy-data-into-the-customer-table)
  - [Insert dummy data into the order table with order details](#insert-dummy-data-into-the-order-table-with-order-details)

### Overview

The schema includes SQL scripts to insert dummy data into the database tables. This data can be used for testing and development purposes.

### Insert dummy data into the category table

        CREATE OR REPLACE FUNCTION InsertCategories(total_rows INT, batch_size INT)
        RETURNS integer AS $$
        DECLARE
            i INT = 0;
            category_name VARCHAR(100);
            category_parent_id INT;
            current_batch INT = 0;
            remaining_rows INT = total_rows;
        BEGIN
            -- LOOP TO INSERT CATEGORIES IN BATCHES
            WHILE remaining_rows > 0 LOOP
                EXIT WHEN remaining_rows <= 0 OR i >= total_rows;
                -- START NEW BATCH
                current_batch := 0;

                FOR current_batch IN 1..batch_size LOOP

                        i := i + 1;
                        remaining_rows := remaining_rows - 1;
                        category_name := 'Category ' || i;

                        -- RANDOMLY DECIDE IF CATEGORY SHOULD NOT HAVE PARENT
                        IF i = 0 or random() < 0.2 THEN
                            category_parent_id := NULL;
                        ELSE
                            -- Randomly assign a parent_category_id (only to existing categories)
                            -- category_parent_id := FLOOR(random() * (i - 1)) + 1;
                            select category_id into category_parent_id from category order by random() limit 1;
                        END IF;

                        -- INSERT ROW INTO CATEGORY TABLE
                        INSERT INTO category (category_name, category_parent_id)
                        VALUES (category_name, category_parent_id);
                END LOOP;
            END LOOP;

            return count(*) from category;
        END;
        $$ LANGUAGE plpgsql;

        SELECT InsertCategories(1000, 100);

### Insert dummy data into the product table

    create or replace function InsertProducts(total_rows INT, batch_size INT)
        returns integer as $$
        declare
            i integer = 0;
            product_category_id integer;
            product_name varchar(150);
            product_description text;
            product_price numeric(10,2);
            product_stock_quantity integer;

            current_batch INT = 0;
            remaining_rows INT = total_rows;
        begin
            while remaining_rows > 0 loop
                EXIT WHEN remaining_rows <= 0 OR i >= total_rows;

                current_batch = 0;
                for current_batch in 1 .. batch_size loop
                    i = i + 1;
                    select category_id into product_category_id from category order by RANDOM() limit 1;
                    product_name = 'product ' || i;
                    product_description = 'product description ' || i;
                    product_price = ROUND((RANDOM() * 1000)::numeric, 2) + 1; -- Random price between zero and 1000
                    product_stock_quantity = FLOOR(RANDOM() * 100) + 1; -- Random stock between zero and 100

                    -- insert product
                    INSERT INTO product (product_category_id, product_name, product_description, product_price, product_stock_quantity)
                    VALUES(product_category_id, product_name, product_description, product_price, product_stock_quantity);
                end loop;
            end loop;
            return count(*) from product;
        end;
        $$ LANGUAGE plpgsql;


        SELECT InsertProducts(15000, 1000);

### Insert dummy data into the customer table

    CREATE OR REPLACE FUNCTION InsertCustomers(total_rows INT, batch_size INT)
        RETURNS integer AS $$
        DECLARE
            i INT = 0;
            customer_firstname VARCHAR(50);
            customer_lastname VARCHAR(50);
            customer_email VARCHAR(350);
            customer_password VARCHAR;

            current_batch INT = 0;
            remaining_rows INT = total_rows;
        BEGIN
            -- LOOP TO INSERT CATEGORIES IN BATCHES
            WHILE remaining_rows > 0 LOOP
                EXIT WHEN remaining_rows <= 0 OR i >= total_rows;
                -- START NEW BATCH
                current_batch := 0;

                FOR current_batch IN 1..batch_size LOOP

                        i := i + 1;
                        remaining_rows := remaining_rows - 1;
                        customer_firstname= 'Customer ' || i;
                        customer_lastname= 'dummy-' || i;
                        customer_email= 'customer-'|| i || '@example.com';
                        customer_password= 'password';

                        -- INSERT ROW INTO CUSTOMER TABLE
                        INSERT INTO customer (customer_firstname, customer_lastname, customer_email, customer_password)
                        VALUES(customer_firstname, customer_lastname, customer_email, customer_password);
                    END LOOP;
            END LOOP;

            return count(*) from customer;
        END;
        $$ LANGUAGE plpgsql;

        SELECT InsertCustomers(10000, 500);

### Insert dummy data into the order table with order details

    create or replace function InsertOrderWithSales(total_rows INT, batch_size INT)
        returns void as $$
        declare
            i integer = 0;

            row_order_id integer;
            order_customer_id integer;
            order_date timestamp;

            sales_order_id integer;
            sales_product_id integer;
            sales_quantity integer;
            sales_unit_price numeric(10,2);

            temp_sale_total_amount numeric(10,2) = 0;
            temp_current_products_per_order integer = 1;
            temp_total_product_per_order integer = 0;

            current_batch INT = 0;
            remaining_rows INT = total_rows;
        begin
            while remaining_rows > 0 loop
                EXIT WHEN remaining_rows <= 0 OR i >= total_rows;

                current_batch = 0;
                for current_batch in 1 .. batch_size loop
                    i = i + 1;
                    temp_current_products_per_order = 1;
                    temp_sale_total_amount = 0;
                    current_batch = current_batch + 1;

                    temp_total_product_per_order = FLOOR(RANDOM() * 3) + 1; -- random number of products per order between 1 and 5

                    -- 1) create order
                    -- 1.1 get random customer id
                    select customer_id into order_customer_id from customer order by random() limit 1;

                    -- 1.2 select random timestamp between 2022 - 2025
                    SELECT TIMESTAMP '2022-01-01 00:00:00' + RANDOM() * (TIMESTAMP '2025-01-01 00:00:00' - TIMESTAMP '2022-01-01 00:00:00') into order_date;

                    INSERT INTO orders (order_customer_id, order_date, order_total_amount)
                    VALUES(order_customer_id, order_date, 1) RETURNING order_id into row_order_id;

                    -- 2) create order sales
                        for current_products_per_order in 1 .. temp_total_product_per_order loop

                            -- get sales_product_id, sales_unit_price from product table
                            select
                                product_id, product_price::numeric
                            into
                                sales_product_id, sales_unit_price
                            from product order by RANDOM() limit 1;

                            sales_order_id = row_order_id;
                            sales_quantity = FLOOR(RANDOM() * 3) + 1;

                            INSERT INTO sales_order (sales_order_id, sales_product_id, sales_quantity, sales_unit_price)
                            VALUES(sales_order_id, sales_product_id, sales_quantity, sales_unit_price);

                            -- accumulate total order amount
                            temp_sale_total_amount = temp_sale_total_amount + (sales_unit_price * sales_quantity);
                        end loop;

                    -- 3) update order total amount
                    update orders set order_total_amount = temp_sale_total_amount where order_id = row_order_id;

                end loop;
            end loop;
        end;
        $$ LANGUAGE plpgsql;
        SELECT InsertOrderWithSales(100000, 1000);
