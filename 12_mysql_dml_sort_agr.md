### Группировка и сортировка данных, использование групповых функций
#### группировки с ипользованием CASE, HAVING, ROLLUP, GROUPING() :
1. для магазина к предыдущему списку продуктов добавить максимальную и минимальную цену и кол-во предложений;
2. сделать выборку показывающую самый дорогой и самый дешевый товар в каждой категории;
3. сделать `rollup` с количеством товаров по категориям.

#### Структура таблиц
```
CREATE TABLE IF NOT EXISTS categories (
    category_id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    title VARCHAR(32) NOT NULL
);

CREATE TABLE IF NOT EXISTS products (
    product_id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    title VARCHAR(32) NOT NULL,
    category_id BIGINT UNSIGNED NOT NULL,
    price DECIMAL(10, 2) NOT NULL,
    rating DECIMAL(3, 1),
    status ENUM('В наличии', 'Распродан') NOT NULL DEFAULT 'В наличии',
    
    FOREIGN KEY (category_id) REFERENCES categories(category_id) ON DELETE RESTRICT ON UPDATE CASCADE
);
```
#### Решение
1. для магазина к предыдущему списку продуктов добавить максимальную и минимальную цену и кол-во предложений:
```
SELECT product_id, title, category_id, price, rating, status,
    MAX(price) OVER () AS max_price,
    MIN(price) OVER () AS min_price
FROM products
ORDER BY price ASC LIMIT 5;

+------------+-------------+-------------+-------+--------+--------------------+-----------+-----------+
| product_id | title       | category_id | price | rating | status             | max_price | min_price |
+------------+-------------+-------------+-------+--------+--------------------+-----------+-----------+
|          1 | Product 1-1 |           1 |  0.01 |    3.8 | Распродан          |   2000.00 |      0.01 |
|          2 | Product 1-2 |           1 |  0.02 |    2.1 | Распродан          |   2000.00 |      0.01 |
|          3 | Product 1-3 |           1 |  0.03 |    4.1 | В наличии          |   2000.00 |      0.01 |
|          4 | Product 1-4 |           1 |  0.04 |    1.9 | В наличии          |   2000.00 |      0.01 |
|          5 | Product 1-5 |           1 |  0.05 |    2.4 | В наличии          |   2000.00 |      0.01 |
+------------+-------------+-------------+-------+--------+--------------------+-----------+-----------+
5 rows in set (0.57 sec)
```
2. сделать выборку показывающую самый дорогой и самый дешевый товар в каждой категории:
```
WITH data_1 AS (
    SELECT 
        c.category_id,
        c.title AS category_title,
        p.title AS product_title,
        p.price,
        ROW_NUMBER() OVER (PARTITION BY p.category_id ORDER BY p.price ASC)  AS data_min,
        ROW_NUMBER() OVER (PARTITION BY p.category_id ORDER BY p.price DESC) AS data_max
    FROM categories c
    JOIN products p ON c.category_id = p.category_id
)
SELECT 
    category_id,
    category_title,
    MAX(CASE WHEN data_min = 1 THEN product_title END) AS cheapest_product,
    MAX(CASE WHEN data_min = 1 THEN price END)         AS cheapest_price,
    MAX(CASE WHEN data_max = 1 THEN product_title END) AS most_expensive_product,
    MAX(CASE WHEN data_max = 1 THEN price END)         AS most_expensive_price
FROM data_1
GROUP BY category_id, category_title
ORDER BY category_id;

+-------------+----------------+------------------+----------------+------------------------+----------------------+
| category_id | category_title | cheapest_product | cheapest_price | most_expensive_product | most_expensive_price |
+-------------+----------------+------------------+----------------+------------------------+----------------------+
|           1 | Category 1     | Product 1-1      |           0.01 | Product 1-10000        |               100.00 |
|           2 | Category 2     | Product 2-1      |         100.01 | Product 2-10000        |               200.00 |
|           3 | Category 3     | Product 3-1      |         200.01 | Product 3-10000        |               300.00 |
|           4 | Category 4     | Product 4-1      |         300.01 | Product 4-10000        |               400.00 |
|           5 | Category 5     | Product 5-1      |         400.01 | Product 5-10000        |               500.00 |
|           6 | Category 6     | Product 6-1      |         500.01 | Product 6-10000        |               600.00 |
|           7 | Category 7     | Product 7-1      |         600.01 | Product 7-10000        |               700.00 |
|           8 | Category 8     | Product 8-1      |         700.01 | Product 8-10000        |               800.00 |
|           9 | Category 9     | Product 9-1      |         800.01 | Product 9-10000        |               900.00 |
|          10 | Category 10    | Product 10-1     |         900.01 | Product 10-10000       |              1000.00 |
|          11 | Category 11    | Product 11-1     |        1000.01 | Product 11-10000       |              1100.00 |
|          12 | Category 12    | Product 12-1     |        1100.01 | Product 12-10000       |              1200.00 |
|          13 | Category 13    | Product 13-1     |        1200.01 | Product 13-10000       |              1300.00 |
|          14 | Category 14    | Product 14-1     |        1300.01 | Product 14-10000       |              1400.00 |
|          15 | Category 15    | Product 15-1     |        1400.01 | Product 15-10000       |              1500.00 |
|          16 | Category 16    | Product 16-1     |        1500.01 | Product 16-10000       |              1600.00 |
|          17 | Category 17    | Product 17-1     |        1600.01 | Product 17-10000       |              1700.00 |
|          18 | Category 18    | Product 18-1     |        1700.01 | Product 18-10000       |              1800.00 |
|          19 | Category 19    | Product 19-1     |        1800.01 | Product 19-10000       |              1900.00 |
|          20 | Category 20    | Product 20-1     |        1900.01 | Product 20-10000       |              2000.00 |
+-------------+----------------+------------------+----------------+------------------------+----------------------+
20 rows in set (0.38 sec)
```
3. сделать `rollup` с количеством товаров по категориям:
```
SELECT 
    CASE WHEN GROUPING(c.category_id) = 1 THEN 'ИТОГО' ELSE c.title END AS category_title,
    COUNT(p.product_id) AS product_count
FROM categories c
LEFT JOIN products p ON c.category_id = p.category_id
GROUP BY c.category_id, c.title WITH ROLLUP
ORDER BY GROUPING(c.category_id), c.category_id;

+----------------+---------------+
| category_title | product_count |
+----------------+---------------+
| Category 1     |         10000 |
| NULL           |         10000 |
| Category 2     |         10000 |
| NULL           |         10000 |
| Category 3     |         10000 |
| NULL           |         10000 |
| Category 4     |         10000 |
| NULL           |         10000 |
| Category 5     |         10000 |
| NULL           |         10000 |
| Category 6     |         10000 |
| NULL           |         10000 |
| Category 7     |         10000 |
| NULL           |         10000 |
| Category 8     |         10000 |
| NULL           |         10000 |
| Category 9     |         10000 |
| NULL           |         10000 |
| Category 10    |         10000 |
| NULL           |         10000 |
| Category 11    |         10000 |
| NULL           |         10000 |
| Category 12    |         10000 |
| NULL           |         10000 |
| Category 13    |         10000 |
| NULL           |         10000 |
| Category 14    |         10000 |
| NULL           |         10000 |
| Category 15    |         10000 |
| NULL           |         10000 |
| Category 16    |         10000 |
| NULL           |         10000 |
| Category 17    |         10000 |
| NULL           |         10000 |
| Category 18    |         10000 |
| NULL           |         10000 |
| Category 19    |         10000 |
| NULL           |         10000 |
| Category 20    |         10000 |
| NULL           |         10000 |
| ИТОГО          |        200000 |
+----------------+---------------+
41 rows in set (0.15 sec)
```
