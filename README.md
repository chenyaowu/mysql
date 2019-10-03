# MySql

## 影响数据库性能几个方面

### 服务器硬件

- CPU资源

  - CPU密集型场景和复杂SQL，频率越高越好（Mysql不支持多CPU对同一SQL并发处理）
  - 并发量高，选择更多CPU。（mysql5.0以前，对多核支持不好。mysql5.6,mysql5.7很好兼容多核）

- 内存

  - 主板支持的最大内存频率
  - 根据数据库大小选择内存

- I/O子系统

  - 磁盘

    1. 传统机器磁盘

    2. 使用RAID增强传统机器硬盘的性能

       ![RAID级别选择](https://github.com/chenyaowu/mysql/blob/master/img/RAID.jpg)

    3. 使用固态存储SSD和PCI-E卡

       - 适用于存在大量随机I/O的场景
       - 适用于解决单线程负载的I/O瓶颈

    4. 使用网络存储NAS和SAN

  - 网络

    1. 延迟

    2. 带宽

    3. 建议：

       - 采用高性能和高带宽的网络接口设备和交换机
       - 对多个网卡进行绑定，增强可用性和宽带
       - 尽可能的进行网络隔离


### 服务器系统

- Linux系统参数优化

  内核相关参数(/etc/sysctl.conf)

  - 网络参数

  ```bash
  # 端口最大监听队列最大长度
  net.core.somaxcon=65535 
  # 每个网路接口接收数据包的速率比内核处理速率快时，允许发送到队列的最大数目
  net.core.netdev_max_backlog=65535
  # 还没获得对方连接请求保存在队列的最大数目
  net.ipv4.tcp_max_syn_backlog=65535
  # 加快TCP连接回收
  net.ipv4.tcp_fin_timeout=10
  net.ipv4.tcp_tw_reuse=1
  net.ipv4.tcp_tw_recycle=1
  # TCP连接接收和发送缓冲区大小的默认值和最大值
  net.core.wmem_defaule=87380
  net.core.wmem_max=16777216
  net.core.rmem_default=87380
  net.core.rmem_max=16777216
  #减少失效连接所占用TCP系统资源数量，加快资源回收效率
  net.ipv4.tcp_keepalive_time=120 // tcp探测消息的时间间隔，单位s，确认TCP连接是否有效
  net.ipv4.tcp_keepalive_intvl=30 // 控制当探测消息未获得相应时，重发该消息的时间间隔，单位s
  net.ipv4.tcp_keepalive_probes=3 //在认定TCP失效之前，最多发送多少keepalive_time消息
  ```

  - 内存参数

  ```bash
  # 用于定义单个共享内存段的最大值
  # 这个参数应该设置的足够大，以便能在一个共享内存段下容纳下整个的Innodb缓冲池的大小
  # 这个值的大小对于64位Linux系统，可取的最大值为物理内存值-1byte，建议值为大于物理内存的一半，一般取值大于Innodb缓冲池的大小即可，可以取物理内存-1byte。
  kernel.shmmax=4294967295
  # 这个参数当内存不足时会对性能产生比较明显的影响（除非内存占满了，否则不使用交换分区）
  vm.swappiness=0
  ```
  
  增加资源限制(/etc/security/limit.conf)
  
  - 这个文件实际上是Linux PAM也就是插入式认证模块的配置文件
  - 打开文件数的限制。
  
  ```bash
  # * 表示对所有用户有效
  # soft 指的是当前系统生效的设置
  # hard 表明系统中所能设定的最大值
  # nofile 表示所限制的资源是打开文件的最大数目
  # 65535 就是限制的数量
  # 追加到limit.conf文件末尾即可
  * soft nofile 65535
  * hard nofile 65535
  ```

  磁盘调度策略(/sys/block/devname/queue/scheduler)

  - noop（电梯式调度策略）
  
  - deadline（截止时间调度策略）
  - anticipatory（预料I/O调度策略）
  
- 文件系统对性能的影响

  - 最好的是XFS

  - EXT3/4系统的挂载参数(/etc/fstab)

    ```bash
    # writeback 源数据写入和日志写入不是同步 （Innodb最好）
    # ordered 只记录源数据 （最安全）
    # journal 原子日志 （最慢）
    data= writeback | ordered | journal
    # 禁止记录文件读取时间
    noatime, nodiratime
    # eg:
    /dev/sda1/ext4 noatime,nodiratime,data=writeback 1 1
    ```

### MySql体系客户端

![MySql体系客户端](https://github.com/chenyaowu/mysql/blob/master/img/Mysql_System1.jpg)




### 数据库存储引擎

  - MyISAM：不支持事务，表级锁。

    - 并发性与锁级别 - 同时读写并发性不太好
    - 表损坏修复 - 可能造成数据丢失 （命令： repair table tablename）
    - 全文索引,blog,text类型支持前150字符前置索引
    - 支持数据压缩 （命令：myisampack -b -f tablename.MYI）
    - 适用场景：1.非事务性应用 2.只读应用 3.空间类应用

  - InnoDB：事务级存储引擎，完美支持行级锁，事务ACID特性

    - 使用表空间进行 数据存储 

      - 系统空间表无法简单的收缩文件大小
      - 独立表空间可以通过optimize table命令收缩系统文件
      - 系统表空间会产生IO瓶颈
      - 独立表空间可以同时向多个文件刷新数据
      - 建议：对Innodb使用独立表空间

    - Innodb是一种事务性存储引擎

    - 完全支持事务的ACID特性

    - Redo Log和Undo Log

    - Innodb支持行级锁

    - 行级锁可以最大程度的支持并发

    - 行级锁是由存储引擎层实现的

    - Innodb状态检查

      ```bash
      show engine innodb status
      ```
    
- 如何选择正确的存储引擎

  - 参考条件：
    1. 事务
    2. 备份
    3. 崩溃恢复
    4. 存储引擎的特有特性
### 数据库参数配置

- Mysql获取配置信息路径

  - 命令行参数

    ```bash
    mysqld_safe --datadir=/data/sql_data
    ```

  - 配置文件

     ```bash
    mysqld --help --verbose | grep -A 1 'Default options'
    ```

    /etc/my.cnf

  - 配置参数作用域

    - 全局参数

      set global 参数名=参数值

    - 会话参数

      set [session] 参数名=参数值

  - 内存配置相关参数

    - 确定可以使用的内存的上限
    
	  - 确定每个连接使用的内存
  
      ```bash
      # 单个线程
      sort_buffer_size // 排序缓冲区
      join_buffer_size // 连接缓冲区
      read_buffer_size // 读缓冲区
      read_rnd_buffer_size // 索引缓存区
      ```
    
    - 确定需要为操作系统保留多少内存
    
    - 为缓存池分配内存
    
      ```bash
      Innodb_buffer_pool_size // 总内存- （每个线程所需内存 * 连接数） - 系统保留内存
      key_buffer_size // MyISAM存储引擎用
      ```
    
    - Innodb I/O配置
    
      ```bash
      Innodb_log_file_size    // 单个日志文件大小
      Innodb_log_files_in_group
      Innodb_log_buffer_size  // 事务缓存区大小
      /*
      * 0:每秒进行一个log写入cache，并flush log到磁盘
      * 1[默认]: 在每次事务提交执行log写入cache，并flush log到磁盘
      * 2[建议]: 每次事务提交，执行log数据写入到cache，每秒执行一次flush log到磁盘
      */
      Innodb_flush_log_at_trx_commit 
      Innodb_flush_method = O_DIRECT // 通知操作系统不要缓存数据（关闭操作系统的缓存）
      Inndb_file_per_table = 1 // 控制Inndb如何控制表空间
      Inndb_doublewrite = 1 // 是否使用双写缓存
      ```
    
    - MyISAM I/O配置
    
      ```bash
      /*
      * OFF : 每次写操作后刷新键缓存中的脏块到磁盘
      * ON  : 只对在键表时指定了delay_key_write选项的表使用延迟刷新
      * ALL : 对所有MyISAM表都使用延迟键写入
      /
      delay_key_write 
      ```
    
    - 安全配置
    
      ```bash
      expire_logs_days // 指定自动清除binlog的天数
      max_allowed_packet // 控制Mysql可以接收的包的大小
      skip_name_resolve // 禁用DNS查找
      sysdate_is_now // 确保sysdate()返回确定性日期
      read_only // 禁止非super权限的用户写权限
      skip_slave_start // 禁用Slave自动恢复
      sql_mode // 设置Mysql所使用的SQL模式
      
      ```
    
    - 其他常用配置
    
      ```bash
      sync_binlog // 控制MySql如何向磁盘刷新binlog
      tmp_table_size 和 max_heap_table_size // 控制内存临时表大小
      max_connections // 控制允许的最大连接数
      
      ```
  
  
  

### 数据库结构设计和SQL语句

- 数据库设计对性能的影响
  - 过分的反范式化为表建立太多的列
  - 过分的范式化造成太多的表关联
  - 在OLTP环境中使用不恰当的分区表
  - 使用外键保证数据的完整性

### Mysql基准测试

- 目的
  - 建立Mysql服务器的性能基准线
  - 模拟比当前系统更高的负载，以找出系统的拓展瓶颈
  - 测试不同的硬件、软件和操作系统配置
  - 证明新的硬件设备是否配置正确

- 如何进行

  - 对整个系统进行基准测试

    - 优点：能过测试整个系统的性能，包括web服务器缓存、数据库等；能反应出系统中各个组件接口间的性能			问题，体现真实性能状况
    - 缺点：测试设计复杂，消耗时间长
    
  - 单独对Mysql进行基准测试 
  
  - 优点：测试设计简单，所需耗费时间短
  - 缺点： 无法全面了解整个系统的性能基线
  
    
  
- Mysql进行基准测试常见指标
  
  - 单位时间内所处理的事务数（TPS）
  
  - 单位时间内所处理的查询数（QPS）
  
  - 响应时间
  
  - 并发量：同时处理的查询请求的数量
  
    
  
    
  
    
  
    




