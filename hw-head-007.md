#DOC!
# Логический уровень PostgreSQL 


Создал ВМ из готвого снапшота, в котром установлена сервернася и клиенсткая часть PostgreSQL.
```
yc compute instance create --name "vm-pg-hw007-host-1" --zone ru-central1-a --network-interface subnet-id=e9bl3fu9obful2c43udh,nat-ip-version=ipv4 --create-boot-disk image-folder-id=standard-images,image-family=ubuntu-2004-lts --ssh-key ~/.ssh/id_rsa_first.pub --labels tag=otus-lb
done (28s)


yc compute instance list | grep hw007
| fhmj32ah48gl0vn5qs13 | vm-pg-hw007-host-1 | ru-central1-a | RUNNING | 62.84.112.61  | 10.128.0.35 

```

Подключился по SSH и установил все для PostgreSQL 13
```
postgres@fhmj32ah48gl0vn5qs13:~$ psql --version 
psql (PostgreSQL) 13.7 (Ubuntu 13.7-1.pgdg20.04+1)

postgres@fhmj32ah48gl0vn5qs13:~$ pg_lsclusters 
Ver Cluster Port Status Owner    Data directory              Log file
13  main    5432 online postgres /var/lib/postgresql/13/main /var/log/postgresql/postgresql-13-main.log

``` 

Под пользователем postgres создал БД testdb, в ней схему tesnm (не testnm) и наполнил данными:
```
postgres=# create database testdb;
CREATE DATABASE

postgres=# \c testdb 
You are now connected to database "testdb" as user "postgres".

testdb=# create schema tesnm;
CREATE SCHEMA
testdb=# \dn+
                          List of schemas
  Name  |  Owner   |  Access privileges   |      Description       
--------+----------+----------------------+------------------------
 public | postgres | postgres=UC/postgres+| standard public schema
        |          | =UC/postgres         | 
 tesnm  | postgres |                      | 
(2 rows)

testdb=# insert into t1 values ('10'),('20'),('30'),('40');
INSERT 0 4

testdb=# \dl
      Large objects
 ID | Owner | Description 
----+-------+-------------
(0 rows)

testdb=# select * from t1;
 c1 
----
 10
 20
 30
 40
(4 rows)
```

Создал роль readonly и пользователя testread, которому предоставил права в БД testdb и схему tesnm
- роль readonly
```
testdb=# create role readonly;
CREATE ROLE

testdb=# \du+
                                          List of roles
 Role name |                         Attributes                         | Member of | Description 
-----------+------------------------------------------------------------+-----------+-------------
 postgres  | Superuser, Create role, Create DB, Replication, Bypass RLS | {}        | 
 readonly  | Cannot login                                               | {}        | 

```
- пользователь testread
```
testdb=# grant CONNECT on DATABASE testdb to readonly;
GRANT

testdb=# grant USAGE ON SCHEMA tesnm to readonly;
GRANT

testdb=# grant SELECT ON ALL tables in schema tesnm to readonly;
GRANT

testdb=# create user testread password 'test123';
CREATE ROLE

testdb=# grant readonly to testread;
GRANT ROLE

testdb=# \du+
                                           List of roles
 Role name |                         Attributes                         | Member of  | Description 
-----------+------------------------------------------------------------+------------+-------------
 postgres  | Superuser, Create role, Create DB, Replication, Bypass RLS | {}         | 
 readonly  | Cannot login                                               | {}         | 
 testread  |                                                            | {readonly} | 
```

Подключение к testdb от пользователя testread
```
testdb=# \c testdb testread;
connection to server on socket "/var/run/postgresql/.s.PGSQL.5432" failed: FATAL:  Peer authentication failed for user "testread"
Previous connection kept
```
Обламинго. А потому что не разрешено от других пользователей подключаться к сокету. 
Добавил строку `/etc/postgresql/13/main/pg_hba.conf` разрешающая роли readonly подклбчаться через сокеты.
```
root@fhmj32ah48gl0vn5qs13:~# grep ^local /etc/postgresql/13/main/pg_hba.conf 
local   all             postgres                                peer
local   all             readonly                                peer
local   all             all                                     peer
local   replication     all                                     md5

```

После манипуляций все удалось подключиться
```
root@fhmj32ah48gl0vn5qs13:~# psql -h localhost -U testread testdb
Password for user testread: 
psql (13.7 (Ubuntu 13.7-1.pgdg20.04+1))
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
Type "help" for help.

testdb=> select * from t1;
ERROR:  permission denied for table t1
```

Не смог вывести объекты из таблицы. Почему? Потому что таблица созданно в другой схеме, к которой testread нет разрешения.
```
postgres=# \c testdb 
You are now connected to database "testdb" as user "postgres".

testdb=# \dn;
  List of schemas
  Name  |  Owner   
--------+----------
 public | postgres
 tesnm  | postgres
(2 rows)

testdb=# \dt
        List of relations
 Schema | Name | Type  |  Owner   
--------+------+-------+----------
 public | t1   | table | postgres
(1 row)

testdb=# show search_path;
   search_path   
-----------------
 "$user", public
(1 row)
```
```
testdb=# \dn+;
                          List of schemas
  Name  |  Owner   |  Access privileges   |      Description       
--------+----------+----------------------+------------------------
 public | postgres | postgres=UC/postgres+| standard public schema
        |          | =UC/postgres         | 
 tesnm  | postgres | postgres=UC/postgres+| 
        |          | readonly=U/postgres  | 
(2 rows)
```

Пересоздал таблицу t1 в схеме tesnm
```
testdb=# drop table t1;
DROP TABLE

testdb=# create table tesnm.t1(c1 integer);
CREATE TABLE

testdb=# insert into tesnm.t1 values('10'),('100'),('1000'),('10000');
INSERT 0 4
```

Зашел от пользователя testread и получил отворот поворот
```
postgres@fhmj32ah48gl0vn5qs13:~$ psql -h localhost -U testread testdb
Password for user testread: 
psql (13.7 (Ubuntu 13.7-1.pgdg20.04+1))
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)

testdb=> select * from tesnm.t1;
ERROR:  permission denied for table t1

testdb=> \conninfo 
You are connected to database "testdb" as user "testread" on host "localhost" (address "::1") at port "5432".
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
```
Почему не понял =(
Смотрел права на схему и таблицу.
Подсмотрел шпаргалку 
> 29 потому что grant SELECT on all TABLEs in SCHEMA testnm TO readonly дал доступ только для существующих на тот момент времени таблиц а t1 пересоздавалась

> 30 \c testdb postgres; 
> ALTER default privileges in SCHEMA testnm grant SELECT on TABLEs to readonly; 
> \c testdb testread;

т.е. нужно внести изменения на уровне привилегий по умолчанию "ALTER", в то время как "GRANT" дает изменения на имеющиеся объекты.

```
testdb=# ALTER default privileges in schema tesnm grant select on tables TO readonly;
ALTER DEFAULT PRIVILEGES

```
Что бы все создавалось в схеме tesnm можно поправить переменную show_path
```
testdb=# show search_path 
testdb-# ;
   search_path   
-----------------
 "$user", public
(1 row)

testdb=# set search_path TO "$user", tesnm, public;
SET

testdb=# show search_path;
      search_path       
------------------------
 "$user", tesnm, public
(1 row)
```

Подсключился к testdb от пользователя testread и поробовал вывести таблицу, облом. Похоже на то что ALTER на все таблицы создан на будущие объекты, использовал команду GRANT на текущий обчект так отработалоло.
```
testdb=# \c testdb testread 
Password for user testread: 
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
You are now connected to database "testdb" as user "testread".
testdb=> \dt tesnm.t1 
        List of relations
 Schema | Name | Type  |  Owner   
--------+------+-------+----------
 tesnm  | t1   | table | postgres
(1 row)

testdb=> select * from tesnm.t1;
ERROR:  permission denied for table t1

testdb=> \c testdb postgres 
Password for user postgres: 
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
You are now connected to database "testdb" as user "postgres".

testdb=# grant SELECT on ALL tables in schema tesnm TO readonly;
GRANT

testdb=# \c testdb testread;
Password for user testread: 
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
You are now connected to database "testdb" as user "testread".

testdb=> select * from tesnm.t1;
  c1   
-------
    10
   100
  1000
 10000
(4 rows)
```


Попробовал создать таблицу t2 от testread не вышло
```
testdb=> create table t2(c1 integer);
ERROR:  permission denied for schema tesnm
LINE 1: create table t2(c1 integer);
                     ^
```
А потом что у нас было разрешение только на SELECT в простарнстве tesnm.
Можно попробовать убрать и схему publuc для пользователя testread и добавить в разрешение CREATE/INSERT.
```
testdb=# grant CREATE on SCHEMA tesnm TO readonly;
GRANT

testdb=# \c testdb testread 
Password for user testread: 
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)
You are now connected to database "testdb" as user "testread".
                     ^
testdb=> create table tesnm.t2(c1 integer);
CREATE TABLE

testdb=> \dt tesnm.*
        List of relations
 Schema | Name | Type  |  Owner   
--------+------+-------+----------
 tesnm  | t1   | table | postgres
 tesnm  | t2   | table | testread
(2 rows)

```

Решил из переменной search_path оставить только значение схемы tesnm. И так все отработало, создал в этой схеме таблицы и наполнил их содержимым.

```
testdb=> show search_path 
testdb-> ;
   search_path   
-----------------
 "$user", public
(1 row)


testdb=> set search_path TO tesnm;
SET

testdb=> create table t3(c1 integer);
CREATE TABLE

testdb=> \dt tesnm.*
        List of relations
 Schema | Name | Type  |  Owner   
--------+------+-------+----------
 tesnm  | t1   | table | postgres
 tesnm  | t2   | table | testread
 tesnm  | t3   | table | testread
(3 rows)

testdb=> insert into t2 values ('2'),('3'),('4');
INSERT 0 3
testdb=> insert into t3 values ('8'),('16'),('32');
INSERT 0 3
testdb=> select * from t2;
 c1 
----
  2
  3
  4
(3 rows)

testdb=> select * from tesnm.t3;
 c1 
----
  8
 16
 32
(3 rows)
```


С правами надо разобраться на практики, к ознакомлению информацию для себя -> https://postgrespro.ru/docs/postgresql/13/sql-grant
