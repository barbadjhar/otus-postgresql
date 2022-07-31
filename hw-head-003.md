# DOC!

Все окурежниие созадвал в Яндекс Облаке. Первым делом созад типовой инстанс через cli yc.

```
$ yc compute instance create --name "vm-postgres-p1" --zone ru-central1-a --network-interface subnet-id=e9bl3fu9obful2c43udh,nat-ip-version=ipv4 --create-boot-disk image-folder-id=standard-images,image-family=ubuntu-2004-lts --ssh-key ~/.ssh/id_rsa_first.pub 

done (21s)
(вывод убрал)
```

Проверил существование ВМ 
```
$ yc compute instance get --full fhm2rr1reb9iomjrrrg4

id: fhm2rr1reb9iomjrrrg4
...
```


Подлюкчился к ВМ по SSH
```
	$ ssh yc-user@51.250.8.89 -i ~/.ssh/id_rsa_first 
	yc-user@fhm2rr1reb9iomjrrrg4:~$ exit
	logout
	Connection to 51.250.8.89 closed.
```


Установил docker, psql и создал диреткорию для хранения данных СУБД
```

root@fhm2rr1reb9iomjrrrg4:~# dpkg -l | grep -e docker -e postgres
ii  docker                                1.5-2                             all          transitional package
ii  docker-ce                             5:20.10.17~3-0~ubuntu-focal       amd64        Docker: the open-source application container engine
ii  docker-ce-cli                         5:20.10.17~3-0~ubuntu-focal       amd64        Docker CLI: the open-source application container engine
ii  docker-ce-rootless-extras             5:20.10.17~3-0~ubuntu-focal       amd64        Rootless support for Docker.
ii  docker-compose-plugin                 2.6.0~ubuntu-focal                amd64        Docker Compose (V2) plugin for the Docker CLI.
ii  docker-scan-plugin                    0.17.0~ubuntu-focal               amd64        Docker scan cli plugin.
ii  wmdocker                              1.5-2                             amd64        System tray for KDE3/GNOME2 docklet applications
ii  postgresql-client                     12+214ubuntu0.1                   all          front-end programs for PostgreSQL (supported version)
ii  postgresql-client-12                  12.11-0ubuntu0.20.04.1            amd64        front-end programs for PostgreSQL 12
ii  postgresql-client-common              214ubuntu0.1                      all          manager for multiple PostgreSQL client versions


yc-user@fhm2rr1reb9iomjrrrg4:~$ ls -ld /var/lib/postgres/
drwx------ 19 systemd-coredump root 4096 Jul 21 09:59 /var/lib/postgres/

yc-user@fhm2rr1reb9iomjrrrg4:~$ id yc-user
uid=1000(yc-user) gid=1001(yc-user) groups=1001(yc-user),998(docker)
```

Поднял контейнер с серверной и клиентской частью Postgres
```
yc-user@fhm2rr1reb9iomjrrrg4:~$ docker image ls
REPOSITORY        TAG       IMAGE ID       CREATED         SIZE
postgres          14        1133a9cdc367   9 days ago      376MB
postgres          latest    1133a9cdc367   9 days ago      376MB
ubuntu/postgres   latest    a4d827229fcf   2 weeks ago     392MB
busybox           latest    62aedd01bd85   6 weeks ago     1.24MB
hello-world       latest    feb5d9fea6a5   10 months ago   13.3kB


yc-user@fhm2rr1reb9iomjrrrg4:~$ docker network create pg-net
e473002321c22e3943a3229a58d3cf140e00c1f8b808849bb1f265b754358933


yc-user@fhm2rr1reb9iomjrrrg4:~$ docker run --name pg-server --network pg-net -e POSTGRES_PASSWORD=god1234 -d -p 5432:5432 -v /var/lib/postgres:/var/lib/postgresql/data postgres:14

# в другой сессии контейнер с клиентом запустил
yc-user@fhm2rr1reb9iomjrrrg4:~$ docker run -it --rm --network pg-net --name pg-client postgres:14 psql -h pg-server -U postgres
 Password for user postgres: 
 psql (14.4 (Debian 14.4-1.pgdg110+1))
 Type "help" for help.
```


Проверил что контейнеры запущены
```
yc-user@fhm2rr1reb9iomjrrrg4:~$ docker ps -a
CONTAINER ID   IMAGE         COMMAND                  CREATED          STATUS                    PORTS                                       NAMES
4dcee4f829a4   postgres:14   "docker-entrypoint.s…"   42 minutes ago   Up 42 minutes             5432/tcp                                    pg-client
da61d5f992ec   postgres:14   "docker-entrypoint.s…"   49 minutes ago   Up 49 minutes             0.0.0.0:5432->5432/tcp, :::5432->5432/tcp   pg-server
c53a321e8ffe   hello-world   "/hello"                 18 hours ago     Exited (0) 18 hours ago                                               wizardly_easley
```


Из контейнра pg-client создал БД с таблицой и добавил данные. Плюс проверил подключение с локального клиента.
```
postgres=# CREATE DATABASE test;
CREATE DATABASE
postgres=# \l
                                 List of databases
   Name    |  Owner   | Encoding |  Collate   |   Ctype    |   Access privileges   
-----------+----------+----------+------------+------------+-----------------------
 postgres  | postgres | UTF8     | en_US.utf8 | en_US.utf8 | 
 template0 | postgres | UTF8     | en_US.utf8 | en_US.utf8 | =c/postgres          +
           |          |          |            |            | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_US.utf8 | en_US.utf8 | =c/postgres          +
           |          |          |            |            | postgres=CTc/postgres
 test      | postgres | UTF8     | en_US.utf8 | en_US.utf8 | 
(4 rows)

postgres=# \c test
You are now connected to database "test" as user "postgres".
test=# 
test=# CREATE TABLE gogrgy (
test(# id BI

test(# id BIGSERIAL,
test(# name VARCHAR(50),
test(# email VARCHAR(150));
CREATE TABLE

test=# \d
              List of relations
 Schema |     Name      |   Type   |  Owner   
--------+---------------+----------+----------
 public | gogrgy        | table    | postgres
 public | gogrgy_id_seq | sequence | postgres
(2 rows)
```



Добавил данных
```
test=# INSERT INTO gogrgy (name, email)
test-# VALUES ('John', 'abc@mail.ru');
INSERT 0 1
test=# select * from gogrgy;
 id | name |    email    
----+------+-------------
  1 | John | abc@mail.ru
(1 row)
```



Проверил что локально подключается 
```
yc-user@fhm2rr1reb9iomjrrrg4:~$ psql -h localhost -U postgres -d postgres
postgres=# \c test
psql (12.11 (Ubuntu 12.11-0ubuntu0.20.04.1), server 14.4 (Debian 14.4-1.pgdg110+1))
WARNING: psql major version 12, server major version 14.
         Some psql features might not work.
You are now connected to database "test" as user "postgres".
test=# \d
              List of relations
 Schema |     Name      |   Type   |  Owner   
--------+---------------+----------+----------
 public | gogrgy        | table    | postgres
 public | gogrgy_id_seq | sequence | postgres
(2 rows)

test=# select * from gogrgy;
 id |  name  |      email       
----+--------+------------------
  1 | John   | abc@mail.ru
  2 | Verona | deb@mail.it
  3 | Mark   | donetelo@mail.ru
(3 rows)
```



Удалил контейнер с сервером pd-server  `docker rm pg-server` проверил что данные в `/var/lib/postgres` не пустое и есть файлы.
Создал заново контейнер pg-server, подключился к нему клиентом, убедился что данные есть с примонтированного каталога и добавил еще одну строку.
```
yc-user@fhm2rr1reb9iomjrrrg4:~$ docker rm pg-server 
pg-server

yc-user@fhm2rr1reb9iomjrrrg4:~$ docker ps -a
CONTAINER ID   IMAGE         COMMAND    CREATED        STATUS                    PORTS     NAMES
c53a321e8ffe   hello-world   "/hello"   18 hours ago   Exited (0) 18 hours ago             wizardly_easley

yc-user@fhm2rr1reb9iomjrrrg4:~$ docker run --name pg-server --network pg-net -e POSTGRES_PASSWORD=god1234 -d -p 5432:5432 -v /var/lib/postgres:/var/lib/postgresql/data postgres:14 
91e3853101d320fe2af83dbd5b8ecd74d0e041a942aa845ec2a78ae6919bcf71

yc-user@fhm2rr1reb9iomjrrrg4:~$ docker ps -a
CONTAINER ID   IMAGE         COMMAND                  CREATED         STATUS                    PORTS                                       NAMES
91e3853101d3   postgres:14   "docker-entrypoint.s…"   5 seconds ago   Up 4 seconds              0.0.0.0:5432->5432/tcp, :::5432->5432/tcp   pg-server
c53a321e8ffe   hello-world   "/hello"                 18 hours ago    Exited (0) 18 hours ago 


yc-user@fhm2rr1reb9iomjrrrg4:~$ sudo du -sh /var/lib/postgres/
50M	/var/lib/postgres/
```




Клиентом подключился 
```
yc-user@fhm2rr1reb9iomjrrrg4:~$ docker run -it --rm --network pg-net --name pg-client postgres:14 psql -h pg-server -U postgres
Password for user postgres: 
psql (14.4 (Debian 14.4-1.pgdg110+1))
Type "help" for help.

postgres=# \d
Did not find any relations.

postgres=# \l
                                 List of databases
   Name    |  Owner   | Encoding |  Collate   |   Ctype    |   Access privileges   
-----------+----------+----------+------------+------------+-----------------------
 postgres  | postgres | UTF8     | en_US.utf8 | en_US.utf8 | 
 template0 | postgres | UTF8     | en_US.utf8 | en_US.utf8 | =c/postgres          +
           |          |          |            |            | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_US.utf8 | en_US.utf8 | =c/postgres          +
           |          |          |            |            | postgres=CTc/postgres
 test      | postgres | UTF8     | en_US.utf8 | en_US.utf8 | 
(4 rows)

postgres=# \conninfo 
You are connected to database "postgres" as user "postgres" on host "pg-server" (address "172.18.0.2") at port "5432".

postgres=# \c test
You are now connected to database "test" as user "postgres".

test=# \d
              List of relations
 Schema |     Name      |   Type   |  Owner   
--------+---------------+----------+----------
 public | gogrgy        | table    | postgres
 public | gogrgy_id_seq | sequence | postgres
(2 rows)

test=# SELECT name FROM gogrgy;
  name  
--------
 John
 Verona
 Mark
(3 rows)

test=# \q
```




Проверил с локального клиента что доступен БД
```
yc-user@fhm2rr1reb9iomjrrrg4:~$ psql -h localhost -U postgres -d postgres
Password for user postgres: 
psql (12.11 (Ubuntu 12.11-0ubuntu0.20.04.1), server 14.4 (Debian 14.4-1.pgdg110+1))
WARNING: psql major version 12, server major version 14.
         Some psql features might not work.
Type "help" for help.

postgres=# \l
                                 List of databases
   Name    |  Owner   | Encoding |  Collate   |   Ctype    |   Access privileges   
-----------+----------+----------+------------+------------+-----------------------
 postgres  | postgres | UTF8     | en_US.utf8 | en_US.utf8 | 
 template0 | postgres | UTF8     | en_US.utf8 | en_US.utf8 | =c/postgres          +
           |          |          |            |            | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_US.utf8 | en_US.utf8 | =c/postgres          +
           |          |          |            |            | postgres=CTc/postgres
 test      | postgres | UTF8     | en_US.utf8 | en_US.utf8 | 
(4 rows)

postgres=# \c test
psql (12.11 (Ubuntu 12.11-0ubuntu0.20.04.1), server 14.4 (Debian 14.4-1.pgdg110+1))
WARNING: psql major version 12, server major version 14.
         Some psql features might not work.
You are now connected to database "test" as user "postgres".
test=# \d
              List of relations
 Schema |     Name      |   Type   |  Owner   
--------+---------------+----------+----------
 public | gogrgy        | table    | postgres
 public | gogrgy_id_seq | sequence | postgres
(2 rows)

test=# select * from gogrgy;
 id |  name  |      email       
----+--------+------------------
  1 | John   | abc@mail.ru
  2 | Verona | deb@mail.it
  3 | Mark   | donetelo@mail.ru
(3 rows)

test=# insert INTO gogrgy (name, email)
test-# values ('Ana', 'demart@mail.nl');
INSERT 0 1
test=# select * from gogrgy;
 id |  name  |      email       
----+--------+------------------
  1 | John   | abc@mail.ru
  2 | Verona | deb@mail.it
  3 | Mark   | donetelo@mail.ru
  4 | Ana    | demart@mail.nl
(4 rows)
```


Done!




