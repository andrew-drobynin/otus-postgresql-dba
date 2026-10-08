###### Подключение из контейнера с клиентом к контейнеру с сервером
```sh
docker exec -it psql psql -h postgres -U postgres
```

###### Запросы для таблицы orders_test
```sql
CREATE TABLE orders_test (  
    test TEXT NULL  
);

INSERT INTO orders_test (test)
VALUES ('test-1'),
       ('test-2');

SELECT *
FROM orders_test;
```

###### Подключение и выполнение запросов в контейнере с psql
![](images/psql-1.png)

###### Подключение с хоста через pgAdmin
![](images/pgAdmin-1.png)

###### Выполнение запроса с хоста через pgAdmin
![](images/pgAdmin-2.png)

###### Остановка и удаление контейнера с сервером
```sh
docker ps -a

docker stop postgres

docker rm postgres
```
![](images/docker-stop-rm.png)

###### Повторное подключение и запроса запросов в контейнере с psql
Строки в таблице orders_test сохранились

![](images/psql-2.png)

###### Повторное выполнение запроса с хоста через pgAdmin
![](images/pgAdmin-3.png)
