### Демонстрация работы индексов
#### Создание индекса
В таблице `terminals` создать индекс на поле `serial_number`
```
CREATE INDEX idx_terminals_serial ON terminals(serial_number);
```
#### План запроса
```
EXPLAIN SELECT * FROM terminals WHERE serial_number = 'TERM-001-2024';

                                       QUERY PLAN                                       
----------------------------------------------------------------------------------------
 Index Scan using idx_terminals_serial on terminals  (cost=0.14..8.16 rows=1 width=524)
   Index Cond: ((serial_number)::text = 'TERM-001-2024'::text)
(2 строки)
```
#### Реализация индекса для полнотекстового поиска
```
CREATE INDEX idx_terminals_model_fts ON terminals USING GIN (to_tsvector('english', model));

EXPLAIN SELECT * FROM terminals WHERE to_tsvector('english', model) @@ to_tsquery('english', 'VideoLink');

                                             QUERY PLAN                                             
----------------------------------------------------------------------------------------------------
 Bitmap Heap Scan on terminals  (cost=8.52..12.78 rows=1 width=524)
   Recheck Cond: (to_tsvector('english'::regconfig, (model)::text) @@ '''videolink'''::tsquery)
   ->  Bitmap Index Scan on idx_terminals_model_fts  (cost=0.00..8.52 rows=1 width=0)
         Index Cond: (to_tsvector('english'::regconfig, (model)::text) @@ '''videolink'''::tsquery)
(4 строки)
```
#### Частичный индекс и индекс на поле с функцией
Частичный индекс — это индекс, который создаётся только для строк, удовлетворяющих определённому условию (WHERE)
```
CREATE INDEX idx_terminals_offline ON terminals (institution_id) WHERE status = 'MAINTENANCE';
SELECT * FROM terminals WHERE status = 'MAINTENANCE' AND institution_id = 1;
```
Функциональный индекс — это индекс, который создаётся на результат применения функции к колонке.
```
CREATE INDEX idx_terminals_model_lower ON terminals (LOWER(model));
SELECT * FROM terminals WHERE LOWER(model) = 'videolink pro';
```
#### Составной индекс
Индекс на несколько колонок
```
CREATE INDEX idx_terminals_inst_status ON terminals (institution_id, status);

-- Использует индекс (обе колонки)
SELECT * FROM terminals WHERE institution_id = 2 AND status = 'MAINTENANCE';

-- Использует индекс (только первая колонка)
SELECT * FROM terminals WHERE institution_id = 1;

-- Не использует индекс (только вторая колонка)
SELECT * FROM terminals WHERE status = 'MAINTENANCE';
```
