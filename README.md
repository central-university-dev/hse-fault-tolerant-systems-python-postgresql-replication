1. Создадим папку для сохранения данных наших PostgreSQL баз данных:
```bash
mkdir postgresql_data
```

2. Запускаем два инстанса PostgreSQL и pgAdmin:
```bash
docker compose up -d
```

3. Убедимся что в папке создались данные для обоих баз:
```bash
sudo ls -la postgresql_data/postgresql_01
sudo ls -la postgresql_data/postgresql_02
```

4. Создадим на базе 1 пользователя, который будет использоваться для репликации, зададим ему пароль:
```bash
# "Заходим" в консоль Docker контейнера
docker exec -it postgresql-01 bash

# От имени пользователя "admin" (указывается в docker-compose.yml) создаем пользователя "repluser" с правами на репликацию ("--replication")
createuser --replication -P repluser -U admin
```

5. Внесем изменения в конфигурацию postgresql-01
```bash
sudo nano postgresql_data/postgresql_01/postgresql.conf
# Проставить следующие значения:
# wal_level = logical
# max_wal_senders = 2
# max_replication_slots = 2
# hot_standby = on
# hot_standby_feedback = on
```

6. Так как PostgreSQL по умолчанию оперирует доступа в зависимости от IP адреса подключающегося - нам нужно разрешить взаимодействие двух БД внутри подсети Docker'а. Разрешим репликацию в рамках подсети:
```bash
# Узнаем, какая подсеть назначилась postgresql-network
docker network inspect postgresql-network | grep Subnet

# Пропишем разрешение для доступа в рамках этой подсети
sudo nano postgresql_data/postgresql_01/pg_hba.conf
# Прописать следующее значение в самый низ:
# host replication all <наша_docker_подсеть> md5
```

7. Перезапустим контейнер, чтобы БД "подхватила" новые настройки:
```bash
docker restart postgresql-01
```

8. PostgreSQL требует, чтобы настройки двух инстансов master-slave были полностью идентичными. Скопируем все настройки на инстанс 02 с инстанса 01 силами PostgreSQL:
```bash
# "Заходим" в консоль Docker контейнера
docker exec -it postgresql-02 bash

# Говорим, что нужно скопировать все данные с хоста "postgresql-01" используя созданного пользователя "repluser" и положить эти данные в папку "/var/lib/postgresql/data/basebackup"
pg_basebackup --host=postgresql-01 --username=repluser --pgdata=/var/lib/postgresql/data/basebackup --wal-method=stream --write-recovery-conf
```

9. Теперь подложим скопированные с 01 инстанса настройки в качестве основных настроек инстанса 02:
```bash
# Временно остановим наш "кластер"
docker compose stop

# Так как данные postgresql-02 примонтированы на хост (настройка в docker-compose.yml) - переместим их в другое место для сохранности
sudo mv postgresql_data/postgresql_02/basebackup postgresql_data

# Теперь полностью удалим все данные и настройки postgresql-02
# За подробностями почему команда выглядит так - https://unix.stackexchange.com/questions/687407/sudo-rm-rf-directory-does-not-work-with-certain-permission-settings
sudo /bin/sh -c 'rm -rf postgresql_data/postgresql_02/*'

# После очистки всех настроек второго инстанса - подложим на второй инстанс все ранее скопированные настройки с первого инстанса
sudo mv postgresql_data/postgresql_02/basebackup/* postgresql_data/postgresql_02/

# Снова запустим наш "кластер"
sudo docker compose start
```

10. Проверить статусы репликации можно выполнив запрос на системной БД postgres (не нашей созданной admin):
```sql
/* На master: */
select * from pg_stat_replication;

/* На slave: */
select * from pg_stat_wal_receiver;
```

11. Запустим миграцию на master, обратим внимание, что в liquibase.properties указан адрес именно master базы, на slave изменения должны отреплицировать:
```bash
sudo docker run --rm --network="postgresql-network" -v "$(pwd)/migrations":/app liquibase/liquibase:4.19.0 --defaultsFile=/app/liquibase.properties update
```

12. Можно подключиться к хостам postgresql-01 и postgresql-02 через pgAdmin и убедиться, что базы синхронизируются между собой.
