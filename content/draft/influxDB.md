---
author: "Bowser"
date: 2019-07-28
title: "influxDB使用"
draft: true
tags: [
    "DB",
    "influxdb"
]
categories: [
    "index"
]
---

时序数据库influxDB初步使用

<!-- more -->

### 1. 安装

安装的时候去[官网](https://portal.influxdata.com/downloads/)

```
wget https://dl.influxdata.com/influxdb/releases/influxdb_1.7.7_amd64.deb
#建议不要用wget，速度很慢，并且可能会和中断，用Aria2c！
#  aria2c -s 32 https://dl.influxdata.com/influxdb/releases/influxdb_1.7.7_amd64.deb
sudo dpkg -i influxdb_1.7.7_amd64.deb
```

### 2. 基本概念

示例数据

```
name: census
-————————————
time             butterflies     honeybees     location   scientist
```

|       名词       |                             解释                             |                             举例                             |
| :--------------: | :----------------------------------------------------------: | :----------------------------------------------------------: |
|     database     |                          数据库名字                          |                             mydb                             |
|   measurement    |                      类似于sql中的table                      |                            census                            |
|    field key     |                           字段的键                           |                         butterflies                          |
|     tag key      | tag区别于field可以用来索引，后者是存储的数据，tag是帮助查询的 |                           location                           |
|      value       |                      上面两个key的value                      |                              12                              |
| retention policy |                           储存策略                           |                           autogen                            |
|     tag set      |                        相同tag的集合                         | location = 1, scientist = langstroth location = 2, scientist = langstroth两组 |
|    field set     |                      相同的field的集合                       |                                                              |
|      series      |  series是相同retention policy，measurement和tag set的集合。  |                                                              |
|      point       |      point指具有相同timestamp的相同series的filed的集合       |                                                              |
|                  |                                                              |                                                              |

### 3. 初次使用

我们使用`influxDB`自带的influx工具与DB通信，也是cs架构。

```
sudo systemctl start influxdb.service
#开启influxDB服务，可以多尝试使用systemctl命令来进行服务的关闭开启
#sudo systemctl stop influxdb.service就是关闭服务influx -precision rfc3339
#precision的意思是选择返回时间戳的格式和精度#rfc3339是返回RFC339格式(YYYY-MM-DDTHH:MM:SS.nnnnnnnnnZ)的时间戳。
```

然后我们就进入了类似于其他DB的CRUD环境，使用`exit`可以退出到终端

CRUD

```
> CREATE DATABASE mydb#创建一个名字为mydb的数据库#如果没有返回信息说明没有问题，没有消息就是好消息
INSERT cpu,host=serverA, region=us, value=64#插入一条数据，measurement是cpu(类似于sql中的table)，tag-value有三对SELECT "host", "region", "value" FROM "cpu"
#应为influxDB默认使用UTC时区，但是我们是UTC+8，所以有两种解决方法#查出来数据之后加8使用#tz命令，选择时区，查询的就是上海/中国时区。
SELECT "host", "region", "value" FROM "cpu"tz("Asia/Shanghai")
```

### 4. 使用`http`接口CRUD

##### 创建查询

```
post
http://localhost:8086/query
Headers: Content-Type:application/x-www-form-urlencoded
Body: q:CREATE DATABSE testdb #就是创建一个数据库testdb
post
http://localhost:8086/write?db=mydb
#data: binary格式
body: 'cpu_load_short,host=server01,region=us-west value=0.64 1434055562000000000'的二进制文件格式
```

可以利用`crul`

```
curl -i -XPOST http://localhost:8086/query --data-urlencode "q=CREATE DATABASE mydb"
#创建数据库curl -i -XPOST 'http://localhost:8086/write?db=mydb' --data-binary 'cpu_load_short,host=server01,region=us-west value=0.64 1434055562000000000'
#写入数据到mydb，最后面的是时间戳即自定义时间，注意依旧是UTC时间格式curl -i -XPOST 'http://localhost:8086/write?db=mydb' --data-binary 'cpu_load_short,host=server02 value=0.67 cpu_load_short,host=server02,region=us-west value=0.55 1422568543702900257  cpu_load_short,direction=in,host=server01,region=us-west value=2.0 1422568543702900257'
#同时写入多个数据curl -i -XPOST 'http://localhost:8086/write?db=mydb' --data-binary @cpu_data.txt
#写入文件中的数据，文件格式必须如下
#cpu_load_short,host=server02 value=0.67
#cpu_load_short,host=server02,region=us-west value=0.55 1422568543702900257
#cpu_load_short,direction=in,host=server01,region=us-west value=2.0 1422568543702900257
```

##### 查询

```
GEThttp://localhost:8086/query?pretty=true#pretty=true是为了使输出json格式化Headers: Content-Type:application/x-www-form-urlencodedBody: "db=mydb", "q=SELECT \"value\" FROM \"cpu_load_short\" WHERE \"region\"='us-west'"#注意这里的转义字符#返回的是json值
curl -G 'http://localhost:8086/query?pretty=true' --data-urlencode "db=mydb" --data-urlencode "q=SELECT \"value\" FROM \"cpu_load_short\" WHERE \"region\"='us-west'"#查找serious为"cpu_load_short","region"="us-west"的value
```

查询可选参数

- 时间戳格式epoch 可选[h,m,s,ms,u,ns]的其中一个

  ```
  curl -G 'http://localhost:8086/query' --data-urlencode "db=mydb" --data-urlencode "epoch=s" --data-urlencode "q=SELECT \"value\" FROM \"cpu_load_short\" WHERE \"region\"='us-west'"
  ```

- …….

##### HTTP返回值

- 2xx：如果返回`HTTP 204 No Content`，那就操作成功
- 4xx: 表示不能识别请求
- 5xx： 系统过载，或者是应用受损

### 5. `HTTPS`设置

1. 安装`SSL`/`TLS`证书

   将私钥文件（.key）和签名的证书文件（`.crt`）或单个捆绑文件（`.pem`）放在`/etc/ssl`目录中。

2. 确保文件权限

   ```
   sudo chown root:root /etc/ssl/<CA-certificate-file>sudo chmod 644 /etc/ssl/<CA-certificate-file>sudo chmod 600 /etc/ssl/<private-key-file>
   ```

3. 在`influxDB`设置中打开

   默认`HTTPS`是关闭的，在`InfluxDB`的配置文件`/etc/influxdb/influxdb.conf`的`[http]`部分通过如下设置开启`HTTPS`：

   - `https-enabled`设为true
   - http-certificate设为/etc/ssl/.crt(或者/etc/ssl/.pem)
   - http-private-key设为/etc/ssl/.key(或者/etc/ssl/.pem)

4. 重启influxDB

   ```
   sudo systemctl restart influxdb
   ```

5. 验证

   ```
   influx -ssl -host localhost.com
   ```

### 6. `influxDB`数据保留策略（Retention Policies）

influxDB每秒可以处理成千上万条数据，要将这些数据全部保存下来会占用大量的存储空间，有时我们可能并不需要将所有历史数据进行存储，因此，InfluxDB推出了数据保留策略（Retention Policies），用来让我们自定义数据的保留时间。
InfluxDB的数据保留策略（RP） 用来定义数据在InfluxDB中存放的时间，或者定义保存某个期间的数据。
一个数据库可以有多个保留策略，但每个策略必须是独一无二的。

1. 查询某数据库的policies

   ```
   >  SHOW RETENTION POLICIES mydbname    duration shardGroupDuration replicaN default----          --------       ------------------               --------       -------autogen 0s            168h0m0s                       1                true
   ```

   数据库mydb只有一个策略，名字是`autogen`，各字段解释

   - name：名称，此示例名称为 autogen。当你创建一个数据库的时候，InfluxDB会自动为数据库创建一个名叫 autogen 的策略，这个策略会永久保存数据。你可以重命名这个策略，并且在InfluxDB的配置文件中禁止掉自动创建策略。
   - duration：数据保存时间，0代表无限制
   - shardGroupDuration：shardGroup的存储时间，shardGroup是InfluxDB的一个基本储存结构。
   - replicaN：全称是REPLICATION，副本个数
   - default：是否是默认策略。

2. 两个概念

   - shard

     shard 在 InfluxDB 中是一个比较重要的概念，它和 retention policy 相关联。每一个存储策略下会存在许多 shard，每一个 shard 存储一个指定时间段内的数据，并且不重复，例如 7点-8点 的数据落入 shard0 中，8点-9点的数据则落入 shard1 中。

     创建数据库时会自动创建一个默认存储策略，永久保存数据，对应的在此存储策略下的 shard 所保存的数据的时间段为 7 天，也就是上面查询时看到的168h。计算的函数如下

     ```javascript
     func shardGroupDuration(d time.Duration) time.Duration {    
     	if d >= 180*24*time.Hour || d == 0 { 
             // 6 months or 0        
             return 7 * 24 * time.Hour    
         } 
     	else if d >= 2*24*time.Hour { 
             // 2 days        
             return 1 * 24 * time.Hour    
         }    
         return 1 * time.Hour
     }
     ```

     如果创建一个新的 retention policy 设置数据的保留时间为 1 天，则单个 shard 所存储数据的时间间隔为 1 小时，超过1个小时的数据会被存放到下一个 shard 中。

   - shard group

     shard group是shards的逻辑容器

3. 创建策略

   ```
   CREATE RETENTION POLICY <retention_policy_name> ON <database_name> DURATION <duration> REPLICATION <n> [SHARD DURATION <duration>] [DEFAULT]
   ```

   其中：SHARD DURATION子句决定了每个shard group存储的时间间隔，在永久存储的策略里这个子句是无效的。这个子句是可选的。shard group duration默认由策略的 duration 决定。`上面的函数`

   | Retention Policy’s DURATION  | Shard Group Duration |
   | :--------------------------: | :------------------: |
   |           <2 days            |        1 hour        |
   |          > 6 months          |        7 days        |
   | hour>= 2 days && <= 6 months |        1 day         |

   ```
   CREATE RETENTION POLICY "one_day_only" ON "mydb" DURATION 1d REPLICATION 
   #创建策略
   CREATE RETENTION POLICY "one_day_only" ON "mydb" DURATION 23h60m REPLICATION 1 DEFAULT
   #创建默认策略
   ALTER RETENTION POLICY <retention_policy_name> ON <database_name> DURATION <duration> REPLICATION <n> SHARD DURATION <duration> DEFAULT
   #修改策略
   DROP RETENTION POLICY <retention_policy_name> ON <database_name>
   #删除策略
   ```

### 7. 一些小问题

1. 有些时候where语句值必须用 ‘’,不可以用””

### 8.`Influxsql`

```mysql
SELECT COUNT(*) FROM "insight"."autogen"."data1" WHERE time > 1564416000/timestamp AND "mianCat"='pageView' AND "pid"='ccnubox' 
#返回某时间之后的所有页面的PV
SELECT COUNT(*) FROM "insight"."autogen"."data1" WHERE time > 1564416000 AND "mianCat"='pageView' AND "pid"='ccnubox' AND "subCat"='com.muxistudio.grade.main'
#返回某时间之后的成绩请求页面PV
SELECT COUNT(*) FROM "insight"."autogen"."data1" WHERE time > 1564416000 AND "mianCat"='pageView' AND "pid"='ccnubox' AND "subCat"='com.muxistudio.course.main'
#返回某时间之后的课程请求页面PV
SELECT COUNT(*) FROM "insight"."autogen"."data1" WHERE time > 1564416000 AND "mianCat"='pageView' AND "pid"='ccnubox' GROUP BY subCat
SELECT COUNT(*) FROM data1 WHERE time > now()-1d  AND "mianCat"='pageView' AND "pid"='ccnubox' GROUP BY subCat
```

 