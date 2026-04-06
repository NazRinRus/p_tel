### Индексы в MySQL
#### Структура таблиц
```
-- ============================================
-- Справочник регионов РФ
-- ============================================
CREATE TABLE regions (
    id INT PRIMARY KEY AUTO_INCREMENT COMMENT 'Уникальный идентификатор',
    name VARCHAR(100) NOT NULL COMMENT 'Название региона',
    code VARCHAR(10) UNIQUE COMMENT 'Код субъекта РФ',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP COMMENT 'Дата создания',
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT 'Дата обновления'
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='Регионы РФ';

-- ============================================
-- Учреждения
-- ============================================
CREATE TABLE institutions (
    id INT PRIMARY KEY AUTO_INCREMENT COMMENT 'Уникальный идентификатор',
    name VARCHAR(255) NOT NULL COMMENT 'Название учреждения',
    region_id INT NOT NULL COMMENT 'Регион расположения',
    location VARCHAR(50) NOT NULL COMMENT 'Временная зона относительно UTC (напр. Europe/Moscow)',
    address CHAR(100) DEFAULT NULL COMMENT 'Юридический адрес',
    contact_phone CHAR(20) DEFAULT NULL COMMENT 'Контактный телефон',
    fax CHAR(20) DEFAULT NULL COMMENT 'Факс',
    contact_email CHAR(70) DEFAULT NULL COMMENT 'Контактный email',
    is_active TINYINT(1) DEFAULT 1 COMMENT 'Активность учреждения (1=активно)',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP COMMENT 'Дата создания',
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT 'Дата обновления',
    
    FOREIGN KEY (region_id) REFERENCES regions(id) ON DELETE RESTRICT ON UPDATE CASCADE,
    INDEX idx_region_id (region_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='Учреждения';

-- ============================================
-- Терминалы связи
-- ============================================
CREATE TABLE terminals (
    id INT PRIMARY KEY AUTO_INCREMENT COMMENT 'Уникальный идентификатор',
    institution_id INT NOT NULL COMMENT 'Учреждение установки',
    serial_number VARCHAR(50) UNIQUE NOT NULL COMMENT 'Заводской номер',
    ip_address VARCHAR(15) DEFAULT NULL COMMENT 'IP-адрес терминала (IPv4)',
    model VARCHAR(100) DEFAULT NULL COMMENT 'Модель оборудования',
    firmware_version VARCHAR(20) DEFAULT NULL COMMENT 'Версия прошивки',
    status VARCHAR(20) DEFAULT 'OFFLINE' COMMENT 'Статус: ONLINE/OFFLINE',
    last_heartbeat TIMESTAMP NULL DEFAULT NULL COMMENT 'Последнее соединение',
    installed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP COMMENT 'Дата установки',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP COMMENT 'Дата создания',
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT 'Дата обновления',
    
    FOREIGN KEY (institution_id) REFERENCES institutions(id) ON DELETE RESTRICT ON UPDATE CASCADE,
    INDEX idx_institution_id (institution_id),
    INDEX idx_status (status)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='Терминалы связи';

-- ============================================
-- Общая информация об абонентах (базовая таблица)
-- ============================================
CREATE TABLE abonents_base (
    id INT PRIMARY KEY AUTO_INCREMENT COMMENT 'Уникальный идентификатор',
    abonent_type ENUM('ABONENT_A','ABONENT_B') NOT NULL COMMENT 'Тип абонента: спецконтингент или родственник',
    first_name VARCHAR(50) NOT NULL COMMENT 'Имя',
    last_name VARCHAR(50) NOT NULL COMMENT 'Фамилия',
    middle_name VARCHAR(50) DEFAULT NULL COMMENT 'Отчество',
    date_of_birth DATE NOT NULL COMMENT 'Дата рождения абонента',
    phone VARCHAR(20) NOT NULL COMMENT 'Телефон для связи',
    email VARCHAR(100) DEFAULT NULL COMMENT 'Email адрес',
    documents JSON NOT NULL COMMENT 'Паспортные данные в формате JSON',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP COMMENT 'Дата создания',
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT 'Дата обновления',
    
    INDEX idx_phone (phone),
    INDEX idx_abonent_type (abonent_type)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='Общая информация об абонентах';

-- ============================================
-- Информация о спецконтингенте (расширение abonents_base)
-- ============================================
CREATE TABLE abonents_a (
    id INT PRIMARY KEY COMMENT 'Ссылка на abonents_base.id',
    institution_id INT NOT NULL COMMENT 'Идентификатор учреждения',
    restriction_type TINYINT NOT NULL COMMENT 'Тип ограничений (2=только разрешённые номера)',
    daily_limit_active TINYINT(1) NOT NULL COMMENT 'Признак ограничения "Суточный лимит"',
    daily_limit INT NOT NULL COMMENT 'Суточный лимит в секундах',
    daily_call_duration INT NOT NULL COMMENT 'Количество выговоренных секунд за сутки',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP COMMENT 'Дата создания',
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT 'Дата обновления',

    FOREIGN KEY (id) REFERENCES abonents_base(id) ON DELETE CASCADE ON UPDATE CASCADE,
    FOREIGN KEY (institution_id) REFERENCES institutions(id) ON DELETE RESTRICT ON UPDATE CASCADE,
    INDEX idx_institution_id (institution_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='Информация о спецконтингенте';

-- ============================================
-- Информация о родственниках (расширение abonents_base)
-- ============================================
CREATE TABLE abonents_b (
    id INT PRIMARY KEY COMMENT 'Ссылка на abonents_base.id',
    uid_keycloack BINARY(16) NOT NULL COMMENT 'Идентификатор родственника из Keycloak (UUID)',
    open_calls TINYINT(1) NOT NULL COMMENT 'Разрешение опции "Всегда на связи"',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP COMMENT 'Дата создания',
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT 'Дата обновления',

    FOREIGN KEY (id) REFERENCES abonents_base(id) ON DELETE CASCADE ON UPDATE CASCADE,
    INDEX idx_uid_keycloack (uid_keycloack)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT='Информация о родственниках';
```
#### Наполнение тестовыми данными
```
START TRANSACTION;

SET FOREIGN_KEY_CHECKS = 0;
DELETE FROM terminals;
DELETE FROM institutions;
DELETE FROM regions;
SET FOREIGN_KEY_CHECKS = 1;

ALTER TABLE regions AUTO_INCREMENT = 1;
ALTER TABLE institutions AUTO_INCREMENT = 1;
ALTER TABLE terminals AUTO_INCREMENT = 1;

INSERT INTO regions (name, code) VALUES
('г. Москва', '77'),
('г. Санкт-Петербург', '78'),
('Московская область', '50'),
('Ленинградская область', '47'),
('Новосибирская область', '54'),
('Свердловская область', '66'),
('Республика Татарстан', '16'),
('Краснодарский край', '23');

INSERT INTO institutions (name, region_id, location, address, contact_phone, contact_email) VALUES
('Центральный офис связи', 1, 'Europe/Moscow', '123456, г. Москва, ул. Тверская, д. 1', '+7 (495) 100-10-10', 'moscow@org.ru'),
('Филиал «Северо-Запад»', 2, 'Europe/Moscow', '190000, г. Санкт-Петербург, Невский пр., д. 100', '+7 (812) 200-20-20', 'spb@org.ru'),
('Подразделение «Химки»', 3, 'Europe/Moscow', '141400, г. Химки, ул. Ленинградская, д. 15', '+7 (498) 300-30-30', 'khimki@org.ru'),
('Отдел Ленинградской обл.', 4, 'Europe/Moscow', '188000, г. Гатчина, пр. 25-го Октября, д. 5', '+7 (813) 400-40-40', 'lenobl@org.ru'),
('Новосибирский узел связи', 5, 'Asia/Novosibirsk', '630099, г. Новосибирск, Красный пр., д. 50', '+7 (383) 500-50-50', 'nsk@org.ru'),
('Екатеринбургский филиал', 6, 'Asia/Yekaterinburg', '620000, г. Екатеринбург, ул. Ленина, д. 25', '+7 (343) 600-60-60', 'ekb@org.ru'),
('Казанский центр обработки данных', 7, 'Europe/Moscow', '420000, г. Казань, ул. Баумана, д. 30', '+7 (843) 700-70-70', 'kazan@org.ru'),
('Краснодарский региональный узел', 8, 'Europe/Moscow', '350000, г. Краснодар, ул. Красная, д. 100', '+7 (861) 800-80-80', 'krasnodar@org.ru'),
('Московская дирекция №2', 1, 'Europe/Moscow', '127000, г. Москва, ул. Садовая, д. 10', '+7 (495) 900-90-90', 'moscow2@org.ru'),
('СПб сервисный центр', 2, 'Europe/Moscow', '191000, г. Санкт-Петербург, ул. Рубинштейна, д. 15', '+7 (812) 111-22-33', 'spb_service@org.ru'),
('Тестовый полигон МО', 3, 'Europe/Moscow', '142000, г. Домодедово, ул. Каширское шоссе, д. 5', '+7 (496) 444-55-66', 'test_mo@org.ru'),
('Удаленный пункт НСО', 5, 'Asia/Novosibirsk', '630100, г. Бердск, ул. Ленина, д. 10', '+7 (383) 777-88-99', 'berdsk@org.ru');

INSERT INTO terminals (institution_id, serial_number, ip_address, model, firmware_version, status, last_heartbeat) VALUES
(1, 'SN-2026-001', '192.168.1.10', 'Router-X100', '1.2.0', 'ONLINE', '2026-04-06 10:00:15'),
(1, 'SN-2026-002', '192.168.1.11', 'Terminal-S5', '1.2.0', 'OFFLINE', NULL),
(2, 'SN-2026-003', '10.0.1.5', 'Gateway-M1', '2.0.1', 'ONLINE', '2026-04-06 09:30:22'),
(2, 'SN-2026-004', '10.0.1.6', 'Controller-V2', '2.0.1', 'ONLINE', '2026-04-06 09:31:05'),
(3, 'SN-2026-005', '172.16.10.1', 'Modem-Pro3', '1.5.0', 'OFFLINE', NULL),
(3, 'SN-2026-006', '172.16.10.2', 'Router-X100', '1.5.1', 'ONLINE', '2026-04-06 08:00:00'),
(4, 'SN-2026-007', '192.168.20.50', 'Terminal-S5', '1.3.0', 'OFFLINE', NULL),
(4, 'SN-2026-008', '192.168.20.51', 'Gateway-M1', '1.3.1', 'ONLINE', '2026-04-06 07:45:12'),
(5, 'SN-2026-009', '10.50.0.10', 'Controller-V2', '2.1.0', 'ONLINE', '2026-04-06 07:00:00'),
(5, 'SN-2026-010', '10.50.0.11', 'Modem-Pro3', '2.1.0', 'ONLINE', '2026-04-06 07:01:33'),
(6, 'SN-2026-011', '10.66.1.5', 'Router-X100', '1.4.0', 'OFFLINE', NULL),
(6, 'SN-2026-012', '10.66.1.6', 'Terminal-S5', '1.4.1', 'ONLINE', '2026-04-06 06:30:00'),
(7, 'SN-2026-013', '172.20.5.100', 'Gateway-M1', '1.6.0', 'ONLINE', '2026-04-06 10:15:00'),
(7, 'SN-2026-014', '172.20.5.101', 'Controller-V2', '1.6.1', 'ONLINE', '2026-04-06 10:16:10'),
(8, 'SN-2026-015', '192.168.50.10', 'Modem-Pro3', '2.2.0', 'OFFLINE', NULL),
(8, 'SN-2026-016', '192.168.50.11', 'Router-X100', '2.2.1', 'ONLINE', '2026-04-06 09:45:00'),
(9, 'SN-2026-017', '10.100.0.1', 'Terminal-S5', '1.7.0', 'ONLINE', '2026-04-06 10:30:00'),
(10, 'SN-2026-018', '172.30.10.50', 'Gateway-M1', '2.3.0', 'OFFLINE', NULL),
(11, 'SN-2026-019', '10.200.5.5', 'Controller-V2', '1.8.0', 'ONLINE', '2026-04-06 11:00:00'),
(12, 'SN-2026-020', '10.50.1.15', 'Router-X100', '1.5.2', 'ONLINE', '2026-04-06 07:15:00');

COMMIT;
```
#### Индексы
Индексы на таблице `regions`
```
mysql> SHOW INDEX FROM regions;
+---------+------------+----------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+---------+------------+
| Table   | Non_unique | Key_name | Seq_in_index | Column_name | Collation | Cardinality | Sub_part | Packed | Null | Index_type | Comment | Index_comment | Visible | Expression |
+---------+------------+----------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+---------+------------+
| regions |          0 | PRIMARY  |            1 | id          | A         |           8 |     NULL |   NULL |      | BTREE      |         |               | YES     | NULL       |
| regions |          0 | code     |            1 | code        | A         |           8 |     NULL |   NULL | YES  | BTREE      |         |               | YES     | NULL       |
+---------+------------+----------+--------------+-------------+-----------+-------------+----------+--------+------+------------+---------+---------------+---------+------------+
2 rows in set (0.01 sec)
```
План запроса с использованием индекса:
```
mysql> EXPLAIN SELECT name FROM regions WHERE code = '66';
+----+-------------+---------+------------+-------+---------------+------+---------+-------+------+----------+-------+
| id | select_type | table   | partitions | type  | possible_keys | key  | key_len | ref   | rows | filtered | Extra |
+----+-------------+---------+------------+-------+---------------+------+---------+-------+------+----------+-------+
|  1 | SIMPLE      | regions | NULL       | const | code          | code | 43      | const |    1 |   100.00 | NULL  |
+----+-------------+---------+------------+-------+---------------+------+---------+-------+------+----------+-------+
1 row in set, 1 warning (0.00 sec)
```
План запроса без индекса - для неявного приведения типов, значение `code` указал без кавычек:
```
mysql> EXPLAIN SELECT name FROM regions WHERE code = 66;
+----+-------------+---------+------------+------+---------------+------+---------+------+------+----------+-------------+
| id | select_type | table   | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra       |
+----+-------------+---------+------------+------+---------------+------+---------+------+------+----------+-------------+
|  1 | SIMPLE      | regions | NULL       | ALL  | code          | NULL | NULL    | NULL |    8 |    12.50 | Using where |
+----+-------------+---------+------------+------+---------------+------+---------+------+------+----------+-------------+
1 row in set, 3 warnings (0.00 sec)
```
#### Полнотекстовый поиск
Поиск включения в текст слова:
```
EXPLAIN SELECT name FROM institutions WHERE address LIKE '%Ленинград%';
+----+-------------+--------------+------------+------+---------------+------+---------+------+------+----------+-------------+
| id | select_type | table        | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra       |
+----+-------------+--------------+------------+------+---------------+------+---------+------+------+----------+-------------+
|  1 | SIMPLE      | institutions | NULL       | ALL  | NULL          | NULL | NULL    | NULL |   12 |    11.11 | Using where |
+----+-------------+--------------+------------+------+---------------+------+---------+------+------+----------+-------------+
1 row in set, 1 warning (0.00 sec)
```
Создание индекса для поиска текста:
```
ALTER TABLE institutions ADD FULLTEXT INDEX ft_address (address);

EXPLAIN SELECT name FROM institutions WHERE MATCH(address) AGAINST('Ленинградская' IN NATURAL LANGUAGE MODE);
+----+-------------+--------------+------------+----------+---------------+------------+---------+-------+------+----------+-------------------------------+
| id | select_type | table        | partitions | type     | possible_keys | key        | key_len | ref   | rows | filtered | Extra                         |
+----+-------------+--------------+------------+----------+---------------+------------+---------+-------+------+----------+-------------------------------+
|  1 | SIMPLE      | institutions | NULL       | fulltext | ft_address    | ft_address | 0       | const |    1 |   100.00 | Using where; Ft_hints: sorted |
+----+-------------+--------------+------------+----------+---------------+------------+---------+-------+------+----------+-------------------------------+
1 row in set, 1 warning (0.00 sec)

mysql> SELECT name FROM institutions WHERE MATCH(address) AGAINST('Ленинградская' IN NATURAL LANGUAGE MODE);
+-------------------------------------------+
| name                                      |
+-------------------------------------------+
| Подразделение «Химки»                     |
+-------------------------------------------+
1 row in set (0.00 sec)
```
В таком исполнении нужно указывать конкретное слово. Любые производные от слова "Ленинград" не будут найдены, придется использовать `LIKE '%...%'`
