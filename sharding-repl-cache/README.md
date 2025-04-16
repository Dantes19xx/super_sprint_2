1. Поднимаем контейнеры
docker-compose up -d

2. Инициализация реплика-сетов
docker exec -it configSrv mongosh --port 27017 #Заходим в контейнер сервера
----------------------------------------
rs.initiate({
  _id: "config_server",
  configsvr: true,
  members: [
    { _id: 0, host: "configSrv:27017" }
  ]
}) #Инициализируем сервер
#Выходим
--------------------------------------------------------
docker exec -it shard1 mongosh --port 27018 #Заходим в шард1
rs.initiate({ _id: "shard1", members: [ { _id: 0, host: "shard1:27018" }, { _id: 1, host: "shard1-1:27021" }, { _id: 2, host: "shard1-2:27022" } ] })
#Инициализируем шард 1
#Выходим
---------------------------------------------------------
docker exec -it shard2 mongosh --port 27019 #Заходим в шард2
rs.initiate({ _id: "shard2", members: [ { _id: 0, host: "shard2:27019" }, { _id: 1, host: "shard2-1:27023" }, { _id: 2, host: "shard2-2:27024" } ] })
#Инициализируем шард 2
#Выходим
------------------------------------
3. Включение шардирования.
docker exec -it mongos_router mongosh --port 27020 #заходим в роутер

sh.addShard("shard1/shard1:27018,shard1-1:27021,shard1-2:27022");
sh.addShard("shard2/shard2:27019,shard2-1:27023,shard2-2:27024");
#Добавляем шарды в роутер

sh.enableSharding("somedb") #включаем шардирование для базы данных.
db.helloDoc.createIndex({ "name": 1 }) #Создаем индекс
sh.shardCollection("somedb.helloDoc", { "name" : "hashed" } ) #указываем ключ для шардирования
#Выходим из роутера

4. Наполнение БД
docker exec -it mongos_router mongosh --port 27020 #заходим в роутер

use somedb #Меняем конекст БД
for(var i = 0; i < 1000; i++) db.helloDoc.insertOne({age:i, name:"ly"+i})
#Наполняем БД документами, выходим из роутера
-------------------------------------------------------
5. Проверка шардов и заполнености БД по шардам.
docker exec -it mongo-sharding-router mongosh --port 27020 #заходим в роутер
db.helloDoc.getShardDistribution() # Проверяем данные шардов
db.helloDoc.countDocuments() # Проверяем общее количество документов
#Выходим из роутера

docker exec -it shard1 mongosh --port 27018 #Заходим в шард1
db.helloDoc.countDocuments() # Проверяем общее количество документов
#Выходим из шарда 1

docker exec -it shard2 mongosh --port 27019 #Заходим в шард2
db.helloDoc.countDocuments() # Проверяем общее количество документов
#Выходим из шарда 2
