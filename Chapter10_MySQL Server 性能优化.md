# MySQL Server 性能优化

1. MySQL 安装优化

    1. 选择合适的发行版本

    1. 源码安装
        
        源码包的编译参数中默认会以Debug 模式生成二进制代码，而Debug 模式给MySQL 带来的性能损失是比较大的，所以当我们编译准备安装的产品代码的时候，一定不要忘记使用“—without-debug”参数禁用Debug 模式。

        而“—with-mysqld-ldflags”和“—with-client-ldflags”两个编译参数如果设置为“-allstatic”的话，可以告诉编译器以静态方式编译来使编译结果代码得到最高的性能。使用静态编译和动态方式编译的代码相比，性能差距可能会达到5%到10%之多。

2. MySQL 日志设置优化

    在默认情况下，系统仅仅打开错误日志，关闭了其他所有日志，以达到尽可能减少IO 损耗提高系统性能的目的。但是在一般稍微重要一点的实际应用场景中，都至少需要打开二进制日志，因为这是MySQL很多存储引擎进行增量备份的基础，也是MySQL 实现复制的基本条件

    * “max_binlog_cache_size” binlog使用的cache内存大小。 如果不够大的话, 在事物过程中会报错。

    * “max_binlog_size”：Binlog 日志最大值，一般来说设置为512M 或者1G，但不能超过1G。该大小并不能非常严格控制Binlog 大小（当接近尾部，但遇到大的事务过程，binlog会将该事务过程放在一起而不会进行切分。）

    * “sync_binlog”：这个参数是对于MySQL 系统来说是至关重要的，他不仅影响到Binlog 对MySQL所带来的性能损耗，而且还影响到MySQL 中数据的完整性。

        * sync_binlog=0，当事务提交之后，MySQL 不做fsync 之类的磁盘同步指令刷新binlog_cache 中的信息到磁盘，而让Filesystem 自行决定什么时候来做同步，或者cache 满了之后才同步到磁盘

        * sync_binlog=n，当每进行n 次事务提交之后，MySQL 将进行一次fsync 之类的磁盘同步指令来将binlog_cache 中的数据强制写入磁盘。

    * MySQL 中Binlog 的产生量是没办法改变的，只要我们的Query 改变了数据库中的数据，那么就必须将该Query 所对应的Event 记录到Binlog 中。在MySQL 复制环境中，实际上是是有8 个参数可以让我们控制需要复制或者需要忽略而不进行复制的DB 或者Table

        * Binlog_Do_DB：设定哪些数据库（Schema）需要记录Binlog；
        * Binlog_Ignore_DB：设定哪些数据库（Schema）不要记录Binlog；

        * Replicate_Do_DB：设定需要复制的数据库（Schema），多个DB 用逗号（“,”）分隔；
        * Replicate_Ignore_DB：设定可以忽略的数据库（Schema）；
        * Replicate_Do_Table：设定需要复制的Table；
        * Replicate_Ignore_Table：设定可以忽略的Table；
        * Replicate_Wild_Do_Table：功能同Replicate_Do_Table，但可以带通配符来进行设置；
        * Replicate_Wild_Ignore_Table：功能同Replicate_Ignore_Table，可带通配符设置；

        在Master 端设置也存在一定的弊端，因为MySQL 的判断是否需要复制某个Event 不是根据产生该Event 的Query 所更改的数据所在的DB，而是根据执行Query 时刻所在的默认Schema，也就是我们登录时候指定的DB 或者运行“USE DATABASE”中所指定的DB。只有当前默认DB 和配置中所设定的DB 完全吻合的时候IO线程才会将该Event 读取给Slave 的IO 线程

        如果是在Slave 端设置后面的六个参数，在性能优化方面可能比在Master 端要稍微逊色一点，因为不管是需要还是不需要复制的Event 都被会被IO 线程读取到Slave 端，这样不仅仅增加了网络IO 量，也给Slave 端的IO 线程增加了Relay Log 的写入量。但是仍然可以减少Slave的SQL 线程在Slave 端的日志应用量。虽然性能方面稍有逊色，但是在Slave 端设置复制过滤机制，可以保证不会出现因为默认Schema 的问题而造成Slave 和Master 数据不一致或者复制出错的问题。

    * Slow Query Log 相关参数及使用建议

        打开Slow Query Log 功能对系统性能的整体影响没有Binlog 那么大，毕竟Slow Query Log 的数据量比较小，带来的IO 损耗也就较小，但是，系统需要计算每一条Query 的执行时间，所以消耗总是会有一些的，主要是CPU 方面的消耗。如果大家的系统在CPU 资源足够丰富的时候，可以不必在乎这一点点损耗，毕竟他可能会给我们带来更大性能优化的收获。但如果我们的CPU 资源也比较紧张的时候，也完全可以在大部分时候关闭该功能，而只需要间断性的打开Slow Query Log 功能来定位可能存在的慢查询。

3. Query Cache 优化

    1. Query Cache 实现原理

        将客户端请求的Query语句通过一定的hash 算法进行一个计算而得到一个hash 值，存放在一个hash 桶中。同时将该Query 的结果集（Result Set）也存放在一个内存Cache 中的。存放Queryhash 值的链表中的每一个hash 值所在的节点中同时还存放了该Query 所对应的Result Set 的Cache 所在的内存地址，以及该Query 所涉及到的所有Table 的标识等其他一些相关信息。系统接受到任何一个SELECT 类型的Query 的时候，首先计算出其hash 值，然后通过该hash 值到Query Cache 中去匹配，如果找到了完全相同的Query，则直接将之前所Cache 的Result Set 返回给客户端而完全不需要进行后面的任何步骤即可完成这次请求。而后端的任何一个表的任何一条数据发生变化之后，也会通知QueryCache，需要将所有与该Table 有关的Query 的Cache 全部失效，并释放出之前占用的内存地址，以便后面其他的Query 能够使用。

        该原理对应的缺点：

        * Query 语句的hash 运算以及hash 查找资源消耗。

        * Query Cache 的失效问题。

            对于更改较多的表来说，该Cache原理会导致经常失效。

        * Query Cache 中缓存的是Result Set ，而不是数据页，也就是说，存在同一条记录被Cache 多次的可能性存在。从而造成内存资源的过渡消耗。

            可通过限定 Query Cache大小，但随之会导致命中率下降。
    
    2. 适度使用Query Cache

        根据Query Cache 失效机制来判断哪些表适合使用Query 哪些表不适合. 应该避免在查询变化频繁的Table 的Query 上使用，而应该在那些查询变化频率较小的Table 的Query 上面使用。

        MySQL 中针对Query Cache 有两个专用的SQL Hint（提示）：SQL_NO_CACHE 和SQL_CACHE，分别代表强制不使用Query Cache 和强制使用Query Cache

        有些SQL 的Result Set 很大，如果使用Query Cache 很容易造成Cache 内存的不足，或者将之前一些老的Cache 冲刷出去。对于这一类Query 我们有两种方法可以解决，一是使用SQL_NO_CACHE 参数来强制他不使用Query Cache 而每次都直接从实际数据中去查找， 另一种方法是通过设定“query_cache_limit”参数值来控制Query Cache 中所Cache 的最大Result Set ，系统默认为1M（1048576）。当某个Query 的Result Set 大于“query_cache_limit”所设定的值的时候，QueryCache 是不会Cache 这个Query 的。

    3. Query Cache 的相关系统参数变量和状态变量
    
3. MySQL Server 其他常用优化除  

    * 网络连接与连接线程

        * max_conecctions：整个MySQL 允许的最大连接数

        * max_user_connections：每个用户允许的最大连接数；

        * net_buffer_length：网络包传输中，传输消息之前的net buffer 初始化大小；

        * max_allowed_packet：在网络传输中，一次传消息输量的最大值；

        * back_log：在MySQL 的连接请求等待队列中允许存放的最大连接请求数。

    * 与每一个客户端连接相对应的连接线程。

        * thread_cache_size：Thread Cache 池中应该存放的连接线程数。

            如果我们的应用程序使用的短连接，Thread Cache 池的功效是最明显的。因为在短连接的数据库应用中，数据库连接的创建和销毁是非常频繁的. 在短连接的应用系统中，thread_cache_size 的值应该设置的相对大一些，不应该小于应用系统对数据库的实际并发请求数。即使是在使用长连接的应用环境中，Thread Cache 机制的利用仍然是对性能大有帮助的。只不过在长连接的环境中我们不需要将thread_cache_size 参数设置太大，一般来说可能50 到100 之间应该就可以了。
    
        * thread_stack：每个连接线程被创建的时候，MySQL 给他分配的内存大小

            不过一般来说如果不是对MySQL 的连接线程处理机制十分熟悉的话，不应该轻易调整该参数的大小，使用系统的默认值（192KB）基本上可以所有的普通应用环境

        该怎样检验上面所做的设置是否合理，是否有需要调整的地方。我们可以通过在系统中执行如下的几个命令来获得相关的状态信息来帮助大家检验设置的合理性：

            mysql> show variables like 'thread%';

            mysql> show status like 'connections';

            mysql> show status like '%thread%';

        Threads_Cache_Hit = (Connections - Threads_created) / Connections * 100%

        一般来说，当系统稳定运行一段时间之后，我们的Thread Cache 命中率应该保持在90%左右甚至更高的比率才算正常。

    * Table Cache 相关的优化

        我们先来看一下MySQL 打开表的相关机制。由于多线程的实现机制，为了尽可能的提高性能，在MySQL 中每个线程都是独立的打开自己需要的表的文件描述符，而不是通过共享已经打开的表的文件描述符的机制来实现

        为了解决打开表文件描述符太过频繁的问题，MySQL 在系统中实现了一个Table Cache 的机制，和前面介绍的Thread Cache 机制有点类似，主要就是Cache 打开的所有表文件的描述符，当有新的请求的时候不需要再重新打开，使用结束的时候也不用立即关闭

    * Sort Buffer，Join Buffer 和Read Buffer

        如果应用系统中很少有Join 语句出现，则可以不用太在乎join_buffer_size 参数的大小设置，但是如果Join 语句不是很少的话，个人建议可以适当增大join_buffer_size 的设置到1MB 左右，如果内存充足甚至可以设置为2MB。对于sort_buffer_size 参数来说，一般设置为2MB 到4MB 之间可以满足大多数应用的需求。当然，如果应用系统中的排序都比较大，内存充足且并发量不是特别的大的时候，也可以继续增大sort_buffer_size 的设置。在这两个Buffer 设置的时候，最需要注意的就是不要忘记是每个Thread 都会创建自己独立的Buffer，而不是整个系统共享的Buffer，不要因为设置过大而造成系统内存不足。










