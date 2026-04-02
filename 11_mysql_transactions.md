### Пример миграции данных из одной БД в другую
- Требуется перенести абонентов из БД сайта (joomla) `zonatelecom` в БД `a2billing`, по большей части интересуют uuid-токены. Новые абоненты уже попадают в обе БД, реализованно через API, требуется перенести старых абонентов, с изменением структуры строк и типов данных.
- Обе БД живут на PXC MySQL 5.7 
#### Сделать бекап таблиц
В таком же порядке, т.к. там связи и зависимости:
- abonents_base
- cc_card
- abonent_b_cards
```
sudo mysqldump --set-gtid-purged=OFF \
  --single-transaction \
  --routines \
  --triggers \
  --events \
  --hex-blob \
  --quick \
  --max_allowed_packet=512M \
  --add-drop-table \
  --result-file=/data/backup/a2billing_abonents_base_cc_card_abonent_b_cards.dump \
  a2billing abonents_base cc_card abonent_b_cards

r.nazarov@galera-node03:/tmp/INFR-4969$ sudo ls -lah /data/backup/a2billing_abonents_base_cc_card_abonent_b_cards.dump
-rw-r--r-- 1 root root 3,5G мар 26 08:33 /data/backup/a2billing_abonents_base_cc_card_abonent_b_cards.dump
```
#### Зачистка таблицы abonents_b
- Зачистить от новых записей, т.к. из БД источника эти же записи подтянутся;
- `abonents_base` - общие поля для всех абонентов;
- `abonents_b` - поля присущие только для абонентов категории Б;
1. Структура таблиц, связи:
```
mysql> SELECT COUNT(1) FROM abonents_b;
+----------+
| COUNT(1) |
+----------+
|  2096281 |
+----------+
1 row in set (0,37 sec)

mysql> SELECT COUNT(1) FROM abonents_base;
+----------+
| COUNT(1) |
+----------+
|  3151351 |
+----------+
1 row in set (0,54 sec)

mysql> SHOW CREATE TABLE abonents_b\G
*************************** 1. row ***************************
       Table: abonents_b
Create Table: CREATE TABLE `abonents_b` (
  `id` bigint(20) NOT NULL,
  `uid_keycloack` binary(16) NOT NULL,
  `open_calls` tinyint(1) NOT NULL DEFAULT '0',
  `incoming_calls_from_any_card` tinyint(1) NOT NULL DEFAULT '0',
  PRIMARY KEY (`id`),
  UNIQUE KEY `unique_uid_keycloack` (`uid_keycloack`),
  CONSTRAINT `fk_abonents_b_base` FOREIGN KEY (`id`) REFERENCES `abonents_base` (`id`) ON DELETE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin
1 row in set (0,01 sec)

mysql> SHOW CREATE TABLE abonents_base\G
*************************** 1. row ***************************
       Table: abonents_base
Create Table: CREATE TABLE `abonents_base` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT,
  `abonent_type` enum('ABONENT_A','ABONENT_B') COLLATE utf8_bin NOT NULL,
  `first_name` varchar(50) COLLATE utf8_bin NOT NULL COMMENT 'Имя',
  `last_name` varchar(50) COLLATE utf8_bin NOT NULL COMMENT 'Фамилия',
  `middle_name` varchar(50) COLLATE utf8_bin NOT NULL COMMENT 'Отчество',
  `phone_number` varchar(20) COLLATE utf8_bin NOT NULL COMMENT 'Номер телефона',
  `sync_date` timestamp NULL DEFAULT NULL COMMENT 'Дата последней синхронизации',
  PRIMARY KEY (`id`),
  KEY `phone_number` (`phone_number`)
) ENGINE=InnoDB AUTO_INCREMENT=18142019 DEFAULT CHARSET=utf8 COLLATE=utf8_bin
1 row in set (0,00 sec)
```
`abonents_b` зависит от `abonents_base`, по сути это горизонтально разрезанная таблица, каскад удалит из `abonents_b` если я удалю зависимую строку в `abonents_base`
2. Сырой запрос удаления (не для прода):
```
DELETE base FROM abonents_base AS base WHERE base.id IN (SELECT id FROM (SELECT id FROM abonents_b LIMIT 10) AS tmp_ids);
```
3. Скрипт с процедурой: `vim /tmp/INFR-4969/create_procedure_delete_abonents_base.sql`
```
tee /tmp/INFR-4969/create_procedure_delete_abonents_base.sql << 'EOF'
DELIMITER //
CREATE PROCEDURE `delete_abonents_base`(
    IN batch INT,
    IN sleep_t DECIMAL(5,2)
)
BEGIN
DECLARE rows_i INT DEFAULT 1;
DECLARE rows_count INT DEFAULT 0;
DECLARE iterations INT DEFAULT 0;

SET batch = IFNULL(batch, 1000);
SET sleep_t = IFNULL(sleep_t, 1);
SET rows_i = 1;
SET rows_count = 0;
SET iterations = 0;

WHILE rows_i > 0 DO 
    START TRANSACTION;
        DELETE base FROM abonents_base AS base WHERE base.id IN (SELECT id FROM (SELECT id FROM abonents_b LIMIT batch) AS tmp_ids);
        SET rows_i = ROW_COUNT();
    COMMIT;
    SET iterations = iterations + 1;
    SET rows_count = rows_count + rows_i;
    SELECT CONCAT('Итерация: ', iterations, ', Удалено строк: ', rows_i, ', Всего удалено строк: ', rows_count) AS 'Status';
    SELECT CONCAT(sleep_t, ' seconds...') AS 'Sleep';
    DO SLEEP(sleep_t);
END WHILE;
END
//
DELIMITER ;
EOF
```
- Накатить процедуру: `sudo mysql a2billing < /tmp/INFR-4969/create_procedure_delete_abonents_base.sql`
- Вызвать процедуру: `CALL delete_abonents_base(1000, 1);`

#### Экспорт данных из zonatelecom
1. Выгрузка данных, удовлетворяющих условию, из БД `zonatelecom` со slave-хоста источника, находясь на целевом хосте:

Скрипт выгрузки данных `vim /tmp/INFR-4969/export_users.sh`
```
#!/bin/bash

DB_HOST="хост_источник"
DB_PORT="3306"
DB_NAME="zonatelecom"
DB_USER="rnazarov"
DB_PASS="***"
OUTPUT_FILE="/tmp/INFR-4969/ztlc_users.csv"
LOG_FILE="/tmp/INFR-4969/export_users.log"

mkdir -p "$(dirname "$OUTPUT_FILE")"
rm -f "$OUTPUT_FILE"

SQL_QUERY="SELECT
	u.id, u.name, u.username, 
	MAX(CASE WHEN zup.profile_key = 'profile71.uuid' THEN zup.profile_value END) as uid,
	MAX(CASE WHEN zup.profile_key = 'profile71.personal_account' THEN zup.profile_value END) as account,
	MAX(CASE WHEN zup.profile_key = 'profile71.user_open_calls' THEN zup.profile_value END) as open_calls,
	MAX(CASE WHEN zup.profile_key = 'profile71.user_open_calls_all' THEN zup.profile_value END) as incoming_calls_from_any_card
FROM ztlc_users u
LEFT JOIN ztlc_user_profiles zup ON u.id = zup.user_id 
	AND zup.profile_key IN ('profile71.uuid', 'profile71.personal_account', 'profile71.user_open_calls', 'profile71.user_open_calls_all')
WHERE u.block = 0
GROUP BY u.id, u.name, u.username;"

echo "[$(date)] Начало выгрузки..." | tee -a "$LOG_FILE"

# Параметры выгрузки:
# -N: без заголовков столбцов (имен полей)
# -B: пакетный режим (табуляция вместо пробелов)
# Вручную добавлю заголовок и сконвертирую табуляцию в символ "|"
# Делиметр обязательно "|", а не "," т.к. каким-то образом есть записи типа "Фамилия,И,О"
{
    echo "id|name|username|uid|account|open_calls|incoming_calls_from_any_card"
    mysql -h "$DB_HOST" -P "$DB_PORT" -u "$DB_USER" -p"$DB_PASS" "$DB_NAME" -N -B -e "$SQL_QUERY" | sed 's/\t/|/g'
} > "$OUTPUT_FILE" 2>> "$LOG_FILE"

if [ $? -eq 0 ]; then
    FILE_SIZE=$(du -h "$OUTPUT_FILE" | cut -f1)
    ROW_COUNT=$(wc -l < "$OUTPUT_FILE")
    echo "[$(date)] Успешно! Файл: $OUTPUT_FILE, Размер: $FILE_SIZE, Строк: $ROW_COUNT" | tee -a "$LOG_FILE"
    exit 0
else
    echo "[$(date)] Ошибка при выгрузке!" | tee -a "$LOG_FILE"
    exit 1
fi
```
Результат:
```
r.nazarov@galera-node03:/tmp/INFR-4969$ sudo bash export_users.sh
[Ср мар 18 09:26:05 MSK 2026] Начало выгрузки...
[Ср мар 18 09:43:56 MSK 2026] Успешно! Файл: /tmp/INFR-4969/ztlc_users.csv, Размер: 282M, Строк: 2951055
```
2. Потребуется перенести файл `ztlc_users.csv` в директорию `SHOW VARIABLES LIKE 'secure_file_priv';`
```
mysql> SHOW VARIABLES LIKE 'secure_file_priv';
+------------------+-----------------------+
| Variable_name    | Value                 |
+------------------+-----------------------+
| secure_file_priv | /var/lib/mysql-files/ |
+------------------+-----------------------+
1 row in set (0.01 sec)
```
Копирование файла: `sudo cp /tmp/INFR-4969/ztlc_users.csv /var/lib/mysql-files/`
3. Загрузка данных с файла `/tmp/INFR-4969/ztlc_users.csv` в во временную таблицу `ztlc_users_tmp` БД `a2billing`:
```
SET NAMES utf8mb4;
SET CHARACTER SET utf8mb4;
SET character_set_connection = utf8mb4;

CREATE TABLE IF NOT EXISTS ztlc_users_tmp (
  id INT NOT NULL,
  uid binary(16) NOT NULL,
  first_name varchar(50) COLLATE utf8_bin NOT NULL COMMENT 'Имя',
  last_name varchar(50) COLLATE utf8_bin NOT NULL COMMENT 'Фамилия',
  middle_name varchar(50) COLLATE utf8_bin NOT NULL COMMENT 'Отчество',
  phone_number varchar(20) COLLATE utf8_bin NOT NULL COMMENT 'Номер телефона',
  cc_card_useralias VARCHAR(50) NOT NULL,
  open_calls tinyint(1) NOT NULL,
  incoming_calls_from_any_card tinyint(1) NOT NULL,
  PRIMARY KEY (id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

LOAD DATA INFILE '/var/lib/mysql-files/ztlc_users.csv'
INTO TABLE ztlc_users_tmp
CHARACTER SET utf8mb4
FIELDS TERMINATED BY '|' 
ENCLOSED BY '"' 
LINES TERMINATED BY '\n'
IGNORE 1 LINES
(
    @v_id, 
    @v_fullname, 
    @v_phone, 
    @v_uid_str, 
    @v_account, 
    @v_open, 
    @v_incoming
)
SET
    id = IFNULL(@v_id, 0),
    uid = CASE 
        WHEN NULLIF(@v_uid_str, '') IS NULL THEN ''
        WHEN REPLACE(NULLIF(@v_uid_str, ''), '-', '') REGEXP '^[0-9A-Fa-f]+$' 
        THEN UNHEX(REPLACE(NULLIF(@v_uid_str, ''), '-', ''))
        ELSE ''
    END,
    last_name = LEFT(TRIM(SUBSTRING_INDEX(SUBSTRING_INDEX(@v_fullname, ' ', 1), ' ', -1)), 50),
    first_name = CASE 
        WHEN LENGTH(@v_fullname) - LENGTH(REPLACE(@v_fullname, ' ', '')) >= 1 
        THEN LEFT(TRIM(SUBSTRING_INDEX(SUBSTRING_INDEX(@v_fullname, ' ', 2), ' ', -1)), 50)
        ELSE '' 
    END,
    middle_name = CASE 
        WHEN LENGTH(@v_fullname) - LENGTH(REPLACE(@v_fullname, ' ', '')) >= 2 
        THEN LEFT(TRIM(SUBSTRING_INDEX(@v_fullname, ' ', -1)), 50)
        ELSE '' 
    END,
    phone_number = LEFT(IFNULL(NULLIF(@v_phone, ''), ''), 20),
    cc_card_useralias = IFNULL(NULLIF(@v_account, ''), ''),
    open_calls = IF(IFNULL(@v_open, '') = '' OR @v_open = 'NULL', 0, CAST(@v_open AS UNSIGNED)),
    incoming_calls_from_any_card = IF(IFNULL(@v_incoming, '') = '' OR @v_incoming = 'NULL', 0, CAST(@v_incoming AS UNSIGNED));

Query OK, 2959279 rows affected (44,16 sec)
Records: 2959279  Deleted: 0  Skipped: 0  Warnings: 0
```
Не валидные данные дублируются, т.к. `uid` обнуляется, надо их все найти и удалить, т.к. они не будут вставляться в таблицу из-за ограничения уникальности. Далее с ними разберемся.

**Таблицы abonents_base + abonents_b связаны друг с другом одним PK - по факту это вертикально разрезанная таблица. И придется писать в них в 2 захода.**
#### INSERT в abonents_base
1. Добавить поле `ztlc_user_id` в таблицу `abonents_base` для сведения с `ztlc_users_tmp.id` и индекс на это поле:
```
sudo pt-online-schema-change --alter "ADD COLUMN ztlc_user_id INT, ADD INDEX ztlc_user_id_idx (ztlc_user_id)" D=a2billing,t=abonents_base --preserve-triggers --alter-foreign-keys-method auto --chunk-size 1000 --sleep 0.3 --recursion-method=none --max-load "Threads_running=50" --critical-load "Threads_running=150"  --dry-run
```
2. Вставка данных:
Сырой запрос
```
insert into abonents_base (ztlc_user_id, abonent_type, first_name, last_name, middle_name, phone_number, sync_date)
    select id, 'ABONENT_B', first_name, last_name, middle_name, phone_number, now()
    from ztlc_users_tmp 
    -- я делал батчами иначе памяти не хватало
    where id > IFNULL((select max(ztlc_user_id) from abonents_base), 0)
    order by id limit 500000; -- 1m 20s
```
Скрипт с процедурой: `vim /tmp/INFR-4969/create_procedure_insert_abonents_base.sql`
```
tee /tmp/INFR-4969/create_procedure_insert_abonents_base.sql << 'EOF'
DELIMITER //
CREATE PROCEDURE `insert_abonents_base`(
    IN batch INT,
    IN sleep_t DECIMAL(5,2)
)
BEGIN
DECLARE rows_i INT DEFAULT 1;
DECLARE rows_count INT DEFAULT 0;
DECLARE iterations INT DEFAULT 0;

SET batch = IFNULL(batch, 1000);
SET sleep_t = IFNULL(sleep_t, 1);
SET rows_i = 1;
SET rows_count = 0;
SET iterations = 0;

WHILE rows_i > 0 DO 
    START TRANSACTION;
        INSERT INTO abonents_base (ztlc_user_id, abonent_type, first_name, last_name, middle_name, phone_number, sync_date)
            SELECT id, 'ABONENT_B', first_name, last_name, middle_name, phone_number, now() FROM ztlc_users_tmp 
            WHERE id > IFNULL((SELECT max(ztlc_user_id) FROM abonents_base), 0) 
            ORDER BY id LIMIT batch;
        SET rows_i = ROW_COUNT();
    COMMIT;
    SET iterations = iterations + 1;
    SET rows_count = rows_count + rows_i;
    SELECT CONCAT('Итерация: ', iterations, ', Добавлено строк: ', rows_i, ', Всего добавлено строк: ', rows_count) AS 'Status';
    SELECT CONCAT(sleep_t, ' seconds...') AS 'Sleep';
    DO SLEEP(sleep_t);
END WHILE;
END
//
DELIMITER ;
EOF
```
- Накатить процедуру: `sudo mysql a2billing < /tmp/INFR-4969/create_procedure_insert_abonents_base.sql`
- Вызвать процедуру: `CALL insert_abonents_base(1000, 1);`

#### Удаление из временной таблицы ztlc_users_tmp записей, уже имеющихся в abonents_b
Таблица `abonents_b` регулярно заполняется новыми данными из БД `zonatelecom`, эти данные так же попали во временную таблицу `ztlc_users_tmp` при импорте. Их надо удалить из `ztlc_users_tmp`.
1. Количество записей в таблице `ztlc_users_tmp`, которые имеют соответствие в таблице `abonents_b` по полям `uid` и `uid_keycloack`:
```
SELECT COUNT(*) AS matched_count FROM ztlc_users_tmp zt INNER JOIN abonents_b ab ON zt.uid = ab.uid_keycloack;
+---------------+
| matched_count |
+---------------+
|          6239 |
+---------------+
1 row in set (4,83 sec)
```
2. Удаление записей в таблице `ztlc_users_tmp`, которые имеют соответствие в таблице `abonents_b` по полям `uid` и `uid_keycloack`:
```
DELETE zt FROM ztlc_users_tmp zt INNER JOIN abonents_b ab ON zt.uid = ab.uid_keycloack;
Query OK, 6239 rows affected (7,54 sec)
```

#### INSERT в abonent_b
1. Добавить дополнительные поля
```
sudo pt-online-schema-change --alter "ADD COLUMN cc_card_useralias VARCHAR(50), ADD COLUMN card_id INT, ADD COLUMN id_tmp BIGINT, ADD INDEX idx_cc_card_useralias (cc_card_useralias), ADD INDEX idx_id_tmp (id_tmp)" D=a2billing,t=abonents_b --preserve-triggers --alter-foreign-keys-method auto --chunk-size 1000 --sleep 0.3 --recursion-method=none --max-load "Threads_running=50" --critical-load "Threads_running=150"  --dry-run
```
2. Загрузка данных:
Сырой запрос:
```
INSERT INTO abonents_b (id, uid_keycloack, cc_card_useralias, open_calls, incoming_calls_from_any_card)
    SELECT ab.id, z.uid, z.cc_card_useralias, z.open_calls, z.incoming_calls_from_any_card
    FROM ztlc_users_tmp z INNER JOIN abonents_base ab ON ab.ztlc_user_id=z.id
    WHERE ab.id > IFNULL((SELECT max(id) FROM abonents_b), 0) 
    ORDER BY ab.id LIMIT 1000;
```
Предварительно удалить FK для ускорения (не имеет смысла удалять, если батчами т.к. +- 1ms на батч):
```
ALTER TABLE abonents_b DROP FOREIGN KEY _fk_abonents_b_base;
```
для восстановления:
```
ALTER TABLE abonents_b ADD CONSTRAINT fk_abonents_b_base FOREIGN KEY (id) REFERENCES abonents_base (id) ON DELETE CASCADE;
```
Создать индекс:
```
mysql> ALTER TABLE abonents_base DROP INDEX ztlc_user_id_idx;
Query OK, 0 rows affected (0.05 sec)
Records: 0  Duplicates: 0  Warnings: 0

mysql> CREATE INDEX idx_abonents_base_join_sort ON abonents_base (ztlc_user_id, id);
Query OK, 0 rows affected (20.39 sec)
Records: 0  Duplicates: 0  Warnings: 0
```
Скрипт с процедурой: `vim /tmp/INFR-4969/create_procedure_insert_abonents_b.sql`
```
tee /tmp/INFR-4969/create_procedure_insert_abonents_b.sql << 'EOF'
DELIMITER //
CREATE PROCEDURE `insert_abonents_b`(
    IN batch INT,
    IN sleep_t DECIMAL(5,2)
)
BEGIN
DECLARE rows_i INT DEFAULT 1;
DECLARE rows_count INT DEFAULT 0;
DECLARE iterations INT DEFAULT 0;
DECLARE last_processed_id BIGINT DEFAULT 0;

SET batch = IFNULL(batch, 1000);
SET sleep_t = IFNULL(sleep_t, 1);
SET rows_i = 1;
SET rows_count = 0;
SET iterations = 0;

SELECT IFNULL(MAX(id_tmp), 0) INTO last_processed_id FROM abonents_b;

WHILE rows_i > 0 DO 
    START TRANSACTION;
        INSERT INTO abonents_b (id, uid_keycloack, cc_card_useralias, open_calls, incoming_calls_from_any_card, id_tmp)
            SELECT ab.id, z.uid, z.cc_card_useralias, z.open_calls, z.incoming_calls_from_any_card, ab.id
            FROM ztlc_users_tmp z INNER JOIN abonents_base ab ON ab.ztlc_user_id=z.id
            WHERE ab.id > last_processed_id
            ORDER BY ab.id LIMIT batch;
        SET rows_i = ROW_COUNT();

        IF rows_i > 0 THEN
            SELECT MAX(id_tmp) INTO last_processed_id FROM abonents_b;
        END IF;

    COMMIT;
    SET iterations = iterations + 1;
    SET rows_count = rows_count + rows_i;
    SELECT CONCAT('Итерация: ', iterations, ', Добавлено строк: ', rows_i, ', Всего добавлено строк: ', rows_count) AS 'Status';
    SELECT CONCAT(sleep_t, ' seconds...') AS 'Sleep';
    DO SLEEP(sleep_t);
END WHILE;
END
//
DELIMITER ;
EOF
```
- Накатить процедуру: `sudo mysql a2billing < /tmp/INFR-4969/create_procedure_insert_abonents_b.sql`
- Вызвать процедуру: `CALL insert_abonents_b(10000, 1);`

#### UPDATE cc_card
Обновить `cc_card.abonent_id` = `abonents_b.id` (где `abonents_b.cc_card_useralias` = `cc_card.useralias`)

Снять бэкап отдельно таблицы `cc_card`:
```
sudo mysqldump --set-gtid-purged=OFF \
  --single-transaction \
  --routines \
  --triggers \
  --events \
  --hex-blob \
  --quick \
  --max_allowed_packet=512M \
  --add-drop-table \
  --result-file=/data/backup/a2billing_cc_card.dump \
  a2billing cc_card

r.nazarov@galera-node03:/tmp/INFR-4969$ sudo ls -lah /data/backup/a2billing_cc_card.dump
-rw-r--r-- 1 root root 3,4G мар 26 10:02 /data/backup/a2billing_cc_card.dump
```
На случай, если придется восстанавливать:
```
sudo mysql a2billing < /data/backup/a2billing_cc_card.dump
```

1. Сырой запрос (на проде заменить на процедуру из п.3):
```
UPDATE cc_card AS c
INNER JOIN abonents_b AS ab ON c.useralias = ab.cc_card_useralias
SET c.abonent_id=ab.id;

Query OK, 582274 rows affected (2 min 53.49 sec)
Rows matched: 582274  Changed: 582274  Warnings: 0
```
2. Требуется индекс:
```
sudo pt-online-schema-change --alter "ADD INDEX idx_abonent_id_null (abonent_id, id)" D=a2billing,t=cc_card --preserve-triggers --alter-foreign-keys-method auto --chunk-size 1000 --sleep 0.3 --recursion-method=none --max-load "Threads_running=50" --critical-load "Threads_running=150"  --dry-run
```
3. Скрипт с процедурой:
```
tee /tmp/INFR-4969/create_procedure_update_cc_card.sql << 'EOF'
DELIMITER //
CREATE PROCEDURE update_cc_card_abonent_batch(
    IN batch_size INT,
    IN pause_sec DECIMAL(5,2)
)
BEGIN
    DECLARE updated INT DEFAULT 1;
    DECLARE total INT DEFAULT 0;
    DECLARE iter INT DEFAULT 0;
    DECLARE start_batch BIGINT DEFAULT 1;
    DECLARE end_batch BIGINT DEFAULT 1;
    DECLARE row_cnt INT DEFAULT 1;

    SET batch_size = IFNULL(batch_size, 1000);
    SET pause_sec = IFNULL(pause_sec, 0.5);
    SET end_batch = start_batch + batch_size;
    
CREATE TABLE IF NOT EXISTS ids_cc_card_tmp(
    id BIGINT(20) NOT NULL AUTO_INCREMENT,
    id_cc_card BIGINT(20) NOT NULL,
    abonent_id BIGINT(20) NOT NULL,
    PRIMARY KEY (id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
SELECT 'Создана временная таблица для хранения cc_card.id изменяемых строк;' AS progress;
TRUNCATE TABLE ids_cc_card_tmp;
SELECT 'Временная таблица очищена;' AS progress;
SELECT 'Вставка значений во временную таблицу' AS progress;
INSERT INTO ids_cc_card_tmp (id_cc_card, abonent_id)
    SELECT c.id, ab.id FROM cc_card AS c JOIN abonents_b AS ab ON c.useralias = ab.cc_card_useralias;
SELECT 'Временная таблица заполнена' AS progress;

SELECT 'Обновление таблицы cc_card' AS progress;
SELECT COUNT(1) INTO row_cnt FROM ids_cc_card_tmp;
WHILE start_batch < row_cnt DO 
    START TRANSACTION;
        UPDATE cc_card AS c
        JOIN (SELECT id_cc_card, abonent_id FROM ids_cc_card_tmp WHERE id >= start_batch AND id < end_batch) AS ab ON c.id = ab.id_cc_card
        SET c.abonent_id=ab.abonent_id;
        SET updated = ROW_COUNT();
    COMMIT;
    SET start_batch = end_batch;
    SET end_batch = start_batch + batch_size;
    SET iter = iter + 1;
    SET total = total + updated;
    SELECT CONCAT('Итерация: ', iter, ', Обновлено строк: ', updated, ', Всего обновлено строк: ', total) AS 'Status';
    SELECT CONCAT(pause_sec, ' seconds...') AS 'Sleep';
    DO SLEEP(pause_sec);
END WHILE;    
END//
DELIMITER ;
EOF
```
- Накатить процедуру: `sudo mysql a2billing < /tmp/INFR-4969/create_procedure_update_cc_card.sql`
- Вызвать процедуру: `CALL update_cc_card_abonent_batch(10000, 1);`

#### Загрузка таблицы ztlc_srv_zcards_users
Требуется загрузить таблицу `ztlc_srv_zcards_users` из БД `zonatelecom` со slave-хоста источника, находясь на целевом хосте:
```
mysqldump -h slave-хоста_источника -u rnazarov -p'****' --set-gtid-purged=OFF --default-character-set=utf8mb4 zonatelecom ztlc_srv_zcards_users | sed 's/USE `zonatelecom`/USE `a2billing`/g' | sudo mysql a2billing
```
#### INSERT abonent_b_cards
Сырой запрос (на проде только пачками):
```
SELECT COUNT(1) FROM 
(SELECT a.id, zu.card_id, zu.open_calls, zu.accept, zu.datetime 
  FROM ztlc_srv_zcards_users AS zu
  INNER JOIN abonents_base AS a ON zu.user_id=a.ztlc_user_id) AS data_1;
+----------+
| COUNT(1) |
+----------+
|   908205 |
+----------+
1 row in set (2,17 sec)

INSERT IGNORE INTO abonent_b_cards (abonent_id, card_id, open_calls_enabled, accept, created_at)
SELECT a.id, zu.card_id, zu.open_calls, zu.accept, zu.datetime 
  FROM ztlc_srv_zcards_users AS zu
  INNER JOIN abonents_base AS a ON zu.user_id=a.ztlc_user_id;

Query OK, 907959 rows affected, 246 warnings (38,27 sec)
Records: 908205  Duplicates: 246  Warnings: 246

mysql> SELECT COUNT(1) FROM abonent_b_cards;
+----------+
| COUNT(1) |
+----------+
|   908145 |
+----------+
1 row in set (0.15 sec)
```

#### Удалить временные объекты
```
ALTER TABLE abonents_base DROP COLUMN ztlc_user_id;
ALTER TABLE abonents_b
    DROP COLUMN cc_card_useralias,
    DROP COLUMN card_id,
    DROP COLUMN id_tmp;
DROP TABLE ztlc_users_tmp;
DROP TABLE ids_cc_card_tmp;
DROP PROCEDURE delete_abonents_base;
DROP PROCEDURE insert_abonents_base;
DROP PROCEDURE insert_abonents_b;
DROP PROCEDURE update_cc_card_abonent_batch;

sudo rm /var/lib/mysql-files/ztlc_users.csv
```
на проде:
```
sudo pt-online-schema-change --alter "DROP COLUMN ztlc_user_id" D=a2billing,t=abonents_base --preserve-triggers --alter-foreign-keys-method auto --chunk-size 1000 --sleep 0.3 --recursion-method=none --max-load "Threads_running=50" --critical-load "Threads_running=150"  --dry-run

sudo pt-online-schema-change --alter "DROP COLUMN cc_card_useralias, DROP COLUMN card_id, DROP COLUMN id_tmp" D=a2billing,t=abonents_b --preserve-triggers --alter-foreign-keys-method auto --chunk-size 1000 --sleep 0.3 --recursion-method=none --max-load "Threads_running=50" --critical-load "Threads_running=150"  --dry-run

DROP TABLE ztlc_users_tmp;
DROP TABLE ids_cc_card_tmp;
DROP PROCEDURE delete_abonents_base;
DROP PROCEDURE insert_abonents_base;
DROP PROCEDURE insert_abonents_b;
DROP PROCEDURE update_cc_card_abonent_batch;

sudo rm /var/lib/mysql-files/ztlc_users.csv
```
