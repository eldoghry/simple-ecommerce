### Dummy Data Insertion

The schema includes SQL scripts to insert dummy data into the database tables. This data can be used for testing and development purposes.

-- INSERT DUMMY DATA
-- Insert dummy data into the category table
INSERT INTO category (name)
VALUES
('Electronics'),
('Clothing'),
('Books'),
('Home Goods'),
('Food'),
('Watches');

-- Insert dummy data into the product table
INSERT INTO product (category*id, name, description, price, stock_quantity)
SELECT
(random() * 5 + 1)::int, -- Random category*id between 1 and 5
'Product ' || generate_series(1, 100),
'Product description',
random() * 100 + 10, -- Random price between 10 and 110
random() \* 100 -- Random stock_quantity between 0 and 100
FROM generate_series(1, 100);

-- Insert dummy data into the customer table
INSERT INTO customer (firstname, lastname, email, password)
SELECT
'FirstName' || generate_series(1, 100),
'LastName' || generate_series(1, 100),
substr(md5(random()::text), 1, 10) || '@example.com',
'$2y$10$/.y6O4t6JREU0cVK1qwY/Nu.yQicrIYUceWwwdfqbw9pG4/YriVHSa' -- Example bcrypt hash (replace with actual hashing)
FROM generate_series(1, 100);

-- Insert dummy data into the order table
INSERT INTO "order" (customer*id, order_date, total_amount)
SELECT
(random() * 100 + 1)::int, -- Random customer*id between 1 and 100
current_date - ((random() * 365)::int || ' days')::interval, -- Corrected order_date calculation
random() \* 1000 + 100 -- Random total_amount between 100 and 1100
FROM generate_series(1, 1001);

-- Insert dummy data into the order*details table
INSERT INTO order_details (order_id, product_id, quantity, unit_price)
SELECT
(random() * 1000 + 1)::int, -- Random order*id between 1 and 1000
(random() * 100 + 1)::int, -- Random product*id between 1 and 100
random() * 5 + 1, -- Random quantity between 1 and 5
(SELECT price FROM product WHERE id = (random() \_ 100 + 1)::int) -- Get unit_price from the product table
FROM generate_series(1, 5000); -- Adjust the number of order_details as needed
