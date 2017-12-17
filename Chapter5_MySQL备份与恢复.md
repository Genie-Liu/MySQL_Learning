# MySQL 备份与恢复


1. 备份使用场景

    * 数据丢失应用场景

        1. 人为操作失误造成某些数据被误操作；
        
        2. 软件BUG 造成数据部分或者全部丢失；
        
        3. 硬件故障造成数据库数据部分或全部丢失；
        
        4. 安全漏洞被入侵数据被恶意破坏；

    * 非数据丢失应用场景

        1. 特殊应用场景下基于时间点的数据恢复；

        6. 开发测试环境数据库搭建；

        7. 相同数据库的新环境搭建；

        8. 数据库或者数据迁移；

2. 逻辑备份与恢复测试

    * 生成INSERT语句备份

            Dumping definition and data mysql database or table
            Usage: mysqldump [OPTIONS] database [tables]
            OR mysqldump [OPTIONS] --databases [OPTIONS] DB1 [DB2 DB3...]
            OR mysqldump [OPTIONS] --all-databases [OPTIONS]

        由于数据库处在一个动态变化过程，该方法有可能导致备份数据不是最新的，甚至无法保证完整性约束。

        保证数据库中的数据一致：

        1. 同时取出所有数据

            对于事务支持的存储引擎，通过控制将整个备份过程控制在同一个事务中，来达到备份数据的一致性和完整性通过“--single-transaction”选项，可以不影响数据库的任何正常服务

        2. 数据库中数据处于静止状态

            需要备份的表锁定，只允许读取而不允许写入。 。mysqldump 程序自己也提供了相关选项如“--lock-tables”和“--lock-all-tables”。如果你需要dump 的表分别在多个不同的数据库中，一定要使用“--lock-all-tables”才能确保数据的一致完整性。

    * 生成特定格式的纯文本备份数据文件备份

        同一个备份文件中不能存在多个表的备份数据

        * 通过执行SELECT ... TO OUTFILE FROM ...命令来实现

                > SELECT * INTO OUTFILE '/tmp/dump.text'
                > FIELDS TERMINATED BY ',' OPTIONALLY ENCLOSED BY '"'
                > LINES TERMINATED BY '\n'
                > FROM test_outfile limit 100;

            若报错，则需要把secure_file_pri在my.ini设置成空值或者指定目录

        * 也可以使用mysqldump导出

                > mysqldump -uroot -T/tmp/mysqldump database table_name 
                --fields-enclosed -by=\" --fields-terminated-by=,
        
            这样的输出结构对我们做为备份来使用是非常合适的，当然如果一次有多个表需要被dump，就会针对每个表都会生成两个相对应的文件

3. 逻辑备份恢复方法

    * INSERT 语句文件的恢复：

            mysql < backup.sql
            source /path/backup.sql

    * 纯数据文本备份的恢复：

            LOAD DATA INFILE
            mysqlimport

    逻辑备份能做什么？

    1. 通过逻辑备份，我们可以通过执行相关SQL 或者命令将数据库中的相关数据完全恢复到备份时候所处的状态，而不影响不相关的数据；

    2. 通过全库的逻辑备份，我们可以在新的MySQL 环境下完全重建出一个于备份时候完全一样的数据库，并且不受MySQL 所处的平台类型限制；

    3. 通过特定条件的逻辑备份，我们可以将某些特定数据轻松迁移（或者同步）到其他的MySQL 或者另外的数据库环境；

    4. 通过逻辑备份，我们可以仅仅恢复备份集中的部分数据而不需要全部恢复。


    逻辑备份恢复测试

    * INSERT 语句的逻辑备份

        1. 准备好备份文件，copy 到某特定目录，如“/tmp”下；

        2. 通过执行如下命令执行备份集中的相关命令：

                mysql -uusername -p < backup.sql
                > source /tmp/backup.sql

        3. 再到数据库中检查相应的数据库对象，看是否已经齐全；

        4. 抽查几个表中的数据进行人工校验

    * 特殊分隔符分隔的纯数据文本文件

        1. 将最接近崩溃时刻的备份文件准备好

        2. 通过特定工具或者命令将数据导入如到数据库中：

                mysqlimport --user=name --password=pwd test --fields-enclosed-by=\" 
                --fields-terminated-by=, /tmp/test_outfile.txt

                LOAD DATA INFILE '/tmp/test_outfile.txt' INTO TABLE test_outfile FIELDS
                TERMINATED BY '"' ENCLOSED BY ',';


4. 数据库物理备份

    * MyISAM 存储引擎

        每一个MyISAM 存储引擎表都会有三个文件存在，分别为记录表结构元数据的“.frm”文件，存储表数据的“.MYD”文件，以及存储索引数据的“.MYI”文件。

        由于MyISAM 属于非事务性存储引擎，所以他没有自己的日志文件. 只需要备份上面的三种文件即可

    * Innodb 存储引擎

        1. innodb_data_home_dir

        2. innodb_data_file_path

        3. “datadir”中相应数据库目录下的所有Innodb 存储引擎表的“.frm”文件；

        4. “datadir”中相应数据库目录下的所有“.idb”文件

        5. innodb_log_group_home_dir

        
    * NDB Cluster 存储引擎

        1、元数据（Metadata）：包含所有的数据库以及表的定义信息；

        2、表数据（Table Records）：保存实际数据的文件；

        3、事务日志数据（Transaction Log）：维持事务一致性和完整性，以及恢复过程中所需要的事务信息。

    热备份方法（冷备份直接copy文件即可）：

    * MyISAM 存储引擎

        将MyISAM 的物理文件copy 出来即可

        MySQL 自己提供了一个使用程序mysqlhotcopy，这个程序就是专门用来备份MyISAM 存储引擎的。

            mysqlhotcopy db_name[./table_regex/] [new_db_name | directory]
        可以通过登录数据库中手工加锁，然后再通过操作系统的命令来复制相关文件执行热物理备份，且在完成文件copy之前，不能退出加锁的session（因为退出会自动解锁），如下：

            FLUSH TABLES WITH READ LOCK;

        不退出mysql，在新的终端下做如下备份：

            cp -R test /tmp/backup/test

        然后再在之前的执行锁定命令的session 中解锁

            unlock tables；

    * Innodb 存储引擎

        Innodb 存储引擎由于是事务性存储引擎，有redo 日志和相关的undo 信息，而且对数据的一致性和完整性的要求也比MyISAM 要严格很多，所以Innodb 的在线（热）物理备份要比MyISAM 复杂很多，一般很难简单的通过几个手工命令来完成，大都是通过专门的Innodb在线物理备份软件来完成。

        Innodb 存储引擎的开发者（Innobase 公司）开发了一款名为ibbackup 的商业备份软件

5. 物理备份恢复

    * MyISAM 存储引擎

        如果是通过停机冷备份或者是在运行状态通过锁定写入操作后的备份集来恢复，仅仅只需要将该备份集直接通过操作系统的拷贝命令将相应的数据文件复制到对应位置来覆盖现有文件即可

    * Innodb 存储引擎

        对于冷备份，Innodb 存储引擎进行恢复所需要的操作和其他存储引擎没有什么差别，同样是备份集文件（包括数据文件和日志文件）复制到相应的目录即可

        通过其他备份软件所进行的备份，就需要根据备份软件本身的要求来进行了(。比如通过ibbackup 来进行的备份)


6. 备份策略的设计思路

    为了应对在线应用的数据丢失的问题，我们的备份就需要快速恢复，而且最好是仅仅需要增量恢复就能找回所需数据。对于这类需求，最好是有在线的，且部分延迟恢复的备用数据库。也可以通过物理备份来解决，毕竟物理备份的恢复速度要比逻辑备份的快很多。

    对于非数据丢失的应用场景，大多数时候恢复时间的要求并不是太高，只要可以恢复出一个完整可用的数据库就可以了。所以不论是物理备份还是逻辑备份，影响都不大。

    可以根据不同的需求不同的级别通过如下的几个思路来设计出合理的备份策略：

    * 对于较为核心的在线应用系统，需要有在线备用主机通过MySQL的复制进行相应的备份，复制线程可以一直开启，恢复线程可以每天恢复一次，尽量让备机的数据延后主机在一定的时间段之内

    * 对于重要级别稍微低一些的应用，恢复时间要求不是太高的话，为了节约硬件成本，不必要使用在线的备份主机来单独运行备用MySQL，而是通过每一定的时间周期内进行一次物理全备份，同时每小时（或者其他合适的时间段）内将产生的二进制日志进行备份。这样虽然没有第一种备份方法恢复快，但是数据的丢失会比较少。恢复所需要的时间由全备周期长短所决定。

    * 而对于恢复基本没有太多时间要求，但是不希望太多数据丢失的应用场景，则可以通过每一定时间周期内进行一次逻辑全备份，同时也备份相应的二进制日志。使用逻辑备份而不使用物理备份的原因是因为逻辑备份实现简单，可以完全在线联机完成，备份过程不会影响应用提供服务。

    * 对于一些搭建临时数据库的备份应用场景，则仅仅只需要通过一个逻辑全备份即可满足需求，都不需要用二进制日志来进行恢复，因为这样的需求对数据并没有太苛刻的要求