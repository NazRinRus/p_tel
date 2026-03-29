### Поднятие MySQL в конейтнере
Контейнер будет поднят на виртуальной машине `192.168.0.140`
#### Склонировать репозиторий
```
cd /tmp
git clone https://github.com/aeuge/otus-mysql-docker.git
```
#### Добавить свои таблицы в init.sql
```
CREATE database otus;
USE otus;
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
#### Поднять сервис
```
cd otus-mysql-docker/
sudo docker-compose up otusdb

(for_databases) nazrinrus@test-host:~$ sudo docker ps
CONTAINER ID   IMAGE          COMMAND                  CREATED         STATUS         PORTS                                                  NAMES
6b4bfc0b2044   mysql:8.0.15   "docker-entrypoint.s…"   2 minutes ago   Up 2 minutes   33060/tcp, 0.0.0.0:3309->3306/tcp, :::3309->3306/tcp   otus-mysql-docker_otusdb_1
```
#### Подключиться к контейнеру с удаленного терминала
Подключение с удаленного хоста с адресом `192.168.0.100`
```
(ansible) nazrinrus@desktop:~/PGMech$ mysql -h 192.168.0.140 -u root -p12345 --port=3309 --protocol=tcp otus -e 'SHOW DATABASES;'
mysql: [Warning] Using a password on the command line interface can be insecure.
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| otus               |
| performance_schema |
| sys                |
+--------------------+

(ansible) nazrinrus@desktop:~/PGMech$ mysql -h 192.168.0.140 -u root -p12345 --port=3309 --protocol=tcp otus -e 'SHOW TABLES;'
mysql: [Warning] Using a password on the command line interface can be insecure.
+----------------+
| Tables_in_otus |
+----------------+
| abonents_a     |
| abonents_b     |
| abonents_base  |
| institutions   |
| regions        |
| terminals      |
+----------------+
```
