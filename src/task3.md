## Задание 3

1. Создайте таблицу с большим количеством данных:
    ```sql
    CREATE TABLE test_cluster AS 
    SELECT 
        generate_series(1,1000000) as id,
        CASE WHEN random() < 0.5 THEN 'A' ELSE 'B' END as category,
        md5(random()::text) as data;
    ```

2. Создайте индекс:
    ```sql
    CREATE INDEX test_cluster_cat_idx ON test_cluster(category);
    ```

3. Измерьте производительность до кластеризации:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM test_cluster WHERE category = 'A';
    ```
    
    *План выполнения:*
   ![all texts](/image_3/image_1.png)
    
    *Объясните результат:*
    
    Выполняется `Bitmap Index Scan`, используя добавленный индекс `test_cluster_cat_idx`. Выполняется именно `Bitmap Index Scan` вместо `Index Scan` так как строк удовлетворяющих условию много (>500k). Поиск оптимизирован, относительго запроса без индекса, тогда выполнялся бы просто `Seq Scan`.

4. Выполните кластеризацию:
    ```sql
    CLUSTER test_cluster USING test_cluster_cat_idx;
    ```
    
    *Результат:*
   ![all texts](/image_3/image_2.png)

5. Измерьте производительность после кластеризации:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM test_cluster WHERE category = 'A';
    ```
    
    *План выполнения:*
   ![all texts](/image_3/image_3.png)
    
    *Объясните результат:*
    Мы видим, что `Heap Blocks: exact=4170` (количество блоков таблицы, которые PostgreSQL прочитал для выполнения запроса), он уменшился в два раза. Опять же выполняется `Bitmap Index Scan` и `Bitmap Heap Scan`.

6. Сравните производительность до и после кластеризации:
    
    *Сравнение:*
    
    Если сравнивать поиск с кластеризацией и без, мы не видим прям сильных улучшений после кластеризации. Но уменьшается I/O, чтение данных меньше в два раза.

    После того, как я сделал `ANALYZE test_cluster` и снова выполнил запрос с 3 задания, вывод такой:

       ![all texts](/image_3/image_4.png)

    Это произошло так, как после обновление статистики, планировщик видит, что у нас упорядочены данные после кластеризации.



    ![all texts](/image_3/кама.jpg)