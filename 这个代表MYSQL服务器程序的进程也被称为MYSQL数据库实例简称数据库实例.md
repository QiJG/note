这个代表MYSQL服务器程序的进程也被称为MYSQL数据库实例简称数据库实例

进程的唯一编号（进程ID）由操作系统决定，进程的名称，由编写程序的人自己定义。

Mysql服务器进程默认的名称为mysqld，Mysql客户端进程默认的名称为mysql

类UNIX 中mysql启动实例

* mysqlId 
* mysqld_safe 启动脚本，会间接的调用mysqld，同时还会启动另外一个监控进程，当mysql服务器进程挂掉时，可以帮助重启它，同时也会将服务器程序出错的信息和其他诊断信息重定向到某个文件中，产生出错日志
* mysql.server 启动脚本 会间接调用mysqld_safe，比如 mysql.server start/stop 启动或停止mysql

windows 的启动方式（提供了手动启动和以服务的形式进行启动）

* mysqld

* 以服务的方式运行服务器程序

  如果无论

  ``` sql
  -- 默认都是8小时
  show global variables like 'wati_timeout';-- 非交互式超时时间 如JDBC程序
  show global variables like 'innteractive_timeout';-- 交互式超时时间 如数据库工具
  
  -- 查看线程的状态|
  show global status like 'Thread%';
  -- Threads_cached：缓存中的线程连接数
  -- Threads_connected：当前打开的连接数
  -- Threads_created：为处理连接创建的线程数
  -- Threads_running：非睡眠状态的连接数，通常指并发连接数
  ```

  

从MySQL 5.7.20开始，不推荐使用查询缓存，并在MySQL 8.0中删除。

INNODB 比 MYISAM 优点

* 支持事务
* 支持外键
* 支持MVCC
* 行级锁



如果 global 和session 级别的 default_storage_engine 



