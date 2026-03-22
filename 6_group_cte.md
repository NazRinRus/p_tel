### Посчитать кол-во очков по всем игрокам за текущий год и за предыдущий
#### Структура таблиц и наполнение данными
```
CREATE TABLE statistic(
    player_name VARCHAR(100) NOT NULL,
    player_id INT NOT NULL,
    year_game SMALLINT NOT NULL CHECK (year_game > 0),
    points DECIMAL(12,2) CHECK (points >= 0),
    PRIMARY KEY (player_name,year_game)
);

INSERT INTO
    statistic(player_name, player_id, year_game, points)
VALUES
    ('Mike',1,2018,18),
    ('Jack',2,2018,14),
    ('Jackie',3,2018,30),
    ('Jet',4,2018,30),
    ('Luke',1,2019,16),
    ('Mike',2,2019,14),
    ('Jack',3,2019,15),
    ('Jackie',4,2019,28),
    ('Jet',5,2019,25),
    ('Luke',1,2020,19),
    ('Mike',2,2020,17),
    ('Jack',3,2020,18),
    ('Jackie',4,2020,29),
    ('Jet',5,2020,27);
```
#### Написать запрос суммы очков с группировкой и сортировкой по годам
```
SELECT 
    year_game,
    SUM(points) AS total_points,
    COUNT(*) AS games_count,
    ROUND(AVG(points), 2) AS avg_points
FROM statistic
GROUP BY year_game
ORDER BY year_game;

 year_game | total_points | games_count | avg_points 
-----------+--------------+-------------+------------
      2018 |        92.00 |           4 |      23.00
      2019 |        98.00 |           5 |      19.60
      2020 |       110.00 |           5 |      22.00
(3 строки)
```
#### Написать cte показывающее тоже самое
```
WITH yearly_stats AS (
    SELECT 
        year_game,
        SUM(points) AS total_points,
        COUNT(*) AS games_count,
        ROUND(AVG(points), 2) AS avg_points
    FROM statistic
    GROUP BY year_game
)
SELECT * 
FROM yearly_stats
ORDER BY year_game;

 year_game | total_points | games_count | avg_points 
-----------+--------------+-------------+------------
      2018 |        92.00 |           4 |      23.00
      2019 |        98.00 |           5 |      19.60
      2020 |       110.00 |           5 |      22.00
(3 строки)
```
#### Используя функцию LAG вывести кол-во очков по всем игрокам за текущий год и за предыдущий
```
p_tel=# SELECT 
    player_name,
    player_id,
    year_game,
    points AS current_year_points,
    LAG(points, 1) OVER (
        PARTITION BY player_name 
        ORDER BY year_game
    ) AS previous_year_points,
    points - LAG(points, 1) OVER (
        PARTITION BY player_name 
        ORDER BY year_game
    ) AS points_difference
FROM statistic
ORDER BY player_name, year_game;
 player_name | player_id | year_game | current_year_points | previous_year_points | points_difference 
-------------+-----------+-----------+---------------------+----------------------+-------------------
 Jack        |         2 |      2018 |               14.00 |                      |                  
 Jack        |         3 |      2019 |               15.00 |                14.00 |              1.00
 Jack        |         3 |      2020 |               18.00 |                15.00 |              3.00
 Jackie      |         3 |      2018 |               30.00 |                      |                  
 Jackie      |         4 |      2019 |               28.00 |                30.00 |             -2.00
 Jackie      |         4 |      2020 |               29.00 |                28.00 |              1.00
 Jet         |         4 |      2018 |               30.00 |                      |                  
 Jet         |         5 |      2019 |               25.00 |                30.00 |             -5.00
 Jet         |         5 |      2020 |               27.00 |                25.00 |              2.00
 Luke        |         1 |      2019 |               16.00 |                      |                  
 Luke        |         1 |      2020 |               19.00 |                16.00 |              3.00
 Mike        |         1 |      2018 |               18.00 |                      |                  
 Mike        |         2 |      2019 |               14.00 |                18.00 |             -4.00
 Mike        |         2 |      2020 |               17.00 |                14.00 |              3.00
(14 строк)
```
