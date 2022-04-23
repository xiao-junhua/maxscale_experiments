# 基于Maxscale的读写分离实验

## 配置MySQL Master-Slave Replication
采用dockers-compose配置，相关配置见[docker-compose.yml](docker-compose.yml).

使用`docker-compose up -v`启动master container和slave container，两者暂未建立主从复制关系,
需要分别进入master container和slave container配置主从复制关系:
```sql
/* 主节点 */
CREATE USER 'repl'@'%' IDENTIFIED BY '222';
GRANT REPLICATION SLAVE ON *.* FOR 'repl'@'%';
SHOW MASTER STATUS; # 得到MASTER_LOG_FILE和MASTER_LOG_POS
/* 从节点 */
CHANGE MASTER TO MASTER_HOST='172.20.0.2', MASTER_USER='repl', MASTER_PASSWORD='222',  MASTER_LOG_FILE='mysql-bin.000003', MASTER_LOG_POS=595;

START SLAVE;
```
**结果展示:**
主节点上创建:

![image.png](https://s2.loli.net/2022/04/22/dpPEnlUZ8yqvj2H.png)
从节点上出现同样的数据库:

![image.png](https://s2.loli.net/2022/04/22/AbyXLpjnRGO91t5.png)

## Maxscale 读写分析
1. 首先在master节点上创建一个maxscale要用的用户:  
```sql
mysql> CREATE USER 'maxuser'@'%' IDENTIFIED BY 'maxpwd';
Query OK, 0 rows affected (0.01 sec)

mysql> GRANT ALL ON *.* TO 'maxuser'@'%';
Query OK, 0 rows affected (0.01 sec)
```
2. 其次配置maxscale的配置文件即`/etc/maxscale.cnf`:  
```sql
[server1]
type=server
address=172.20.0.2
port=3306
protocol=MySQLBackend

[server2]
type=server
address=172.20.0.3
port=3306
protocol=MySQLBackend

[MySQL-Monitor]
type=monitor
module=mysqlmon
servers=server1,server2
user=maxuser
password=maxpwd
monitor_interval=1000
detect_replication_lag=true
detect_stale_master=true

[Read-Write-Service]
type=service
router=readwritesplit
servers=server1,server2
user=maxuser
password=maxpwd
master_failure_mode=fail_on_write

[Read-Write-Listener]
type=listener
service=Read-Write-Service
protocol=MySQLClient
port=4006
```
**可以看到,maxscale已经能够看到相关server:**
![image.png](https://s2.loli.net/2022/04/22/drhneMlNzbDUq7k.png)
3. 使用mysql登录maxscale，并且实现读写分离
![image.png](https://s2.loli.net/2022/04/22/bfZJ9gUdCyaH3cW.png)
![image.png](https://s2.loli.net/2022/04/22/bfZJ9gUdCyaH3cW.png)