### Поднятие MySQL в конейтнере
Контейнер будет поднят на виртуальной машине `192.168.0.140`
#### Склонировать репозиторий
```
cd /tmp
git clone https://github.com/aeuge/otus-mysql-docker.git
```
#### Поднять сервис
```
cd otus-mysql-docker/
sudo docker-compose up otusdb

(for_databases) nazrinrus@test-host:~$ sudo docker ps
CONTAINER ID   IMAGE          COMMAND                  CREATED         STATUS         PORTS                                                  NAMES
6b4bfc0b2044   mysql:8.0.15   "docker-entrypoint.s…"   2 minutes ago   Up 2 minutes   33060/tcp, 0.0.0.0:3309->3306/tcp, :::3309->3306/tcp   otus-mysql-docker_otusdb_1
```
#### Подключиться к контейнеру с удаленного терминала
Подключение с удаленного хоста с адресом `192.168.0.100`
```
(ansible) nazrinrus@desktop:~/PGMech$ mysql -h 192.168.0.140 -u root -p12345 --port=3309 --protocol=tcp otus -e 'SHOW DATABASES;'
mysql: [Warning] Using a password on the command line interface can be insecure.
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| otus               |
| performance_schema |
| sys                |
+--------------------+
```
