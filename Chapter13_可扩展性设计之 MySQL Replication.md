# 可扩展性设计之 MySQL Replication

MySQL Replication 是MySQL 非常有特色的一个功能. 他能够将一个MySQL Server的Instance 中的数据完整的复制到另外一个MySQL Server 的Instance 中. 虽然复制过程并不是实时而是异步进行的，但是由于其高效的性能设计，延时非常之少。MySQL 的Replication 功能在实际应用场景中被非常广泛的用于保证系统数据的安全性和系统可扩展设计中

1. Replication 对可扩展性设计的意义
    
    通过MySQL 的Replication 功能，我们可以非常方便的将一个数据库中的数据复制到很多台MySQL 主机上面，组成一个MySQL 集群，然后通过这个MySQL 集群来对外提供服务。这样，每台MySQL 主机所需要承担的负载就会大大降低，整个MySQL 集群的处理能力也很容易得到提升

2. Replication 机制的实现原理

    Mysql 的Replication 是一个异步的复制过程，从一个Mysql instace（我们称之为Master）复制到另一个Mysql instance（我们称之Slave）。在Master 与Slave 之间的实现整个复制过程主要由三个线程来完成，其中两个线程（Sql 线程和IO 线程）在Slave 端，另外一个线程（IO 线程）在Master 端。

    必须打开Master 端的Binary Log功能. 

    复制的基本过程如下：

    1. Slave 上面的IO 线程连接上Master，并请求从指定日志文件的指定位置（或者从最开始的日志）之后的日志内容

    2. Master 接收到来自Slave 的IO 线程的请求后，通过负责复制的IO 线程根据请求信息读取指定日志指定位置之后的日志信息，返回给Slave 端的IO 线程。返回信息中除了日志所包含的信息之外，还包括本次返回的信息在Master 端的Binary Log文件的名称以及在Binary Log 中的位置；

    3. Slave 的IO 线程接收到信息后，将接收到的日志内容依次写入到Slave 端的Relay Log 文件(mysql-relay-bin.xxxxxx)的最末端，并将读取到的Master 端的binlog的文件名和位置记录到master-info 文件中，以便在下一次读取的时候能够清楚的高速Master“我需要从某个bin-log 的哪个位置开始往后的日志内容，请发给我”

    4. Slave 的SQL 线程检测到Relay Log 中新增加了内容后，会马上解析该Log 文件中的内容成为在Master 端真实执行时候的那些可执行的Query 语句，并在自身执行这些Query。这样，实际上就是在Master 端和Slave 端执行了同样的Query，所以两端的数据是完全一样的。

    复制实现级别:

    MySQL 的复制可以是基于一条语句（Statement Level），也可以是基于一条记录（Rowlevel），可以在MySQL 的配置参数中设定这个复制级别，不同复制级别的设置会影响到Master 端的Binary Log 记录成不同的形式。

    1. Row Level：Binary Log 中会记录成每一行数据被修改的形式，然后在Slave 端再对相同的数据进行修改。

    2. Statement Level:每一条会修改数据的Query 都会记录到Master 的BinaryLog 中。Slave 在复制的时候SQL 线程会解析成和原来Master 端执行过的相同的Query来再次执行。


1. Replication 常用架构

    1. 常规复制架构(Master - Slaves)

        在实际应用场景中，MySQL 复制90% 以上都是一个Master 复制到一个或者多个Slave 的架构模式，主要用于读压力比较大的应用的数据库端廉价扩展解决方案.

    2. Dual Master 复制架构(Master - Master)

        通过Dual Master 复制架构，我们不仅能够避免因为正常的常规维护操作需要的停机所带来的重新搭建Replication 环境的操作，因为我们任何一端都记录了自己当前复制到对方的什么位置了，当系统起来之后，就会自动开始从之前的位置重新开始复制，而不需要人为去进行任何干预，大大节省了维护成本。

        不仅仅如此，Dual Master 复制架构和一些第三方的HA 管理软件结合，还可以在我们当前正在使用的Master 出现异常无法提供服务之后，非常迅速的自动切换另外一端来提供相应的服务，减少异常情况下带来的停机时间，并且完全不需要人工干预

        在正常情况下，我们都只会将其中一端开启写服务，另外一端仅仅只是提供读服务，或者完全不提供任何服务，仅仅只是作为一个备用的机器存在

    3. 级联复制架构(Master - Slaves - Slaves ...)

        在有些应用场景中，可能读写压力差别比较大，读压力特别的大，一个Master 可能需要上10 台甚至更多的Slave 才能够支撑注读的压力。这时候，Master 就会比较吃力了，因为仅仅连上来的Slave IO 线程就比较多了，这样写的压力稍微大一点的时候，Master 端因为复制就会消耗较多的资源，很容易造成复制的延时。

        这时候我们就可以利用MySQL 可以在Slave 端记录复制所产生变更的Binary Log 信息的功能，也就是打开—log-slave-update 选项。然后，通过二级（或者是更多级别）复制来减少Master 端因为复制所带来的压力

    4. Dual Master 与级联复制结合架构(Master - Master - Slaves)

        级联复制在一定程度上面确实解决了Master 因为所附属的Slave 过多而成为瓶颈的问题，但是他并不能解决人工维护和出现异常需要切换后可能存在重新搭建Replication的问题。这样就很自然的引申出了Dual Master 与级联复制结合的Replication 架构，我称之为Master - Master - Slaves 架构

1. Replication 搭建实现
    
    1. Master 端准备工作

        要保证Master 端MySQL 记录Binary Log 的选项打开. 启动MySQL Server 的时候使用—log-bin 选项或者在MySQL的配置文件my.cnf 中配置log-bin[=path for binary log]参数选项

        我们还需要准备一个用于复制的MySQL 用户。可以通过给一个现有帐户授予复制相关的权限，也可以创建一个全新的专用于复制的帐户。当然，我还是建议用一个专用于复制的帐户来进行复制。在之前“MySQL 安全管理”部分也已经介绍过了，通过特定的帐户处理特定一类的工作，不论是在安全策略方面更有利，对于维护来说也有更大的便利性。实现MySQL Replication 仅仅只需要“REPLICATION SLAVE”权限即可。可以通过如下方式来创建这个用户:

            > CREATE USER 'repl'@'192.168.0.2'
            -> IDENTIFIED BY 'password';

            > GRANT REPLICATION SLAVE ON *.*
            -> TO 'repl'@'192.168.0.2';

    2. 获取Master 端的备份“快照”

        这里所说的Master 端的备份“快照”，并不是特指通过类似LVM 之类的软件所做的snapshot，而是所有数据均是基于某一特定时刻的，数据完整性和一致性都可以得到保证的备份集。同时还需要取得该备份集时刻所对应的Master 端Binary Log 的准确LogPosition，因为在后面配置Slave 的时候会用到。

        * 通过数据库全库冷备份

            对于可以停机的数据库, copy 所有数据文件和日志文件到需要搭建Slave 的主机中合适的位置这样我们还仅仅只是得到了一个满足要求的备份集，我们还需要这个备份集所对应的日志位置才能可以.

                > SHOW Master STATUS

        * 通过LVM 或者ZFS 等具有snapshot 功能的软件进行“热备份”

            为了保证我们的备份集数据能够完整且一致，我们需要在进行Snapshot 过程中通过相关命令（FLUSH TABLES WITH READ LOCK）来锁住所有表的写操作. 因为从我们锁定了所有表的写入操作开始到解锁之前，数据库不能进行任何写入操作，这个时间段之内任何时候通过执行SHOW MASTER STATUS 明令都可以得到准确的Log Position

        * 通过mysqldump 客户端程序

            为了让我们的备份集具有一致性和完整性，我们必须让dump 数据的这个过程处于同一个事务中，或者锁住所有需要复制的表的写操作. 我们可以在执行mysqldump 程序的时候通过添加—single-transaction 选项来做到. 如果我们的存储引擎并不支持事务，或者是需要dump 表仅仅只有部分支持事务的时候，我们就只能先通过FLUSH TABLES WITH READ LOCK 命令来暂停所有写入服务，然后再dump 数据. 

            mysqldump 程序增加了另外一个参数选项来帮助我们获取到对应的Log Position，这个参数选项就是—master-data 。当我们添加这个参数选项之后，mysqldump 会在dump 文件中产生一条CHANGE MASTER TO 命令，命令中记录了dump 时刻所对应的详细的Log Position 信息
            
                $ mysqldump --master-data -usky -p example group_message > group_message.sql

        * 通过现有某一个Slave 端进行“热备份”

            通过现有Slave 来获取备份集的方式，不仅仅得到数据库备份的方式很简单，连所需要Log Position，甚至是新Slave 后期的配置等相关动作都可以省略掉，只需要新的Slave 完全基于这个备份集来启动，就可以正常从Master 进行复制了
            
    3. Slave 端恢复备份“快照”

        * 恢复全库冷备份集

            根据Slave 上my.cnf 配置文件的设置，将文件存放在相应的目录，覆盖现有所有的数据和日志等相关文件，然后再启动Slave 端的MySQL，就完成了整个恢复过程。

        * 恢复对Master 进行Snapshot 得到的备份集

            对于通过对Master 进行Snapshot 所得到的备份集，实际上和全库冷备的恢复方法基本一样

        * 恢复mysqldump 得到的备份集

            通过mysqldump 客户端程序所得到的备份集，和前面两种备份集的恢复方式有较大的差别。因为前面两种备份集的都属于物理备份，而通过mysqldump 客户端程序所做的备份属于逻辑备份

            使用mysql 客户端程序在Slave 端恢复之前，建议复制出通过—master-data所得到的CHANGE MASTER TO 命令部分，然后在备份文件中注销掉该部分，再进行恢复。因为该命令并不是一个完整的CHANGE MASTER TO 命令，如果在配置文件（my.cnf）中没有配置MASTER_HOST,MASTER_USER,MASTER_PASSWORD 这三个参数的时候，该语句是无法有效完成的。

        * 恢复通过现有Slave 所得到的热备份

            通过现有Slave 所得到的备份集和上面第一种或者第二种备份集也差不多。

    4. 配置并启动Slave

        通过CHANGE MASTER TO 命令来配置然后再启动Slave. 

        CHANGE MASTER TO 命令总共需要设置5 项内容，分别为：

        1. MASTER_HOST：Master 的主机名（或者IP 地址）；
        2. MASTER_USER：Slave 连接Master 的用户名，实际上就是之前所创建的repl 用户；
        2. MASTER_PASSWORD：Slave 连接Master 的用户的密码；
        2. MASTER_LOG_FILE：开始复制的日志文件名称；
        2. MASTER_LOG_POS：开始复制的日志文件的位置，也就是在之前介绍备份集过程中一致提到的Log Position。

        命令：

            > CHANGE MASTER TO
            -> MASTER_HOST='192.168.0.1',
            -> MASTER_USER='repl',
            -> MASTER_PASSWORD='password',
            -> MASTER_LOG_FILE='mysql-bin.000035',
            -> MASTER_LOG_POS=399;

        执行完CHANGE MASTER TO 命令之后，就可以通过如下命令启动SLAVE 了：

            > START SLAVE;


