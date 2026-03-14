### Проект p_tel
Спроектировать структуру БД для системы voip телефонии для режимных объектов

### Описание таблиц
#### Справочники:
Таблица: `regions` (Регионы РФ):
| Поле | Тип | Обязательное | Описание | Пример |
|----|----|--------|--------|--------|
|id|SERIAL|PK|Уникальный идентификатор|1|
|name|VARCHAR(100)|NOT NULL|Название региона|Москва|
|code|VARCHAR(10)|UNIQUE|Код субъекта РФ|77|
|created_at|TIMESTAMP|DEFAULT|Дата создания|2024-01-01|
|updated_at|TIMESTAMP|DEFAULT|Дата обновления|2024-01-01|

Таблица: `tariffs` (Тарифы):
Поле|Тип|Обязательное|Описание|Пример
----|----|--------|--------|--------
id|SERIAL|PK|Уникальный идентификатор|1
name|VARCHAR(100)|NOT NULL|Название тарифа|Базовый
price_per_minute|"NUMERIC(10|2)"|NOT NULL|Стоимость минуты (руб)|5.00
connection_fee|"NUMERIC(10|2)"|DEFAULT 0|Плата за соединение (руб)|0.00
is_active|BOOLEAN|DEFAULT TRUE|Активность тарифа|TRUE
description|TEXT|-|Описание тарифа|Стандартный тариф
created_at|TIMESTAMP|DEFAULT|Дата создания|-
updated_at|TIMESTAMP|DEFAULT|Дата обновления|-

Таблица: `promotions` (Акции и скидки):
Поле|Тип|Обязательное|Описание|Пример
----|----|--------|--------|--------
id|SERIAL|PK|Уникальный идентификатор|1
name|VARCHAR(100)|NOT NULL|Название акции|Новый клиент
type|VARCHAR(20)|NOT NULL|Тип скидки|PERCENT/FIXED
value|"NUMERIC(10|2)"|NOT NULL|Величина скидки|10.00
start_date|TIMESTAMP|NOT NULL|Начало действия|2024-01-01
end_date|TIMESTAMP|NOT NULL|Окончание действия|2024-12-31
is_active|BOOLEAN|DEFAULT TRUE|Активность акции|TRUE
description|TEXT|-|Описание|Скидка 10%
created_at|TIMESTAMP|DEFAULT|Дата создания|-
updated_at|TIMESTAMP|DEFAULT|Дата обновления|-

Таблица: `relationship_types` (Типы отношений):
----|----|--------|--------|--------
Поле|Тип|Обязательное|Описание|Пример
id|SERIAL|PK|Уникальный идентификатор|1
name|VARCHAR(50)|NOT NULL UNIQUE|Название типа|Супруг/Супруга
description|TEXT|-|Описание типа|Официальный брак
created_at|TIMESTAMP|DEFAULT|Дата создания|-

#### Инфраструктура
Таблица: `institutions` (Учреждения):
----|----|--------|--------|--------
Поле|Тип|Обязательное|Описание|Пример
id|SERIAL|PK|Уникальный идентификатор|1
name|VARCHAR(200)|NOT NULL|Название учреждения|ИК-5 УФСИН
region_id|INTEGER|FK → regions|Регион расположения|1
address|TEXT|-|Полный адрес|"г. Москва| ул. Примерная| 1"
type|VARCHAR(50)|-|Тип учреждения|ИК/СИЗО/КЛИНИКА
contact_phone|VARCHAR(20)|-|Контактный телефон|+7 495 000-00-01
contact_email|VARCHAR(100)|-|Контактный email|ik5@ufsin.ru
is_active|BOOLEAN|DEFAULT TRUE|Активность учреждения|TRUE
created_at|TIMESTAMP|DEFAULT|Дата создания|-
updated_at|TIMESTAMP|DEFAULT|Дата обновления|-

Таблица: `terminals` (Терминалы связи):
Поле|Тип|Обязательное|Описание|Пример
----|----|--------|--------|--------
id|SERIAL|PK|Уникальный идентификатор|1
institution_id|INTEGER|FK → institutions|Учреждение установки|1
serial_number|VARCHAR(50)|UNIQUE NOT NULL|Заводской номер|TERM-001-2024
ip_address|INET|-|IP-адрес терминала|192.168.1.101
model|VARCHAR(100)|-|Модель оборудования|VideoLink Pro
firmware_version|VARCHAR(20)|-|Версия прошивки|2.1.0
status|VARCHAR(20)|DEFAULT 'OFFLINE'|Статус|ONLINE/OFFLINE
last_heartbeat|TIMESTAMP|-|Последнее соединение|2024-01-15 14:30:00
installed_at|TIMESTAMP|DEFAULT|Дата установки|2024-01-01
created_at|TIMESTAMP|DEFAULT|Дата создания|-
updated_at|TIMESTAMP|DEFAULT|Дата обновления|-

#### Клиенты и счета
Таблица: `clients` (Все клиенты):
Поле|Тип|Обязательное|Описание|Пример
----|----|--------|--------|--------
id|SERIAL|PK|Уникальный идентификатор|1
full_name|VARCHAR(150)|NOT NULL|ФИО полностью|Иванов Иван Иванович
is_special_contingent|BOOLEAN|DEFAULT FALSE|Флаг спецконтингента|TRUE/FALSE
institution_id|INTEGER|FK → institutions|Учреждение (только для СК)|1
internal_id|VARCHAR(50)|-|Внутренний номер дела|Дело-12345
passport_series|VARCHAR(4)|-|Серия паспорта|4500
passport_number|VARCHAR(6)|-|Номер паспорта|123456
passport_issued_by|TEXT|-|Кем выдан|ОВД района...
passport_issue_date|DATE|-|Дата выдачи|2010-05-15
phone|VARCHAR(20)|NOT NULL|Телефон для связи|+7 900 000-00-01
email|VARCHAR(100)|-|Email адрес|ivanov@example.com
registration_address|TEXT|-|Адрес регистрации|г. Москва...
actual_address|TEXT|-|Фактический адрес|г. Москва...
birth_date|DATE|-|Дата рождения|1980-05-15
status|VARCHAR(20)|DEFAULT 'ACTIVE'|Статус клиента|ACTIVE/BLOCKED
release_date|DATE|-|Дата освобождения (СК)|2025-06-01
created_at|TIMESTAMP|DEFAULT|Дата создания|-
updated_at|TIMESTAMP|DEFAULT|Дата обновления|-

Таблица: `accounts` (Лицевые счета):
Поле|Тип|Обязательное|Описание|Пример
----|----|--------|--------|--------
id|SERIAL|PK|Уникальный идентификатор|1
client_id|INTEGER|UNIQUE FK → clients|Только для спецконтингента|1
tariff_id|INTEGER|FK → tariffs|Текущий тариф|1
balance|"NUMERIC(12|2)"|DEFAULT 0.00|Баланс счета (руб)|500.00
currency|VARCHAR(3)|DEFAULT 'RUB'|Валюта счета|RUB
payment_system_type|VARCHAR(50)|-|Тип платежной системы|BANK_CARD/SBP
status|VARCHAR(20)|DEFAULT 'ACTIVE'|Статус счета|ACTIVE/BLOCKED
credit_limit|"NUMERIC(12|2)"|DEFAULT 0.00|Кредитный лимит (руб)|0.00
created_at|TIMESTAMP|DEFAULT|Дата создания|-
updated_at|TIMESTAMP|DEFAULT|Дата обновления|-

Таблица: `account_promotions` (Привязка акций к счетам):
Поле|Тип|Обязательное|Описание|Пример
----|----|--------|--------|--------
account_id|INTEGER|"PK| FK → accounts"|Лицевой счет|1
promotion_id|INTEGER|"PK| FK → promotions"|Акция|1
applied_at|TIMESTAMP|DEFAULT|Дата применения|2024-01-01
expires_at|TIMESTAMP|-|Дата истечения|2024-12-31
is_active|BOOLEAN|DEFAULT TRUE|Активность привязки|TRUE

#### Разрешенные связи
Таблица: `allowed_connections` (Разрешенные связи):
Поле|Тип|Обязательное|Описание|Пример
----|----|--------|--------|--------
id|SERIAL|PK|Уникальный идентификатор|1
sc_client_id|INTEGER|NOT NULL FK → clients|Спецконтингент|1
external_client_id|INTEGER|NOT NULL FK → clients|Внешний контакт|4
relationship_type_id|INTEGER|FK → relationship_types|Тип отношения|1
relationship_description|VARCHAR(100)|-|Текстовое описание|Супруга
verification_status|VARCHAR(20)|DEFAULT 'PENDING'|Статус верификации|APPROVED
verified_by_admin_id|INTEGER|-|Кто верифицировал|1
verified_at|TIMESTAMP|-|Дата верификации|2024-01-01
rejection_reason|TEXT|-|Причина отказа|-
valid_from|DATE|-|Начало действия|2024-01-01
valid_until|DATE|-|Окончание действия|2025-12-31
max_calls_per_day|INTEGER|-|Лимит звонков в день|3
max_duration_per_call|INTEGER|-|Лимит длительности (сек)|900
is_active|BOOLEAN|DEFAULT TRUE|Активность связи|TRUE
notes|TEXT|-|Примечания|-
created_at|TIMESTAMP|DEFAULT|Дата создания|-
updated_at|TIMESTAMP|DEFAULT|Дата обновления|-

#### Сеансы и транзакции
Таблица: `sessions` (Сеансы связи):
Поле|Тип|Обязательное|Описание|Пример
----|----|--------|--------|--------
id|SERIAL|PK|Уникальный идентификатор|1
terminal_id|INTEGER|FK → terminals|Терминал|1
account_id|INTEGER|FK → accounts|Счет спецконтингента|1
allowed_connection_id|INTEGER|FK → allowed_connections|Разрешенная связь|1
sc_client_id|INTEGER|FK → clients|Дублирование для скорости|1
external_client_id|INTEGER|FK → clients|Дублирование для скорости|4
scheduled_start|TIMESTAMP|-|Запланированное начало|2024-01-15 10:00:00
actual_start|TIMESTAMP|-|Фактическое начало|2024-01-15 10:05:00
actual_end|TIMESTAMP|-|Фактическое окончание|2024-01-15 10:20:00
duration_seconds|INTEGER|DEFAULT 0|Длительность (сек)|900
cost|"NUMERIC(12|2)"|DEFAULT 0.00|Стоимость (руб)|75.00
status|VARCHAR(20)|DEFAULT 'COMPLETED'|Статус|COMPLETED
failure_reason|TEXT|-|Причина неудачи|-
call_type|VARCHAR(20)|DEFAULT 'VOICE'|Тип звонка|VOICE/VIDEO
recording_url|TEXT|-|Ссылка на запись|s3://...
operator_notes|TEXT|-|Заметки оператора|-
created_at|TIMESTAMP|DEFAULT|Дата создания|-
updated_at|TIMESTAMP|DEFAULT|Дата обновления|-

Таблица: `transactions` (Транзакции):
Поле|Тип|Обязательное|Описание|Пример
----|----|--------|--------|--------
id|SERIAL|PK|Уникальный идентификатор|1
account_id|INTEGER|FK → accounts|Лицевой счет|1
session_id|INTEGER|FK → sessions|Связанный сеанс|1
type|VARCHAR(20)|NOT NULL|Тип операции|CHARGE
amount|"NUMERIC(12|2)"|NOT NULL|Сумма|-75.00
balance_before|"NUMERIC(12|2)"|NOT NULL|Баланс до операции|575.00
balance_after|"NUMERIC(12|2)"|NOT NULL|Баланс после операции|500.00
payment_gateway_ref|VARCHAR(100)|-|ID в платежной системе|pay_123456
payment_method|VARCHAR(50)|-|Способ оплаты|BANK_CARD
description|TEXT|-|Описание операции|Оплата сеанса #1
created_at|TIMESTAMP|DEFAULT|Дата создания|-

#### Администрирование
Таблица: `system_users` (Пользователи системы):
Поле|Тип|Обязательное|Описание|Пример
----|----|--------|--------|--------
id|SERIAL|PK|Уникальный идентификатор|1
username|VARCHAR(50)|UNIQUE NOT NULL|Логин|admin
password_hash|VARCHAR(255)|NOT NULL|Хеш пароля (bcrypt)|2b$12$...
email|VARCHAR(100)|UNIQUE NOT NULL|Email|admin@system.com
full_name|VARCHAR(150)|-|ФИО|Администратор
role|VARCHAR(30)|DEFAULT 'OPERATOR'|Роль|ADMIN/OPERATOR
institution_id|INTEGER|FK → institutions|Привязка к учреждению|1
is_active|BOOLEAN|DEFAULT TRUE|Активность учетной записи|TRUE
last_login|TIMESTAMP|-|Последний вход|2024-01-15 09:00:00
created_at|TIMESTAMP|DEFAULT|Дата создания|-
updated_at|TIMESTAMP|DEFAULT|Дата обновления|-

Таблица: `audit_log` (Журнал аудита):
Поле|Тип|Обязательное|Описание|Пример
----|----|--------|--------|--------
id|SERIAL|PK|Уникальный идентификатор|1
user_id|INTEGER|FK → system_users|Пользователь|1
action|VARCHAR(50)|NOT NULL|Действие|CREATE/UPDATE
table_name|VARCHAR(50)|-|Таблица|accounts
record_id|INTEGER|-|ID записи|1
old_values|JSONB|-|Старые значения|"{""balance"": 500}"
new_values|JSONB|-|Новые значения|"{""balance"": 425}"
ip_address|INET|-|IP адрес|192.168.1.100
user_agent|TEXT|-|User Agent|Mozilla/5.0...
created_at|TIMESTAMP|DEFAULT|Дата создания|-
