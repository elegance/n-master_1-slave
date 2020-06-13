# 5.7 多主一从

#### 目录结构：

```powershell
➜  n-master_1-slave tree ./
./
├── conf
│   ├── master-1
│   │   └── master-1.cnf
│   ├── master-2
│   │   └── master-2.cnf
│   └── slave
│       └── slave.cnf
├── docker-compose.yml
```

#### master-1.cnf:

```properties
[mysqld]
server_id = 1
gtid_mode = on
enforce_gtid_consistency = on
log-bin = mysql-bin
log-slave-updates = 1
binlog_format = ROW
skip-slave-start = 1
character-set-server = utf8mb4
collation-server = utf8mb4_unicode_ci
```

#### master-2.cnf:

```properties
[mysqld]
server_id = 2
gtid_mode = on
enforce_gtid_consistency = on
log-bin = mysql-bin
log-slave-updates = 1
binlog_format = ROW
skip-slave-start = 1
character-set-server = utf8mb4
collation-server = utf8mb4_unicode_ci
```

#### slave.cnf:

```properties
[mysqld]
server_id = 666
gtid_mode = on
enforce_gtid_consistency = on
log-bin = mysql-bin
log-slave-updates = 1
binlog_format = ROW
skip-slave-start = 1
binlog-ignore-db = mysql
binlog_ignore_db = information_schema
binlog_ignore_db = performation_schema
binlog_ignore_db = sys
character-set-server = utf8mb4
collation-server = utf8mb4_unicode_ci
master_info_repository  = table
relay_log_info_repository = table
```

#### docker-compose.yml:

```yaml
version: '3.5'
services:
  master-1:
    image: mysql:5.7
    container_name: mysql-master-1
    environment:
      - "MYSQL_ROOT_PASSWORD=root"
    volumes:
      - master-1-data:/var/lib/mysql
      - ./conf/master-1:/etc/mysql/conf.d/
  master-2:
    image: mysql:5.7
    container_name: mysql-master-2
    environment:
      - "MYSQL_ROOT_PASSWORD=root"
    volumes:
      - master-2-data:/var/lib/mysql
      - ./conf/master-2:/etc/mysql/conf.d/
  slave:
    image: mysql:5.7
    container_name: mysql-slave
    environment:
      - "MYSQL_ROOT_PASSWORD=root"
    volumes:
      - slave-data:/var/lib/mysql
      - ./conf/slave:/etc/mysql/conf.d/
volumes:
  master-1-data:
  master-2-data:
  slave-data:
```

#### 启动

```powershell
cd n-master_1-slave
# 将配置文件设置为 0444 权限，否则mysql 认为 World-writable config file，将忽略改配置文件
➜ chmod 0444 conf/master-1/master-1.cnf
➜ chmod 0444 conf/master-2/master-2.cnf
➜ chmod 0444 conf/slave/slave.cnf

# 启动
➜ docker-compose up -d
Creating network "n-master_1-slave_default" with the default driver
...

# 查看下docker 容器
➜ docker ps

# 分别进入2个 master 容器建立下复制账户 repl
# master-1 容器
➜ docker exec -it mysql-master-1 bash
root@a54575fe08af:/# mysql -uroot -p
mysql> GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%' IDENTIFIED BY 'repl';

# master-2 容器
➜ docker exec -it mysql-master-2 bash
root@4b1ef0d49885:/# mysql -uroot -p
mysql> GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%' IDENTIFIED BY 'repl';

# slave 服务 设置主从配置，并启动 slave
➜  n-master_1-slave docker exec -it mysql-slave bash
root@b6d19b9c202c:/# mysql -uroot -p

# 作为 master-1 的从服务
mysql> CHANGE MASTER TO MASTER_HOST='master-1', 
MASTER_USER='repl',
MASTER_PASSWORD='repl', 
MASTER_PORT=3306, 
MASTER_AUTO_POSITION=1, 
MASTER_CONNECT_RETRY=10 
for channel 'master-1';

# 作为 master-2 的从服务
mysql> CHANGE MASTER TO MASTER_HOST='master-2', 
MASTER_USER='repl',
MASTER_PASSWORD='repl', 
MASTER_PORT=3306, 
MASTER_AUTO_POSITION=1, 
MASTER_CONNECT_RETRY=10 
for channel 'master-2';

# 启动所有 slave (也可单独对channel 启用：start slave for channel 'master-1')
mysql> start slave;

# 查看slave状态，关注下：Slave_IO_Running, Slave_SQL_Running, Last_Error,  Seconds_Behind_Master : 主从延时
mysql> show slave status \G;

```

#### 测试同步

```mysql
# master-1
> create database test1;
> use test1;
> create table a(id int);
> insert a(id) values(1);

# master-2
> create database test2;
> use test2;
> create table b(id int);
> insert b(id) values(1);

# slave 查看
> show databases;
> use test1;
> show tables;
> select * from a;

> use test2;
> show tables;
> select * from b;
```

#### 环境清理

```bash
docker-compose down
docker volume prune
```

#### 命令集：

```mysql
> show global variables like '%gtid%';
> show slave hosts;
> show processlist;
> show full processlist \G;
> show slave status \G;
> show slave status for channel 'master-1' \G;
> start slave;
> start slave for channel 'master-1';
> start slave for channel 'master-1';
> set gtid_next='xxxxxxx:n';
> begin; commit;
> set gtid_next='AUTOMATIC';
```
