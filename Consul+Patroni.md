### Описание установки паторони под консул

#### Ставим консул

Установка производится на 3 вм 
```
postgresql-patroni0
postgresql-patroni1
postgresql-patroni2
```


Разбрасываем бинарник на все машины:
```
scp -i .\.ssh\mykeys\yo .\Desktop\otus_teacher\Postgres\Patroni\consul konstantin@158.160.103.149:\tmp
``` 

```
mv /tmp/consul /usr/bin/
chmod +x /usr/bin/consul
useradd -r -c 'Consul DCS service' consul
mkdir -p /var/lib/consul /etc/consul.d
chown consul:consul /var/lib/consul /etc/consul.d
chmod 775 /var/lib/consul /etc/consul.d
```  

Сгенерируем ключ для консула на любой из нод кластера  
`consul keygen`

Создаем конфигурационный файл для консула [/etc/consul.d/config.json](examples/config.json)

<details><summary>Параметры конфига</summary>

**bind_addr** — адрес, на котором будет слушать наш сервер консул. Это может быть IP любого из наших сетевых интерфейсов или, как в данном примере, все.  
**bootstrap_expect** — ожидаемое количество серверов в кластере.  
**client_addr** — адрес, к которому будут привязаны клиентские интерфейсы.  
**datacenter** — привязка сервера к конкретному датацентру. Нужен для логического разделения. Серверы с одинаковым датацентром должны находиться в одной локальной сети.  
**data_dir** — каталог для хранения данных.  
**domain** — домен, в котором будет зарегистрирован сервис.  
**enable_script_checks** — разрешает на агенте проверку работоспособности.  
**dns_config** — параметры для настройки DNS.  
**enable_syslog** — разрешение на ведение лога.  
**encrypt** — ключ для шифрования сетевого трафика. В качестве значения используем сгенерированный ранее.    
**leave_on_terminate** — при получении сигнала на остановку процесса консула, корректно отключать ноду от кластера.    
**log_level** — минимальный уровень события для отображения в логе. Возможны варианты "trace", "debug", "info", "warn", and "err".    
**rejoin_after_leave** — по умолчанию, нода покидающая кластер не присоединяется к нему автоматически. Данная опция позволяет управлять данным поведением.  
**retry_join** — перечисляем узлы, к которым можно присоединять кластер. Процесс будет повторяться, пока не завершиться успешно.  
**server** — режим работы сервера.  
**start_join** — список узлов кластера, к которым пробуем присоединиться при загрузке сервера.  
**ui_config** — конфигурация для графического веб-интерфейса.  

</details>

Проверяем корректность конфигурационного файла.  
`consul validate /etc/consul.d/config.json`

В завершение настройки создадим юнит в systemd для возможности автоматического запуска сервиса  
[consul.service](examples/consul.service)

Перечитываем конфигурацию systemd:  
`systemctl daemon-reload`

Стартуем наш сервис:  
`systemctl start consul`

Также разрешаем автоматический старт при запуске сервера:  
`systemctl enable consul`

Смотрим текущее состояние работы сервиса:  
`systemctl status consul`

Мы должны увидеть состояние:  
```
...
Active: active (running) since ...
...
```

Состояние нод кластера мы можем посмотреть командой:  
`consul members`

А данной командой можно увидеть дополнительную информацию:  
`consul members -detailed`

Также у нас должен быть доступен веб-интерфейс по адресу:  
`http://<IP-адрес любого сервера консул>:8500`  
Перейдя по нему, мы должны увидеть страницу со статусом нашего кластера.  
Консул установлен.


#### Установка Postgresql и Patroni

Устанавливаем Postgres и необходимы пакеты на машинах **postgresql-patroni0-2**  
```
wget -qO- https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo tee /etc/apt/trusted.gpg.d/pgdg.asc &>/dev/null
sudo apt update -y
apt install postgresql-15 postgresql15-contrib postgresql15-devel
``` 

Удаляем директорию PGDATA и отключаем сервис для запуска кластера, так как запуском и остановкой кластера будет заниматься Patroni
```
systemctl disable postgresql --now
rm -rf /var/lib/postgresql/15/main/*
```

Устанавливаем Patroni и необходимые пакеты
```
apt install -y python3 python3-pip python3-psycopg2 && \
pip3 install patroni[consul] && \
mkdir /etc/patroni
```
Создаем конфигурационный файл [/etc/patroni/patroni.yml](examples/patroni.yml)
<details><summary>Параметры конфига</summary>

**name** — имя узла, на котором настраивается данный конфиг.  
**scope** — имя кластера. Его мы будем использовать при обращении к ресурсу, а также под этим именем будет зарегистрирован сервис в consul.  
**consul-token** — если наш кластер consul использует ACL, необходимо указать токен.  
**restapi-connect_address** — адрес на настраиваемом сервере, на который будут приходить подключения к patroni.   
**restapi-auth** — логин и пароль для аутентификации на интерфейсе API.  
**pg_hba** — блок конфигурации pg_hba для разрешения подключения к СУБД и ее базам. Необходимо обратить внимание на подсеть для  
строки host replication replicator. Она должна соответствовать той, которая используется в вашей инфраструктуре.  
**postgresql-pgpass** — путь до файла, который создаст патрони. В нем будет храниться пароль для подключения к postgresql.  
**postgresql-connect_address** — адрес и порт, которые будут использоваться для подключения к СУДБ.  
**postgresql - data_dir** — путь до файлов с данными базы.  
**postgresql - bin_dir** — путь до бинарников postgresql.  
**pg_rewind, replication, superuser** — логины и пароли, которые будут созданы для базы.  

</details>

Создадим юнит [patroni.service](examples/patroni.service) в systemd для patroni.
*обратите внимание, что в официальной документации предлагается не перезапускать автоматически службу (Restart=no).*  
*Это дает возможность разобраться в причине падения базы*

Перечитываем конфигурацию systemd:  
`systemctl daemon-reload`

Разрешаем автозапуск и стартуем сервис:  
`systemctl enable patroni --now`  

Проверяем статусы сервиса на обоих серверах:  
`systemctl status patroni`  

Посмотреть список нод  
`patronictl -c /etc/patroni/patroni.yml list`

Мы увидим следующую картину
```
root@postgresql-patroni0:~# patronictl -c /etc/patroni/patroni.yml list
+ Cluster: postgres --+-----------------+---------+---------+----+-----------+
| Member              | Host            | Role    | State   | TL | Lag in MB |
+---------------------+-----------------+---------+---------+----+-----------+
| postgresql-patroni0 | 158.160.106.183 | Replica | running |  2 |         0 |
| postgresql-patroni1 | 158.160.103.149 | Leader  | running |  2 |           |
| postgresql-patroni2 | 158.160.105.130 | Replica | running |  2 |         0 |
+---------------------+-----------------+---------+---------+----+-----------+
```


Изменим параметры с помощью команды

```
patronictl -c /etc/patroni/patroni.yml edit-config
```

И посмотрим на то что для применинения параметров требуется рестарт:

```
root@postgresql-patroni0:~# patronictl -c /etc/patroni/patroni.yml list postgres
+ Cluster: postgres --+-----------------+---------+---------+----+-----------+-----------------+
| Member              | Host            | Role    | State   | TL | Lag in MB | Pending restart |
+---------------------+-----------------+---------+---------+----+-----------+-----------------+
| postgresql-patroni0 | 158.160.106.183 | Replica | running |  2 |         0 | *               |
| postgresql-patroni1 | 158.160.103.149 | Leader  | running |  2 |           | *               |
| postgresql-patroni2 | 158.160.105.130 | Replica | running |  2 |         0 | *               |
+---------------------+-----------------+---------+---------+----+-----------+-----------------+

```

Или 

```
curl -s http://localhost:8008/patroni | grep pend
{"state": "running", "postmaster_start_time": "2023-05-28 20:12:35.829118+00:00", "role": "replica", "server_version": 150003, "xlog": {"received_location": 117440512, "replayed_location": 117440512, "replayed_timestamp": "2023-05-28 20:14:05.570839+00:00", "paused": false}, "timeline": 2, "dcs_last_seen": 1685349522, "database_system_identifier": "7238325751068226103", "pending_restart": true, "patroni": {"version": "3.0.2", "scope": "postgres"}}
```
Обращаем внимание на **"pending_restart": true**

Рестарт одной ноды

```
patronictl -c /etc/patroni/patroni.yml restart postgres postgresql-patroni1
```

Рестарт всего кластера
```
patronictl -c /etc/patroni/patroni.yml restart postgres
```

Рестарт reload кластера
```
patronictl -c /etc/patroni/patroni.yml reload postgres
```

Плановое переключение
```
patronictl -c /etc/patroni/patroni.yml switchover postgres
```

Реинициализации ноды
```
patronictl -c /etc/patroni/patroni.yml reinit postgres postgresql-patroni2
```


---Немного про sysbench---

apt install git
git clone https://github.com/Percona-Lab/sysbench-tpcc.git
curl -s https://packagecloud.io/install/repositories/akopytov/sysbench/script.deb.sh | sudo bash
sudo apt -y install sysbench

./tpcc.lua --pgsql-port=5432 --pgsql-user=postgres --pgsql-password=postgres --pgsql-db=postgres --time=10 --threads=1 --tables=1 --scale=1 --report-interval=5 --db-driver=pgsql prepare

./tpcc.lua --pgsql-port=5432 --pgsql-user=postgres --pgsql-password=postgres --pgsql-db=postgres --time=10 --threads=1 --tables=1 --scale=1 --report-interval=5 --db-driver=pgsql run

./tpcc.lua --pgsql-port=5432 --pgsql-user=postgres --pgsql-password=postgres --pgsql-db=postgres --time=10 --threads=1 --tables=1 --scale=1 --report-interval=5 --db-driver=pgsql cleanup