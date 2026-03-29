### Перенести структуру БД из PostgreSQL в MySQL
- Тип `BOOLEAN` в `TINYINT(1)`
- Документы абонентов можно хранить в JSON


### Описание таблиц
#### Справочники:
Таблица: `regions` (Регионы РФ):
| Поле | Тип | Обязательное | Описание | Пример |
|----|----|--------|--------|--------|
|id|INT|PK|Уникальный идентификатор|1|
|name|VARCHAR(100)|NOT NULL|Название региона|Москва|
|code|VARCHAR(10)|UNIQUE|Код субъекта РФ|77|
|created_at|TIMESTAMP|DEFAULT|Дата создания|2024-01-01|
|updated_at|TIMESTAMP|DEFAULT|Дата обновления|2024-01-01|

#### Инфраструктура
Таблица: `institutions` (Учреждения):
Поле|Тип|Обязательное|Описание|Пример
----|----|--------|--------|--------
id|INT|PK|Уникальный идентификатор|1
name|VARCHAR(255)|NOT NULL|Название учреждения|ИК-5 УФСИН
region_id|INT|FK → regions|Регион расположения|1
location|VARCHAR(50)|NOT NULL|Временная зона относительно UTS|'Europe/Moscow'
address|CHAR(100)|-|Юридический адрес|"г. Москва ул. Примерная 1"
contact_phone|CHAR(20)|-|Контактный телефон|+7 495 000-00-01
fax|CHAR(20)|-|Факс|+7 495 000-00-01
contact_email|CHAR(70)|-|Контактный email|ik5@ufsin.ru
is_active|TINYINT(1)|DEFAULT|Активность учреждения|TRUE
created_at|TIMESTAMP|DEFAULT|Дата создания|2024-01-01
updated_at|TIMESTAMP|DEFAULT|Дата обновления|2024-01-01

Таблица: `terminals` (Терминалы связи):
Поле|Тип|Обязательное|Описание|Пример
----|----|--------|--------|--------
id|INT|PK|Уникальный идентификатор|1
institution_id|INT|FK → institutions|Учреждение установки|1
serial_number|VARCHAR(50)|UNIQUE NOT NULL|Заводской номер|TERM-001-2024
ip_address|VARCHAR(15)|-|IP-адрес терминала|192.168.1.101
model|VARCHAR(100)|-|Модель оборудования|VideoLink Pro
firmware_version|VARCHAR(20)|-|Версия прошивки|2.1.0
status|VARCHAR(20)|DEFAULT 'OFFLINE'|Статус|ONLINE/OFFLINE
last_heartbeat|TIMESTAMP|-|Последнее соединение|2024-01-15 14:30:00
installed_at|TIMESTAMP|DEFAULT|Дата установки|2024-01-01
created_at|TIMESTAMP|DEFAULT|Дата создания|2024-01-01
updated_at|TIMESTAMP|DEFAULT|Дата обновления|2024-01-01

#### Клиенты и счета
Таблица: `abonents_base` (Общая информация об абонентах):
Поле|Тип|Обязательное|Описание|Пример
----|----|--------|--------|--------
id|INT|PK|Уникальный идентификатор|1
abonent_type|ENUM('ABONENT_A','ABONENT_B')|NOT NULL|Тип абонента - спецконтингент или родственник|ABONENT_A
first_name|VARCHAR(50)|NOT NULL|Имя|Иван
last_name|VARCHAR(50)|NOT NULL|Фамилия|Иванов
middle_name|VARCHAR(50)|NULL|Отчество|Иванович
date_of_birth|DATE|NOT NULL|Дата рождения абонента|2000-12-13
phone|VARCHAR(20)|NOT NULL|Телефон для связи|+7 900 000-00-01
email|VARCHAR(100)|-|Email адрес|ivanov@example.com
documents|JSON|NOT NULL|Паспортные данные|
created_at|TIMESTAMP|DEFAULT|Дата создания|2024-01-01
updated_at|TIMESTAMP|DEFAULT|Дата обновления|2024-01-01

Таблица: `abonents_a` (Информация о спецконтингенте):
Поле|Тип|Обязательное|Описание|Пример
----|----|--------|--------|--------
id|INT|PK|Уникальный идентификатор|1
institution_id|INT|NOT NULL|Идентификатор учреждения|1029
restriction_type|TINYINT(1)|NOT NULL|2, если разрешены вызовы только на номера из списка разрешенных|0
daily_limit_active|TINYINT(1)|NOT NULL|Признак наличия ограничение "Суточный лимит"|1
daily_limit|INT|NOT NULL|Суточный лимит в секундах|900
daily_call_duration|INT|NOT NULL|Количество выговоренных секунд за сутки'|435
created_at|TIMESTAMP|DEFAULT|Дата создания|2024-01-01
updated_at|TIMESTAMP|DEFAULT|Дата обновления|2024-01-01

Таблица: `abonents_b` (Общая информация об абонентах):
Поле|Тип|Обязательное|Описание|Пример
----|----|--------|--------|--------
id|INT|PK|Уникальный идентификатор|1
uid_keycloack|BINARY(16)|NOT NULL|Идентификатор родственника из Keycloack|
open_calls|TINYINT(1)|NOT NULL|Признак разрешения родственнику использовать опцию "Всегда на связи"|1
created_at|TIMESTAMP|DEFAULT|Дата создания|2024-01-01
updated_at|TIMESTAMP|DEFAULT|Дата обновления|2024-01-01


### SQL-скрипт
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
#### Пример вставки абонента с паспортными данными в виде JSON
```
INSERT INTO abonents_base (
    abonent_type,
    first_name,
    last_name,
    middle_name,
    date_of_birth,
    phone,
    email,
    documents
) VALUES (
    'ABONENT_B',
    'Мария',
    'Петрова',
    'Сергеевна',
    '1990-12-18',
    '+7 911 933-64-41',
    'maria.petrova@example.com',
    JSON_OBJECT(
        'document_type', 'passport_rf',
        'series', '4005',
        'number', '654321',
        'issue_date', '2015-07-10',
        'issued_by', 'Территориальный пункт УФМС России по Московской области',
        'department_code', '500-015',
        'place_of_birth', 'г. Подольск, Московская обл.',
        'registration_address', JSON_OBJECT(
            'postal_code', '142100',
            'region', 'Московская область',
            'city', 'Подольск',
            'street', 'пр. Ленина',
            'building', '8',
            'apartment', '15'
        ),
        'snils', '987-654-321 01',
        'inn', '500198765432'
    )
);

Query OK, 1 row affected (0.01 sec)
```
#### Примеры вывода содержимого JSON
```
SELECT 
    id, 
    last_name, 
    JSON_VALID(documents) AS is_valid,
    JSON_UNQUOTE(JSON_EXTRACT(documents, '$.series')) AS passport_series,
    JSON_UNQUOTE(JSON_EXTRACT(documents, '$.number')) AS passport_number,
    JSON_EXTRACT(documents, '$.registration_address.city') AS registration_city
FROM abonents_base
WHERE abonent_type = 'ABONENT_B';

+----+----------------+----------+-----------------+-----------------+--------------------+
| id | last_name      | is_valid | passport_series | passport_number | registration_city  |
+----+----------------+----------+-----------------+-----------------+--------------------+
|  1 | Петрова        |        1 | 4005            | 654321          | "Подольск"         |
+----+----------------+----------+-----------------+-----------------+--------------------+
1 row in set (0.00 sec)

SELECT * FROM abonents_base 
WHERE JSON_EXTRACT(documents, '$.number') = '654321'\G

           id: 1
 abonent_type: ABONENT_B
   first_name: Мария
    last_name: Петрова
  middle_name: Сергеевна
date_of_birth: 1992-11-28
        phone: +7 910 987-65-43
        email: maria.petrova@example.com
    documents: {"inn": "500198765432", "snils": "987-654-321 01", "number": "654321", "series": "4005", "issued_by": "Территориальный пункт УФМС России по Московской области", "issue_date": "2015-07-10", "document_type": "passport_rf", "place_of_birth": "г. Подольск, Московская обл.", "department_code": "500-015", "registration_address": {"city": "Подольск", "region": "Московская область", "street": "пр. Ленина", "building": "8", "apartment": "15", "postal_code": "142100"}}
   created_at: 2026-03-29 19:31:19
   updated_at: 2026-03-29 19:31:19
1 row in set (0.00 sec)
```
