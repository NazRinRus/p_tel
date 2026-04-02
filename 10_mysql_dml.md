### DML в MySQL
Тестирование DML в MySQL
#### Схема БД для тестов
Исходная структура таблиц:
```
CREATE TABLE categories IF NOT EXISTS (
    category_id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    title VARCHAR(32) NOT NULL
);

CREATE TABLE products IF NOT EXISTS (
    product_id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    title VARCHAR(32) NOT NULL,
    category_id VARCHAR(32) REFERENCES categories (category_id),
    price INT,
    rating INT,
    status VARCHAR(32) NOT NULL -- "В наличии" или "Распродан"
);
```
#### Задачи
1. Проанализировать и скорректировать типы данных в полях таблиц, чтобы исключить вставку невалидных с точки зрения бизнес-логики данных
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
Пояснение изменений:
- `CREATE TABLE IF NOT EXISTS` - `IF NOT EXISTS` ставятся сразу после `CREATE TABLE`;
- `category_id BIGINT UNSIGNED` - тип должен соответствовать типу первичного ключа в таблице `categories`. Добавлен `NOT NULL`  для внешних ключей;
- `price DECIMAL(10, 2)` - денежные значения обычно имеют тип `DECIMAL`;
- `rating DECIMAL(3, 1)` - позволяет хранить дробные оценки (например, 4.5). Поддерживает диапазон 0.0 – 99.9.;
- `status ENUM('В наличии', 'Распродан')` - `ENUM` ограничивает значения только указанными строками. Вставка другого значения вызовет ошибку. Добавлен `DEFAULT` ;
- Добавлен внешний ключ, ограничивающий удаление категории, если в ней есть товар, каскадное обновление.

2. Написать хранимую процедуру (или скрипт на любом языке программирования) со следующей функциональностью:
- сгенерировать 20 категорий
- после каждой вставки в таблицу категорий получить текущий `category_id` с помощью `LAST_INSERT_ID()` и сгенерировать 10000 товаров в текущей категории
- ВАЖНО: все цены товаров должны быть уникальными
- суммарно должно получиться 200 тысяч товаров (по 10 тысяч в каждой категории) 

- SQL-скрипт для накатки процедуры `vim /tmp/create_procedure_generate_data.sql`:
```
DELIMITER $$

CREATE PROCEDURE generate_data()
BEGIN
    DECLARE i INT DEFAULT 1;
    DECLARE j INT DEFAULT 1;
    DECLARE current_category_id BIGINT UNSIGNED;
    DECLARE price_counter DECIMAL(10, 2) DEFAULT 0.01;
    
    SET FOREIGN_KEY_CHECKS = 0;
    SET AUTOCOMMIT = 0;
    
    START TRANSACTION;
    
    WHILE i <= 20 DO
        INSERT INTO categories (title) 
        VALUES (CONCAT('Category ', i));
        
        SET current_category_id = LAST_INSERT_ID();
        
        SET j = 1;
        WHILE j <= 10000 DO
            INSERT INTO products (title, category_id, price, rating, status) 
            VALUES (
                CONCAT('Product ', i, '-', j),
                current_category_id,
                price_counter,
                ROUND(RAND() * 4 + 1, 1),
                IF(RAND() > 0.3, 'В наличии', 'Распродан')
            );
            
            SET price_counter = price_counter + 0.01;
            SET j = j + 1;
        END WHILE;
        
        SET i = i + 1;
    END WHILE;
    
    COMMIT;
    
    SET FOREIGN_KEY_CHECKS = 1;
    SET AUTOCOMMIT = 1;
    
    SELECT 'Генерация завершена: 20 категорий, 200000 товаров' AS result;
END$$

DELIMITER ;
```
- Накатить процедуру `mysql -h 192.168.0.140 -u root -p12345 --port=3309 --protocol=tcp otus < /tmp/create_procedure_generate_data.sql`
- Вызвать процедуру `CALL generate_data();`
```
mysql> CALL generate_data();
+-------------------------------------------------------------------------------------+
| result                                                                              |
+-------------------------------------------------------------------------------------+
| Генерация завершена: 20 категорий, 200000 товаров                                   |
+-------------------------------------------------------------------------------------+
1 row in set (8.46 sec)

Query OK, 0 rows affected (8.46 sec)

mysql> SELECT COUNT(1) FROM products;
+----------+
| COUNT(1) |
+----------+
|   200000 |
+----------+
1 row in set (0.02 sec)

mysql> SELECT COUNT(1) FROM categories;
+----------+
| COUNT(1) |
+----------+
|       20 |
+----------+
1 row in set (0.00 sec)
```
3. Написать запрос, который выведет товары в следующем порядке:
- сначала все товары в наличии, отсортированные по возрастанию цены
- затем все распроданные товары, тоже отсортированные по возрастанию цены

Запрос должен поддерживать постраничную выдачу (по 50 товаров на страницу). Продумайте наиболее эффективный способ организации такой выдачи с учётом особенностей `LIMIT` в MySQL. Все цены в таблице `products` уникальные, это поможет оптимизировать запрос.

```
SET @min_price = 0.00;

SELECT 
    product_id,
    title,
    category_id,
    price,
    rating,
    status,
    @max_price := price AS max_price_in_batch
FROM products
WHERE status = 'В наличии' AND price >= @min_price
ORDER BY price ASC LIMIT 50;

SELECT @max_price AS max_price;
+-----------+
| max_price |
+-----------+
|      0.73 |
+-----------+
1 row in set (0.00 sec)

SET @min_price = @max_price + 0.01;
```
Пояснение:
- операции `SET @min_price = 0.00;` и `SET @min_price = @max_price + 0.01;` даются на откуп приложению, на уровне бэкэнда сохраняются минимальные и максимальные значения, вставляются как переменные в запрос;
- можно использовать `LIMIT ... OFFSET ...`, но в таком случае перечитываются все записи до `OFFSET`, а позже откидываются. Например, `LIMIT 50 OFFSET 100000` находит 100 050 строк, отбрасывает 100 000, возвращает 50.

Аналогичный запрос с распроданными товарами:
```
SET @min_price = 0.00;

SELECT 
    product_id,
    title,
    category_id,
    price,
    rating,
    status,
    @max_price := price AS max_price_in_batch
FROM products
WHERE status = 'Распродан' AND price >= @min_price
ORDER BY price ASC LIMIT 50;

SELECT @max_price AS max_price;
+-----------+
| max_price |
+-----------+
|      1.44 |
+-----------+
1 row in set (0.00 sec)

SET @min_price = @max_price + 0.01;
```
