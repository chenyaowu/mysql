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

#### 数据库设计对性能的影响

- 过分的反范式化为表建立太多的列
- 过分的范式化造成太多的表关联
- 在OLTP环境中使用不恰当的分区表
- 使用外键保证数据的完整性

#### 数据库结构优化的目的

- 减少数据冗余
- 尽量避免数据维护中出现更新，插入和删除异常
- 节约数据存储空间
- 提交查询效率

#### 数据库结构设计的步骤

- 需求分析：全面了解产品设计的存储需求、存储需求、数据处理需求、数据的安全性和完整性
- 逻辑设置：设计数据的逻辑存储结构、设局实体之间的逻辑关系。解决数据冗余和数据维护异常
- 物理设置：根据所使用的数据库特点进行表结构设计
- 维护优化： 根据实际情况对索引、存储结构等进行优化

#### 数据库设计范式

- 第一范式

  - 数据库表中的所有字段都只具有单一属性
  - 单一属性的列是由基本的数据类型所构成的
  - 设计出来的表都是简单的二维表

- 第二范式

  - 要求一个表中只具有一个业务主键，也就是说符合第二范式的表中不能存在非主键列对只对部分主键的依赖关系

- 第三范式

  - 指每一个非主属性既不部分依赖于也不传递依赖于业务主键，也就是在第二范式的基础上消除了非主属性对主键的传递依赖

- 范式化设计优点：

  - 可以尽量的减少数据冗余，数据表更新快体积小
  - 范式化的更新操作比反范式化更快
  - 范式化的表通常比反范式化更小

- 范式化设计缺点：

  - 对于查询需要对多个表进行关联
  - 更难进行索引优化

- 反范式化优点：

  - 可以减少表的关联
  - 可以更好的进行索引优化

- 反范式化缺点：
    - 存在数据冗余及数据维护异常
    - 对数据的修改需要更多的成本

#### 物理设计

- 定义数据库、表及字段的命名规范

  -  可读性原则
  - 表意性原则
  - 长名原则

- 选择合适的存储引擎

  ![存储引擎](https://github.com/chenyaowu/mysql/blob/master/img/engine.jpg)

- 为表中的字段选择合适的数据类型

  ![int](https://github.com/chenyaowu/mysql/blob/master/img/int.jpg)

  ![double](https://github.com/chenyaowu/mysql/blob/master/img/double.jpg)

  - VARCHAR

    - 存储特点
      - varchar用于存储变长字符串，只占用必要的存储空间
      - 列的最大长度小于255则只占用一个额外字节用于记录字符串长度
      - 列的最大长度大于255则要占用两个额外字节用于记录字符串长度
    - 长度的选择问题
      - 使用最小的符合需求的长度
      - varchar(5)和varchar(200)存储'MySql'字符串性能不同
    - 适合场景
      - 字符串列的最大长度比平均长度大很多
      - 字符串列很好被更新
      - 使用了多字节字符集存储字符串

  - CHAR

    - 存储特点
      - CHAR类型是定长的
      - 字符串存储在CHAR类型的列中会删除末尾的空格
      - CHAR类型的最大宽度为255
    - 适合场景
      - 存储长度近似的值
      - 存储短字符串
      - 存储经常更新的字符串列

  - 日期类型

    - DATETIME类型

      - 以YYYY-MM-DD HH:MM:SS[.fraction] 格式存储日期时间
      - datetime = YYYY-MM-DD HH:MM:SS
      - datetime(6) = YYYY-MM-DD HH:MM:SS.fraction
      - DATETIME类型与时区无关，占用8个字节的存储空间
      - 时间范围1000-01-01 00:00:00 到 9999-12-31 23:59:59

    - TIMESTAMP类型

      - 存储了又格林尼治时间1970年1月1日到当前时间的秒数
      - 以YYYY-MM-DD HH:MM:SS.[.fraction] 的格式显示，占用4个字节
      - 时间范围1970-01-01到 2038-01-19
      - TIMESTAMP类型显示依赖于所指定的时区
      - 在行的数据修改时可以自动修改timestamp列的值

    - Date类型

      - 占用的字节数比使用字符串、datetime、int存储要少，使用date类型只需要3个字节

      - 使用Date类型还可以利用日期时间函数进行日期之间的计算
      - 时间范围： 1000-01-01到 9999-12-31之间的日期

    - Time类型

      - 用户存储时间数据，格式为HH:MM:SS

    - 注意事项

      - 不要使用字符串类型来存储日期时间数据
        - 日期时间类型通过比字符串占用的存储空间小
        - 日期时间类型在进行查找过滤时可以利用日期来进行对比
        - 日期时间类型还有着丰富的处理函数，可以方便的对日期类型进行日期计算
      - 使用Int存储日期时间不如使用Timestamp类型

#### 索引优化

- 索引大大减少了存储引擎需要扫描的数据量
- 索引可以帮助我们进行排序以避免使用临时表
- 索引可以把随机I/O变为顺序I/O
- 索引会增加写操作的成本
- 太多索引会增加查询优化器的选择时间 

- BTree索引

  - 特点
    - 以B+树的结构存储数据
    - 能够加快数据的查询速度
    - 更适合进行范围查找
  - 适用
    - 全值匹配的查询
    - 匹配最左前缀的查询
    - 匹配列前缀查询
    - 匹配范围的查询
    - 精确匹配左前列并范围匹配另外一列
    - 只访问索引的查询
  - 限制
    - 如果不是按照索引最左列开始查找，则无法使用索引
    - 使用索引时不能跳过索引中的列
    - Not in 和 <> 操作无法使用索引
    - 如果查询中有某个列的范围查询，则其右边所有列都无法使用索引

- Hash索引

  - 特点
    - 是基于Hash表实现的，只有查询条件精确匹配Hash索引中的所有列时，才能够使用到hash索引。
    - 对于Hash索引中的所有列，存储引擎都会为每一行计算一个Hash码，Hash索引中存储的就是Hash码。
  - 限制
    - 必须进行二次查找
    - 无法用于排序
    - 不支持部分索引查找也不支持范围查找

- 索引优化

  - 索引列上不能使用表达式或函数

  - 前缀索引和索引列的选择性

  - 联合索引

    - 选择索引列的顺序
      - 经常会被使用到的列优先
      - 选择性高的列优先
      - 宽度小的列优先

  - 覆盖索引

    - 优点
      - 可以优化缓存，减少磁盘IO操作
      - 可以减少随机IO，边随机IO操作变为顺序IO操作
      - 可以避免对Innodb主键索引的二次查询
      - 可以避免MyISAM表进行系统调用
    - 缺点
      - 存储引擎不支持覆盖索引
      - 查询中使用了太多的列
      - 使用了双%的like查询

  - 使用索引扫描来优化排序

    - 索引的列排序和Order By子句的顺序完全一致
    - 索引中所有列的方向（升序，降序）和Order By子句完全一致
    - Order By中字段全部在关联表中的第一张表中

  - 模拟Hash索引优化查询

    - 只能处理键值的全值匹配查找
    - 所使用的Hash函数决定着索引键的大小

  - 利用索引优化锁

    - 索引可以减少锁定的行数
    - 索引可以加快处理速度，同时也加快了锁的释放

  - 索引的维护和优化

    - 删除重复和冗余的索引

    - 更新索引统计信息及减少索引碎片

      ```bash
      analyze table table_name
      ```

      

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
  
  
  
- 计划和设计基准测试

  - 对整个系统还是某一组件
  - 使用什么样的数据
  - 准备基准测试及数据收集脚本
    - CPU使用率、IO、网络流量、状态与计数器信息等（Get_Test_info.sh）
  - 运行基准测试
  - 保存及分析基准测试结果（analysis.sh）

- 常用基准测试功能

  - Mysql基准测试工具(mysqlslap)

    - mysql内置

    - 可以模拟服务器负载，并输出相关统计信息

    - 可以指定也可以自动生成查询语句

    - 常用参数

      ```bash
      -- auto-generate-sql 由系统自动生成SQL脚本进行测试
      -- auto-generate-sql-add-autoincrement 在生成的表中增加自增ID
      -- auto-generate-sql-load-type 指定测试中使用的查询类型
      -- auto-generate-sql-write-number 指定初始化数据时生成的数据量
      -- concurrency 指定并发线程的数量
      -- engine 指定要测试表的存储引擎，可以用逗号分隔多个存储引擎
      -- no-drop 指定不清理测试数据
      -- iterations 指定测试运行的次数
      -- number-of-queries 指定每一个线程执行的查询次数
      -- debug-info 指定输出额外的内存及CPU统计信息
      -- number-int-cols 指定测试表中包含的INT类型列的数量
      -- number-char-cols 指定测试表中包含的varchar类型的数量
      -- create-schema 指定了用于执行测试的数据库名字
      -- query 用于指定自定义SQL的脚本
      -- only-print 并不运行测试脚本，而是把生成的脚本打印出来
      ```

  - Mysql基准测试工具(sysbench)

    - 另外安装

    - 常用参数

      - -- test 用于指定所要指定的测试类型，支持一下参数

      - Fileio 文件系统I/O性能测试

      - cpu CPU性能测试

      - memory 内存性能测试

      - Oltp 测试要指定具体的lua脚本

        ```bash
        --mysql-db 用于指定执行基准测试的数据库名
        --mysql-table-engine 用于指定所使用的存储引擎
        --oltp-tables-count 执行测试的表的数量
        --oltp-table-size 指定每个表中的数据行数
        --num-threads 指定测试的并发线程数量
        --max-time 指定最大的测试时间
        --report-interval 指定间隔多长时间输出一个统计信息
        --mysql-user 指定执行测试的Mysql用户
        --mysql-password 指定执行测试的MySQL用户的密码
        prepare 用于准备测试数据
        run 用于实际进行测试
        cleanup 用于清理测试数据
        ```







