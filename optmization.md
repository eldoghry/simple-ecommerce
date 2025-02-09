### Retrieve the Most Recent Orders with Customer Information (1000 Orders)

#### Initial Query

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

### Optimization Techniques

#### Optimization Technique 1: Indexing Foreign Key

- Created an index on `product_category_id` in the `product` table to enhance the sequence scan, reducing it to an index-only scan.

**Optimization Query 1:**

```sql
CREATE INDEX idx_product_category_id ON product (product_category_id);
```

**Execution Time After Optimization (1):** 4.971 ms

---

#### Optimization Technique 2: Pre-Aggregation

- After applying **Optimization Technique 1**, the `product` table is aggregated using a subquery. This reduces the number of rows processed in the `JOIN` operation, improving performance.

**Optimization Query 2:**

```sql
SELECT
    c.category_name,
    p.total_products
FROM category c
LEFT JOIN (
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

**Execution Time After Optimization (2):** 2.744 ms

---

### Retrieve the total number of products in each category

select c.category_name , count(p.product_id) as total_products
from category c left join product p
on c.category_id = p.product_category_id
group by c.category_id
order by total_products desc; -- Execution Time: 5.513 ms

first i create index for category id fk on product table to speedup aggregation by converting seq scan to index only scan on product table

second move aggregation at first by rewriting query to aggregate on product table instead of aggregate join result

explain analyze select c.category_name, foo.total_product
from category c left join ( select p.product_category_id, count(\*) as total_product from product p group by p.product_category_id ) as foo
on c.category_id = foo.product_category_id
order by foo.total_product desc; -- Execution Time: 2.294 ms
