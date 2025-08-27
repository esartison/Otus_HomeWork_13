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
yugabyte=#  CREATE TABLE foo ( id INT PRIMARY KEY,
yugabyte(#                            name VARCHAR(255),
yugabyte(#                            data VARCHAR(4000)
yugabyte(#                            );
CREATE TABLE foo ( id INT PRIMARY KEY,
                           name VARCHAR(255),
                           data VARCHAR(4000)
                           );
CREATE TABLE



INSERT INTO foo(id, name, data) SELECT i, 'name'||i, random() FROM generate_series(1,100000000) i;


```
загрузил таблицу размером 14Gb.

Синтаксис для генерации тестовой таблицы одинаковый на Yugabyte и Postgres. 

## **(3) Установка PostgreSQL **
Установил Postgres и создал базу, использовал настройки по умолчанию и ничего не менял
```
pgdb001:установка Postgres 
sudo apt install postgresql
fixing permissions on existing directory /var/lib/postgresql/16/main ... ok
creating subdirectories ... ok
selecting dynamic shared memory implementation ... posix
selecting default max_connections ... 100
selecting default shared_buffers ... 128MB
selecting default time zone ... Etc/UTC
creating configuration files ... ok
running bootstrap script ... ok
performing post-bootstrap initialization ... ok
syncing data to disk ... ok
Setting up postgresql (16+257build1.1) ...
Processing triggers for man-db (2.12.0-4build2) ...
Processing triggers for libc-bin (2.39-0ubuntu8.5) ...
Scanning processes...
Scanning linux images...

Running kernel seems to be up-to-date.

No services need to be restarted.

No containers need to be restarted.

No user sessions are running outdated binaries.

No VM guests are running outdated hypervisor (qemu) binaries on this host.
```
<img width="1290" height="263" alt="image" src="https://github.com/user-attachments/assets/8d92b0a0-246a-4bb3-8930-722b44cd572b" />


Создал тестовый набор данных
```
mydb=# CREATE TABLE foo (
mydb(#     id INT PRIMARY KEY,
mydb(#     name VARCHAR(255),
mydb(# data VARCHAR(4000)
mydb(# );
CREATE TABLE
mydb=#

INSERT INTO foo(id, name, data) SELECT i, 'name'||i, random() FROM generate_series(1,200000000) i;

mydb=# SELECT pg_size_pretty(pg_relation_size('foo'));
 pg_size_pretty
----------------
 13 GB
(1 row)

Time: 44.158 ms
```
загрузил таблицу размером 13Gb.




## **(4) Сравнение про-ти**

Решил выполнить 2 простых запроса на обоих базах и сравнить про-ть

**CockroachDB**
```
(a) root@localhost:26257/mydb> select max(id) from foo;
     max
-------------
  100000000
(1 row)

Time: 3ms total (execution 3ms / network 0ms)

(б) select count(0) from foo; -- выполнялся более 30 минут и не дождался выполнения
```
Select Count из всей таблицы очень долго выполнялся и в итоге я не дождался, пробовал перезапускать. Возможно, 2Gb RAM слишком мало. 


**PostgreSQL**
```
(a) mydb=#  select max(id) from foo;
    max
-----------
 200000000
(1 row)

Time: 0.339 ms

Time: 3ms total (execution 3ms / network 0ms)

(б) mydb=#  select count(0) from foo;
   count
-----------
 200000000
(1 row)

Time: 241718.252 ms (04:01.718)
```

Первый запрос на обоих базах выполнился быстро на обоих базах и побидитель не выявлен. Второй запрос на Postgres выполнился намного быстрее.

Мой тест показал что Postgres имеет лучшую про-ть при схожих конфигурациях. 




