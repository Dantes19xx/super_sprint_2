1. Поднимаем контейнеры

docker-compose up -d

2. Инициализация реплика-сетов
docker exec -it mongo-sharding-configSrv mongosh --port 27017 #Заходим в контейнер сервера

rs.initiate({
  _id: "config_server",
  configsvr: true,
  members: [
    { _id: 0, host: "configSrv:27017" }
  ]
}) #Инициализируем сервер

docker exec -it mongo-sharding-shard1 mongosh --port 27018 #Заходим в шард1
rs.initiate({
  _id: "shard1",
  members: [
    { _id: 0, host: "mongo-sharding-shard1:27018" }
  ]
}) #Инициализируем шард 1

docker exec -it mongo-sharding-shard2 mongosh --port 27019 #Заходим в шард2
rs.initiate({
  _id: "shard2",
  members: [
    { _id: 0, host: "mongo-sharding-shard2:27019" }
  ]
}) #Инициализируем шард 2

3. Добавление и включение шардирования.
docker exec -it mongo-sharding-router mongosh --port 27020 #заходим в роутер

sh.addShard("shard1/mongo-sharding-shard1:27018")
sh.addShard("shard2/mongo-sharding-shard2:27019") #Добавляем шарды в роутер

sh.enableSharding("somedb") #включаем шардирование для базы данных.
db.helloDoc.createIndex({ age: 1 }) #Создаем индекс для поля age
sh.shardCollection("somedb.helloDoc", { age: 1 }) #указываем ключ для шардирования
sh.splitAt("somedb.helloDoc", { age: 500 }) #Разбиваем порог age для шардирования пополам
sh.moveChunk("somedb.helloDoc", { age: 500 }, "shard1") # Переносим один чанк из второго шарда в первый

#Выходим из роутера

4. Наполнение БД
docker exec -it mongo-sharding-router mongosh --port 27020 #заходим в роутер

use somedb
for(var i = 0; i < 1000; i++) db.helloDoc.insertOne({age:i, name:"ly"+i})
EOF #Наполняем БД документами, выходим из роутера

5. Проверка шардов и заполнености БД по шардам.
docker exec -it mongo-sharding-router mongosh --port 27020 #заходим в роутер

db.helloDoc.getShardDistribution() # Проверяем данные шардов
db.helloDoc.countDocuments() # Проверяем общее количество документов

