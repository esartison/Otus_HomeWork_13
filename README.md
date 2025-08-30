# Домашнее задание Сартисона Евгения №13

🎯 Задание
Разверните Yugabyte или Greenplum в Kubernetes или облаках.
Загрузите датасет 10 Гб+ .
Проведите тест скорости запросов в сравнении с одиночным инстансом PostgreSQL.
Опишите процесс развертывания, настройки, проблемы и результаты.

⭐ Задание повышенной сложности*
Разверните оба варианта (Yugabyte и Greenplum)
Проведите сравнение производительности и подготовьте аналитический отчёт.

Задание повышенной сложности выполнить не получится, так-как GreenPlum перестал быть open-source. 

Решил разворачивать Postgres и Yugabyte в Docker, чтобы были равные условия и можно было сравнить. 



## **(1) Установка Yugabyte и загрузка тестовых данных **

Пошел по иструкции из [Install YugabyteDB in Docker](https://docs.yugabyte.com/preview/quick-start/docker/)

Стащил образ
```
student:~$ docker pull yugabytedb/yugabyte:2.25.2.0-b359
2.25.2.0-b359: Pulling from yugabytedb/yugabyte
09720f817e0c: Pull complete 
81c92ce6d7b6: Pull complete 
809ac79c7b65: Pull complete 
5d55c0eeaad0: Pull complete 
1b55dea911eb: Pull complete 
9158f1db666b: Pull complete 
457bd16d685b: Pull complete 
556765b224e4: Pull complete 
cdc0faf24508: Pull complete 
4f4fb700ef54: Pull complete 
0137975de31d: Pull complete 
6205293b0026: Pull complete 
b327c1b23eac: Pull complete 
80eb0dc40f0a: Pull complete 
c5d2c264cd4c: Pull complete 
8db778a4947d: Pull complete 
7de11e291cc5: Pull complete 
4e45d9cd6e55: Pull complete 
Digest: sha256:f1b868431c9f71013f451df8cc63c59886568bb1254e4cb6f98de14d0232a3ad
Status: Downloaded newer image for yugabytedb/yugabyte:2.25.2.0-b359
docker.io/yugabytedb/yugabyte:2.25.2.0-b359
```

Запустил контейнер
```
student:~$ docker run -d --name yugabyteotus \
        -p 7000:7000  -p 15433:15433 -p 5433:5433 -p 9042:9042 \
        yugabytedb/yugabyte:2.25.2.0-b359 bin/yugabyted start \
        --background=false
e0730e36ee65afdce865c436239657f4e4970dc6c230e98d0b09dbe86140e32d
```

Проверил состояние - все хорошо
```
student:~$ docker ps
CONTAINER ID   IMAGE                               COMMAND                  CREATED          STATUS          PORTS                                                                                                                                                                                                                                                                       NAMES
e0730e36ee65   yugabytedb/yugabyte:2.25.2.0-b359   "/sbin/tini -- bin/y…"   26 seconds ago   Up 26 seconds   0.0.0.0:5433->5433/tcp, [::]:5433->5433/tcp, 6379/tcp, 7100/tcp, 7200/tcp, 0.0.0.0:7000->7000/tcp, [::]:7000->7000/tcp, 9000/tcp, 9100/tcp, 10100/tcp, 11000/tcp, 0.0.0.0:9042->9042/tcp, [::]:9042->9042/tcp, 0.0.0.0:15433->15433/tcp, [::]:15433->15433/tcp, 12000/tcp   yugabyteotus


---------------------------------------------------------------------------------------------------------+
|                                                 yugabyted                                                 |
+-----------------------------------------------------------------------------------------------------------+
| Status              : Running.                                                                            |
| YSQL Status         : Ready                                                                               |
| Replication Factor  : 1                                                                                   |
| YugabyteDB UI       : http://e0730e36ee65:15433                                                           |
| JDBC                : jdbc:postgresql://e0730e36ee65:5433/yugabyte?user=yugabyte&password=yugabyte        |
| YSQL                : bin/ysqlsh -h e0730e36ee65  -U yugabyte -d yugabyte                                 |
| YCQL                : bin/ycqlsh e0730e36ee65 9042 -u cassandra                                           |
| Data Dir            : /root/var/data                                                                      |
| Log Dir             : /root/var/logs                                                                      |
| Universe UUID       : 34f7e676-63db-4c73-b41b-5782e1c2f445                                                |
+-----------------------------------------------------------------------------------------------------------+
student:~$ 
```

Попробовал подключиться и проверил состояние кластера - все хорошо
```
student:~$ docker exec -it yugabyteotus bash -c '/home/yugabyte/bin/ysqlsh --echo-queries --host $(hostname)'
ysqlsh (15.12-YB-2.25.2.0-b0)
Type "help" for help.

yugabyte=# 
yugabyte=# 
yugabyte=# \l
                                                  List of databases
      Name       |  Owner   | Encoding | Collate |    Ctype    | ICU Locale | Locale Provider |   Access privileges   
-----------------+----------+----------+---------+-------------+------------+-----------------+-----------------------
 postgres        | postgres | UTF8     | C       | en_US.UTF-8 |            | libc            | 
 system_platform | postgres | UTF8     | C       | en_US.UTF-8 |            | libc            | 
 template0       | postgres | UTF8     | C       | en_US.UTF-8 |            | libc            | =c/postgres          +
                 |          |          |         |             |            |                 | postgres=CTc/postgres
 template1       | postgres | UTF8     | C       | en_US.UTF-8 |            | libc            | =c/postgres          +
                 |          |          |         |             |            |                 | postgres=CTc/postgres
 yugabyte        | postgres | UTF8     | C       | en_US.UTF-8 |            | libc            | 
(5 rows)

```

Создал тестовый набор данных
```
yugabyte=# \c yugabyte
You are now connected to database "yugabyte" as user "yugabyte".
yugabyte=# 
yugabyte=# 
yugabyte=# CREATE TABLE foo ( id INT ,
yugabyte(#                            name VARCHAR(255),
yugabyte(#                            data VARCHAR(4000)
yugabyte(#                            );
CREATE TABLE foo ( id INT ,
                           name VARCHAR(255),
                           data VARCHAR(4000)
                           );
CREATE TABLE
Time: 290.033 ms

yugabyte=# INSERT INTO foo(id, name, data) SELECT i, 'name'||i, random() FROM generate_series(1,50000000) i;
INSERT INTO foo(id, name, data) SELECT i, 'name'||i, random() FROM generate_series(1,50000000) i;
INSERT 0 50000000
Time: 18225338.785 ms (05:03:45.339)

yugabyte=# INSERT INTO foo(id, name, data) SELECT i, 'name'||i, random() FROM generate_series(1,50000000) i;
INSERT INTO foo(id, name, data) SELECT i, 'name'||i, random() FROM generate_series(1,50000000) i;
INSERT 0 50000000
Time: 33240527.300 ms (09:14:00.527)
yugabyte=# select pg_size_pretty(pg_table_size('foo'));
select pg_size_pretty(pg_table_size('foo'));
 pg_size_pretty 
----------------
 6639 MB
```
загрузил таблицу размером 6.7Gb. Загрузка идет очень долго, возможно сам Yugabyte как-то оптимизирует и сжимает место. 
Вставка происходила десятки часов, хотя на Postgres добежала всего за 10 минут, возможно влияют какие-то настройки.

Синтаксис для генерации тестовой таблицы одинаковый на Yugabyte и Postgres. 

## **(2) Установка PostgreSQL и загрузка тестовых данных**
Поднял образ Postgres с настройками по умолчанию
```
student:~$ docker run --name postdbotus -e POSTGRES_PASSWORD=mysecretpassword -p 5444:5432 -d postgres
60f71fdcbbba10a9d53c9754d6516554f9630d8be2ef586c2de97fd6600648ee
student:~$ docker ps
CONTAINER ID   IMAGE      COMMAND                  CREATED         STATUS         PORTS                                         NAMES
60f71fdcbbba   postgres   "docker-entrypoint.s…"   7 seconds ago   Up 7 seconds   0.0.0.0:5444->5432/tcp, [::]:5444->5432/tcp   postdbotus
```
<img width="1282" height="369" alt="image" src="https://github.com/user-attachments/assets/a966b347-9955-4514-a1a6-edfbce3f0335" />



Создал тестовый набор данных
```
postgres=# \c postgres
You are now connected to database "postgres" as user "postgres".
postgres=# CREATE TABLE foo ( id INT ,
                           name VARCHAR(255),
                           data VARCHAR(4000)
                           );
CREATE TABLE
postgres=# INSERT INTO foo(id, name, data) SELECT i, 'name'||i, random() FROM generate_series(1,50000000) i;
INSERT 0 50000000
Time: 119098.271 ms (01:59.098)
postgres=# 


postgres=# INSERT INTO foo(id, name, data) SELECT i, 'name'||i, random() FROM generate_series(1,50000000) i;
INSERT 0 50000000
Time: 159322.516 ms (02:39.323)


postgres=# INSERT INTO foo(id, name, data) SELECT i, 'name'||i, random() FROM generate_series(1,5000000) i;
INSERT 0 5000000
Time: 7122.283 ms (00:07.122)



postgres=# select pg_size_pretty(pg_table_size('foo'));
 pg_size_pretty 
----------------
 6837 MB
(1 row)

Time: 13.684 ms
```
загрузил таблицу размером 6.8Gb. Загрузить можно было и больше, но на Yugabyte вставка 50m строк занимала 10 часов. 




## **(3) Сравнение про-ти**

Решил выполнить 2 простых запроса на обоих базах и сравнить про-ть

**Yugabyte**
```
(a) yugabyte=# select max(id) from foo;
select max(id) from foo;

   max    
----------
 50000000
(1 row)

Time: 399059.192 ms (06:39.059)

(б) select count(0) from foo;
yugabyte=# select count(0) from foo; 
select count(0) from foo;

   count   
-----------
 105000000
(1 row)

Time: 342794.048 ms (05:42.794)

```



**PostgreSQL**
```
(a) postgres=# select max(id) from foo;
   max    
----------
 50000000
(1 row)

Time: 42826.000 ms (00:42.826)


(б) postgres=# select count(0) from foo;
   count   
-----------
 105000000
(1 row)

Time: 4470.065 ms (00:04.470)
```

В обоих случаях Postgres показал про-ть на порядки лучше. Я брал настройки по умолчанию, возможно Yugabyte нужно донастроить, чтобы он хорошо работал. 

Postgres вне конкуренции. 

Мой тест показал что Postgres имеет лучшую про-ть при схожих конфигурациях. 




