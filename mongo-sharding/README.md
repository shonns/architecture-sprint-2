## Как запустить в Docker Desktop для Windows.

> **_NOTE:_**  Если у вас уже есть запущенные контейнеры, то их можно остановить и удалить при помощи нижеуказанных команд
```shell
// Остановить все контейнеры
docker stop $(docker ps -aq)
// Удалить все контейнеры
docker rm $(docker ps -aq)
```

1. Перейти по пути проекта:

```shell
cd mongo-sharding
```
2. Инициировать запуск контейнеров по конфигурационному файлу compose.yaml
```shell
docker compose up -d
```
Если в докере уже были подняты контейнеры, то их можно убрать при помощи команды: docker compose down

3. Инициализируем сервер конфигурации mongosh

```shell
docker exec -it configSrv mongosh --port 27017 

rs.initiate({
  _id: "config_server",
  configsvr: true,
  members: [{ _id: 0, host: "configSrv:27017" }]
});
```

4. Инициализируем shard1

```shell
docker exec -it shard1 mongosh --port 27018 

rs.initiate({
  _id: "shard1",
  members: [{ _id: 0, host: "shard1:27018" }]
});
```

5. Инициализируем shard2

```shell
docker exec -it shard2 mongosh --port 27019 

rs.initiate({
  _id: "shard2",
  members: [{ _id: 0, host: "shard2:27019" }]
});
```

6. Инициализируемс роутер и передаем ему информацию о шардах
```shell
docker exec -it mongos_router mongosh --port 27020 
sh.addShard("shard1/shard1:27018");
sh.addShard("shard2/shard2:27019");
```

7. Делаем доступным шардирования для somedb 

```shell
sh.enableSharding("somedb");
sh.shardCollection("somedb.helloDoc", { "name" : "hashed" });
```

8. Наполняем тестовыми данными через роутер
```shell
use somedb

for(var i = 0; i < 1000; i++) db.helloDoc.insertOne({age:i, name:"ly"+i})
```
9. Можем проверить, что данные были добавлены при помощи метода countDocuments() на каждом из шардов.
```shell
docker exec -it shard1 mongosh --port 27018
use somedb;
db.helloDoc.countDocuments();
exit(); 
  
docker exec -it shard2 mongosh --port 27019
use somedb;
db.helloDoc.countDocuments();
exit();  
```
10. Если данные по количеству документов на шардах отобразились, то все настроено корректно. В противном случае необходимо внимательно свериться с вышеуказанными пунктами.