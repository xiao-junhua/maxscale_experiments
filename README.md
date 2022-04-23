# 入职任务

1. 基础工具学习
- [√] 学习使用dockers
  相关总结见[docker学习总结](./notes/docker.md)
- [√] 学习使用git
  相关总结见[git学习总结](./notes/git.md)
- [x] 学习shell相关命令
  暂时没有系统学习
- [x] tshark
  遇到了问题，采用的filter未能将TCP packet转换未MySQL packet
- [x] pgshark
  **TODO**

2. MaxScale读写分离实验, 具体内容参考[Maxscale readwrite split](./maxscale_readwritesplit/)
 - 搭建了MySQL Master-Slave Replication
 - 参考[Building MariaDB MaxScale from Source Code](https://mariadb.com/kb/en/mariadb-maxscale-22-building-mariadb-maxscale-from-source-code/)编译Maxscale
 - 使用MaxScale实现了读写分离，能够正常登录，写会转发到master节点，而读会转发到slave节点
3. MaxScale query_classifier 模块调研
- query_classifier调研,详细内容见[query classfier分析](./notes/query_classifier.md)
- 扩展新的语法(**TODO**)

4. MySQL Client/Server Protocol
- [√] 仔细阅读了[MySQL Client/Server 文档](https://mariadb.com/kb/en/mariadb-maxscale-22-building-mariadb-maxscale-from-source-code/),详细内容参考[MySQL Client/Server Protocol分析](./notes/mysql_protocol.md)
- [x] 使用tshark抓包分析(**TODO**)
- [x] 分析Maxscale中针对mysql的连接池实现(**TODO**)

5. Postgsql Front/End Protocol
**TODO**
