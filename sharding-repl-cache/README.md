# sharding-repl-cache

Задания 2, 3 и 4. MongoDB с шардированием и репликацией и Redis для кеширования запросов приложения к MongoDB.

| Сервис                             | Роль                         | Порт в сети / на хосте      |
|------------------------------------|------------------------------|-----------------------------|
| `configSrv`                        | сервер конфигурации          | 27017 / 27017               |
| `shard1-1`, `shard1-2`, `shard1-3` | группа репликации `shard1`   | 27018 / 27021, 27022, 27023 |
| `shard2-1`, `shard2-2`, `shard2-3` | группа репликации `shard2`   | 27019 / 27024, 27025, 27026 |
| `mongos_router`                    | роутер                       | 27020 / 27020               |
| `redis`                            | кеш                          | 6379 / 6379                 |
| `pymongo_api`                      | приложение                   | 8080 / 8080                 |

Кеширование включается переменной окружения приложения `REDIS_URL: "redis://redis:6379"`.

## Как запустить

Запускаем MongoDB, Redis и приложение:

```shell
docker compose up -d
```

Настраиваем репликацию и шардирование, заполняем MongoDB данными:

```shell
./scripts/mongo-init.sh
```

Скрипт выполняет те же шаги, что описаны в [README задания 3](../mongo-sharding-repl/README.md#что-делает-скрипт-инициализации): инициализирует сервер конфигурации, группы репликации `shard1` и `shard2` по три реплики, добавляет шарды в роутер, шардирует коллекцию `somedb.helloDoc` по хешу поля `name`, вставляет 1000 документов и выводит количество документов и реплик в каждом шарде.

## Как проверить

http://localhost:8080 — общее количество документов (`collections`), шарды со списком реплик (`shards`) и `"cache_enabled": true`.

Количество документов и реплик в каждом шарде:

```shell
docker compose exec -T shard1-1 mongosh --port 27018 --quiet <<EOF
use somedb
db.helloDoc.countDocuments()
rs.status().members.length
EOF

docker compose exec -T shard2-1 mongosh --port 27019 --quiet <<EOF
use somedb
db.helloDoc.countDocuments()
rs.status().members.length
EOF
```

Скорость повторных запросов к кешируемому эндпоинту `/<collection_name>/users`. Первый вызов идёт в MongoDB (около 1 с), следующие в течение минуты отдаются из Redis:

```shell
for i in 1 2 3; do curl -s -o /dev/null -w "call $i: %{time_total}s\n" http://localhost:8080/helloDoc/users; done
```

```text
call 1: 1.016243s
call 2: 0.005547s
call 3: 0.005205s
```

Swagger: http://localhost:8080/docs

## Как остановить

```shell
docker compose down -v
```
