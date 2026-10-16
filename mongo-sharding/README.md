# mongo-sharding

Задание 2. MongoDB с шардированием: приложение `pymongo-api` работает с кластером из двух шардов через роутер `mongos`.

| Сервис          | Роль                         | Порт  |
|-----------------|------------------------------|-------|
| `configSrv`     | сервер конфигурации          | 27017 |
| `shard1`        | шард 1                       | 27018 |
| `shard2`        | шард 2                       | 27019 |
| `mongos_router` | роутер                       | 27020 |
| `pymongo_api`   | приложение                   | 8080  |

## Как запустить

Запускаем MongoDB и приложение:

```shell
docker compose up -d
```

Инициализируем шардирование и заполняем MongoDB данными:

```shell
./scripts/mongo-init.sh
```

## Что делает скрипт инициализации

Те же шаги можно выполнить вручную.

1. Инициализировать сервер конфигурации:

```shell
docker compose exec -T configSrv mongosh --port 27017 --quiet <<EOF
rs.initiate({
  _id: "config_server",
  configsvr: true,
  members: [{ _id: 0, host: "configSrv:27017" }]
})
EOF
```

2. Инициализировать шарды:

```shell
docker compose exec -T shard1 mongosh --port 27018 --quiet <<EOF
rs.initiate({ _id: "shard1", members: [{ _id: 0, host: "shard1:27018" }] })
EOF

docker compose exec -T shard2 mongosh --port 27019 --quiet <<EOF
rs.initiate({ _id: "shard2", members: [{ _id: 0, host: "shard2:27019" }] })
EOF
```

3. Добавить шарды в роутер, включить шардирование коллекции `somedb.helloDoc` по хешу поля `name` и заполнить её данными:

```shell
docker compose exec -T mongos_router mongosh --port 27020 --quiet <<EOF
sh.addShard("shard1/shard1:27018")
sh.addShard("shard2/shard2:27019")
sh.enableSharding("somedb")
sh.shardCollection("somedb.helloDoc", { "name": "hashed" })
use somedb
for (var i = 0; i < 1000; i++) db.helloDoc.insertOne({ age: i, name: "ly" + i })
db.helloDoc.countDocuments()
EOF
```

## Как проверить

Общее количество документов и список шардов: http://localhost:8080 (поля `collections` и `shards`).

Количество документов в каждом шарде:

```shell
docker compose exec -T shard1 mongosh --port 27018 --quiet <<EOF
use somedb
db.helloDoc.countDocuments()
EOF

docker compose exec -T shard2 mongosh --port 27019 --quiet <<EOF
use somedb
db.helloDoc.countDocuments()
EOF
```

Swagger: http://localhost:8080/docs

## Как остановить

```shell
docker compose down -v
```
