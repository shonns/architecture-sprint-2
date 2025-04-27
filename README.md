## Файл draw.io с пятью схемами расположен в корневой директории. Название: sprint2_scheme.drawio

## Как запустить в Docker Desktop для Windows.

> **_NOTE:_** Если у вас уже есть запущенные контейнеры, то их можно остановить и удалить при помощи нижеуказанных команд

```shell
// Остановить все контейнеры
docker stop $(docker ps -aq)
// Удалить все контейнеры
docker rm $(docker ps -aq)
```

1. Перейти по пути проекта:

```shell
cd sharding-repl-cache
```

2. Инициировать запуск контейнеров по конфигурационному файлу compose.yaml

```shell
docker compose up -d
```

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
docker exec -it shard1-1 mongosh --port 27018

rs.initiate({
  _id: "shard1",
  members: [{ _id : 0, host : "shard1-1:27018" },    { _id : 1, host : "shard1-2:27018" },    { _id : 2, host : "shard1-3:27018" }]
});
```

5. Инициализируем shard2

```shell
docker exec -it shard2-1 mongosh --port 27019

rs.initiate({
  _id: "shard2",
  members: [{ _id : 0, host : "shard2-1:27019" },    { _id : 1, host : "shard2-2:27019" },    { _id : 2, host : "shard2-3:27019" }]
});
```

6. Инициализируем роутер и передаем ему информацию о шардах

```shell
docker exec -it mongos_router mongosh --port 27020

sh.addShard("shard1/shard1-1:27018");
sh.addShard("shard2/shard2-1:27019");
```

7. Делаем доступным шардирования для somedb

```shell
sh.enableSharding("somedb");
sh.shardCollection("somedb.helloDoc", { "name" : "hashed" });
```

8. Наполняем данными через роутер

```shell
use somedb

for(var i = 0; i < 1000; i++) db.helloDoc.insertOne({age:i, name:"ly"+i})
```

9. Можем проверить, что данные были добавлены при помощи метода countDocuments() на каждом из шардов.

```shell
docker exec -it shard1-1 mongosh --port 27018
use somedb;
db.helloDoc.countDocuments();
exit();

docker exec -it shard2-1 mongosh --port 27019
use somedb;
db.helloDoc.countDocuments();
exit();
```

10. Чтобы проверить реплицируются ли наши данные проверим статус на shard1 и shard2

```shell
docker exec -it shard1-1 mongosh --port 27018
rs.status();
exit();

docker exec -it shard2-1 mongosh --port 27019
rs.status();
exit();
```

11. Если данные по количеству документов на шардах отобразились и по командам п.10 отобразились данные о репиках, то все настроено корректно.
    В противном случае необходимо внимательно свериться с вышеуказанными пунктами.

> **_NOTE:_** Согласно приложенной схеме sprint2.scheme.drawio узлы shard1-1 и shard2-1 являются PRIMARY. Но в MongoDB выбор какой узел является PRIMARY происходит автоматически через механизм выбора лидера в репликасете. Чтобы данные соответствовали схеме можно явно указать приоритеты в рамках инициализации репликасета.

```shell
docker exec -it shard1-1 mongosh --port 27018

rs.initiate({
 _id: "shard2",
 members: [{ _id : 0, host : "shard1-1:27018", priority: 10},    { _id : 1, host : "shard1-2:27019", priority: 5 },    { _id : 2, host : "shard1-3:27019", priority: 1 }]
});
```

```shell
docker exec -it shard2-1 mongosh --port 27019

rs.initiate({
  _id: "shard2",
  members: [{ _id : 0, host : "shard2-1:27019", priority: 10 },    { _id : 1, host : "shard2-2:27019", priority: 5 },    { _id : 2, host : "shard2-3:27019", priority: 1 }]
});
```

> **_NOTE:_** Либо сделать реконфигурацию с помощью скрипта уже после инициализации:

```shell
docker exec -it shard1-1 mongosh --port 27018
```

```javascript
cfg = rs.conf();
cfg.members[0].priority = 10;
rs.reconfig(cfg, { force: true });
```

```shell
docker exec -it shard2-1 mongosh --port 27019
```

```javascript
cfg = rs.conf();
cfg.members[0].priority = 10;
rs.reconfig(cfg, { force: true });
```

12. После включения в приложении Redis можем проверить его работоспособность.
    Для этого можно следовать таким этапам:

- Остановить запущенный контейнер redis
- Зайти на url http://localhost:8080/helloDoc/users в браузере. Ресурс выдаст 500 ошибку "Internal Server Error"
- Включить контейнер redis
- Открыть в браузере средства разработчика на вкладке "Networks" и обновить ресурс http://localhost:8080/helloDoc/users в браузере.
  На данном этапе в redis еще не сохранен кеш данных и загрузка выполняется ~1 секунду
- Обновить страницу в браузере, Redis должен отработать и загрузка ускорится до < 100 миллисекунд
