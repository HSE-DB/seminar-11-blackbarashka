# Задание 1: BRIN индексы и bitmap-сканирование

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

4. Создайте BRIN индекс по колонке category:a
   ```sql
   CREATE INDEX t_books_brin_cat_idx ON t_books USING brin(category);
   ```

5. Найдите книги с NULL значением category:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books WHERE category IS NULL;
   ```
   
   *План выполнения:*
   ![all texts](/image/image_1.png)
   
   *Объясните результат:*

   Если мы будем делать запрос без brin индекса, то `cost = 2725`, а с индексом brin `cost` намного меньше. Это связано с тем, что после создания brin-индекса PostgreSQL читает метаданные этого индекса и сканирует только те блоки, где у нас есть NULL. BRIN-индекс эффективен для поиска NULL, так как хранит информацию о минимальных и максимальных значениях в каждом диапазоне блоков. Если NULL присутствует в диапазоне, весь диапазон помечается для проверки.
6. Создайте BRIN индекс по автору:
   ```sql
   CREATE INDEX t_books_brin_author_idx ON t_books USING brin(author);
   ```

7. Выполните поиск по категории и автору:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books 
   WHERE category = 'INDEX' AND author = 'SYSTEM';
   ```
   
   *План выполнения:*
   ![all texts](/image/image_2.png)
   
   *Объясните результат (обратите внимание на bitmap scan):*
   
   PostgreSQL делает bitmap scan и использует индекс по category, а не по author, потому что оптимизатор оценил его как более селективный. Еще фактически отсканировал всю таблицу. Это происходит из-за фактического отсутсвия значений `INDEX` и `SYSTEM`. PostgreSQL использовал. 

8. Получите список уникальных категорий:
   ```sql
   EXPLAIN ANALYZE
   SELECT DISTINCT category 
   FROM t_books 
   ORDER BY category;
   ```
   
   *План выполнения:*
   ![all texts](/image/image_3.png)
   
   *Объясните результат:*

   Тут у нас происходит сортировка таблицы, `PostgreSQL` сперва сканирует всю таблицу используя `Seq Scan`, после из-за наличия в запросе `DISTINCT`, создается хэш-таблица (`HashAggregate`), чтоб найти уникальные значения. Brin-индекс не используется так как, нет условия `WHERE`.

9. Подсчитайте книги, где автор начинается на 'S':
   ```sql
   EXPLAIN ANALYZE
   SELECT COUNT(*) 
   FROM t_books 
   WHERE author LIKE 'S%';
   ```
   
   *План выполнения:*
   ![all texts](/image/image_4.png)

   
   *Объясните результат:*
   Тут происходит Seq Scan всех 150к строк. Brin-индекс не используется так как, brin-индексы не работают с LIKE. Я создал b-tree индексы и повторил запрос, но PostgreSQL все еще считает Seq Scan эффективнее чем Index Scan, это происходит так как, у нас в столбце `author` нет значений начинающихся с S. 

10. Создайте индекс для регистронезависимого поиска:
    ```sql
    CREATE INDEX t_books_lower_title_idx ON t_books(LOWER(title));
    ```

11. Подсчитайте книги, начинающиеся на 'O':
    ```sql
    EXPLAIN ANALYZE
    SELECT COUNT(*) 
    FROM t_books 
    WHERE LOWER(title) LIKE 'o%';
    ```
   
   *План выполнения:*
   ![all texts](/image/image_6.png)

   *Объясните результат:*

   Происходит `Seq Scan`, то есть не используется  созданный индекс. PostgreSQL считает, что `Seq Scan` эффективнее чем `Index Scan`, также он предпологает что 15 строк подходят под условие. Я сделал запрос без использования паттерна `%`, тогда индекс заработал.

12. Удалите созданные индексы:
    ```sql
    DROP INDEX t_books_brin_cat_idx;
    DROP INDEX t_books_brin_author_idx;
    DROP INDEX t_books_lower_title_idx;
    ```

13. Создайте составной BRIN индекс:
    ```sql
    CREATE INDEX t_books_brin_cat_auth_idx ON t_books 
    USING brin(category, author);
    ```

14. Повторите запрос из шага 7:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books 
    WHERE category = 'INDEX' AND author = 'SYSTEM';
    ```
   
   *План выполнения:*
   ![all texts](/image/image_6.png)
   
   *Объясните результат:*

   Создание составного brin-индекса по категории и автору  ускорило выполнение запроса (с 32 мс до 4 мс), так как индекс позволил отфильтровать 94% блоков таблицы. Однако из-за lossy-режима и случайного распределения данных всё равно было прочитано 73 диапазона блоков и проверено 8855 строк. Это демонстрирует как преимущества составных brin-индексов для многокритериального поиска, так и их ограничения при работе с некоррелированными данными.