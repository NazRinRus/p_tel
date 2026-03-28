### Создание кластера с физической и логической репликацией
Весь стенд собирается на виртуальных машинах `Yandex Cloud`, CLI уже [настроен](https://yandex.cloud/ru/docs/cli/operations/install-cli)
#### Физическая репликация:
- Настроить физическую репликации между двумя кластерами базы данных;
- Репликация должна работать используя "слот репликации";
- Реплика должна отставать от мастера на 5 минут;
#### Логическая репликация:
- Создать на первом кластере базу данных, таблицу и наполнить ее данными (на ваше усмотрение);
- На нем же создать публикацию этой таблицы;
- На новом кластере подписаться на эту публикацию;
- Убедиться что она среплицировалась. Добавить записи в эту таблицу на основном сервере и убедиться, что они видны на логической реплике;

#### Создать сетевую инфраструктуру:
```
yc vpc network create --name test-net --description "main-net"
yc vpc subnet create --name dbs-subnet --range 192.168.0.0/24 --network-name test-net --description "dbs-subnet"
```
#### Создать виртуальные машины для PostgreSQL:
при создании ВМ с параметром `--ssh-key`, создается пользователь под именем `yc-user`, к ВМ подключаются командой типа `ssh -i ~/.ssh/id_rsa yc-user@213.165.219.149`
- Создать VM1:
```
yc compute instance create --name pg-node1-vm --hostname pg-node1-vm --cores 2 --memory 4 --create-boot-disk size=25G,type=network-hdd,image-folder-id=standard-images,image-family=ubuntu-2404-lts --network-interface subnet-name=dbs-subnet,nat-ip-version=ipv4 --ssh-key ~/.ssh/id_rsa.pub 
```
- Создать VM2:
```
yc compute instance create --name pg-node2-vm --hostname pg-node2-vm --cores 2 --memory 4 --create-boot-disk size=25G,type=network-hdd,image-folder-id=standard-images,image-family=ubuntu-2404-lts --network-interface subnet-name=dbs-subnet,nat-ip-version=ipv4 --ssh-key ~/.ssh/id_rsa.pub 
```
- Создать VM3:
```
yc compute instance create --name pg-node3-vm --hostname pg-node3-vm --cores 2 --memory 4 --create-boot-disk size=25G,type=network-hdd,image-folder-id=standard-images,image-family=ubuntu-2404-lts --network-interface subnet-name=dbs-subnet,nat-ip-version=ipv4 --ssh-key ~/.ssh/id_rsa.pub 
```
#### Созданные виртуальные машины:
```
yc compute instance list
+----------------------+-------------+---------------+---------+-----------------+--------------+
|          ID          |    NAME     |    ZONE ID    | STATUS  |   EXTERNAL IP   | INTERNAL IP  |
+----------------------+-------------+---------------+---------+-----------------+--------------+
| epd12o9pjrml700cl2j0 | pg-node2-vm | ru-central1-b | RUNNING | 158.160.8.60    | 192.168.0.33 |
| epd698bd7nktptrhlto3 | pg-node1-vm | ru-central1-b | RUNNING | 158.160.19.141  | 192.168.0.17 |
| epdml104lkoanojl7bkg | pg-node3-vm | ru-central1-b | RUNNING | 213.165.221.132 | 192.168.0.5  |
+----------------------+-------------+---------------+---------+-----------------+--------------+
```
#### Подключиться к VM в трех терминалах:
```
vm_ip_address=$(yc compute instance show --name pg-node1-vm | grep -E ' +address' | tail -n 1 | awk '{print $2}') && ssh -o StrictHostKeyChecking=no -i ~/.ssh/id_rsa yc-user@$vm_ip_address

vm_ip_address=$(yc compute instance show --name pg-node2-vm | grep -E ' +address' | tail -n 1 | awk '{print $2}') && ssh -o StrictHostKeyChecking=no -i ~/.ssh/id_rsa yc-user@$vm_ip_address

vm_ip_address=$(yc compute instance show --name pg-node3-vm | grep -E ' +address' | tail -n 1 | awk '{print $2}') && ssh -o StrictHostKeyChecking=no -i ~/.ssh/id_rsa yc-user@$vm_ip_address
```
#### Установим PostgreSQL на первой ноде:
скрипт установки тестового стенда (доступ со всех хостов, пароль `postgres:postgres`): `vim /tmp/install_postgresql.sh`
```
#!/bin/bash
sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'
sudo wget -qO- https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo tee /etc/apt/trusted.gpg.d/pgdg.asc &>/dev/null 
sudo apt update
PGVERSION=18
sudo apt install postgresql-${PGVERSION?} postgresql-${PGVERSION?}-repack postgresql-${PGVERSION?}-dbgsym postgresql-client-${PGVERSION?} unzip -y
echo "listen_addresses = ' * '" | sudo tee -a /etc/postgresql/${PGVERSION?}/main/postgresql.conf > /dev/null
sudo sed -i 's/^#*\s*wal_level\s*=\s*replica/wal_level = logical/' /etc/postgresql/${PGVERSION?}/main/postgresql.conf
echo 'host    all             all             0.0.0.0/0              scram-sha-256' | sudo tee -a /etc/postgresql/${PGVERSION?}/main/pg_hba.conf > /dev/null
echo 'host    replication     replica         0.0.0.0/0              scram-sha-256' | sudo tee -a /etc/postgresql/${PGVERSION?}/main/pg_hba.conf > /dev/null
sudo sed -i 's|host    all             all             127.0.0.1/32            scram-sha-256|host    all             all             127.0.0.1/32            trust|g' /etc/postgresql/${PGVERSION?}/main/pg_hba.conf
sudo systemctl restart postgresql.service
sudo psql -U postgres -h 127.0.0.1 -c "alter user postgres with password 'postgres';"
sudo psql -U postgres -h 127.0.0.1 -c "create user replica replication password 'replica';"
sudo wget https://edu.postgrespro.ru/demo-medium.zip -O /tmp/demo-medium.zip
sudo unzip /tmp/demo-medium.zip -d /tmp/
sudo psql -U postgres -h 127.0.0.1 < /tmp/demo-medium-20170815.sql
```
запуск скрипта: `bash /tmp/install_postgresql.sh`

#### Установим PostgreSQL на остальных нодах:
```
sudo apt update && sudo apt upgrade -y -q && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt -y install postgresql && sudo apt install unzip && sudo apt -y install mc
```
Конфигурирование PostgreSQL
```
PGVERSION=18
echo "listen_addresses = ' * '" | sudo tee -a /etc/postgresql/${PGVERSION?}/main/postgresql.conf > /dev/null
sudo sed -i 's/^#*\s*wal_level\s*=\s*replica/wal_level = logical/' /etc/postgresql/${PGVERSION?}/main/postgresql.conf
echo 'host    all             all             0.0.0.0/0              scram-sha-256' | sudo tee -a /etc/postgresql/${PGVERSION?}/main/pg_hba.conf > /dev/null
echo 'host    replication     replica         0.0.0.0/0              scram-sha-256' | sudo tee -a /etc/postgresql/${PGVERSION?}/main/pg_hba.conf > /dev/null
sudo sed -i 's|host    all             all             127.0.0.1/32            scram-sha-256|host    all             all             127.0.0.1/32            trust|g' /etc/postgresql/${PGVERSION?}/main/pg_hba.conf
sudo systemctl restart postgresql.service
sudo psql -U postgres -h 127.0.0.1 -c "alter user postgres with password 'postgres';"
sudo psql -U postgres -h 127.0.0.1 -c "create user replica replication password 'replica';"
```
перезагрузить для применения конфигурации:
```
sudo pg_ctlcluster 18 main restart
```
#### Настройка репликации на 2 VM
- удалить директорию с данными:
```
sudo pg_ctlcluster 18 main stop
sudo rm -rf /var/lib/postgresql/18/main
```
Сделать бэкап БД. Ключ `-R` создаст заготовку управляющего файла `recovery.conf`, `-C -S replica02` - создаст слот репликации.
```
sudo -u postgres pg_basebackup -D /var/lib/postgresql/18/main -R -X stream -C -S replica02 -v -P -h 192.168.0.17 -U replica -W
```
проверить:
```
sudo cat /var/lib/postgresql/18/main/postgresql.auto.conf
# Do not edit this file manually!
# It will be overwritten by the ALTER SYSTEM command.
primary_conninfo = 'user=replica password=replica channel_binding=prefer host=192.168.0.17 port=5432 sslmode=prefer sslnegotiation=postgres sslcompression=0 sslcertmode=allow sslsni=1 ssl_min_protocol_version=TLSv1.2 gssencmode=prefer krbsrvname=postgres gssdelegation=0 target_session_attrs=any load_balance_hosts=disable'
primary_slot_name = 'replica02'
```
- Обнулить лог: `sudo rm -rf /var/log/postgresql/`
- Стартовать кластер: `sudo pg_ctlcluster 18 main start`
- Смотреть как стартовал: `pg_lsclusters`
- Проверим состояние репликации на мастере:
```
sudo -u postgres psql -d demo
SELECT * FROM pg_stat_replication \gx
SELECT * FROM pg_current_wal_lsn();
```
вывод:
```
demo=# SELECT * FROM pg_stat_replication \gx
-[ RECORD 1 ]----+------------------------------
pid              | 4268
usesysid         | 16549
usename          | replica
application_name | 18/main
client_addr      | 192.168.0.33
client_hostname  | 
client_port      | 47984
backend_start    | 2026-03-28 19:37:59.923813+00
backend_xmin     | 
state            | streaming
sent_lsn         | 0/3F000168
write_lsn        | 0/3F000168
flush_lsn        | 0/3F000168
replay_lsn       | 0/3F000168
write_lag        | 
flush_lag        | 
replay_lag       | 
sync_priority    | 0
sync_state       | async
reply_time       | 2026-03-28 19:41:19.439282+00

demo=# SELECT * FROM pg_current_wal_lsn();
 pg_current_wal_lsn 
--------------------
 0/3F000168
(1 row)
```
- Проверим состояние репликации на реплике:
```
sudo -u postgres psql -d demo
select * from pg_stat_wal_receiver \gx
select pg_last_wal_receive_lsn();
select pg_last_wal_replay_lsn();
```
вывод:
```
demo=# select * from pg_stat_wal_receiver \gx
-[ RECORD 1 ]

pid                   | 9323
status                | streaming
receive_start_lsn     | 0/3F000000
receive_start_tli     | 1
written_lsn           | 0/3F000168
flushed_lsn           | 0/3F000168
received_tli          | 1
last_msg_send_time    | 2026-03-28 19:41:59.482125+00
last_msg_receipt_time | 2026-03-28 19:41:59.444524+00
latest_end_lsn        | 0/3F000168
latest_end_time       | 2026-03-28 19:39:29.422844+00
slot_name             | replica02
sender_host           | 192.168.0.17
sender_port           | 5432
conninfo              | user=replica password=******** channel_binding=prefer dbname=replication host=192.168.0.17 port=5432 fallback_application_name=18/main sslmode=prefer sslnegotiation=postgres sslcompression=0 sslcertmode=allow sslsni=1 ssl_min_protocol_version=TLSv1.2 gssencmode=prefer krbsrvname=postgres gssdelegation=0 target_session_attrs=any load_balance_hosts=disable

demo=# select pg_last_wal_receive_lsn();
 pg_last_wal_receive_lsn 
-------------------------
 0/3F000168
(1 row)

demo=# select pg_last_wal_replay_lsn();
 pg_last_wal_replay_lsn 
------------------------
 0/3F000168
(1 row)
```
#### Настройка отложенной реплики:
Так как реплика задерживает применение данных на 5 минут, слот репликации на мастере также будет отставать на 5 минут. Мастер будет хранить все WAL-файлы за эти 5 минут.
- На хосте с репликой:
```
sudo -u postgres psql -c "ALTER SYSTEM SET recovery_min_apply_delay = '5min';"
sudo systemctl restart postgresql
```
#### Настройка логической репликации
- На первой виртуальной машине (мастер)
```
-- выдать права на таблицу и схему
GRANT select ON TABLE bookings.airports_data TO replica;
GRANT USAGE ON SCHEMA bookings TO replica;
GRANT SELECT ON TABLE bookings.airports_data TO replica;

-- создать публикацию
CREATE PUBLICATION test_pub FOR TABLE bookings.airports_data;

-- Просмотр созданной публикации
\dRp+

                                      Publication test_pub
  Owner   | All tables | Inserts | Updates | Deletes | Truncates | Generated columns | Via root 
----------+------------+---------+---------+---------+-----------+-------------------+----------
 postgres | f          | t       | t       | t       | t         | none              | f
Tables:
    "bookings.airports_data"
```
- На третьей виртуальной машине:
```
CREATE DATABASE test_db;
\c test_db 
CREATE SCHEMA bookings;
CREATE TABLE bookings.airports_data (
    airport_code character(3) NOT NULL,
    airport_name jsonb NOT NULL,
    city jsonb NOT NULL,
    coordinates point NOT NULL,
    timezone text NOT NULL
);

--создадим подписку к БД по Порту с Юзером и Паролем и Копированием данных=true
CREATE SUBSCRIPTION test_sub 
CONNECTION 'host=192.168.0.17 port=5432 user=replica password=replica dbname=demo' 
PUBLICATION test_pub WITH (copy_data = true);

\dRs
SELECT * FROM pg_stat_subscription \gx
```
#### Удаление VM
```
yc compute instance delete pg-node3-vm
yc compute instance delete pg-node2-vm
yc compute instance delete pg-node1-vm
yc vpc subnet delete dbs-subnet
yc vpc network delete test-net
```
