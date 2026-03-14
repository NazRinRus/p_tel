```
-- ============================================================================
-- ИНДЕКСЫ ДЛЯ КЛИЕНТОВ
-- ============================================================================

-- Поиск клиента по номеру телефона
-- Используется для: авторизации, поиска клиента в системе, проверки дубликатов
-- Пример запроса: SELECT * FROM clients WHERE phone = '+7 900 000-00-01';
CREATE INDEX idx_clients_phone ON clients(phone);

-- Поиск клиента по email
-- Используется для: отправки уведомлений, восстановления пароля, поиска
-- Пример запроса: SELECT * FROM clients WHERE email = 'ivanov@example.com';
CREATE INDEX idx_clients_email ON clients(email);

-- Фильтрация клиентов по типу (спецконтингент или внешний контакт)
-- Используется для: разделения СК и внешних контактов в отчетах и интерфейсе
-- Пример запроса: SELECT * FROM clients WHERE is_special_contingent = TRUE;
CREATE INDEX idx_clients_is_sc ON clients(is_special_contingent);

-- Поиск всех клиентов в конкретном учреждении
-- Используется для: получения списка СК в учреждении, отчетности по учреждению
-- Пример запроса: SELECT * FROM clients WHERE institution_id = 1;
CREATE INDEX idx_clients_institution ON clients(institution_id);

-- Фильтрация клиентов по статусу
-- Используется для: поиска активных/заблокированных/освобожденных клиентов
-- Пример запроса: SELECT * FROM clients WHERE status = 'ACTIVE';
CREATE INDEX idx_clients_status ON clients(status);

-- ============================================================================
-- ИНДЕКСЫ ДЛЯ СЧЕТОВ
-- ============================================================================

-- Поиск счета по клиенту (основной JOIN для биллинга)
-- Используется для: получения счета клиента при авторизации, проверке баланса
-- Пример запроса: SELECT * FROM accounts WHERE client_id = 1;
CREATE INDEX idx_accounts_client ON accounts(client_id);

-- Фильтрация счетов по статусу
-- Используется для: поиска активных/заблокированных/закрытых счетов
-- Пример запроса: SELECT * FROM accounts WHERE status = 'ACTIVE';
CREATE INDEX idx_accounts_status ON accounts(status);

-- Поиск счетов по балансу
-- Используется для: выявления должников, клиентов с низким балансом
-- Пример запроса: SELECT * FROM accounts WHERE balance < 100.00;
CREATE INDEX idx_accounts_balance ON accounts(balance);

-- ============================================================================
-- ИНДЕКСЫ ДЛЯ УЧРЕЖДЕНИЙ И ТЕРМИНАЛОВ
-- ============================================================================

-- Фильтрация учреждений по региону
-- Используется для: отчетности по регионам, географической аналитики
-- Пример запроса: SELECT * FROM institutions WHERE region_id = 1;
CREATE INDEX idx_institutions_region ON institutions(region_id);

-- Фильтрация учреждений по типу
-- Используется для: разделения ИК/СИЗО/клиник в отчетах и интерфейсе
-- Пример запроса: SELECT * FROM institutions WHERE type = 'ИК';
CREATE INDEX idx_institutions_type ON institutions(type);

-- Поиск всех терминалов в учреждении
-- Используется для: мониторинга оборудования, статистики по учреждению
-- Пример запроса: SELECT * FROM terminals WHERE institution_id = 1;
CREATE INDEX idx_terminals_institution ON terminals(institution_id);

-- Мониторинг терминалов по статусу
-- Используется для: поиска онлайн/офлайн терминалов, дашборда мониторинга
-- Пример запроса: SELECT * FROM terminals WHERE status = 'ONLINE';
CREATE INDEX idx_terminals_status ON terminals(status);

-- Поиск терминала по серийному номеру
-- Используется для: инвентаризации, поиска оборудования при обслуживании
-- Пример запроса: SELECT * FROM terminals WHERE serial_number = 'TERM-001-2024';
CREATE INDEX idx_terminals_serial ON terminals(serial_number);

-- ============================================================================
-- ИНДЕКСЫ ДЛЯ РАЗРЕШЕННЫХ СВЯЗЕЙ
-- ============================================================================

-- Поиск всех разрешенных связей спецконтингента
-- Используется для: получения списка контактов СК, проверки перед звонком
-- Пример запроса: SELECT * FROM allowed_connections WHERE sc_client_id = 1;
CREATE INDEX idx_allowed_connections_sc ON allowed_connections(sc_client_id);

-- Поиск всех связей внешнего контакта
-- Используется для: проверки кому может звонить внешний контакт (обратный поиск)
-- Пример запроса: SELECT * FROM allowed_connections WHERE external_client_id = 4;
CREATE INDEX idx_allowed_connections_external ON allowed_connections(external_client_id);

-- Фильтрация связей по статусу верификации
-- Используется для: поиска одобренных/ожидающих/отклоненных связей
-- Пример запроса: SELECT * FROM allowed_connections WHERE verification_status = 'APPROVED';
CREATE INDEX idx_allowed_connections_status ON allowed_connections(verification_status);

-- Поиск только активных связей
-- Используется для: проверки доступности связи перед началом звонка
-- Пример запроса: SELECT * FROM allowed_connections WHERE is_active = TRUE;
CREATE INDEX idx_allowed_connections_active ON allowed_connections(is_active);

-- Уникальность пары СК-внешний контакт (защита от дублирования связей)
-- Используется для: предотвращения создания дубликатов связей между одной парой
-- Пример запроса: Автоматическая проверка при INSERT/UPDATE
CREATE UNIQUE INDEX idx_allowed_connections_unique_pair ON allowed_connections(sc_client_id, external_client_id);

-- ============================================================================
-- ИНДЕКСЫ ДЛЯ СЕАНСОВ СВЯЗИ
-- ============================================================================

-- История сеансов по лицевому счету
-- Используется для: получения истории звонков по счету, детализации
-- Пример запроса: SELECT * FROM sessions WHERE account_id = 1;
CREATE INDEX idx_sessions_account ON sessions(account_id);

-- Статистика по терминалу
-- Используется для: анализа загрузки терминала, отчетности по оборудованию
-- Пример запроса: SELECT * FROM sessions WHERE terminal_id = 1;
CREATE INDEX idx_sessions_terminal ON sessions(terminal_id);

-- История по разрешенной связи
-- Используется для: получения истории звонков между конкретной парой клиентов
-- Пример запроса: SELECT * FROM sessions WHERE allowed_connection_id = 1;
CREATE INDEX idx_sessions_connection ON sessions(allowed_connection_id);

-- История звонков спецконтингента
-- Используется для: получения всех звонков СК (основной запрос в личном кабинете)
-- Пример запроса: SELECT * FROM sessions WHERE sc_client_id = 1;
CREATE INDEX idx_sessions_sc_client ON sessions(sc_client_id);

-- Поиск звонков внешнему контакту
-- Используется для: проверки кому звонил СК, статистики по контактам
-- Пример запроса: SELECT * FROM sessions WHERE external_client_id = 4;
CREATE INDEX idx_sessions_external_client ON sessions(external_client_id);

-- Фильтрация сеансов по статусу
-- Используется для: поиска активных/завершенных/неудачных звонков
-- Пример запроса: SELECT * FROM sessions WHERE status = 'IN_PROGRESS';
CREATE INDEX idx_sessions_status ON sessions(status);

-- Поиск сеансов по времени начала
-- Используется для: отчетности за период, поиска звонков по дате
-- Пример запроса: SELECT * FROM sessions WHERE actual_start >= '2024-01-01';
CREATE INDEX idx_sessions_actual_start ON sessions(actual_start);

-- Хронологический список сеансов
-- Используется для: получения списка звонков в порядке создания
-- Пример запроса: SELECT * FROM sessions ORDER BY created_at DESC;
CREATE INDEX idx_sessions_created_at ON sessions(created_at);

-- ============================================================================
-- ИНДЕКСЫ ДЛЯ ТРАНЗАКЦИЙ
-- ============================================================================

-- История операций по счету (выписка)
-- Используется для: получения финансовой истории счета, детализации платежей
-- Пример запроса: SELECT * FROM transactions WHERE account_id = 1;
CREATE INDEX idx_transactions_account ON transactions(account_id);

-- Поиск транзакций по сеансу
-- Используется для: проверки оплаты конкретного звонка
-- Пример запроса: SELECT * FROM transactions WHERE session_id = 1;
CREATE INDEX idx_transactions_session ON transactions(session_id);

-- Фильтрация по типу операции
-- Используется для: разделения пополнений/списаний/возвратов в отчетах
-- Пример запроса: SELECT * FROM transactions WHERE type = 'CHARGE';
CREATE INDEX idx_transactions_type ON transactions(type);

-- Хронология операций
-- Используется для: получения истории операций в порядке времени
-- Пример запроса: SELECT * FROM transactions ORDER BY created_at DESC;
CREATE INDEX idx_transactions_created_at ON transactions(created_at);

-- ============================================================================
-- СОСТАВНЫЕ ИНДЕКСЫ ДЛЯ ЧАСТЫХ ЗАПРОСОВ
-- ============================================================================

-- История звонков по счету с сортировкой по дате (оптимизация личного кабинета)
-- Используется для: быстрого получения истории звонков с сортировкой по убыванию
-- Пример запроса: SELECT * FROM sessions WHERE account_id = 1 ORDER BY created_at DESC LIMIT 100;
CREATE INDEX idx_sessions_account_date ON sessions(account_id, created_at DESC);

-- История звонков спецконтингента с сортировкой по дате
-- Используется для: быстрого получения истории СК с сортировкой по убыванию
-- Пример запроса: SELECT * FROM sessions WHERE sc_client_id = 1 ORDER BY created_at DESC LIMIT 100;
CREATE INDEX idx_sessions_sc_date ON sessions(sc_client_id, created_at DESC);

-- Выписка по счету с сортировкой по дате (оптимизация финансовой истории)
-- Используется для: быстрого получения последних транзакций счета
-- Пример запроса: SELECT * FROM transactions WHERE account_id = 1 ORDER BY created_at DESC LIMIT 50;
CREATE INDEX idx_transactions_account_date ON transactions(account_id, created_at DESC);

-- ============================================================================
-- ИНДЕКСЫ ДЛЯ ТАРИФОВ И АКЦИЙ
-- ============================================================================

-- Поиск только активных тарифов
-- Используется для: отображения доступных тарифов при подключении
-- Пример запроса: SELECT * FROM tariffs WHERE is_active = TRUE;
CREATE INDEX idx_tariffs_active ON tariffs(is_active);

-- Поиск только активных акций
-- Используется для: отображения действующих акций, применения скидок
-- Пример запроса: SELECT * FROM promotions WHERE is_active = TRUE;
CREATE INDEX idx_promotions_active ON promotions(is_active);

-- Поиск акций по периоду действия
-- Используется для: проверки действующих акций на текущую дату
-- Пример запроса: SELECT * FROM promotions WHERE start_date <= NOW() AND end_date >= NOW();
CREATE INDEX idx_promotions_dates ON promotions(start_date, end_date);

-- ============================================================================
-- ИНДЕКСЫ ДЛЯ АУДИТА
-- ============================================================================

-- Действия конкретного пользователя
-- Используется для: проверки активности пользователя, расследования инцидентов
-- Пример запроса: SELECT * FROM audit_log WHERE user_id = 1;
CREATE INDEX idx_audit_log_user ON audit_log(user_id);

-- Фильтрация по типу действия
-- Используется для: поиска операций CREATE/UPDATE/DELETE/LOGIN
-- Пример запроса: SELECT * FROM audit_log WHERE action = 'UPDATE';
CREATE INDEX idx_audit_log_action ON audit_log(action);

-- Хронология аудита
-- Используется для: получения лога действий в порядке времени
-- Пример запроса: SELECT * FROM audit_log ORDER BY created_at DESC LIMIT 1000;
CREATE INDEX idx_audit_log_created_at ON audit_log(created_at);

-- Изменения по конкретной таблице
-- Используется для: отслеживания изменений в определенной таблице
-- Пример запроса: SELECT * FROM audit_log WHERE table_name = 'accounts';
CREATE INDEX idx_audit_log_table ON audit_log(table_name);
```
