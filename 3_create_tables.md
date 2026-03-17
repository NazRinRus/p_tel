-- ============================================================================
-- СКРИПТ СОЗДАНИЯ БАЗЫ ДАННЫХ
-- Проект: p_tel - Система связи для закрытых учреждений
-- СУБД: PostgreSQL 14+
-- ============================================================================

-- Создание базы данных (выполняется отдельно)
-- CREATE DATABASE p_tel;

-- Подключение к БД
-- \c p_tel

-- ============================================================================
-- 1. СПРАВОЧНИКИ И ГЕОГРАФИЯ
-- ============================================================================

-- Таблица: Регионы РФ
CREATE TABLE regions (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    code VARCHAR(10) UNIQUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Таблица: Тарифы
CREATE TABLE tariffs (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    price_per_minute NUMERIC(10, 2) NOT NULL,
    connection_fee NUMERIC(10, 2) DEFAULT 0,
    is_active BOOLEAN DEFAULT TRUE,
    description TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Таблица: Акции и скидки
CREATE TABLE promotions (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    type VARCHAR(20) NOT NULL,
    value NUMERIC(10, 2) NOT NULL,
    start_date TIMESTAMP NOT NULL,
    end_date TIMESTAMP NOT NULL,
    is_active BOOLEAN DEFAULT TRUE,
    description TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Таблица: Типы отношений между клиентами
CREATE TABLE relationship_types (
    id SERIAL PRIMARY KEY,
    name VARCHAR(50) NOT NULL UNIQUE,
    description TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- ============================================================================
-- 2. ИНФРАСТРУКТУРА (УЧРЕЖДЕНИЯ И ОБОРУДОВАНИЕ)
-- ============================================================================

-- Таблица: Учреждения (ФСИН, больницы, СИЗО и т.д.)
CREATE TABLE institutions (
    id SERIAL PRIMARY KEY,
    name VARCHAR(200) NOT NULL,
    region_id INTEGER REFERENCES regions(id) ON DELETE SET NULL,
    address TEXT,
    type VARCHAR(50),
    contact_phone VARCHAR(20),
    contact_email VARCHAR(100),
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Таблица: Терминалы связи
CREATE TABLE terminals (
    id SERIAL PRIMARY KEY,
    institution_id INTEGER REFERENCES institutions(id) ON DELETE CASCADE,
    serial_number VARCHAR(50) UNIQUE NOT NULL,
    ip_address INET,
    model VARCHAR(100),
    firmware_version VARCHAR(20),
    status VARCHAR(20) DEFAULT 'OFFLINE',
    last_heartbeat TIMESTAMP,
    installed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- ============================================================================
-- 3. КЛИЕНТЫ (СПЕЦКОНТИНГЕНТ И ВНЕШНИЕ КОНТАКТЫ)
-- ============================================================================

-- Таблица: Все клиенты (и СК и внешние контакты)
CREATE TABLE clients (
    id SERIAL PRIMARY KEY,
    full_name VARCHAR(150) NOT NULL,
    is_special_contingent BOOLEAN DEFAULT FALSE,
    institution_id INTEGER REFERENCES institutions(id) ON DELETE SET NULL,
    internal_id VARCHAR(50),
    passport_series VARCHAR(4),
    passport_number VARCHAR(6),
    passport_issued_by TEXT,
    passport_issue_date DATE,
    phone VARCHAR(20) NOT NULL,
    email VARCHAR(100),
    registration_address TEXT,
    actual_address TEXT,
    birth_date DATE,
    status VARCHAR(20) DEFAULT 'ACTIVE',
    release_date DATE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Таблица: Лицевые счета (только для спецконтингента)
CREATE TABLE accounts (
    id SERIAL PRIMARY KEY,
    client_id INTEGER UNIQUE REFERENCES clients(id) ON DELETE CASCADE,
    tariff_id INTEGER REFERENCES tariffs(id) ON DELETE SET NULL,
    balance NUMERIC(12, 2) DEFAULT 0.00,
    currency VARCHAR(3) DEFAULT 'RUB',
    payment_system_type VARCHAR(50),
    status VARCHAR(20) DEFAULT 'ACTIVE',
    credit_limit NUMERIC(12, 2) DEFAULT 0.00,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Таблица: Привязка акций к счетам (многие-ко-многим)
CREATE TABLE account_promotions (
    account_id INTEGER REFERENCES accounts(id) ON DELETE CASCADE,
    promotion_id INTEGER REFERENCES promotions(id) ON DELETE CASCADE,
    applied_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    expires_at TIMESTAMP,
    is_active BOOLEAN DEFAULT TRUE,
    PRIMARY KEY (account_id, promotion_id)
);

-- ============================================================================
-- 4. РАЗРЕШЕННЫЕ СВЯЗИ (СВОДНАЯ ТАБЛИЦА СК И КОНТАКТОВ)
-- ============================================================================

-- Таблица: Разрешенные связи между спецконтингентом и внешними клиентами
CREATE TABLE allowed_connections (
    id SERIAL PRIMARY KEY,
    sc_client_id INTEGER NOT NULL REFERENCES clients(id) ON DELETE CASCADE,
    external_client_id INTEGER NOT NULL REFERENCES clients(id) ON DELETE CASCADE,
    relationship_type_id INTEGER REFERENCES relationship_types(id) ON DELETE SET NULL,
    relationship_description VARCHAR(100),
    verification_status VARCHAR(20) DEFAULT 'PENDING',
    verified_by_admin_id INTEGER,
    verified_at TIMESTAMP,
    rejection_reason TEXT,
    valid_from DATE,
    valid_until DATE,
    max_calls_per_day INTEGER,
    max_duration_per_call INTEGER,
    is_active BOOLEAN DEFAULT TRUE,
    notes TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- ============================================================================
-- 5. СЕАНСЫ СВЯЗИ И ТРАНЗАКЦИИ
-- ============================================================================

-- Таблица: Сеансы связи (Звонки)
CREATE TABLE sessions (
    id SERIAL PRIMARY KEY,
    terminal_id INTEGER REFERENCES terminals(id) ON DELETE SET NULL,
    account_id INTEGER REFERENCES accounts(id) ON DELETE SET NULL,
    allowed_connection_id INTEGER REFERENCES allowed_connections(id) ON DELETE SET NULL,
    sc_client_id INTEGER REFERENCES clients(id) ON DELETE SET NULL,
    external_client_id INTEGER REFERENCES clients(id) ON DELETE SET NULL,
    scheduled_start TIMESTAMP,
    actual_start TIMESTAMP,
    actual_end TIMESTAMP,
    duration_seconds INTEGER DEFAULT 0,
    cost NUMERIC(12, 2) DEFAULT 0.00,
    status VARCHAR(20) DEFAULT 'COMPLETED',
    failure_reason TEXT,
    call_type VARCHAR(20) DEFAULT 'VOICE',
    recording_url TEXT,
    operator_notes TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Таблица: Транзакции (История платежей)
CREATE TABLE transactions (
    id SERIAL PRIMARY KEY,
    account_id INTEGER REFERENCES accounts(id) ON DELETE CASCADE,
    session_id INTEGER REFERENCES sessions(id) ON DELETE SET NULL,
    type VARCHAR(20) NOT NULL,
    amount NUMERIC(12, 2) NOT NULL,
    balance_before NUMERIC(12, 2) NOT NULL,
    balance_after NUMERIC(12, 2) NOT NULL,
    payment_gateway_ref VARCHAR(100),
    payment_method VARCHAR(50),
    description TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- ============================================================================
-- 6. АДМИНИСТРИРОВАНИЕ И ЛОГИРОВАНИЕ
-- ============================================================================

-- Таблица: Пользователи системы (Администраторы, Операторы)
CREATE TABLE system_users (
    id SERIAL PRIMARY KEY,
    username VARCHAR(50) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    email VARCHAR(100) UNIQUE NOT NULL,
    full_name VARCHAR(150),
    role VARCHAR(30) DEFAULT 'OPERATOR',
    institution_id INTEGER REFERENCES institutions(id) ON DELETE SET NULL,
    is_active BOOLEAN DEFAULT TRUE,
    last_login TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Таблица: Журнал аудита (Логирование действий)
CREATE TABLE audit_log (
    id SERIAL PRIMARY KEY,
    user_id INTEGER REFERENCES system_users(id) ON DELETE SET NULL,
    action VARCHAR(50) NOT NULL,
    table_name VARCHAR(50),
    record_id INTEGER,
    old_values JSONB,
    new_values JSONB,
    ip_address INET,
    user_agent TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- ============================================================================
-- 7. ИНДЕКСЫ ДЛЯ ПРОИЗВОДИТЕЛЬНОСТИ
-- ============================================================================

-- Индексы для клиентов
CREATE INDEX idx_clients_phone ON clients(phone);
CREATE INDEX idx_clients_email ON clients(email);
CREATE INDEX idx_clients_is_sc ON clients(is_special_contingent);
CREATE INDEX idx_clients_institution ON clients(institution_id);
CREATE INDEX idx_clients_status ON clients(status);

-- Индексы для счетов
CREATE INDEX idx_accounts_client ON accounts(client_id);
CREATE INDEX idx_accounts_status ON accounts(status);
CREATE INDEX idx_accounts_balance ON accounts(balance);

-- Индексы для учреждений и терминалов
CREATE INDEX idx_institutions_region ON institutions(region_id);
CREATE INDEX idx_institutions_type ON institutions(type);
CREATE INDEX idx_terminals_institution ON terminals(institution_id);
CREATE INDEX idx_terminals_status ON terminals(status);
CREATE INDEX idx_terminals_serial ON terminals(serial_number);

-- Индексы для разрешенных связей
CREATE INDEX idx_allowed_connections_sc ON allowed_connections(sc_client_id);
CREATE INDEX idx_allowed_connections_external ON allowed_connections(external_client_id);
CREATE INDEX idx_allowed_connections_status ON allowed_connections(verification_status);
CREATE INDEX idx_allowed_connections_active ON allowed_connections(is_active);
CREATE UNIQUE INDEX idx_allowed_connections_unique_pair ON allowed_connections(sc_client_id, external_client_id);

-- Индексы для сеансов связи
CREATE INDEX idx_sessions_account ON sessions(account_id);
CREATE INDEX idx_sessions_terminal ON sessions(terminal_id);
CREATE INDEX idx_sessions_connection ON sessions(allowed_connection_id);
CREATE INDEX idx_sessions_sc_client ON sessions(sc_client_id);
CREATE INDEX idx_sessions_external_client ON sessions(external_client_id);
CREATE INDEX idx_sessions_status ON sessions(status);
CREATE INDEX idx_sessions_actual_start ON sessions(actual_start);
CREATE INDEX idx_sessions_created_at ON sessions(created_at);

-- Индексы для транзакций
CREATE INDEX idx_transactions_account ON transactions(account_id);
CREATE INDEX idx_transactions_session ON transactions(session_id);
CREATE INDEX idx_transactions_type ON transactions(type);
CREATE INDEX idx_transactions_created_at ON transactions(created_at);

-- Составные индексы для частых запросов
CREATE INDEX idx_sessions_account_date ON sessions(account_id, created_at DESC);
CREATE INDEX idx_sessions_sc_date ON sessions(sc_client_id, created_at DESC);
CREATE INDEX idx_transactions_account_date ON transactions(account_id, created_at DESC);

-- Индексы для тарифов и акций
CREATE INDEX idx_tariffs_active ON tariffs(is_active);
CREATE INDEX idx_promotions_active ON promotions(is_active);
CREATE INDEX idx_promotions_dates ON promotions(start_date, end_date);

-- Индексы для аудита
CREATE INDEX idx_audit_log_user ON audit_log(user_id);
CREATE INDEX idx_audit_log_action ON audit_log(action);
CREATE INDEX idx_audit_log_created_at ON audit_log(created_at);
CREATE INDEX idx_audit_log_table ON audit_log(table_name);

-- ============================================================================
-- 8. ФУНКЦИИ ДЛЯ БИЗНЕС-ЛОГИКИ
-- ============================================================================

-- Функция: Автоматическое обновление updated_at
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = CURRENT_TIMESTAMP;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Функция: Проверка что клиент является спецконтингентом
CREATE OR REPLACE FUNCTION validate_client_is_special_contingent(p_client_id INTEGER)
RETURNS BOOLEAN AS $$
DECLARE
    v_is_sc BOOLEAN;
BEGIN
    SELECT is_special_contingent INTO v_is_sc
    FROM clients
    WHERE id = p_client_id;
    
    RETURN COALESCE(v_is_sc, FALSE);
END;
$$ LANGUAGE plpgsql;

-- Функция: Валидация счета (только для спецконтингента)
CREATE OR REPLACE FUNCTION validate_account_sc_client()
RETURNS TRIGGER AS $$
BEGIN
    IF NOT validate_client_is_special_contingent(NEW.client_id) THEN
        RAISE EXCEPTION 'Счет может быть создан только для спецконтингента (client_id=%)', NEW.client_id;
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Функция: Валидация разрешенной связи
CREATE OR REPLACE FUNCTION validate_allowed_connection()
RETURNS TRIGGER AS $$
DECLARE
    v_sc_is_special BOOLEAN;
    v_ext_is_special BOOLEAN;
BEGIN
    -- Проверка что SC является спецконтингентом
    SELECT is_special_contingent INTO v_sc_is_special
    FROM clients WHERE id = NEW.sc_client_id;
    
    IF COALESCE(v_sc_is_special, FALSE) = FALSE THEN
        RAISE EXCEPTION 'sc_client_id должен быть спецконтингентом (id=%)', NEW.sc_client_id;
    END IF;
    
    -- Проверка что внешний контакт НЕ является спецконтингентом
    SELECT is_special_contingent INTO v_ext_is_special
    FROM clients WHERE id = NEW.external_client_id;
    
    IF COALESCE(v_ext_is_special, FALSE) = TRUE THEN
        RAISE EXCEPTION 'external_client_id не может быть спецконтингентом (id=%)', NEW.external_client_id;
    END IF;
    
    -- Проверка что клиенты разные
    IF NEW.sc_client_id = NEW.external_client_id THEN
        RAISE EXCEPTION 'sc_client_id и external_client_id должны быть разными';
    END IF;
    
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Функция: Автоматическое заполнение sc_client_id и external_client_id в sessions
CREATE OR REPLACE FUNCTION fill_session_client_ids()
RETURNS TRIGGER AS $$
BEGIN
    IF NEW.allowed_connection_id IS NOT NULL AND NEW.sc_client_id IS NULL THEN
        SELECT sc_client_id, external_client_id 
        INTO NEW.sc_client_id, NEW.external_client_id
        FROM allowed_connections 
        WHERE id = NEW.allowed_connection_id;
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Функция: Валидация что счет в сессии принадлежит СК
CREATE OR REPLACE FUNCTION validate_session_account()
RETURNS TRIGGER AS $$
DECLARE
    v_client_is_sc BOOLEAN;
BEGIN
    IF NEW.account_id IS NOT NULL THEN
        SELECT is_special_contingent INTO v_client_is_sc
        FROM clients c
        JOIN accounts a ON c.id = a.client_id
        WHERE a.id = NEW.account_id;
        
        IF COALESCE(v_client_is_sc, FALSE) = FALSE THEN
            RAISE EXCEPTION 'Счет в сессии должен принадлежать спецконтингенту';
        END IF;
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Функция: Расчет стоимости сеанса
CREATE OR REPLACE FUNCTION calculate_session_cost(
    p_duration_seconds INTEGER,
    p_tariff_id INTEGER,
    p_account_id INTEGER
)
RETURNS NUMERIC(12, 2) AS $$
DECLARE
    v_price_per_minute NUMERIC(10, 2);
    v_connection_fee NUMERIC(10, 2);
    v_total_cost NUMERIC(12, 2);
    v_discount_percent NUMERIC(10, 2) := 0;
BEGIN
    SELECT price_per_minute, connection_fee 
    INTO v_price_per_minute, v_connection_fee
    FROM tariffs 
    WHERE id = p_tariff_id AND is_active = TRUE;
    
    IF v_price_per_minute IS NULL THEN
        RAISE EXCEPTION 'Тариф не найден или не активен';
    END IF;
    
    v_total_cost := v_connection_fee + (v_price_per_minute * p_duration_seconds / 60.0);
    
    SELECT COALESCE(MAX(ap.value), 0)
    INTO v_discount_percent
    FROM account_promotions ap
    JOIN promotions p ON ap.promotion_id = p.id
    WHERE ap.account_id = p_account_id
      AND ap.is_active = TRUE
      AND p.is_active = TRUE
      AND p.type = 'PERCENT'
      AND CURRENT_TIMESTAMP BETWEEN p.start_date AND p.end_date;
    
    IF v_discount_percent > 0 THEN
        v_total_cost := v_total_cost * (1 - v_discount_percent / 100.0);
    END IF;
    
    RETURN ROUND(v_total_cost, 2);
END;
$$ LANGUAGE plpgsql;

-- Функция: Создание транзакции
CREATE OR REPLACE FUNCTION create_transaction(
    p_account_id INTEGER,
    p_type VARCHAR(20),
    p_amount NUMERIC(12, 2),
    p_session_id INTEGER DEFAULT NULL,
    p_description TEXT DEFAULT NULL
)
RETURNS INTEGER AS $$
DECLARE
    v_balance_before NUMERIC(12, 2);
    v_balance_after NUMERIC(12, 2);
    v_transaction_id INTEGER;
BEGIN
    SELECT balance INTO v_balance_before
    FROM accounts
    WHERE id = p_account_id
    FOR UPDATE;
    
    IF v_balance_before IS NULL THEN
        RAISE EXCEPTION 'Счет не найден';
    END IF;
    
    v_balance_after := v_balance_before + p_amount;
    
    INSERT INTO transactions (
        account_id, session_id, type, amount, 
        balance_before, balance_after, description
    ) VALUES (
        p_account_id, p_session_id, p_type, p_amount,
        v_balance_before, v_balance_after, p_description
    ) RETURNING id INTO v_transaction_id;
    
    UPDATE accounts 
    SET balance = v_balance_after,
        updated_at = CURRENT_TIMESTAMP
    WHERE id = p_account_id;
    
    RETURN v_transaction_id;
END;
$$ LANGUAGE plpgsql;

-- Функция: Завершение сеанса связи
CREATE OR REPLACE FUNCTION complete_session(
    p_session_id INTEGER
)
RETURNS VOID AS $$
DECLARE
    v_session RECORD;
    v_cost NUMERIC(12, 2);
BEGIN
    SELECT * INTO v_session
    FROM sessions
    WHERE id = p_session_id
    FOR UPDATE;
    
    IF v_session IS NULL THEN
        RAISE EXCEPTION 'Сеанс не найден';
    END IF;
    
    IF v_session.status != 'IN_PROGRESS' THEN
        RAISE EXCEPTION 'Сеанс не находится в процессе';
    END IF;
    
    UPDATE sessions
    SET actual_end = CURRENT_TIMESTAMP,
        duration_seconds = EXTRACT(EPOCH FROM (CURRENT_TIMESTAMP - actual_start))::INTEGER,
        status = 'COMPLETED',
        updated_at = CURRENT_TIMESTAMP
    WHERE id = p_session_id;
    
    SELECT duration_seconds INTO v_session.duration_seconds
    FROM sessions WHERE id = p_session_id;
    
    v_cost := calculate_session_cost(
        v_session.duration_seconds,
        (SELECT tariff_id FROM accounts WHERE id = v_session.account_id),
        v_session.account_id
    );
    
    UPDATE sessions
    SET cost = v_cost,
        updated_at = CURRENT_TIMESTAMP
    WHERE id = p_session_id;
    
    PERFORM create_transaction(
        v_session.account_id,
        'CHARGE',
        -v_cost,
        p_session_id,
        'Оплата сеанса связи #' || p_session_id
    );
END;
$$ LANGUAGE plpgsql;

-- Функция: Проверка лимитов для разрешенной связи
CREATE OR REPLACE FUNCTION check_connection_limits(
    p_connection_id INTEGER,
    p_date DATE DEFAULT CURRENT_DATE
)
RETURNS TABLE (
    calls_today INTEGER,
    max_calls INTEGER,
    can_call BOOLEAN,
    reason TEXT
) AS $$
DECLARE
    v_max_calls INTEGER;
    v_calls_today INTEGER;
BEGIN
    SELECT max_calls_per_day INTO v_max_calls
    FROM allowed_connections
    WHERE id = p_connection_id;
    
    SELECT COUNT(*) INTO v_calls_today
    FROM sessions
    WHERE allowed_connection_id = p_connection_id
      AND DATE(actual_start) = p_date
      AND status = 'COMPLETED';
    
    IF v_max_calls IS NULL THEN
        calls_today := v_calls_today;
        max_calls := NULL;
        can_call := TRUE;
        reason := 'Лимит не установлен';
    ELSIF v_calls_today >= v_max_calls THEN
        calls_today := v_calls_today;
        max_calls := v_max_calls;
        can_call := FALSE;
        reason := 'Превышен дневной лимит звонков';
    ELSE
        calls_today := v_calls_today;
        max_calls := v_max_calls;
        can_call := TRUE;
        reason := 'Лимит не превышен';
    END IF;
    
    RETURN;
END;
$$ LANGUAGE plpgsql;

-- Функция: Создание нового сеанса связи
CREATE OR REPLACE FUNCTION create_session(
    p_terminal_id INTEGER,
    p_allowed_connection_id INTEGER,
    p_scheduled_start TIMESTAMP DEFAULT NULL
)
RETURNS INTEGER AS $$
DECLARE
    v_session_id INTEGER;
    v_account_id INTEGER;
    v_sc_client_id INTEGER;
    v_limits RECORD;
BEGIN
    SELECT sc_client_id INTO v_sc_client_id
    FROM allowed_connections
    WHERE id = p_allowed_connection_id;
    
    IF v_sc_client_id IS NULL THEN
        RAISE EXCEPTION 'Разрешенная связь не найдена';
    END IF;
    
    SELECT id INTO v_account_id
    FROM accounts
    WHERE client_id = v_sc_client_id AND status = 'ACTIVE';
    
    IF v_account_id IS NULL THEN
        RAISE EXCEPTION 'У спецконтингента нет активного счета';
    END IF;
    
    SELECT * INTO v_limits FROM check_connection_limits(p_allowed_connection_id);
    
    IF v_limits.can_call = FALSE THEN
        RAISE EXCEPTION '%', v_limits.reason;
    END IF;
    
    INSERT INTO sessions (
        terminal_id,
        account_id,
        allowed_connection_id,
        sc_client_id,
        external_client_id,
        scheduled_start,
        status
    )
    SELECT
        p_terminal_id,
        v_account_id,
        p_allowed_connection_id,
        ac.sc_client_id,
        ac.external_client_id,
        p_scheduled_start,
        CASE WHEN p_scheduled_start IS NULL THEN 'IN_PROGRESS' ELSE 'SCHEDULED' END
    FROM allowed_connections ac
    WHERE ac.id = p_allowed_connection_id
    RETURNING id INTO v_session_id;
    
    RETURN v_session_id;
END;
$$ LANGUAGE plpgsql;

-- ============================================================================
-- 9. ТРИГГЕРЫ
-- ============================================================================

-- Триггеры для автоматического обновления updated_at
CREATE TRIGGER trg_regions_updated_at BEFORE UPDATE ON regions
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER trg_tariffs_updated_at BEFORE UPDATE ON tariffs
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER trg_promotions_updated_at BEFORE UPDATE ON promotions
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER trg_institutions_updated_at BEFORE UPDATE ON institutions
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER trg_terminals_updated_at BEFORE UPDATE ON terminals
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER trg_clients_updated_at BEFORE UPDATE ON clients
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER trg_accounts_updated_at BEFORE UPDATE ON accounts
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER trg_allowed_connections_updated_at BEFORE UPDATE ON allowed_connections
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER trg_sessions_updated_at BEFORE UPDATE ON sessions
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER trg_system_users_updated_at BEFORE UPDATE ON system_users
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

-- Триггер для валидации счета (только для спецконтингента)
CREATE TRIGGER trg_accounts_validate_sc 
    BEFORE INSERT OR UPDATE ON accounts
    FOR EACH ROW EXECUTE FUNCTION validate_account_sc_client();

-- Триггер для валидации разрешенной связи
CREATE TRIGGER trg_allowed_connections_validate 
    BEFORE INSERT OR UPDATE ON allowed_connections
    FOR EACH ROW EXECUTE FUNCTION validate_allowed_connection();

-- Триггер для автоматического заполнения sc_client_id и external_client_id в sessions
CREATE TRIGGER trg_sessions_fill_client_ids 
    BEFORE INSERT OR UPDATE OF allowed_connection_id ON sessions
    FOR EACH ROW EXECUTE FUNCTION fill_session_client_ids();

-- Триггер для валидации счета в сессии
CREATE TRIGGER trg_sessions_validate_account 
    BEFORE INSERT OR UPDATE ON sessions
    FOR EACH ROW EXECUTE FUNCTION validate_session_account();

-- ============================================================================
-- 10. ПРЕДСТАВЛЕНИЯ (VIEWS)
-- ============================================================================

-- Представление: Все спецконтингенты с счетами
CREATE OR REPLACE VIEW v_special_contingents AS
SELECT 
    c.id AS client_id,
    c.full_name,
    c.internal_id,
    c.phone,
    c.status AS client_status,
    i.name AS institution_name,
    i.type AS institution_type,
    a.id AS account_id,
    a.balance,
    a.status AS account_status,
    t.name AS tariff_name,
    t.price_per_minute
FROM clients c
JOIN institutions i ON c.institution_id = i.id
LEFT JOIN accounts a ON c.id = a.client_id
LEFT JOIN tariffs t ON a.tariff_id = t.id
WHERE c.is_special_contingent = TRUE;

-- Представление: Активные счета спецконтингента
CREATE OR REPLACE VIEW v_active_accounts AS
SELECT 
    a.id AS account_id,
    c.id AS client_id,
    c.full_name AS client_name,
    c.internal_id,
    c.phone AS client_phone,
    i.name AS institution_name,
    a.balance,
    a.status AS account_status,
    t.name AS tariff_name,
    t.price_per_minute
FROM accounts a
JOIN clients c ON a.client_id = c.id
JOIN institutions i ON c.institution_id = i.id
LEFT JOIN tariffs t ON a.tariff_id = t.id
WHERE a.status = 'ACTIVE' AND c.is_special_contingent = TRUE;

-- Представление: Разрешенные связи с подробной информацией
CREATE OR REPLACE VIEW v_allowed_connections_detail AS
SELECT 
    ac.id AS connection_id,
    sc.id AS sc_client_id,
    sc.full_name AS sc_full_name,
    sc.internal_id AS sc_internal_id,
    i.name AS institution_name,
    ext.id AS external_client_id,
    ext.full_name AS external_full_name,
    ext.phone AS external_phone,
    rt.name AS relationship_type,
    ac.relationship_description,
    ac.verification_status,
    ac.verified_at,
    ac.valid_from,
    ac.valid_until,
    ac.is_active,
    ac.max_calls_per_day,
    ac.max_duration_per_call
FROM allowed_connections ac
JOIN clients sc ON ac.sc_client_id = sc.id
JOIN clients ext ON ac.external_client_id = ext.id
JOIN institutions i ON sc.institution_id = i.id
LEFT JOIN relationship_types rt ON ac.relationship_type_id = rt.id;

-- Представление: Статистика по сеансам по учреждениям
CREATE OR REPLACE VIEW v_institution_session_stats AS
SELECT 
    i.id AS institution_id,
    i.name AS institution_name,
    COUNT(s.id) AS total_sessions,
    SUM(s.duration_seconds) AS total_duration_seconds,
    SUM(s.cost) AS total_revenue,
    AVG(s.duration_seconds) AS avg_duration_seconds,
    COUNT(DISTINCT s.sc_client_id) AS unique_sc_clients,
    COUNT(DISTINCT s.external_client_id) AS unique_external_clients
FROM institutions i
LEFT JOIN clients c ON i.id = c.institution_id AND c.is_special_contingent = TRUE
LEFT JOIN sessions s ON c.id = s.sc_client_id
WHERE s.status = 'COMPLETED'
GROUP BY i.id, i.name;

-- Представление: Статистика по спецконтингенту
CREATE OR REPLACE VIEW v_sc_statistics AS
SELECT 
    c.id AS sc_client_id,
    c.full_name,
    c.internal_id,
    i.name AS institution_name,
    a.balance,
    COUNT(s.id) AS total_calls,
    SUM(s.duration_seconds) AS total_duration_seconds,
    SUM(s.cost) AS total_spent,
    MAX(s.actual_start) AS last_call_date
FROM clients c
JOIN institutions i ON c.institution_id = i.id
LEFT JOIN accounts a ON c.id = a.client_id
LEFT JOIN sessions s ON c.id = s.sc_client_id AND s.status = 'COMPLETED'
WHERE c.is_special_contingent = TRUE
GROUP BY c.id, c.full_name, c.internal_id, i.name, a.balance;

-- Представление: Активные разрешенные связи
CREATE OR REPLACE VIEW v_active_connections AS
SELECT 
    ac.id AS connection_id,
    sc.full_name AS sc_name,
    sc.internal_id,
    ext.full_name AS external_name,
    ext.phone AS external_phone,
    ac.relationship_description,
    ac.valid_from,
    ac.valid_until,
    ac.max_calls_per_day,
    ac.max_duration_per_call
FROM allowed_connections ac
JOIN clients sc ON ac.sc_client_id = sc.id
JOIN clients ext ON ac.external_client_id = ext.id
WHERE ac.verification_status = 'APPROVED' 
  AND ac.is_active = TRUE
  AND (ac.valid_until IS NULL OR ac.valid_until >= CURRENT_DATE);

-- ============================================================================
-- 11. НАЧАЛЬНЫЕ ДАННЫЕ (SEED DATA)
-- ============================================================================

-- Регионы
INSERT INTO regions (name, code) VALUES
('Москва', '77'),
('Московская область', '50'),
('Санкт-Петербург', '78'),
('Ленинградская область', '47'),
('Свердловская область', '66');

-- Тарифы
INSERT INTO tariffs (name, price_per_minute, connection_fee, description) VALUES
('Базовый', 5.00, 0.00, 'Стандартный тариф для всех клиентов'),
('Премиум', 3.50, 0.00, 'Сниженная стоимость минуты'),
('Видеосвязь', 10.00, 5.00, 'Тариф для видеозвонков');

-- Акции
INSERT INTO promotions (name, type, value, start_date, end_date, description) VALUES
('Новый клиент', 'PERCENT', 10.00, '2024-01-01', '2024-12-31', 'Скидка 10% для новых клиентов'),
('Праздничная', 'FIXED', 50.00, '2024-12-20', '2025-01-10', 'Бонус 50 рублей на счет');

-- Типы отношений
INSERT INTO relationship_types (name, description) VALUES
('Супруг/Супруга', 'Официальный брак'),
('Родитель', 'Отец или мать'),
('Ребенок', 'Сын или дочь'),
('Брат/Сестра', 'Родные или сводные'),
('Адвокат', 'Юридический представитель'),
('Другой родственник', 'Иные родственные связи'),
('Иное', 'Не является родственником');

-- Учреждения
INSERT INTO institutions (name, region_id, address, type, contact_phone) VALUES
('ИК-5 УФСИН', 1, 'г. Москва, ул. Примерная, 1', 'ИК', '+7 495 000-00-01'),
('СИЗО-1', 1, 'г. Москва, ул. Следственная, 5', 'СИЗО', '+7 495 000-00-02'),
('Психиатрическая клиника №1', 3, 'г. Санкт-Петербург, ул. Медицинская, 10', 'КЛИНИКА', '+7 812 000-00-03');

-- Терминалы
INSERT INTO terminals (institution_id, serial_number, ip_address, model, status) VALUES
(1, 'TERM-001-2024', '192.168.1.101', 'VideoLink Pro', 'ONLINE'),
(1, 'TERM-002-2024', '192.168.1.102', 'VideoLink Pro', 'ONLINE'),
(2, 'TERM-003-2024', '192.168.2.101', 'VoiceBox Standard', 'ONLINE'),
(3, 'TERM-004-2024', '192.168.3.101', 'VideoLink Pro', 'MAINTENANCE');

-- Клиенты - Спецконтингент (is_special_contingent = TRUE)
INSERT INTO clients (full_name, is_special_contingent, institution_id, internal_id, phone, birth_date, status) VALUES
('Иванов Иван Иванович', TRUE, 1, 'Дело-12345', '+7 900 000-00-01', '1980-05-15', 'ACTIVE'),
('Петров Петр Петрович', TRUE, 2, 'Дело-67890', '+7 900 000-00-02', '1975-08-20', 'ACTIVE'),
('Сидоров Алексей Петрович', TRUE, 3, 'Палата-305', '+7 900 000-00-03', '1990-03-10', 'ACTIVE');

-- Клиенты - Внешние контакты (is_special_contingent = FALSE)
INSERT INTO clients (full_name, is_special_contingent, phone, email, birth_date) VALUES
('Иванова Мария Сергеевна', FALSE, '+7 900 111-11-11', 'ivanova@example.com', '1982-07-20'),
('Петрова Анна Ивановна', FALSE, '+7 900 222-22-22', 'petrova@example.com', '1978-12-05'),
('Сидорова Елена Алексеевна', FALSE, '+7 900 333-33-33', 'sidorova@example.com', '1992-01-15'),
('Адвокат Смирнов', FALSE, '+7 900 444-44-44', 'smirnov@lawyer.com', '1970-03-25');

-- Счета (только для спецконтингента)
INSERT INTO accounts (client_id, tariff_id, balance, payment_system_type, status) VALUES
(1, 1, 500.00, 'BANK_CARD', 'ACTIVE'),
(2, 2, 1000.00, 'SBP', 'ACTIVE'),
(3, 1, 250.00, 'CASH_TERMINAL', 'ACTIVE');

-- Разрешенные связи
INSERT INTO allowed_connections (sc_client_id, external_client_id, relationship_type_id, relationship_description, verification_status, verified_at, valid_from, valid_until) VALUES
(1, 4, 1, 'Супруга', 'APPROVED', CURRENT_TIMESTAMP, '2024-01-01', '2025-12-31'),
(2, 5, 1, 'Супруга', 'APPROVED', CURRENT_TIMESTAMP, '2024-01-01', '2025-12-31'),
(3, 6, 2, 'Мать', 'APPROVED', CURRENT_TIMESTAMP, '2024-01-01', '2025-12-31'),
(1, 8, 5, 'Адвокат', 'APPROVED', CURRENT_TIMESTAMP, '2024-01-01', '2025-12-31');

-- Пользователи системы
INSERT INTO system_users (username, password_hash, email, full_name, role) VALUES
('admin', '$2b$12$example_hash_here', 'admin@system.com', 'Администратор Системы', 'ADMIN'),
('operator1', '$2b$12$example_hash_here', 'operator1@system.com', 'Оператор 1', 'OPERATOR'),
('auditor1', '$2b$12$example_hash_here', 'auditor1@system.com', 'Аудитор 1', 'AUDITOR'),
('verifier1', '$2b$12$example_hash_here', 'verifier1@system.com', 'Верификатор 1', 'VERIFIER');

-- ============================================================================
-- КОНЕЦ СКРИПТА
-- ============================================================================

-- Накатить скрипт
-- psql -d p_tel -f create_schema_db_p_tel.sql
/* ==========================================================================

p_tel=# \dt+
                                            Список отношений
 Схема  |         Имя         |   Тип   | Владелец |  Хранение  | Метод доступа |   Размер   | Описание 
--------+---------------------+---------+----------+------------+---------------+------------+----------
 public | account_promotions  | таблица | postgres | постоянное | heap          | 0 bytes    | 
 public | accounts            | таблица | postgres | постоянное | heap          | 8192 bytes | 
 public | allowed_connections | таблица | postgres | постоянное | heap          | 16 kB      | 
 public | audit_log           | таблица | postgres | постоянное | heap          | 8192 bytes | 
 public | clients             | таблица | postgres | постоянное | heap          | 16 kB      | 
 public | institutions        | таблица | postgres | постоянное | heap          | 16 kB      | 
 public | promotions          | таблица | postgres | постоянное | heap          | 16 kB      | 
 public | regions             | таблица | postgres | постоянное | heap          | 8192 bytes | 
 public | relationship_types  | таблица | postgres | постоянное | heap          | 16 kB      | 
 public | sessions            | таблица | postgres | постоянное | heap          | 8192 bytes | 
 public | system_users        | таблица | postgres | постоянное | heap          | 16 kB      | 
 public | tariffs             | таблица | postgres | постоянное | heap          | 16 kB      | 
 public | terminals           | таблица | postgres | постоянное | heap          | 16 kB      | 
 public | transactions        | таблица | postgres | постоянное | heap          | 8192 bytes | 
(14 строк)
   ========================================================================== */
