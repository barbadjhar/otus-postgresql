#DOC!
#Физический уровень PostgreSQL 

1. Создал первую ВМ "vm-pg-hw006-host-1" в Яндекс Облаке

```
yc compute instance create --name "vm-pg-hw006-host-1" --zone ru-central1-a --network-interface subnet-id=e9bl3fu9obful2c43udh,nat-ip-version=ipv4 --create-boot-disk image-folder-id=standard-images,image-family=ubuntu-2004-lts --ssh-key ~/.ssh/id_rsa_first.pub --labels tag=otus-lb
done (46s)
...


yc compute instance list | grep vm-
| fhm2rr1reb9iomjrrrg4 | vm-postgres-p1     | ru-central1-a | STOPPED |               | 10.128.0.12 |
| fhm58udq7pfkfqp03ks9 | vm-pg-hw006-host-1 | ru-central1-a | RUNNING | 51.250.67.122 | 10.128.0.23 |
| fhm5fd25ah7vk42opeen | vm-postgrsql-a     | ru-central1-a | STOPPED |               | 10.128.0.26 |
```

2. Подключился к хосту по SSH. Добавил репозиторий и установил серверную и клиентскую часть PostgreSQL 14.

```json
 dpkg -l | grep postgresq
ii  pgdg-keyring                          2018.2                            all          keyring for apt.postgresql.org
ii  postgresql                            14+241.pgdg20.04+1                all          object-relational SQL database (supported version)
ii  postgresql-14                         14.4-1.pgdg20.04+1                amd64        The World's Most Advanced Open Source Relational Database
ii  postgresql-client-14                  14.4-1.pgdg20.04+1                amd64        front-end programs for PostgreSQL 14
ii  postgresql-client-common              241.pgdg20.04+1                   all          manager for multiple PostgreSQL client versions
ii  postgresql-common                     241.pgdg20.04+1                   all          PostgreSQL database-cluster manager
```

3. Кластер рабоатет
```json
$ sudo -u postgres pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
14  main    5432 online postgres /var/lib/postgresql/14/main /var/log/postgresql/postgresql-14-main.log

```

4. Создал БД с таблице, добавил поле
```json

postgres=# create database marmelad;
CREATE DATABASE

postgres=# \c marmelad 
You are now connected to database "marmelad" as user "postgres".

marmelad=# create table melom(c1 text);
CREATE TABLE

marmelad=# insert into melom values ('1');
INSERT 0 1

marmelad=# select * from melom;
c1 
----
 1
(1 row)
```

5. В это раз зашел под пользователем postgres и остановил сервис
```json
postgres@fhm58udq7pfkfqp03ks9:~$ pg_lsclusters 
Ver Cluster Port Status Owner    Data directory              Log file
14  main    5432 online postgres /var/lib/postgresql/14/main /var/log/postgresql/postgresql-14-main.log

postgres@fhm58udq7pfkfqp03ks9:~$ pg_ctlcluster 14 main stop
Warning: stopping the cluster using pg_ctlcluster will mark the systemd unit as failed. Consider using systemctl:
  sudo systemctl stop postgresql@14-main

postgres@fhm58udq7pfkfqp03ks9:~$ pg_lsclusters 
Ver Cluster Port Status Owner    Data directory              Log file
14  main    5432 down   postgres /var/lib/postgresql/14/main /var/log/postgresql/postgresql-14-main.log
```

6. Создал network-ssd диск и добавил на ВМ
```json
 yc compute disk create --zone ru-central1-a --type network-ssd --size 10G
done (6s)
id: fhm2ma48uhs21jvnktq7
folder_id: b1gomt5860css997fvi5
created_at: "2022-07-31T16:25:55Z"
type_id: network-ssd
zone_id: ru-central1-a
size: "10737418240"
block_size: "4096"
status: READY
disk_placement_policy: {}




yc compute instance attach-disk vm-pg-hw006-host-1 --disk-id fhm2ma48uhs21jvnktq7


yc compute instance get vm-pg-hw006-host-1 | grep -i disk
boot_disk:
  disk_id: fhm9npofvofj2ljdkofi
secondary_disks:
    disk_id: fhm2ma48uhs21jvnktq7
```

Проверил на ВМ видит ли новый диск.
```json
yc-user@fhm58udq7pfkfqp03ks9:~$ lsblk 
NAME   MAJ:MIN RM SIZE RO TYPE MOUNTPOINT
vda    252:0    0   5G  0 disk 
├─vda1 252:1    0   1M  0 part 
└─vda2 252:2    0   5G  0 part /
vdb    252:16   0  10G  0 disk 
```

7. На второ диске создао файловую систему EXT4 и смотнировал, владельцем определил пользователя postgres
```json
root@fhm58udq7pfkfqp03ks9:~# lsblk -f
NAME   FSTYPE LABEL  UUID                                 FSAVAIL FSUSE% MOUNTPOINT
vda                                                                      
├─vda1                                                                   
└─vda2 ext4          82afb880-9c95-44d6-8df9-84129f3f2cd1      2G    54% /
vdb                                                                      
└─vdb1 ext4   datapg 8458118a-0a2f-447f-b117-e1d591f0fac4    9.2G     0% /mnt/data


root@fhm58udq7pfkfqp03ks9:~# ls -ld /mnt/data
drwxr-xr-x 3 postgres postgres 4096 Jul 31 16:34 /mnt/data

```

8. Перенес директорию с фалами БД на смонтированный диск
```json
root@fhm58udq7pfkfqp03ks9:~# du -sh /var/lib/postgresql /mnt/data
12K     /var/lib/postgresql
51M     /mnt/data


root@fhm58udq7pfkfqp03ks9:~# tree -d /mnt/data/
/mnt/data/
├── 14
│   └── main
│       ├── base
│       │   ├── 1
│       │   ├── 13759
│       │   ├── 13760
│       │   └── 16389
│       ├── global
│       ├── pg_commit_ts
│       ├── pg_dynshmem
│       ├── pg_logical
│       │   ├── mappings
│       │   └── snapshots
│       ├── pg_multixact
│       │   ├── members
│       │   └── offsets
│       ├── pg_notify
│       ├── pg_replslot
│       ├── pg_serial
│       ├── pg_snapshots
│       ├── pg_stat
│       ├── pg_stat_tmp
│       ├── pg_subtrans
│       ├── pg_tblspc
│       ├── pg_twophase
│       ├── pg_wal
│       │   └── archive_status
│       └── pg_xact
└── lost+found

```

9. Запустить после переноса дир. с БД не получилось, так как нет файлов БД
```json

postgres@fhm58udq7pfkfqp03ks9:~$ pg_lsclusters 
Ver Cluster Port Status Owner     Data directory              Log file
14  main    5432 down   <unknown> /var/lib/postgresql/14/main /var/log/postgresql/postgresql-14-main.log

postgres@fhm58udq7pfkfqp03ks9:~$ pg_ctlcluster 14 main start
Error: /var/lib/postgresql/14/main is not accessible or does not exist

```

10. Заменил в конф.файле путь до файлов с БД, кластер запустился.	
```json
yc-user@fhm58udq7pfkfqp03ks9:~$ sudo grep -ir "data_dir" /etc/postgresql/14/main/postgresql.conf 
#data_directory = '/var/lib/postgresql/14/main'         # use data in another directory
data_directory = '/mnt/data/14/main'            # use data in another directory

postgres@fhm58udq7pfkfqp03ks9:~$ pg_lsclusters 
Ver Cluster Port Status Owner    Data directory    Log file
14  main    5432 down   postgres /mnt/data/14/main /var/log/postgresql/postgresql-14-main.log

postgres@fhm58udq7pfkfqp03ks9:~$ pg_ctlcluster 14 main start
Warning: the cluster will not be running as a systemd service. Consider using systemctl:
  sudo systemctl start postgresql@14-main

postgres@fhm58udq7pfkfqp03ks9:~$ pg_lsclusters 
Ver Cluster Port Status Owner    Data directory    Log file
14  main    5432 online postgres /mnt/data/14/main /var/log/postgresql/postgresql-14-main.log
```


11. Все файлы принадлежазие ранее созданной БД (marmelad) были целеком в другуй директорию и после старта ничему было терятся.
```json

postgres=# 
postgres=# \l
                                  List of databases
   Name    |  Owner   | Encoding |   Collate   |    Ctype    |   Access privileges   
-----------+----------+----------+-------------+-------------+-----------------------
 marmelad  | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | 
 postgres  | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | 
 template0 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
           |          |          |             |             | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
           |          |          |             |             | postgres=CTc/postgres
(4 rows)

postgres=# \c marmelad 
You are now connected to database "marmelad" as user "postgres".

marmelad=# \d
         List of relations
 Schema | Name  | Type  |  Owner   
--------+-------+-------+----------
 public | melom | table | postgres
(1 row)

marmelad=# select * from melom;
 c1 
----
 1
(1 row)
``` 




12. Задачние с *
12.1 Ранее сдела снепшот с диска с готовым набором серверно и клиентской частью PostgreSQL.
```json
$ yc compute instance create --name "vm-pg-hw006-host-2" --zone ru-central1-a --network-interface subnet-id=e9bl3fu9obful2c43udh,nat-ip-version=ipv4 --create-boot-disk snapshot-name=vm-pg-14 --ssh-key ~/.ssh/id_rsa_first.pub --labels tag=otus-lb
done (44s)
id: fhm0i7ffvce7u4jkdltl


 yc compute instance list | grep "vm-pg"
| fhm0i7ffvce7u4jkdltl | vm-pg-hw006-host-2 | ru-central1-a | RUNNING | 62.84.113.241 | 10.128.0.18 |
| fhm58udq7pfkfqp03ks9 | vm-pg-hw006-host-1 | ru-central1-a | RUNNING | 51.250.67.122 | 10.128.0.23 |
```


12.2  На ВМ vm-pg-hw006-host-1 остановил кластер и отмантировал диск с ранее перенесенной табличным пространством 
```json
postgres@fhm58udq7pfkfqp03ks9:~$ pg_lsclusters 
Ver Cluster Port Status Owner    Data directory    Log file
14  main    5432 online postgres /mnt/data/14/main /var/log/postgresql/postgresql-14-main.log

postgres@fhm58udq7pfkfqp03ks9:~$ pg_ctlcluster 14 main stop

postgres@fhm58udq7pfkfqp03ks9:~$ pg_lsclusters 
Ver Cluster Port Status Owner    Data directory    Log file
14  main    5432 down   postgres /mnt/data/14/main /var/log/postgresql/postgresql-14-main.log

postgres@fhm58udq7pfkfqp03ks9:~$ exit
logout

yc-user@fhm58udq7pfkfqp03ks9:~$ sudo  umount /mnt/data

yc-user@fhm58udq7pfkfqp03ks9:~$ lsblk -f
NAME   FSTYPE LABEL  UUID                                 FSAVAIL FSUSE% MOUNTPOINT
vda                                                                      
├─vda1                                                                   
└─vda2 ext4          82afb880-9c95-44d6-8df9-84129f3f2cd1    2.1G    53% /
vdb                                                                      
└─vdb1 ext4   datapg 8458118a-0a2f-447f-b117-e1d591f0fac4    
``` 

Последовательность отключения диска от инстанса
```json

yc compute instance detach-disk --name vm-pg-hw006-host-1 --disk-id fhm2ma48uhs21jvnktq7
done (9s)
...

yc compute instance get vm-pg-hw006-host-1 | grep disk
boot_disk:
  disk_id: fhm9npofvofj2ljdkofi



yc-user@fhm58udq7pfkfqp03ks9:~$ lsblk -f
NAME   FSTYPE LABEL UUID                                 FSAVAIL FSUSE% MOUNTPOINT
vda                                                                     
├─vda1                                                                  
└─vda2 ext4         82afb880-9c95-44d6-8df9-84129f3f2cd1    2.1G    53% /
yc-user@fhm58udq7pfkfqp03ks9:~$ dmesg -T | tail
[Sun Jul 31 16:30:29 2022] pcieport 0000:80:01.0:   bridge window [io  0xd000-0xdfff]
[Sun Jul 31 16:30:29 2022] pcieport 0000:80:01.0:   bridge window [mem 0xfe600000-0xfe7fffff]
[Sun Jul 31 16:30:29 2022] pcieport 0000:80:01.0:   bridge window [mem 0xfbc00000-0xfbdfffff 64bit pref]
[Sun Jul 31 16:30:29 2022] virtio-pci 0000:82:00.0: enabling device (0000 -> 0003)
[Sun Jul 31 16:30:29 2022] virtio_blk virtio2: [vdb] 20971520 512-byte logical blocks (10.7 GB/10.0 GiB)
[Sun Jul 31 16:33:50 2022]  vdb: vdb1
[Sun Jul 31 16:33:50 2022]  vdb: vdb1
[Sun Jul 31 16:44:28 2022] EXT4-fs (vdb1): mounted filesystem with ordered data mode. Opts: (null)
[Sun Jul 31 17:25:46 2022] pcieport 0000:80:01.0: pciehp: Slot(1): Attention button pressed
[Sun Jul 31 17:25:46 2022] pcieport 0000:80:01.0: pciehp: Slot(1): Powering off due to button press
```



12.3 Подлючил диск и смонтировал на второй ВМ (vm-pg-hw006-host-2)
```json
 yc compute instance attach-disk vm-pg-hw006-host-2 --disk-id fhm2ma48uhs21jvnktq7
done (4s)


yc compute instance get vm-pg-hw006-host-2 | grep disk
boot_disk:
  disk_id: fhm5go0apilrjut1eg2v
secondary_disks:
    disk_id: fhm2ma48uhs21jvnktq7
```

Раздел смонирровал в ново созданную директорию.
```json
root@fhm0i7ffvce7u4jkdltl:~# lsblk -f
NAME   FSTYPE LABEL  UUID                                 FSAVAIL FSUSE% MOUNTPOINT
vda                                                                      
├─vda1                                                                   
└─vda2 ext4          82afb880-9c95-44d6-8df9-84129f3f2cd1    2.1G    53% /
vdb                                                                      
└─vdb1 ext4   datapg 8458118a-0a2f-447f-b117-e1d591f0fac4                

root@fhm0i7ffvce7u4jkdltl:~# mount /dev/vdb1 /mnt/data/

root@fhm0i7ffvce7u4jkdltl:~# ls -l /mnt/data/
total 20
drwxr-xr-x 3 postgres postgres  4096 Jul 31 15:49 14
drwx------ 2 postgres postgres 16384 Jul 31 16:34 lost+found
```

12.4 Поменял в конфиге путь до файлов с БД, т.е. все что делал в задаче без звездочки
```json
root@fhm0i7ffvce7u4jkdltl:~# cd /etc/postgresql/14/main/

root@fhm0i7ffvce7u4jkdltl:/etc/postgresql/14/main# vim postgresql.conf 

root@fhm0i7ffvce7u4jkdltl:/etc/postgresql/14/main# grep data_dir postgresql.conf
#data_directory = '/var/lib/postgresql/14/main'		# use data in another directory
data_directory = '/mnt/data/14/main'		# use data in another directory
```

12.5 Переклюился в пользователя postgres и подключился к БД marmelad. Строку ранее добавленную, вывел на экран.
```json
postgres@fhm0i7ffvce7u4jkdltl:/etc/postgresql/14/main$ pg_lsclusters 
Ver Cluster Port Status Owner    Data directory    Log file
14  main    5432 down   postgres /mnt/data/14/main /var/log/postgresql/postgresql-14-main.log

postgres@fhm0i7ffvce7u4jkdltl:/etc/postgresql/14/main$ pg_ctlcluster 14 main start

sudo su - postgres 
psql 
psql (14.4 (Ubuntu 14.4-1.pgdg20.04+1))
Type "help" for help.

postgres=# \l
                                  List of databases
   Name    |  Owner   | Encoding |   Collate   |    Ctype    |   Access privileges   
-----------+----------+----------+-------------+-------------+-----------------------
 marmelad  | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | 
 postgres  | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | 
 template0 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
           |          |          |             |             | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
           |          |          |             |             | postgres=CTc/postgres
(4 rows)

postgres=# \c marmelad 
You are now connected to database "marmelad" as user "postgres".

marmelad=# \d
         List of relations
 Schema | Name  | Type  |  Owner   
--------+-------+-------+----------
 public | melom | table | postgres
(1 row)

marmelad=# select * from melom;
 c1 
----
 1
(1 row)
``` 
