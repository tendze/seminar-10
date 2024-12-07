# Задание 2: Специальные случаи использования индексов

# Партиционирование и специальные случаи использования индексов

1. Удалите прошлый инстанс PostgreSQL - `docker-compose down` в папке `src` и запустите новый: `docker-compose up -d`.

2. Создайте партиционированную таблицу и заполните её данными:

    ```sql
    -- Создание партиционированной таблицы
    CREATE TABLE t_books_part (
        book_id     INTEGER      NOT NULL,
        title       VARCHAR(100) NOT NULL,
        category    VARCHAR(30),
        author      VARCHAR(100) NOT NULL,
        is_active   BOOLEAN      NOT NULL
    ) PARTITION BY RANGE (book_id);

    -- Создание партиций
    CREATE TABLE t_books_part_1 PARTITION OF t_books_part
        FOR VALUES FROM (MINVALUE) TO (50000);

    CREATE TABLE t_books_part_2 PARTITION OF t_books_part
        FOR VALUES FROM (50000) TO (100000);

    CREATE TABLE t_books_part_3 PARTITION OF t_books_part
        FOR VALUES FROM (100000) TO (MAXVALUE);

    -- Копирование данных из t_books
    INSERT INTO t_books_part 
    SELECT * FROM t_books;
    ```

3. Обновите статистику таблиц:
   ```sql
   ANALYZE t_books;
   ANALYZE t_books_part;
   ```
   
   *Результат:*
   [Вставьте результат выполнения]

4. Выполните запрос для поиска книги с id = 18:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books_part WHERE book_id = 18;
   ```
   
   *План выполнения:*
   ```sql
    Seq Scan on t_books_part_1 t_books_part  (cost=0.00..1032.99 rows=1 width=32) (actual time=0.008..2.998 rows=1 loops=1)
    Filter: (book_id = 18)
    Rows Removed by Filter: 49998
    Planning Time: 0.437 ms
    Execution Time: 3.013 ms
    ```
   
   *Объясните результат:*
   3ms всего

5. Выполните поиск по названию книги:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books_part 
   WHERE title = 'Expert PostgreSQL Architecture';
   ```
   
   *План выполнения:*
   ```sql
    Append  (cost=0.00..3101.01 rows=3 width=33) (actual time=4.603..16.388 rows=1 loops=1)
    ->  Seq Scan on t_books_part_1  (cost=0.00..1032.99 rows=1 width=32) (actual time=4.602..4.603 rows=1 loops=1)
            Filter: ((title)::text = 'Expert PostgreSQL Architecture'::text)
            Rows Removed by Filter: 49998
    ->  Seq Scan on t_books_part_2  (cost=0.00..1034.00 rows=1 width=33) (actual time=5.207..5.207 rows=0 loops=1)
            Filter: ((title)::text = 'Expert PostgreSQL Architecture'::text)
            Rows Removed by Filter: 50000
    ->  Seq Scan on t_books_part_3  (cost=0.00..1034.01 rows=1 width=34) (actual time=6.567..6.567 rows=0 loops=1)
            Filter: ((title)::text = 'Expert PostgreSQL Architecture'::text)
            Rows Removed by Filter: 50001
    Planning Time: 0.317 ms
    Execution Time: 23.759 ms
   ```
   *Объясните результат:*
   Оценка стоимости выполнения запроса составляет от 0.00 до 3101.01, и PostgreSQL ожидает найти 3 строки. Фактически запрос выполнялся 1 раз и занял 4.603-16.388 миллисекунд, вернув 1 строку. Время планирования запроса составило 0.317 миллисекунды, а общее время выполнения — 23.759 миллисекунд. Последовательное сканирование каждой партиции заняло время: 4.603 миллисекунды для t_books_part_1, 5.207 миллисекунды для t_books_part_2 и 6.567 миллисекунды для t_books_part_3. Фильтр удалил 49998 строк из t_books_part_1, 50000 строк из t_books_part_2 и 50001 строку из t_books_part_3.

6. Создайте партиционированный индекс:
   ```sql
   CREATE INDEX ON t_books_part(title);
   ```
   
   *Результат:*
   [Вставьте результат выполнения]

7. Повторите запрос из шага 5:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books_part 
   WHERE title = 'Expert PostgreSQL Architecture';
   ```
   
   *План выполнения:*
   ```sql
    Append  (cost=0.29..24.94 rows=3 width=33) (actual time=0.033..0.136 rows=1 loops=1)
    ->  Index Scan using t_books_part_1_title_idx on t_books_part_1  (cost=0.29..8.31 rows=1 width=32) (actual time=0.032..0.084 rows=1 loops=1)
            Index Cond: ((title)::text = 'Expert PostgreSQL Architecture'::text)
    ->  Index Scan using t_books_part_2_title_idx on t_books_part_2  (cost=0.29..8.31 rows=1 width=33) (actual time=0.026..0.026 rows=0 loops=1)
            Index Cond: ((title)::text = 'Expert PostgreSQL Architecture'::text)
    ->  Index Scan using t_books_part_3_title_idx on t_books_part_3  (cost=0.29..8.31 rows=1 width=34) (actual time=0.023..0.023 rows=0 loops=1)
            Index Cond: ((title)::text = 'Expert PostgreSQL Architecture'::text)
    Planning Time: 0.500 ms
    Execution Time: 0.160 ms
    ```
   
   *Объясните результат:*
   Ну, быстрее стало, было 23.7ms

8. Удалите созданный индекс:
   ```sql
   DROP INDEX t_books_part_title_idx;
   ```
   
   *Результат:*
   [Вставьте результат выполнения]

9. Создайте индекс для каждой партиции:
   ```sql
   CREATE INDEX ON t_books_part_1(title);
   CREATE INDEX ON t_books_part_2(title);
   CREATE INDEX ON t_books_part_3(title);
   ```
   
   *Результат:*
   [Вставьте результат выполнения]

10. Повторите запрос из шага 5:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books_part 
    WHERE title = 'Expert PostgreSQL Architecture';
    ```
    
    *План выполнения:*
    ```sql
    Append  (cost=0.29..24.94 rows=3 width=33) (actual time=0.026..0.054 rows=1 loops=1)
    ->  Index Scan using t_books_part_1_title_idx on t_books_part_1  (cost=0.29..8.31 rows=1 width=32) (actual time=0.025..0.026 rows=1 loops=1)
            Index Cond: ((title)::text = 'Expert PostgreSQL Architecture'::text)
    ->  Index Scan using t_books_part_2_title_idx on t_books_part_2  (cost=0.29..8.31 rows=1 width=33) (actual time=0.013..0.013 rows=0 loops=1)
            Index Cond: ((title)::text = 'Expert PostgreSQL Architecture'::text)
    ->  Index Scan using t_books_part_3_title_idx on t_books_part_3  (cost=0.29..8.31 rows=1 width=34) (actual time=0.012..0.012 rows=0 loops=1)
            Index Cond: ((title)::text = 'Expert PostgreSQL Architecture'::text)
    Planning Time: 0.461 ms
    Execution Time: 0.082 ms
    ```
    
    *Объясните результат:*
     PostgreSQL использует индексы на каждой партиции таблицы t_books_part для поиска строки, где столбец title равен 'Expert PostgreSQL Architecture'. Оценка стоимости выполнения запроса составляет от 0.29 до 24.94, и PostgreSQL ожидает найти 3 строки. Фактически запрос выполнялся 1 раз и занял 0.026-0.054 миллисекунд, вернув 1 строку. Время планирования запроса составило 0.461 миллисекунды, а общее время выполнения — 0.082 миллисекунд. Использование индексов на каждой партиции позволило значительно ускорить выполнение запроса по сравнению с последовательным сканированием, где время выполнения составляло около 23.759 миллисекунд.

11. Удалите созданные индексы:
    ```sql
    DROP INDEX t_books_part_1_title_idx;
    DROP INDEX t_books_part_2_title_idx;
    DROP INDEX t_books_part_3_title_idx;
    ```
    
    *Результат:*
    [Вставьте результат выполнения]

12. Создайте обычный индекс по book_id:
    ```sql
    CREATE INDEX t_books_part_idx ON t_books_part(book_id);
    ```
    
    *Результат:*
    [Вставьте результат выполнения]

13. Выполните поиск по book_id:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books_part WHERE book_id = 11011;
    ```
    
    *План выполнения:*
    ```sql
    Index Scan using t_books_part_1_book_id_idx on t_books_part_1 t_books_part  (cost=0.29..8.31 rows=1 width=32) (actual time=0.017..0.018 rows=1 loops=1)
    Index Cond: (book_id = 11011)
    Planning Time: 0.459 ms
    Execution Time: 0.056 ms
    ```
    
    *Объясните результат:*
    [Ваше объяснение]

14. Создайте индекс по полю is_active:
    ```sql
    CREATE INDEX t_books_active_idx ON t_books(is_active);
    ```
    
    *Результат:*
    [Вставьте результат выполнения]

15. Выполните поиск активных книг с отключенным последовательным сканированием:
    ```sql
    SET enable_seqscan = off;
    EXPLAIN ANALYZE
    SELECT * FROM t_books WHERE is_active = true;
    SET enable_seqscan = on;
    ```
    
    *План выполнения:*
    ```sql
    Bitmap Heap Scan on t_books  (cost=850.12..2831.02 rows=75590 width=33) (actual time=2.455..11.776 rows=75164 loops=1)
    Recheck Cond: is_active
    Heap Blocks: exact=1225
    ->  Bitmap Index Scan on t_books_active_idx  (cost=0.00..831.22 rows=75590 width=0) (actual time=2.298..2.299 rows=75164 loops=1)
            Index Cond: (is_active = true)
    Planning Time: 0.196 ms
    Execution Time: 13.958 ms
    ```
    
    *Объясните результат:*
    Запрос использует индекс t_books_active_idx для фильтрации строк, где столбец is_active равен true. Использование Bitmap Heap Scan и Bitmap Index Scan позволяет PostgreSQL эффективно обрабатывать запрос, что приводит к быстрому выполнению. Общее время выполнения запроса составило 13.958 миллисекунд.

16. Создайте составной индекс:
    ```sql
    CREATE INDEX t_books_author_title_index ON t_books(author, title);
    ```
    
    *Результат:*
    [Вставьте результат выполнения]

17. Найдите максимальное название для каждого автора:
    ```sql
    EXPLAIN ANALYZE
    SELECT author, MAX(title) 
    FROM t_books 
    GROUP BY author;
    ```
    
    *План выполнения:*
    ```sql
    HashAggregate  (cost=3475.00..3485.01 rows=1001 width=42) (actual time=85.800..85.929 rows=1003 loops=1)
    Group Key: author
    Batches: 1  Memory Usage: 193kB
    ->  Seq Scan on t_books  (cost=0.00..2725.00 rows=150000 width=21) (actual time=0.006..10.856 rows=150000 loops=1)
    Planning Time: 0.205 ms
    Execution Time: 85.985 ms
    ```
    
    *Объясните результат:*
    [Ваше объяснение]

18. Выберите первых 10 авторов:
    ```sql
    EXPLAIN ANALYZE
    SELECT DISTINCT author 
    FROM t_books 
    ORDER BY author 
    LIMIT 10;
    ```
    
    *План выполнения:*
    ```sql
    Limit  (cost=0.42..56.61 rows=10 width=10) (actual time=8.557..10.170 rows=10 loops=1)
    ->  Result  (cost=0.42..5625.42 rows=1001 width=10) (actual time=8.556..10.165 rows=10 loops=1)
            ->  Unique  (cost=0.42..5625.42 rows=1001 width=10) (actual time=8.553..10.159 rows=10 loops=1)
                ->  Index Only Scan using t_books_author_title_index on t_books  (cost=0.42..5250.42 rows=150000 width=10) (actual time=8.551..10.016 rows=1402 loops=1)
                        Heap Fetches: 5
    Planning Time: 0.125 ms
    Execution Time: 10.197 ms
    ```
    
    *Объясните результат:*
    [Ваше объяснение]

19. Выполните поиск и сортировку:
    ```sql
    EXPLAIN ANALYZE
    SELECT author, title 
    FROM t_books 
    WHERE author LIKE 'T%'
    ORDER BY author, title;
    ```
    
    *План выполнения:*
    ```sql
    Sort  (cost=3100.27..3100.30 rows=14 width=21) (actual time=15.350..15.351 rows=1 loops=1)
    "  Sort Key: author, title"
    Sort Method: quicksort  Memory: 25kB
    ->  Seq Scan on t_books  (cost=0.00..3100.00 rows=14 width=21) (actual time=15.337..15.338 rows=1 loops=1)
            Filter: ((author)::text ~~ 'T%'::text)
            Rows Removed by Filter: 149999
    Planning Time: 0.161 ms
    Execution Time: 15.372 ms
    ```
    
    *Объясните результат:*
    [Ваше объяснение]

20. Добавьте новую книгу:
    ```sql
    INSERT INTO t_books (book_id, title, author, category, is_active)
    VALUES (150001, 'Cookbook', 'Mr. Hide', NULL, true);
    COMMIT;
    ```
    
    *Результат:*
    [Вставьте результат выполнения]

21. Создайте индекс по категории:
    ```sql
    CREATE INDEX t_books_cat_idx ON t_books(category);
    ```
    
    *Результат:*
    [Вставьте результат выполнения]

22. Найдите книги без категории:
    ```sql
    EXPLAIN ANALYZE
    SELECT author, title 
    FROM t_books 
    WHERE category IS NULL;
    ```
    
    *План выполнения:*
    ```sql
    Index Scan using t_books_cat_idx on t_books  (cost=0.29..8.14 rows=1 width=21) (actual time=0.027..0.028 rows=1 loops=1)
    Index Cond: (category IS NULL)
    Planning Time: 0.250 ms
    Execution Time: 0.043 ms
    ```
    
    *Объясните результат:*
    нашлось за 0.043

23. Создайте частичные индексы:
    ```sql
    DROP INDEX t_books_cat_idx;
    CREATE INDEX t_books_cat_null_idx ON t_books(category) WHERE category IS NULL;
    ```
    
    *Результат:*
    [Вставьте результат выполнения]

24. Повторите запрос из шага 22:
    ```sql
    EXPLAIN ANALYZE
    SELECT author, title 
    FROM t_books 
    WHERE category IS NULL;
    ```
    
    *План выполнения:*
    ```sql
    Index Scan using t_books_cat_null_idx on t_books  (cost=0.12..7.96 rows=1 width=21) (actual time=0.021..0.023 rows=1 loops=1)
    Planning Time: 0.322 ms
    Execution Time: 0.042 ms
    ```
    
    *Объясните результат:*
    та же самая скорость

25. Создайте частичный уникальный индекс:
    ```sql
    CREATE UNIQUE INDEX t_books_selective_unique_idx 
    ON t_books(title) 
    WHERE category = 'Science';
    
    -- Протестируйте его
    INSERT INTO t_books (book_id, title, author, category, is_active)
    VALUES (150002, 'Unique Science Book', 'Author 1', 'Science', true);
    
    -- Попробуйте вставить дубликат
    INSERT INTO t_books (book_id, title, author, category, is_active)
    VALUES (150003, 'Unique Science Book', 'Author 2', 'Science', true);
    
    -- Но можно вставить такое же название для другой категории
    INSERT INTO t_books (book_id, title, author, category, is_active)
    VALUES (150004, 'Unique Science Book', 'Author 3', 'History', true);
    ```
    
    *Результат:*
    [Вставьте результаты всех операций]
    
    *Объясните результат:*
    [Ваше объяснение]