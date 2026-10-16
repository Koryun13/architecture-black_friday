# mongo-sharding-repl

Задание 3. MongoDB с шардированием и репликацией: два шарда, у каждого шарда группа репликации из трёх реплик.

| Сервис                           | Роль                                  | Порт в сети / на хосте     |
|----------------------------------|---------------------------------------|----------------------------|
| `configSrv`                      | сервер конфигурации                   | 27017 / 27017              |
| `shard1-1`, `shard1-2`, `shard1-3` | группа репликации `shard1`          | 27018 / 27021, 27022, 27023 |
| `shard2-1`, `shard2-2`, `shard2-3` | группа репликации `shard2`          | 27019 / 27024, 27025, 27026 |
| `mongos_router`                  | роутер                                | 27020 / 27020              |
| `pymongo_api`                    | приложение                            | 8080 / 8080                |

## Как запустить

Запускаем MongoDB и приложение:

```shell
docker compose up -d
```

Настраиваем репликацию и шардирование, заполняем MongoDB данными:

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

2. Настроить репликацию для первого шарда — группа `shard1` из трёх реплик:

```shell
docker compose exec -T shard1-1 mongosh --port 27018 --quiet <<EOF
rs.initiate({
  _id: "shard1",
  members: [
    { _id: 0, host: "shard1-1:27018" },
    { _id: 1, host: "shard1-2:27018" },
    { _id: 2, host: "shard1-3:27018" }
  ]
})
EOF
```

3. Настроить репликацию для второго шарда — группа `shard2` из трёх реплик:

```shell
docker compose exec -T shard2-1 mongosh --port 27019 --quiet <<EOF
rs.initiate({
  _id: "shard2",
  members: [
    { _id: 0, host: "shard2-1:27019" },
    { _id: 1, host: "shard2-2:27019" },
    { _id: 2, host: "shard2-3:27019" }
  ]
})
EOF
```

4. Дождаться, пока в каждой группе выберется PRIMARY (несколько секунд), и проверить состояние реплик:

```shell
docker compose exec -T shard1-1 mongosh --port 27018 --quiet --eval "rs.status().members.map(m => m.name + ' ' + m.stateStr)"
docker compose exec -T shard2-1 mongosh --port 27019 --quiet --eval "rs.status().members.map(m => m.name + ' ' + m.stateStr)"
```

5. Добавить шарды в роутер со списком всех реплик, включить шардирование коллекции `somedb.helloDoc` и заполнить её данными:

```shell
docker compose exec -T mongos_router mongosh --port 27020 --quiet <<EOF
sh.addShard("shard1/shard1-1:27018,shard1-2:27018,shard1-3:27018")
sh.addShard("shard2/shard2-1:27019,shard2-2:27019,shard2-3:27019")
sh.enableSharding("somedb")
sh.shardCollection("somedb.helloDoc", { "name": "hashed" })
use somedb
for (var i = 0; i < 1000; i++) db.helloDoc.insertOne({ age: i, name: "ly" + i })
db.helloDoc.countDocuments()
EOF
```

## Как проверить

http://localhost:8080 показывает общее количество документов (`collections`) и шарды со списком реплик (`shards`).

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

Swagger: http://localhost:8080/docs

## Как остановить

```shell
docker compose down -v
```
