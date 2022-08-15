# DOC!
# MVCC, vacuum и autovacuum 

Создла инстанс из готового снапшота
```
 yc compute instance create --name vm-pg-hw008-host-1 --zone ru-central1-a --platform standard-v2 --cores 2 --memory 4G --network-interface subnet-id=e9bl3fu9obful2c43udh,ipv4-address=auto,nat-ip-version=ipv4  --labels tag=test --ssh-key=/home/barbadjharxxx/.ssh/id_rsa_work_001.pub --description "test maschine for HW of 008" --create-boot-disk snapshot-name=vm-pg-repo 

yc compute instance list | egrep 'host|NAME'
|          ID          |        NAME        |    ZONE ID    | STATUS  |  EXTERNAL IP  | INTERNAL IP |
| fhm88m4kf0sln0do180s | vm-pg-hw008-host-1 | ru-central1-a | RUNNING | 51.250.89.193 | 10.128.0.37 |
| fhmj32ah48gl0vn5qs13 | vm-pg-hw007-host-1 | ru-central1-a | STOPPED |               | 10.128.0.35 |
```


Установил серверную и клиентскую часть Postgresql 13
```
root@fhm88m4kf0sln0do180s:~# dpkg -l | grep postgre
ii  pgdg-keyring                          2018.2                            all          keyring for apt.postgresql.org
ii  postgresql-13                         13.8-1.pgdg20.04+1                amd64        The World's Most Advanced Open Source Relational Database
ii  postgresql-client-13                  13.8-1.pgdg20.04+1                amd64        front-end programs for PostgreSQL 13
ii  postgresql-client-common              242.pgdg20.04+1                   all          manager for multiple PostgreSQL client versions
ii  postgresql-common                     242.pgdg20.04+1                   all          PostgreSQL database-cluster manager


root@fhm88m4kf0sln0do180s:~# pg_lsclusters 
Ver Cluster Port Status Owner    Data directory              Log file
13  main    5432 online postgres /var/lib/postgresql/13/main /var/log/postgresql/postgresql-13-main.log
```

Настройки кластера можно вывести двумя сособами из подключения через psql
```
show all;

OR

select name, setting from pg_settings; 

OR

select * from pg_file_settings;
```

Поменял значение max_connections через ALTER, применил pg_config_reload(), но он не применился.
```
postgres=# alter system set max_connections TO '400';
ALTER SYSTEM

postgres=# show max_connections;
 max_connections 
-----------------
 100
(1 row)

postgres=# select * from pg_file_settings;
                    sourcefile                    | sourceline | seqno |            name            |                 setting                 | applied |            error             
--------------------------------------------------+------------+-------+----------------------------+-----------------------------------------+---------+------------------------------
 /etc/postgresql/13/main/postgresql.conf          |         42 |     1 | data_directory             | /var/lib/postgresql/13/main             | t       | 
 /etc/postgresql/13/main/postgresql.conf          |         44 |     2 | hba_file                   | /etc/postgresql/13/main/pg_hba.conf     | t       | 
 /etc/postgresql/13/main/postgresql.conf          |         46 |     3 | ident_file                 | /etc/postgresql/13/main/pg_ident.conf   | t       | 
 /etc/postgresql/13/main/postgresql.conf          |         50 |     4 | external_pid_file          | /var/run/postgresql/13-main.pid         | t       | 
 /etc/postgresql/13/main/postgresql.conf          |         64 |     5 | port                       | 5432                                    | t       | 
 /etc/postgresql/13/main/postgresql.conf          |         65 |     6 | max_connections            | 100                                     | f       | 
 /etc/postgresql/13/main/postgresql.conf          |         67 |     7 | unix_socket_directories    | /var/run/postgresql                     | t       | 
 /etc/postgresql/13/main/postgresql.conf          |        101 |     8 | ssl                        | on                                      | t       | 
 /etc/postgresql/13/main/postgresql.conf          |        103 |     9 | ssl_cert_file              | /etc/ssl/certs/ssl-cert-snakeoil.pem    | t       | 
 /etc/postgresql/13/main/postgresql.conf          |        105 |    10 | ssl_key_file               | /etc/ssl/private/ssl-cert-snakeoil.key  | t       | 
 /etc/postgresql/13/main/postgresql.conf          |        122 |    11 | shared_buffers             | 128MB                                   | t       | 
 /etc/postgresql/13/main/postgresql.conf          |        143 |    12 | dynamic_shared_memory_type | posix                                   | t       | 
 /etc/postgresql/13/main/postgresql.conf          |        229 |    13 | max_wal_size               | 1GB                                     | t       | 
 /etc/postgresql/13/main/postgresql.conf          |        230 |    14 | min_wal_size               | 80MB                                    | t       | 
 /etc/postgresql/13/main/postgresql.conf          |        530 |    15 | log_line_prefix            | %m [%p] %q%u@%d                         | t       | 
 /etc/postgresql/13/main/postgresql.conf          |        564 |    16 | log_timezone               | Etc/UTC                                 | t       | 
 /etc/postgresql/13/main/postgresql.conf          |        570 |    17 | cluster_name               | 13/main                                 | t       | 
 /etc/postgresql/13/main/postgresql.conf          |        586 |    18 | stats_temp_directory       | /var/run/postgresql/13-main.pg_stat_tmp | t       | 
 /etc/postgresql/13/main/postgresql.conf          |        679 |    19 | datestyle                  | iso, mdy                                | t       | 
 /etc/postgresql/13/main/postgresql.conf          |        681 |    20 | timezone                   | Etc/UTC                                 | t       | 
 /etc/postgresql/13/main/postgresql.conf          |        695 |    21 | lc_messages                | en_US.UTF-8                             | t       | 
 /etc/postgresql/13/main/postgresql.conf          |        697 |    22 | lc_monetary                | en_US.UTF-8                             | t       | 
 /etc/postgresql/13/main/postgresql.conf          |        698 |    23 | lc_numeric                 | en_US.UTF-8                             | t       | 
 /etc/postgresql/13/main/postgresql.conf          |        699 |    24 | lc_time                    | en_US.UTF-8                             | t       | 
 /etc/postgresql/13/main/postgresql.conf          |        702 |    25 | default_text_search_config | pg_catalog.english                      | t       | 
 /var/lib/postgresql/13/main/postgresql.auto.conf |          3 |    26 | max_connections            | 400                                     | f       | setting could not be applied
(26 rows)

```

Как выеснилось есть настройки требуещие рестрат сервиса Postgresql. Как выяснить какие настройки требует рестарта кластера, можно так
```
postgres=# select name, setting, unit FROm pg_settings where context = 'postmaster';
```

После рестарта кластера настройка применилось и внес остальные изменения через `ALTER SYSTEM SET foo TO 'x';`
```
postgres=# show max_connections;
 max_connections 
-----------------
 400
(1 row)

```

Под изменения попали параметры
```
max_connections = 40
shared_buffers = 1GB
effective_cache_size = 3GB
maintenance_work_mem = 512MB
checkpoint_completion_target = 0.9
wal_buffers = 16MB
default_statistics_target = 500
random_page_cost = 4
effective_io_concurrency = 2
work_mem = 6553kB
min_wal_size = 4GB
max_wal_size = 16GB
```

`pg_reload_conf()` тут не поможет, рестартовал кластре.  

```
yc-user@fhm88m4kf0sln0do180s:~$ sudo systemctl restart postgresql@13-main.service 


postgres=# select * from pg_file_settings;
                    sourcefile                    | sourceline | seqno |             name             |                 setting                 | applied | error 
--------------------------------------------------+------------+-------+------------------------------+-----------------------------------------+---------+-------
 /etc/postgresql/13/main/postgresql.conf          |         42 |     1 | data_directory               | /var/lib/postgresql/13/main             | t       | 
 /etc/postgresql/13/main/postgresql.conf          |         44 |     2 | hba_file                     | /etc/postgresql/13/main/pg_hba.conf     | t       | 
 /etc/postgresql/13/main/postgresql.conf          |         46 |     3 | ident_file                   | /etc/postgresql/13/main/pg_ident.conf   | t       | 
 /etc/postgresql/13/main/postgresql.conf          |         50 |     4 | external_pid_file            | /var/run/postgresql/13-main.pid         | t       | 
 /etc/postgresql/13/main/postgresql.conf          |         64 |     5 | port                         | 5432                                    | t       | 
 /etc/postgresql/13/main/postgresql.conf          |         65 |     6 | max_connections              | 100                                     | f       | 
 /etc/postgresql/13/main/postgresql.conf          |         67 |     7 | unix_socket_directories      | /var/run/postgresql                     | t       | 
 /etc/postgresql/13/main/postgresql.conf          |        101 |     8 | ssl                          | on                                      | t       | 
 /etc/postgresql/13/main/postgresql.conf          |        103 |     9 | ssl_cert_file                | /etc/ssl/certs/ssl-cert-snakeoil.pem    | t       | 
 /etc/postgresql/13/main/postgresql.conf          |        105 |    10 | ssl_key_file                 | /etc/ssl/private/ssl-cert-snakeoil.key  | t       | 
 /etc/postgresql/13/main/postgresql.conf          |        122 |    11 | shared_buffers               | 128MB                                   | f       | 
 /etc/postgresql/13/main/postgresql.conf          |        143 |    12 | dynamic_shared_memory_type   | posix                                   | t       | 
 /etc/postgresql/13/main/postgresql.conf          |        229 |    13 | max_wal_size                 | 1GB                                     | f       | 
 /etc/postgresql/13/main/postgresql.conf          |        230 |    14 | min_wal_size                 | 80MB                                    | f       | 
 /etc/postgresql/13/main/postgresql.conf          |        530 |    15 | log_line_prefix              | %m [%p] %q%u@%d                         | t       | 
 /etc/postgresql/13/main/postgresql.conf          |        564 |    16 | log_timezone                 | Etc/UTC                                 | t       | 
 /etc/postgresql/13/main/postgresql.conf          |        570 |    17 | cluster_name                 | 13/main                                 | t       | 
 /etc/postgresql/13/main/postgresql.conf          |        586 |    18 | stats_temp_directory         | /var/run/postgresql/13-main.pg_stat_tmp | t       | 
 /etc/postgresql/13/main/postgresql.conf          |        679 |    19 | datestyle                    | iso, mdy                                | t       | 
 /etc/postgresql/13/main/postgresql.conf          |        681 |    20 | timezone                     | Etc/UTC                                 | t       | 
 /etc/postgresql/13/main/postgresql.conf          |        695 |    21 | lc_messages                  | en_US.UTF-8                             | t       | 
 /etc/postgresql/13/main/postgresql.conf          |        697 |    22 | lc_monetary                  | en_US.UTF-8                             | t       | 
 /etc/postgresql/13/main/postgresql.conf          |        698 |    23 | lc_numeric                   | en_US.UTF-8                             | t       | 
 /etc/postgresql/13/main/postgresql.conf          |        699 |    24 | lc_time                      | en_US.UTF-8                             | t       | 
 /etc/postgresql/13/main/postgresql.conf          |        702 |    25 | default_text_search_config   | pg_catalog.english                      | t       | 
 /var/lib/postgresql/13/main/postgresql.auto.conf |          3 |    26 | max_connections              | 400                                     | t       | 
 /var/lib/postgresql/13/main/postgresql.auto.conf |          4 |    27 | shared_buffers               | 1GB                                     | t       | 
 /var/lib/postgresql/13/main/postgresql.auto.conf |          5 |    28 | effective_cache_size         | 3GB                                     | t       | 
 /var/lib/postgresql/13/main/postgresql.auto.conf |          6 |    29 | maintenance_work_mem         | 512MB                                   | t       | 
 /var/lib/postgresql/13/main/postgresql.auto.conf |          7 |    30 | checkpoint_completion_target | 0.9                                     | t       | 
 /var/lib/postgresql/13/main/postgresql.auto.conf |          8 |    31 | wal_buffers                  | 16MB                                    | t       | 
 /var/lib/postgresql/13/main/postgresql.auto.conf |          9 |    32 | default_statistics_target    | 500                                     | t       | 
 /var/lib/postgresql/13/main/postgresql.auto.conf |         10 |    33 | random_page_cost             | 4                                       | t       | 
 /var/lib/postgresql/13/main/postgresql.auto.conf |         11 |    34 | effective_io_concurrency     | 2                                       | t       | 
 /var/lib/postgresql/13/main/postgresql.auto.conf |         12 |    35 | work_mem                     | 6553kB                                  | t       | 
 /var/lib/postgresql/13/main/postgresql.auto.conf |         13 |    36 | min_wal_size                 | 4GB                                     | t       | 
 /var/lib/postgresql/13/main/postgresql.auto.conf |         14 |    37 | max_wal_size                 | 16GB                                    | t       | 
(37 rows)
```

Посмотрел что добавилось в файл
```
postgres@fhm88m4kf0sln0do180s:~$ cat /var/lib/postgresql/13/main/postgresql.auto.conf 
# Do not edit this file manually!
# It will be overwritten by the ALTER SYSTEM command.
max_connections = '400'
shared_buffers = '1GB'
effective_cache_size = '3GB'
maintenance_work_mem = '512MB'
checkpoint_completion_target = '0.9'
wal_buffers = '16MB'
default_statistics_target = '500'
random_page_cost = '4'
effective_io_concurrency = '2'
work_mem = '6553kB'
min_wal_size = '4GB'
max_wal_size = '16GB'
```

Запустил бенчмарк
```
postgres@fhm88m4kf0sln0do180s:~$ pgbench -i postgres
dropping old tables...
NOTICE:  table "pgbench_accounts" does not exist, skipping
NOTICE:  table "pgbench_branches" does not exist, skipping
NOTICE:  table "pgbench_history" does not exist, skipping
NOTICE:  table "pgbench_tellers" does not exist, skipping
creating tables...
generating data (client-side)...
100000 of 100000 tuples (100%) done (elapsed 0.09 s, remaining 0.00 s)
vacuuming...
creating primary keys...
done in 0.86 s (drop tables 0.01 s, create tables 0.21 s, client-side generate 0.33 s, vacuum 0.16 s, primary keys 0.14 s).
```

Запустил второй раз бенчмарк с парамтерами: 8 подключений к БД, вывод каждые 60сек, отработать тест длиной в час
```
postgres@fhm88m4kf0sln0do180s:~$ pgbench -c8 -P 60 -T 3600 -U postgres postgres                                                                                                               
starting vacuum...end.                                                                                                                                                                        
progress: 60.0 s, 468.8 tps, lat 17.020 ms stddev 13.338                                                                                                                                      
progress: 120.0 s, 511.6 tps, lat 15.603 ms stddev 12.113                                                                                                                                     
progress: 180.0 s, 487.1 tps, lat 16.384 ms stddev 12.501                                                                                                                                     
progress: 240.0 s, 451.0 tps, lat 17.705 ms stddev 13.838                                                                                                                                     
progress: 300.0 s, 480.1 tps, lat 16.626 ms stddev 12.975                                                                                                                                     
progress: 360.0 s, 500.4 tps, lat 15.954 ms stddev 12.370                                                                                                                                     
progress: 420.0 s, 515.7 tps, lat 15.478 ms stddev 11.589                                                                                                                                     
progress: 480.0 s, 419.4 tps, lat 19.036 ms stddev 14.911                                                                                                                                     
xxx
progress: 1500.0 s, 515.3 tps, lat 15.491 ms stddev 12.193                                                                                                                                    
progress: 1560.0 s, 478.4 tps, lat 16.682 ms stddev 12.803                                                                                                                                    
progress: 1620.0 s, 454.3 tps, lat 17.575 ms stddev 15.269                                                                                                                                    
progress: 1680.0 s, 475.9 tps, lat 16.774 ms stddev 12.738                                                                                                                                    
progress: 1740.0 s, 456.1 tps, lat 17.503 ms stddev 13.665                                                                                                                                    
progress: 1800.0 s, 496.1 tps, lat 16.093 ms stddev 12.000                                                                                                                                    
progress: 1860.0 s, 468.1 tps, lat 17.056 ms stddev 13.973                                                                                                                                    
progress: 1920.0 s, 459.5 tps, lat 17.372 ms stddev 15.106                                                                                                                                    
progress: 1980.0 s, 481.3 tps, lat 16.586 ms stddev 13.496                                                                                                                                    
progress: 2040.0 s, 498.3 tps, lat 16.016 ms stddev 12.746
progress: 2100.0 s, 523.3 tps, lat 15.255 ms stddev 11.961
progress: 2160.0 s, 465.4 tps, lat 17.152 ms stddev 13.352
progress: 2220.0 s, 495.8 tps, lat 16.100 ms stddev 12.259
progress: 2280.0 s, 523.0 tps, lat 15.260 ms stddev 11.935
progress: 2340.0 s, 504.7 tps, lat 15.814 ms stddev 12.594
progress: 2400.0 s, 504.0 tps, lat 15.840 ms stddev 12.552
progress: 2460.0 s, 442.7 tps, lat 18.034 ms stddev 14.412
progress: 2520.0 s, 513.4 tps, lat 15.548 ms stddev 12.016
progress: 2580.0 s, 518.7 tps, lat 15.390 ms stddev 12.172
progress: 2640.0 s, 501.7 tps, lat 15.909 ms stddev 12.560
progress: 2700.0 s, 501.5 tps, lat 15.917 ms stddev 12.251
progress: 2760.0 s, 474.9 tps, lat 16.810 ms stddev 12.920
progress: 2820.0 s, 520.4 tps, lat 15.339 ms stddev 11.545
progress: 2880.0 s, 525.4 tps, lat 15.189 ms stddev 11.607
progress: 2940.0 s, 498.3 tps, lat 16.019 ms stddev 13.055
progress: 3000.0 s, 502.9 tps, lat 15.872 ms stddev 12.078
progress: 3060.0 s, 512.4 tps, lat 15.576 ms stddev 11.981
progress: 3120.0 s, 514.2 tps, lat 15.523 ms stddev 11.902
progress: 3180.0 s, 548.9 tps, lat 14.540 ms stddev 11.011
progress: 3240.0 s, 450.2 tps, lat 17.734 ms stddev 16.246
progress: 3300.0 s, 494.3 tps, lat 16.147 ms stddev 12.733
progress: 3360.0 s, 463.5 tps, lat 17.223 ms stddev 13.110
progress: 3420.0 s, 502.8 tps, lat 15.876 ms stddev 12.587
progress: 3480.0 s, 480.4 tps, lat 16.620 ms stddev 12.970
progress: 3540.0 s, 516.6 tps, lat 15.452 ms stddev 11.844
progress: 3600.0 s, 527.5 tps, lat 15.130 ms stddev 11.820
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
duration: 3600 s
number of transactions actually processed: 1761024
latency average = 16.319 ms
latency stddev = 12.836 ms
tps = 489.169109 (including connections establishing)
tps = 489.169409 (excluding connections establishing)
```

Где средние значение последние 1/6tps соответствует значнию 501.59tps.
Плюс на графиках инстанса в время теста виден рост операци на диск.
Попробуем настроить autovacuum оптимально. Зафиксируем значения до изменения
```
 select * from pg_file_sittings;
 ...
 /var/lib/postgresql/13/main/postgresql.auto.conf |          3 |    26 | max_connections              | 400                                     | t       | 
 /var/lib/postgresql/13/main/postgresql.auto.conf |          4 |    27 | shared_buffers               | 1GB                                     | t       | 
 /var/lib/postgresql/13/main/postgresql.auto.conf |          5 |    28 | effective_cache_size         | 3GB                                     | t       | 
 /var/lib/postgresql/13/main/postgresql.auto.conf |          6 |    29 | maintenance_work_mem         | 512MB                                   | t       | 
 /var/lib/postgresql/13/main/postgresql.auto.conf |          7 |    30 | checkpoint_completion_target | 0.9                                     | t       | 
 /var/lib/postgresql/13/main/postgresql.auto.conf |          8 |    31 | wal_buffers                  | 16MB                                    | t       | 
 /var/lib/postgresql/13/main/postgresql.auto.conf |          9 |    32 | default_statistics_target    | 500                                     | t       | 
 /var/lib/postgresql/13/main/postgresql.auto.conf |         10 |    33 | random_page_cost             | 4                                       | t       | 
 /var/lib/postgresql/13/main/postgresql.auto.conf |         11 |    34 | effective_io_concurrency     | 2                                       | t       | 
 /var/lib/postgresql/13/main/postgresql.auto.conf |         12 |    35 | work_mem                     | 6553kB                                  | t       | 
 /var/lib/postgresql/13/main/postgresql.auto.conf |         13 |    36 | min_wal_size                 | 4GB                                     | t       | 
 /var/lib/postgresql/13/main/postgresql.auto.conf |         14 |    37 | max_wal_size                 | 16GB                                    | t       | 
(37 rows)
```

Выстовил значения для ряда параметров основываясь на статьяз в интернете и в документации к PoetgreSQL 
```
 /var/lib/postgresql/13/main/postgresql.auto.conf |          3 |    26 | shared_buffers               | 1GB                                     | t       | 
 /var/lib/postgresql/13/main/postgresql.auto.conf |          4 |    27 | checkpoint_completion_target | 0.9                                     | t       | 
 /var/lib/postgresql/13/main/postgresql.auto.conf |          5 |    28 | wal_buffers                  | 16MB                                    | t       | 
 /var/lib/postgresql/13/main/postgresql.auto.conf |          6 |    29 | default_statistics_target    | 500                                     | t       | 
 /var/lib/postgresql/13/main/postgresql.auto.conf |          7 |    30 | random_page_cost             | 4                                       | t       | 
 /var/lib/postgresql/13/main/postgresql.auto.conf |          8 |    31 | max_connections              | 100                                     | f       | setting could not be applied
 /var/lib/postgresql/13/main/postgresql.auto.conf |          9 |    32 | effective_cache_size         | 2GB                                     | t       | 
 /var/lib/postgresql/13/main/postgresql.auto.conf |         10 |    33 | maintenance_work_mem         | 214MB                                   | t       | 
 /var/lib/postgresql/13/main/postgresql.auto.conf |         11 |    34 | effective_io_concurrency     | 10                                      | t       | 
 /var/lib/postgresql/13/main/postgresql.auto.conf |         12 |    35 | work_mem                     | 10737kB                                 | t       | 
 /var/lib/postgresql/13/main/postgresql.auto.conf |         13 |    36 | min_wal_size                 | 1GB                                     | t       | 
 /var/lib/postgresql/13/main/postgresql.auto.conf |         14 |    37 | max_wal_size                 | 4GB                                     | t       | 
```

Перестартовал сервис  `root@fhm88m4kf0sln0do180s:~# systemctl status postgresql@13-main.service`. Потворил запуск бенчмарка postgres.
Итоговый вывод первой попытки
```
progress: 1800.0 s, 475.7 tps, lat 16.780 ms stddev 13.320
progress: 1860.0 s, 502.2 tps, lat 15.892 ms stddev 12.599
progress: 1920.0 s, 488.7 tps, lat 16.337 ms stddev 13.199
progress: 1980.0 s, 486.5 tps, lat 16.406 ms stddev 12.680
progress: 2040.0 s, 499.8 tps, lat 15.969 ms stddev 12.230
progress: 2100.0 s, 499.7 tps, lat 15.970 ms stddev 12.574
progress: 2160.0 s, 510.0 tps, lat 15.653 ms stddev 12.475
progress: 2220.0 s, 491.5 tps, lat 16.237 ms stddev 12.248
progress: 2280.0 s, 480.6 tps, lat 16.608 ms stddev 13.208
progress: 2340.0 s, 483.0 tps, lat 16.531 ms stddev 12.436
progress: 2400.0 s, 491.3 tps, lat 16.244 ms stddev 12.575
progress: 2460.0 s, 464.1 tps, lat 17.198 ms stddev 13.259
progress: 2520.0 s, 479.3 tps, lat 16.656 ms stddev 13.852
progress: 2580.0 s, 487.0 tps, lat 16.389 ms stddev 13.234
progress: 2640.0 s, 479.1 tps, lat 16.665 ms stddev 12.818
progress: 2700.0 s, 485.6 tps, lat 16.440 ms stddev 12.586
progress: 2760.0 s, 492.0 tps, lat 16.223 ms stddev 12.704
progress: 2820.0 s, 491.7 tps, lat 16.230 ms stddev 12.900
progress: 2880.0 s, 430.5 tps, lat 18.550 ms stddev 14.439
progress: 2940.0 s, 518.1 tps, lat 15.407 ms stddev 11.415
progress: 3000.0 s, 491.1 tps, lat 16.252 ms stddev 12.571
progress: 3060.0 s, 436.4 tps, lat 18.294 ms stddev 14.075
progress: 3120.0 s, 457.4 tps, lat 17.444 ms stddev 13.137
progress: 3180.0 s, 508.1 tps, lat 15.718 ms stddev 11.969
progress: 3240.0 s, 506.6 tps, lat 15.757 ms stddev 11.799
progress: 3300.0 s, 505.8 tps, lat 15.780 ms stddev 11.980
progress: 3360.0 s, 503.6 tps, lat 15.846 ms stddev 12.212
progress: 3420.0 s, 469.8 tps, lat 16.993 ms stddev 13.254
progress: 3480.0 s, 478.0 tps, lat 16.699 ms stddev 13.008
progress: 3540.0 s, 507.8 tps, lat 15.717 ms stddev 12.006
progress: 3600.0 s, 496.3 tps, lat 16.086 ms stddev 12.329
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
duration: 3600 s
number of transactions actually processed: 1743171
latency average = 16.486 ms
latency stddev = 12.901 ms
tps = 484.208748 (including connections establishing)
tps = 484.209078 (excluding connections establishing)
```
Итоговое значение по 1/6 487.0096 tps.

Немного изменил параметры
```
/var/lib/postgresql/13/main/postgresql.auto.conf |          3 |    26 | shared_buffers               | 1GB                                     | t       | 
 /var/lib/postgresql/13/main/postgresql.auto.conf |          4 |    27 | checkpoint_completion_target | 0.9                                     | t       | 
 /var/lib/postgresql/13/main/postgresql.auto.conf |          5 |    28 | wal_buffers                  | 16MB                                    | t       | 
 /var/lib/postgresql/13/main/postgresql.auto.conf |          6 |    29 | default_statistics_target    | 500                                     | t       | 
 /var/lib/postgresql/13/main/postgresql.auto.conf |          7 |    30 | random_page_cost             | 4                                       | t       | 
 /var/lib/postgresql/13/main/postgresql.auto.conf |          8 |    31 | effective_cache_size         | 2GB                                     | t       | 
 /var/lib/postgresql/13/main/postgresql.auto.conf |          9 |    32 | effective_io_concurrency     | 10                                      | t       | 
 /var/lib/postgresql/13/main/postgresql.auto.conf |         10 |    33 | min_wal_size                 | 1GB                                     | t       | 
 /var/lib/postgresql/13/main/postgresql.auto.conf |         11 |    34 | max_wal_size                 | 4GB                                     | t       | 
 /var/lib/postgresql/13/main/postgresql.auto.conf |         12 |    35 | max_connections              | 400                                     | f       | setting could not be applied
 /var/lib/postgresql/13/main/postgresql.auto.conf |         13 |    36 | work_mem                     | 214MB                                   | t       | 
 /var/lib/postgresql/13/main/postgresql.auto.conf |         14 |    37 | maintenance_work_mem         | 1GB                                     | t       | 
 /var/lib/postgresql/13/main/postgresql.auto.conf |         15 |    38 | autovacuum_vacuum_threshold  | 150                                     | t       | 
 /var/lib/postgresql/13/main/postgresql.auto.conf |         16 |    39 | autovacuum_analyze_threshold | 100                                     | t       | 
```

Запустил бенчмарк. Зафиксировал изменения после часа работы бенчмарка. При внесенных изменениях значение операция стаала равна 487.45 tps. Не особо отличается от прошлого раза.
```
progress: 3300.0 s, 500.4 tps, lat 15.954 ms stddev 12.151
progress: 3360.0 s, 465.4 tps, lat 17.153 ms stddev 12.986
progress: 3420.0 s, 502.9 tps, lat 15.871 ms stddev 12.743
progress: 3480.0 s, 506.4 tps, lat 15.762 ms stddev 12.117
progress: 3540.0 s, 484.4 tps, lat 16.478 ms stddev 12.658
progress: 3600.0 s, 504.5 tps, lat 15.821 ms stddev 11.852
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
duration: 3600 s
number of transactions actually processed: 1759742
latency average = 16.330 ms
latency stddev = 12.697 ms
tps = 488.810854 (including connections establishing)
tps = 488.811176 (excluding connections establishing)
```


Прошлые настройки больше подходят под определение ровное значение tps на горизонте часа.

