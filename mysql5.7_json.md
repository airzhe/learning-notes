##Mysql5.7新特性之一JSON数据类型探索
mysql作为一款开源数据库，被各大公司使用在自己的项目中，目前mysql最新版本为5.7.13,在5.7中,mysql加入了不小新的特性。(号称比mysql5.6快3倍)
1. 性能和可扩展性：改进 InnoDB 的可扩展性和临时表的性能，从而实现更快的网络和大数据加载等操作。
2. JSON支持：使用 MySQL 的 JSON 功能，你可以结合 NoSQL 的灵活和关系数据库的强大。
3. 改进复制 以提高可用性的性能。包括多源复制，多从线程增强，在线 GTIDs，和增强的半同步复制。
4. 性能模式 提供更好的视角。我们增加了许多新的监控功能，以减少空间和过载，使用新的 SYS 模式显著提高易用性。
5. 安全: 我们贯彻“安全第一”的要求，许多 MySQL 5.7 新功能帮助用户保证他们数据库的安全。
6. 优化: 我们重写了大部分解析器，优化器和成本模型。这提高了可维护性，可扩展性和性能。
7. GIS: MySQL 5.7 全新的功能，包括 InnoDB 空间索引，使用 Boost.Geometry，同时提高完整性和标准符合性。

#####新建一个表

```sql
CREATE TABLE `workdb`.`members` (  
`uid` INT NOT NULL AUTO_INCREMENT,  
`payload` JSON NOT NULL,  
PRIMARY KEY (`uid`));  
```

#####插入一条数据
和平常我们json字符串格式一样。
```sql
INSERT INTO members (`uid` , `payload`) VALUES(NULL , '{"username":"lzf","province":"山东","weight":200,"pwd":"123456"}'); 
```

#####查询数据
如何查询数据，在mysql不支持json之前，我们一般是把字符串读取出来先后解析才能读出我们想要的字段，现在我们可以利用mysql提供我们一个函数json_extract来直接在数据库里查询。

######1)查询username
```sql
SELECT json_extract(payload,'$.username') FROM members  
```
######2)从复合json中查询数据
```sql

INSERT INTO members (`uid` , `payload`) VALUES(NULL , '{"username":{"last_name":"lu","first_name":"zhengfei"},"province":"山东","weight":200,"pwd":"123456"}');  
SELECT json_extract(payload,'$.username.last_name') FROM members;  
```
######3)使用->查询
```sql
SELECT payload->'$.pwd' FROM members ORDER BY json_extract(payload,'$.pwd') DESC;  
```

######4)从数组中查询数据
```sql
INSERT INTO members (`uid` , `payload`) VALUES(NULL , '{"username":"lzf","province":"山东","weight":200,"pwd":"123456","fav":["book","movies","coding"]}');  
SELECT json_extract(payload,'$.fav[0]') FROM members;  
```
#####虚拟列

可能大家会觉得每次查询都要带个json_extract那不是太麻烦了，下面来介绍一个5.7的另一个新功能Generated Column。

Generated Column是MySQL 5.7引入的新特性，所谓Cenerated Column，就是数据库中这一列由其他列计算而得,而Generated Column又分为Virtual Column与Stored Column 前者只将Generated Column保存在数据字典中（表的元数据），并不会将这一列数据持久化到磁盘上；后者会将Generated Column持久化到磁盘上，而不是每次读取的时候计算所得。很明显，后者存放了可以通过已有数据计算而得的数据，需要更多的磁盘空间，与Virtual Column相比并没有优势，因此，MySQL 5.7中，不指定Generated Column的类型，默认是Virtual Column。此外： Stored Generated Column性能较差，如果需要Stored Generated Golumn的话，可能在Generated Column上建立索引更加合适。
######1)在原json数据中新建一个虚拟列
```sql
ALTER TABLE members ADD province varchar(128) GENERATED ALWAYS AS (json_extract(payload,'$.province')) VIRTUAL;  
```
重新查看表结构
```sql
desc members;  
```
重新查询数据
```sql
SELECT uid,province FROM members;  
```
mysql因为json新特性具备了nosql数据库数据可扩展的特点，但nosql因为非关系特性而导致不能使用join特性，举个例子像订单表、订单商品表，关系是一对多，如果拿mongodb来存储，如果用一个集合来存储，那每条订单记录下会缀着他的商品列表，这样设计会感觉有些奇怪，如果分两个集合，则查询的时候又会费力不少，很多人感觉nosql因为数据格式的自由性而在前期忽略了集合数据格式的设计，我个人感觉其实nosql对前期数据格式设计要求更高，那mysql使用了json能不能还能用join呢。
```sql
SELECT a.uid,a.title,b.name FROM thread a LEFT JOIN thread_content b ON a.uid=b.uid WHERE b.uid=7374428;  mysql
```

#####如何更新JSON类型的数据
```sql
UPDATE members SET payload = JSON_SET(payload,'$.pwd','99999') where uid=4;  
```
#####添加字段
```sql
ALTER TABLE thread ADD hot int generated always as (json_extract(payload,'$.hot')) VIRTUAL;(0.3秒)  
```
添加索引
```sql
ALTER TABLE thread ADD INDEX idx_views (views); 
```
    
    