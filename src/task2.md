## Задание 2

1. Удалите старую базу данных, если есть:
    ```shell
    docker compose down
    ```

2. Поднимите базу данных из src/docker-compose.yml:
    ```shell
    docker compose down && docker compose up -d
    ```

3. Обновите статистику:
    ```sql
    ANALYZE t_books;
    ```

4. Создайте полнотекстовый индекс:
    ```sql
    CREATE INDEX t_books_fts_idx ON t_books 
    USING GIN (to_tsvector('english', title));
    ```

5. Найдите книги, содержащие слово 'expert':
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books 
    WHERE to_tsvector('english', title) @@ to_tsquery('english', 'expert');
    ```
    
    *План выполнения:*
   ![all texts](/image_2/image_1.png)
    
    *Объясните результат:*

    Как мы видим наш `GIN` индекс значительно оптимизировал  полнтотекстовый поиск. `to_tsvector` разбивает заговолок на лексикомы, `to_tsquery` преобразует наше слово в полнотекстовый запрос.

6. Удалите индекс:
    ```sql
    DROP INDEX t_books_fts_idx;
    ```

7. Создайте таблицу lookup:
    ```sql
    CREATE TABLE t_lookup (
         item_key VARCHAR(10) NOT NULL,
         item_value VARCHAR(100)
    );
    ```

8. Добавьте первичный ключ:
    ```sql
    ALTER TABLE t_lookup 
    ADD CONSTRAINT t_lookup_pk PRIMARY KEY (item_key);
    ```

9. Заполните данными:
    ```sql
    INSERT INTO t_lookup 
    SELECT 
         LPAD(CAST(generate_series(1, 150000) AS TEXT), 10, '0'),
         'Value_' || generate_series(1, 150000);
    ```

10. Создайте кластеризованную таблицу:
     ```sql
     CREATE TABLE t_lookup_clustered (
          item_key VARCHAR(10) PRIMARY KEY,
          item_value VARCHAR(100)
     );
     ```

11. Заполните её теми же данными:
     ```sql
     INSERT INTO t_lookup_clustered 
     SELECT * FROM t_lookup;
     
     CLUSTER t_lookup_clustered USING t_lookup_clustered_pkey;
     ```

12. Обновите статистику:
     ```sql
     ANALYZE t_lookup;
     ANALYZE t_lookup_clustered;
     ```

13. Выполните поиск по ключу в обычной таблице:
     ```sql
     EXPLAIN ANALYZE
     SELECT * FROM t_lookup WHERE item_key = '0000000455';
     ```
     
     *План выполнения:*
   ![all texts](/image_2/image_2.png)
     
     *Объясните результат:*
     Как мы видим происходит `Index Scan`, это происходит потому что в простую таблицу мы добавили первичные ключи, а для них автоматически создается индекс, поэтому при поиске postgreSQL использовал именно `Index Scan` вместо `Seq Scan`.

14. Выполните поиск по ключу в кластеризованной таблице:
     ```sql
     EXPLAIN ANALYZE
     SELECT * FROM t_lookup_clustered WHERE item_key = '0000000455';
     ```
     
     *План выполнения:*
   ![all texts](/image_2/image_3.png)
     
     *Объясните результат:*
     
     Мы видим, что поиск в кластеризованной таблице происходит быстрее, че в обычной, также мы видим, что используется `Index Scan`, то есть индекс создаваемый автоматически из-за наличия первичного ключа. Кластеризованная таблица работает быстрее, потому в отличии от обычной таблице, данные в ней упорядочены по `item_key` после команды `CLUSTER`.


15. Создайте индекс по значению для обычной таблицы:
     ```sql
     CREATE INDEX t_lookup_value_idx ON t_lookup(item_value);
     ```

16. Создайте индекс по значению для кластеризованной таблицы:
     ```sql
     CREATE INDEX t_lookup_clustered_value_idx 
     ON t_lookup_clustered(item_value);
     ```

17. Выполните поиск по значению в обычной таблице:
     ```sql
     EXPLAIN ANALYZE
     SELECT * FROM t_lookup WHERE item_value = 'T_BOOKS';
     ```
     
     *План выполнения:*
   ![all texts](/image_2/image_4.png)
     
     *Объясните результат:*
     
     Как мы видим, используется введенный ранее индекс. Происходит `Index Scan`, запрос оптимизирован.

18. Выполните поиск по значению в кластеризованной таблице:
     ```sql
     EXPLAIN ANALYZE
     SELECT * FROM t_lookup_clustered WHERE item_value = 'T_BOOKS';
     ```
     
     *План выполнения:*
   ![all texts](/image_2/image_5.png)
     
     *Объясните результат:*
     Тут тоже мы видим, используется введенный ранее индекс для кластеризованной таблицы. Происходит `Index Scan`, запрос оптимизирован.

19. Сравните производительность поиска по значению в обычной и кластеризованной таблицах:
     
     *Сравнение:*
     
     Так как в 17 и 18 задании мы делаем запрос по `item_value`, а в кластеризованной таблице упорядочивание происходит по `item_key`, производительность почти не отличается. А в 13 и 14 задании, как я уже отмечал, есть сильное преимущество кластеризованной таблицы по производительности, так как происходит поиск по `item_key`, и в отличии от обычной таблицы, в кластеризованной данные упорядочены.

   ![all texts](/image_2/это_просто_мысли_вслух.jpg)