# Задание 1. B-tree индексы в PostgreSQL

1. Запустите БД через docker compose в ./src/docker-compose.yml:

2. Выполните запрос для поиска книги с названием 'Oracle Core' и получите план выполнения:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books WHERE title = 'Oracle Core';
   ```
   
   *План выполнения:*
   ```sql
    Seq Scan on t_books  (cost=0.00..3100.00 rows=1 width=33) (actual time=19.318..19.323 rows=1 loops=1) 
    Filter: ((title)::text = 'Oracle Core'::text)
    Rows Removed by Filter: 149999
    Planning Time: 0.481 ms
    Execution Time: 19.377 ms
    ```

   
   *Объясните результат:*
   Результат команды EXPLAIN ANALYZE показывает, что PostgreSQL выполняет последовательное сканирование таблицы t_books для поиска строки, где столбец title равен 'Oracle Core'. Оценка стоимости выполнения запроса составляет от 0.00 до 3100.00, и PostgreSQL ожидает найти 1 строку размером 33 байта. Фактически запрос выполнялся 1 раз и занял 19.318-19.323 миллисекунд, вернув 1 строку. Фильтр по столбцу title удалил 149999 строк, которые не соответствовали условию. Время планирования запроса составило 0.481 миллисекунды, а общее время выполнения - 19.377 миллисекунд.

3. Создайте B-tree индексы:
   ```sql
   CREATE INDEX t_books_title_idx ON t_books(title);
   CREATE INDEX t_books_active_idx ON t_books(is_active);
   ```
   
   *Результат:*
   успех

4. Проверьте информацию о созданных индексах:
   ```sql
   SELECT schemaname, tablename, indexname, indexdef
   FROM pg_catalog.pg_indexes
   WHERE tablename = 't_books';
   ```
   
   *Результат:*
   Вывело столбцы: table_name (t_books), indexname (название индексов, в нашем случае t_books_title_idx и t_books_active_idx, и t_books_id_pk) и indexdef (код sql как создавался индекс)
   
   *Объясните результат:*
   В t_books теперь есть три индекса: уникальный индекс t_books_id_pk на столбце book_id, а также два созданных индекса t_books_title_idx и t_books_active_idx на столбцах title и is_active соответственно. Все индексы используют тип индекса btree.

5. Обновите статистику таблицы:
   ```sql
   ANALYZE t_books;
   ```
   
   *Результат:*
   ничего не вывело. analyze просто собирает статистику о таблице, чтобы оптимизатору запросов было легче принимать решение, как лучше выполнить запрос

6. Выполните запрос для поиска книги 'Oracle Core' и получите план выполнения:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books WHERE title = 'Oracle Core';
   ```
   
   *План выполнения:*
   ```sql
    Index Scan using t_books_title_idx on t_books  (cost=0.42..8.44 rows=1 width=33) (actual time=3.417..3.420 rows=1 loops=1)
    Index Cond: ((title)::text = 'Oracle Core'::text)
    Planning Time: 4.452 ms
    Execution Time: 3.445 ms
    ```
   
   *Объясните результат:*
   PostgreSQL использует индекс t_books_title_idx для поиска строки в таблице t_books, где столбец title равен 'Oracle Core'. Оценка стоимости выполнения запроса составляет от 0.42 до 8.44, и PostgreSQL ожидает найти 1 строку размером 33 байта. Фактически запрос выполнялся 1 раз и занял 3.417-3.420 миллисекунд, вернув 1 строку. Условие индекса проверяет, совпадает ли значение столбца title со строкой 'Oracle Core'. Время планирования запроса составило 4.452 миллисекунды, а общее время выполнения — 3.445 миллисекунд.

7. Выполните запрос для поиска книги по book_id и получите план выполнения:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books WHERE book_id = 18;
   ```
   
   *План выполнения:*
   ```sql
    Index Scan using t_books_id_pk on t_books  (cost=0.42..8.44 rows=1 width=33) (actual time=0.019..0.020 rows=1 loops=1)
    Index Cond: (book_id = 18)
    Planning Time: 0.059 ms
    Execution Time: 0.033 ms
    ```

   
   *Объясните результат:*
   Результат команды EXPLAIN ANALYZE показывает, что PostgreSQL использует индекс t_books_id_pk для поиска строки в таблице t_books, где столбец book_id равен 18. Оценка стоимости выполнения запроса составляет от 0.42 до 8.44, и PostgreSQL ожидает найти 1 строку размером 33 байта. Фактически запрос выполнялся 1 раз и занял 0.019-0.020 миллисекунд, вернув 1 строку. Условие индекса проверяет, совпадает ли значение столбца book_id с числом 18. Время планирования запроса составило 0.059 миллисекунды, а общее время выполнения — 0.033 миллисекунд.

8. Выполните запрос для поиска активных книг и получите план выполнения:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books WHERE is_active = true;
   ```
   
   *План выполнения:*
   ```sql
    Seq Scan on t_books  (cost=0.00..2725.00 rows=76270 width=33) (actual time=0.013..25.899 rows=75161 loops=1)
    Filter: is_active
    Rows Removed by Filter: 74839
    Planning Time: 0.129 ms
    Execution Time: 29.670 ms
    ```

   
   *Объясните результат:*
   то же самое только со столбцом is_active

9. Посчитайте количество строк и уникальных значений:
   ```sql
   SELECT 
       COUNT(*) as total_rows,
       COUNT(DISTINCT title) as unique_titles,
       COUNT(DISTINCT category) as unique_categories,
       COUNT(DISTINCT author) as unique_authors
   FROM t_books;
   ```
   
   *Результат:*
   total_rows - 150000, unique_titles - 150000, unique_categories - 6, unique_authors - 1003


10. Удалите созданные индексы:
    ```sql
    DROP INDEX t_books_title_idx;
    DROP INDEX t_books_active_idx;
    ```
    
    *Результат:*
    удалил

11. Основываясь на предыдущих результатах, создайте индексы для оптимизации следующих запросов:
    a. `WHERE title = $1 AND category = $2`
    b. `WHERE title = $1`
    c. `WHERE category = $1 AND author = $2`
    d. `WHERE author = $1 AND book_id = $2`
    
    *Созданные индексы:*
    a. 
    ```sql
    CREATE INDEX idx_title_category ON t_books(title, category);
    ```

    b. 
    ```sql
    CREATE INDEX idx_title ON t_books(title);
    ```
    c. 
    ```sql
    CREATE INDEX idx_category_author ON t_books(category, author);
    ```
    d. 
    ```sql
    CREATE INDEX idx_author_book_id ON t_books(author, book_id);
    ```
    
    *Объясните ваше решение:*
    каждый запрос просто создаёт индекс для оптимизации запросов по нужным столбцам

12. Протестируйте созданные индексы.
    
    *Результаты тестов:*
    a. 
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books WHERE title = 'Oracle Core' AND category = 'Databases';
    ```
    ```sql
    Index Scan using idx_title_category on t_books  (cost=0.42..8.44 rows=1 width=33) (actual time=0.022..0.023 rows=1 loops=1)
    Index Cond: (((title)::text = 'Oracle Core'::text) AND ((category)::text = 'Databases'::text))
    Planning Time: 0.064 ms
    Execution Time: 0.035 ms

    ```
    b. 
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books WHERE title = 'Oracle Core';
    ```
    ```sql
    Index Scan using idx_title on t_books  (cost=0.42..8.44 rows=1 width=33) (actual time=0.027..0.028 rows=1 loops=1)
    Index Cond: ((title)::text = 'Oracle Core'::text)
    Planning Time: 0.234 ms
    Execution Time: 0.044 ms
    ```
    c. 
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books WHERE category = 'Databases' AND author = 'Jonathan Lewis';

    ```

    ```sql
    Index Scan using idx_category_author on t_books  (cost=0.29..8.31 rows=1 width=33) (actual time=0.022..0.024 rows=1 loops=1)
    Index Cond: (((category)::text = 'Databases'::text) AND ((author)::text = 'Jonathan Lewis'::text))
    Planning Time: 0.282 ms
    Execution Time: 0.043 ms

    ```
    
    d.
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books WHERE book_id = 3001 AND author = 'Jonathan Lewis';
    ```

    ```sql
    Index Scan using idx_author_book_id on t_books  (cost=0.42..8.44 rows=1 width=33) (actual time=0.027..0.028 rows=1 loops=1)
    Index Cond: (((author)::text = 'Jonathan Lewis'::text) AND (book_id = 3001))
    Planning Time: 0.294 ms
    Execution Time: 0.044 ms
    ```
    
    *Объясните результаты:*
    Видно, что запрос SELECT без индекса в 1 задании занимает 19ms, в то время запросы с индексами и несколькими условиями занимают почти в 450 раз меньше времени

13. Выполните регистронезависимый поиск по началу названия:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books WHERE title ILIKE 'Relational%';
    ```
    
    *План выполнения:*
    ```sql
    Seq Scan on t_books  (cost=0.00..3100.00 rows=15 width=33) (actual time=85.155..85.156 rows=0 loops=1)
    Filter: ((title)::text ~~* 'Relational%'::text)
    Rows Removed by Filter: 150000
    Planning Time: 1.333 ms
    Execution Time: 85.177 ms
    ```
    
    *Объясните результат:*
    Оценка стоимости выполнения запроса составляет от 0.00 до 3100.00, и PostgreSQL ожидает найти 15 строк. Фактически запрос выполнялся 1 раз и занял 85.155-85.156 миллисекунд, но не вернул ни одной строки, так как фильтр удалил все 150000 строк таблицы. Время планирования запроса составило 1.333 миллисекунды, а общее время выполнения — 85.177 миллисекунд.

14. Создайте функциональный индекс:
    ```sql
    CREATE INDEX t_books_up_title_idx ON t_books(UPPER(title));
    ```
    
    *Результат:*
    success

15. Выполните запрос из шага 13 с использованием UPPER:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books WHERE UPPER(title) LIKE 'RELATIONAL%';
    ```
    
    *План выполнения:*
    ```sql
    Seq Scan on t_books  (cost=0.00..3475.00 rows=750 width=33) (actual time=68.081..68.083 rows=0 loops=1)
    Filter: (upper((title)::text) ~~ 'RELATIONAL%'::text)
    Rows Removed by Filter: 150000
    Planning Time: 7.917 ms
    Execution Time: 68.109 ms
    ```
    
    *Объясните результат:*
    Оценка стоимости выполнения запроса составляет от 0.00 до 3475.00, и PostgreSQL ожидает найти 750 строк. Фактически запрос выполнялся 1 раз и занял 68.081-68.083 миллисекунд, но не вернул ни одной строки, так как фильтр удалил все 150000 строк таблицы. Время планирования запроса составило 7.917 миллисекунды, а общее время выполнения — 68.109 миллисекунд. Несмотря на создание индекса t_books_up_title_idx на столбце UPPER(title), PostgreSQL не использовал его для этого запроса, что привело к аналогичному времени выполнения, как и без индекса.

16. Выполните поиск подстроки:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books WHERE title ILIKE '%Core%';
    ```
    
    *План выполнения:*
    ```sql
    Seq Scan on t_books  (cost=0.00..3100.00 rows=15 width=33) (actual time=83.954..83.959 rows=1 loops=1)
    Filter: ((title)::text ~~* '%Core%'::text)
    Rows Removed by Filter: 149999
    Planning Time: 0.291 ms
    Execution Time: 83.979 ms
    ```

17. Попробуйте удалить все индексы:
    ```sql
    DO $$ 
    DECLARE
        r RECORD;
    BEGIN
        FOR r IN (SELECT indexname FROM pg_indexes 
                  WHERE tablename = 't_books' 
                  AND indexname != 'books_pkey')
        LOOP
            EXECUTE 'DROP INDEX ' || r.indexname;
        END LOOP;
    END $$;
    ```
    
    *Результат:*
    ок

18. Создайте индекс для оптимизации суффиксного поиска:
    ```sql
    -- Вариант 1: с reverse()
    CREATE INDEX t_books_rev_title_idx ON t_books(reverse(title));
    
    -- Вариант 2: с триграммами
    CREATE EXTENSION IF NOT EXISTS pg_trgm;
    CREATE INDEX t_books_trgm_idx ON t_books USING gin (title gin_trgm_ops);
    ```
    
    *Результаты тестов:*
    [Вставьте планы выполнения для обоих вариантов]
    
    *Объясните результаты:*
    [Ваше объяснение]

19. Выполните поиск по точному совпадению:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books WHERE title = 'Oracle Core';
    ```
    
    *План выполнения:*
    ```sql
    Bitmap Heap Scan on t_books  (cost=116.57..120.58 rows=1 width=33) (actual time=0.086..0.088 rows=1 loops=1)
    Recheck Cond: ((title)::text = 'Oracle Core'::text)
    Heap Blocks: exact=1
    ->  Bitmap Index Scan on t_books_trgm_idx  (cost=0.00..116.57 rows=1 width=0) (actual time=0.072..0.073 rows=1 loops=1)
    Index Cond: ((title)::text = 'Oracle Core'::text)
    Planning Time: 0.405 ms
    Execution Time: 0.114 ms

    ```
    
    *Объясните результат:*
     PostgreSQL использует индекс t_books_trgm_idx для поиска строки в таблице t_books, где столбец title равен 'Oracle Core'. Оценка стоимости выполнения запроса составляет от 116.57 до 120.58, и PostgreSQL ожидает найти 1 строку размером 33 байта. Фактически запрос выполнялся 1 раз и занял 0.086-0.088 миллисекунд, вернув 1 строку. Индексный сканирование (Bitmap Index Scan) на индексе t_books_trgm_idx заняло 0.072-0.073 миллисекунды. Время планирования запроса составило 0.405 миллисекунды, а общее время выполнения — 0.114 миллисекунд.



20. Выполните поиск по началу названия:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books WHERE title ILIKE 'Relational%';
    ```
    
    *План выполнения:*
    ```sql
    Bitmap Heap Scan on t_books  (cost=95.15..150.36 rows=15 width=33) (actual time=0.029..0.030 rows=0 loops=1)
    Recheck Cond: ((title)::text ~~* 'Relational%'::text)
    Rows Removed by Index Recheck: 1
    Heap Blocks: exact=1
    ->  Bitmap Index Scan on t_books_trgm_idx  (cost=0.00..95.15 rows=15 width=0) (actual time=0.019..0.020 rows=1 loops=1)
    Index Cond: ((title)::text ~~* 'Relational%'::text)
    Planning Time: 0.200 ms
    Execution Time: 0.045 ms
    ```
    
    *Объясните результат:*
    Оценка стоимости выполнения запроса составляет от 95.15 до 150.36, и PostgreSQL ожидает найти 15 строк. Фактически запрос выполнялся 1 раз и занял 0.029-0.030 миллисекунд, но не вернул ни одной строки, так как фильтр удалил 1 строку. Индексный сканирование (Bitmap Index Scan) на индексе t_books_trgm_idx заняло 0.019-0.020 миллисекунды. Время планирования запроса составило 0.200 миллисекунды, а общее время выполнения — 0.045 миллисекунд. Использование индекса значительно ускорило запрос по сравнению с последовательным сканированием, где время выполнения составляло около 68 миллисекунд.



21. Создайте свой пример индекса с обратной сортировкой:
    ```sql
    CREATE INDEX t_books_desc_idx ON t_books(title DESC);
    ```
    
    *Тестовый запрос:*
    ```sql
    CREATE INDEX t_books_desc_idx ON t_books(title DESC);

    EXPLAIN ANALYZE
    SELECT * FROM t_books WHERE title = 'Oracle Core' ORDER BY title DESC;
    ```
    
    *План выполнения:*
    ```sql
    Index Scan using t_books_desc_idx on t_books  (cost=0.42..8.44 rows=1 width=33) (actual time=0.039..0.040 rows=1 loops=1)
    Index Cond: ((title)::text = 'Oracle Core'::text)
    Planning Time: 1.898 ms
    Execution Time: 0.069 ms
    ```
