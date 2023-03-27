# 12.5 "Индексы" - НиконоровДА FOPS-6

## Задание 1.

Напишите запрос к учебной базе данных, который вернёт процентное отношение общего размера всех индексов к общему размеру всех таблиц.

```SQL
SELECT count(INDEX_LENGTH)/(SELECT sum(DATA_LENGTH) FROM INFORMATION_SCHEMA.TABLES)*100 FROM INFORMATION_SCHEMA.TABLES;
```

![alt text](https://github.com/mxssclxck/hw-12.05/blob/main/img/1.png)

## Доработка задание 1.

```SQL
SELECT ROUND(SUM(INDEX_LENGTH)/SUM(DATA_LENGTH)*100,2) AS Index_to_table_ratio FROM information_schema.TABLES WHERE TABLE_SCHEMA = 'sakila';
```
![alt text](https://github.com/mxssclxck/hw-12.05/blob/main/img/1-1.png)

## Задание 2
Выполните explain analyze следующего запроса:
```SQL
select distinct concat(c.last_name, ' ', c.first_name), sum(p.amount) over (partition by c.customer_id, f.title)
from payment p, rental r, customer c, inventory i, film f
where date(p.payment_date) = '2005-07-30' and p.payment_date = r.rental_date and r.customer_id = c.customer_id and i.inventory_id = r.inventory_id
```
* перечислите узкие места;
* оптимизируйте запрос: внесите корректировки по использованию операторов, при необходимости добавьте индексы.

Использование явного синтаксиса JOIN вместо неявного (через запятую). Это улучшит читаемость и может помочь оптимизатору запросов в MySQL выбрать более эффективный план выполнения запроса.

Использовать явное преобразование типа данных вместо функции date(), чтобы ускорить выполнение запроса.

Использование индексов на столбцах, которые используются для соединения таблиц (customer_id и inventory_id), а также для условия WHERE (payment_date и rental_date).

Удаление лишних столбцов из запроса, которые не участвуют в агрегации (SUM), чтобы уменьшить объем обрабатываемых данных.

Избавиться от использования оконных функций и переписать запрос с использованием группировки и агрегирования данных.

```SQL
EXPLAIN ANALYZE
SELECT DISTINCT CONCAT(c.last_name, ' ', c.first_name), SUM(p.amount)
FROM payment p
JOIN rental r ON p.payment_date = r.rental_date
JOIN customer c ON r.customer_id = c.customer_id 
JOIN inventory i ON i.inventory_id = r.inventory_id
JOIN film f on f.film_id = i.film_id
WHERE p.payment_date BETWEEN '2005-07-30 00:00:00' AND '2005-07-30 23:59:59'
GROUP BY c.customer_id , f.title;
                        
-> Limit: 200 row(s)  (actual time=10.189..10.230 rows=200 loops=1)
    -> Sort with duplicate removal: `CONCAT(c.last_name, ' ', c.first_name)`, `SUM(p.amount)`  (actual time=10.189..10.207 rows=200 loops=1)
        -> Table scan on <temporary>  (actual time=9.681..9.790 rows=634 loops=1)
            -> Aggregate using temporary table  (actual time=9.679..9.679 rows=634 loops=1)
                -> Nested loop inner join  (cost=1022.40 rows=645) (actual time=0.062..8.346 rows=642 loops=1)
                    -> Nested loop inner join  (cost=799.41 rows=645) (actual time=0.058..7.185 rows=642 loops=1)
                        -> Nested loop inner join  (cost=576.43 rows=645) (actual time=0.055..5.998 rows=642 loops=1)
                            -> Nested loop inner join  (cost=350.73 rows=634) (actual time=0.042..2.067 rows=634 loops=1)
                                -> Filter: (r.rental_date between '2005-07-30 00:00:00' and '2005-07-30 23:59:59')  (cost=128.83 rows=634) (actual time=0.033..0.880 rows=634 loops=1)
                                    -> Covering index range scan on r using rental_date over ('2005-07-30 00:00:00' <= rental_date <= '2005-07-30 23:59:59')  (cost=128.83 rows=634) (actual time=0.030..0.385 rows=634 loops=1)
                                -> Single-row index lookup on c using PRIMARY (customer_id=r.customer_id)  (cost=0.25 rows=1) (actual time=0.002..0.002 rows=1 loops=634)
                            -> Index lookup on p using payment_date_idx (payment_date=r.rental_date)  (cost=0.25 rows=1) (actual time=0.003..0.004 rows=1 loops=634)
                        -> Single-row index lookup on i using PRIMARY (inventory_id=r.inventory_id)  (cost=0.25 rows=1) (actual time=0.002..0.002 rows=1 loops=642)
                    -> Single-row index lookup on f using PRIMARY (film_id=i.film_id)  (cost=0.25 rows=1) (actual time=0.002..0.002 rows=1 loops=642)

```

![alt text](https://github.com/mxssclxck/hw-12.05/blob/main/img/2.png)

## Доработка задание 2

```SQL
EXPLAIN ANALYZE
SELECT CONCAT(c.last_name, ' ', c.first_name), SUM(p.amount)
FROM payment p
JOIN rental r ON p.payment_date = r.rental_date
JOIN customer c ON r.customer_id = c.customer_id 
JOIN inventory i ON i.inventory_id = r.inventory_id
WHERE p.payment_date BETWEEN '2005-07-30 00:00:00' AND '2005-07-30 23:59:59'
GROUP BY c.customer_id;
                        
-> Limit: 200 row(s)  (actual time=6.697..6.748 rows=200 loops=1)
    -> Table scan on <temporary>  (actual time=6.695..6.732 rows=200 loops=1)
        -> Aggregate using temporary table  (actual time=6.694..6.694 rows=391 loops=1)
            -> Nested loop inner join  (cost=799.41 rows=645) (actual time=0.049..5.919 rows=642 loops=1)
                -> Nested loop inner join  (cost=576.43 rows=645) (actual time=0.045..4.875 rows=642 loops=1)
                    -> Nested loop inner join  (cost=350.73 rows=634) (actual time=0.035..2.074 rows=634 loops=1)
                        -> Filter: (r.rental_date between '2005-07-30 00:00:00' and '2005-07-30 23:59:59')  (cost=128.83 rows=634) (actual time=0.020..0.911 rows=634 loops=1)
                            -> Covering index range scan on r using rental_date over ('2005-07-30 00:00:00' <= rental_date <= '2005-07-30 23:59:59')  (cost=128.83 rows=634) (actual time=0.018..0.454 rows=634 loops=1)
                        -> Single-row index lookup on c using PRIMARY (customer_id=r.customer_id)  (cost=0.25 rows=1) (actual time=0.002..0.002 rows=1 loops=634)
                    -> Index lookup on p using payment_date_idx (payment_date=r.rental_date)  (cost=0.25 rows=1) (actual time=0.003..0.004 rows=1 loops=634)
                -> Single-row covering index lookup on i using PRIMARY (inventory_id=r.inventory_id)  (cost=0.25 rows=1) (actual time=0.001..0.001 rows=1 loops=642)

```

![alt text](https://github.com/mxssclxck/hw-12.05/blob/main/img/2-1.png)

## Задание 3*
Самостоятельно изучите, какие типы индексов используются в PostgreSQL. Перечислите те индексы, которые используются в PostgreSQL, а в MySQL — нет.

Приведите ответ в свободной форме.

```
1. B-деревья (B-tree): наиболее распространенный тип индекса в PostgreSQL, который используется для индексирования строковых, числовых и датовых столбцов.

2. Хэш-индексы (Hash): используются для быстрого поиска по значению хэш-функции. Хэш-индексы хорошо работают для запросов с оператором "равно" (=), но не поддерживают сортировку.

3. GiST (Generalized Search Tree): поддерживают геометрические и текстовые типы данных, а также пользовательские типы с заданным оператором сравнения.

4. GIN (Generalized Inverted Index): используется для полнотекстового поиска и поиска по массивам и другим составным типам данных.

5. SP-GiST (Space-Partitioned Generalized Search Tree): используется для индексации геометрических типов данных.

6. BRIN (Block Range INdex): используется для индексации больших таблиц и ускорения агрегатных операций.

7. Bloom Filter: используется для оптимизации запросов с оператором "неравно" (!=) и "IN".

Некоторые индексы, которые поддерживаются в PostgreSQL, но не поддерживаются в MySQL:

GiST и SP-GiST
BRIN
Bloom Filter
Также следует отметить, что в MySQL поддерживается индекс типа FULLTEXT, который используется для полнотекстового поиска, но он работает не так эффективно, как GIN в PostgreSQL.
```


