# MySQL Client Server协议分析

## MySQL连接方式
那么MySQL连接方式总共有五种，分别是TCP/IP，TLS/SSL，Unix Sockets，Shared Memory，Named pipes,它们的区别如下图所示:

![image.png](https://s2.loli.net/2022/04/24/iXRsBot1Qfbj8N7.png)

1.Unix Sockets:
```bash
mysql -uroot
```
若你在本机使用这种方式连接MySQL数据库的话，它默认会使用Unix Sockets。

2.TCP/IP：
```bash
mysql --protocol=tcp -uroot
mysql -P3306 -h127.0.0.1 -uroot
```
连接的时候我们指定连接协议，或者指定相应的IP及端口，我们的连接方式就变成了TCP/IP方式。

3.TLS/SSL：
```bash
mysql --protocol=tcp -uroot --ssl=on
mysql -P3306 -h127.0.0.1 -uroot --ssl=on
```
TLS/SSL是基于TCP/IP的，所以我们只需再指定打开ssl配置即可。

然后我们可以通过以下语句来查询目前数据库的连接情况：
```sql
SELECT DISTINCT connection_type FROM performance_schema.threads WHERE connection_type IS NOT NULL
```

## MySQL整体通信流程


![image.png](https://s2.loli.net/2022/04/24/G9wRhJlKm8WBVZz.png)  


### Packet中的基础数据类型

1. Integer  
首先Integer在MySQL协议中有两种编码方式，分别为FixedLengthInteger和LengthEncodedInteger,其中前者用于存储无符号定长整数，后者使用LengthEncodedInteger编码的整数可能会使用1, 3, 4, 或者9 个字节，具体使用字节取决于数值的大小，下表是不同的数据长度的整数所使用的字节数：
![image.png](https://s2.loli.net/2022/04/24/Z6xONa4yIhYKHuv.png)

2. String  
String的编码格式相对Integer来说会复杂一点，主要有以下几种：
 - FixedLengthString（定长方式）
 - NullTerminatedString（Null结尾方式）
 - VariableLengthString（动态计算字符串长度方式）
 - LengthEncodedString（指定字符串长度方式）
 - RestOfPacketString（包末端字符串方式）

### 基本数据包
数据包格式也主要分为两种，一种是Server端向Client端发送的数据包格式，另一种则是Client向Server端发送的数据包。
#### Server to Client
Server向Client发送的数据包有两个原则：
- 每个数据包大小不能超过2^24字节(16MB);
- 每个数据包前都需要加上数据包信息


具体来说，包格式如下:    
![image.png](https://s2.loli.net/2022/04/24/UNjY7RfI2MrWncF.png)

#### Client to Server

主要有包括执行命令和相应参数:  
![image.png](https://s2.loli.net/2022/04/24/cklIpTyunh1PXKC.png)

命令参考:  
![image.png](https://s2.loli.net/2022/04/24/OmrDPWEv7g4yRj6.png)

## 具体过程分析
主要可以分成两个阶段: 数据库账户认证阶段和具体命令阶段

### 数据库账户认证阶段
主要步骤如下：
1. Client与Server进行连接  
2. Server向Client发送Handshake packet  
3. Client与Server发送Auth packet  
4. Server向Client发送OK packet或者ERR packet  

### 命令执行阶段
在我们正确连接数据库后，我们就要执行相应的命令了，比如切换数据库，执行CRUD操作等，这个阶段主要分为两步，Client发送命令，Server端接收命令执行相应的操作。
Server端向我们发送数据包，可分为4类：
- OK包(包括PREPARE_OK)：Server端发送正确处理信息的包，包头标识为0x00；
- Error包： Server端发送错误信息的包，包头标识为0xFF；
- EOF包：用于Server向Client发送结束包，包头标识为0xFE；
- Result Set包：用于Server向Client发送的查询结果包；

每个包之中包括了一个Data Field,主要是数据的一种编码结构，分成三个部分  
![image.png](https://s2.loli.net/2022/04/24/ASQk1zKgCbXNBIh.png)

## MySQL Replication Protocol

Mysql主从复制的整体工作流程如下:  
![image.png](https://s2.loli.net/2022/04/24/yak4qimADS6O5Jt.png)

为了获取到master节点的binlog，主从协议的流程如下所示:  
![image.png](https://s2.loli.net/2022/04/24/svIHzNdjPxoEYA6.png)  

1. client 与 server 之间成功建立连接、完成身份认证  
2. client 向 server 发送 COM_REGISTER_SLAVE 包，表明要注册成为一个 slave ，server 响应 OK_Packet 或者 ERR_Packet，只有成功才能进行后续步骤。
3. client 向 server 发送 COM_BINLOG_DUMP 包，表明要开始获取 binlog 的内容。
4. server 响应数据，可能是：
 - binlog network stream（ binlog 网络流）
 - ERR_Packet，表示有错误发生。
 - EOF_Packet，如果 COM_BINLOG_DUMP 中的 flags 设置为了 0x01 ，则在 binlog 没有更多新事件时发送 EOF_Packet，而不是阻塞连接继续等待后续 binlog event 。

### Binlog Event
客户端注册 slave 成功，并且发送 COM_BINLOG_DUMP 正确，那么 MySQL 就会向客户端发送 binlog network stream 即 binlog 网络流，所谓的 binlog 网络流其实就是源源不断的 binlog event 包（对 MySQL 进行的操作，例如 inset、update、delete 等，在 binlog 中是以一个或多个 binlog event 的形式存在的）。

Replication 的两种方式：

- 异步，默认方式，master 不断地向 slave 发送 binlog event ，无需 slave 进行 ack 确认。•
- 半同步，master 向 slave 每发送一个 binlog event 都需要等待 ack 确认回复。

Binlog 有三种模式：
- statement ，binlog 存储的是原始 SQL 语句。
- row ，binlog 存储的是每行的实际前后变化。
- mixed ，混合模式，binlog 存储的一部分是 SQL 语句，一部分是每行变化。


Binlog Event 的包格式如下图：
![image.png](https://s2.loli.net/2022/04/24/DGIMdZT1XP4Nfz6.png)