# MySQL 数据库 Query 的优化

1. MySQL Query Optimizer基本工作原理
    
    MySQL 的Query Tree是通过优化实现DBXP 的经典数据结构和Tree 构造器而生成的一个指导完成一个Query 语句的请求所需要处理的工作步骤，我们可以简单的认为就是一个的数据处理流程规划，只不过是以一个Tree 的数据结构存放而已。

2. Query 语句优化基本思路和原则

    Query 语句的优化思路和原则主要提现在以下几个方面：

    1. 优化更需要优化的Query； (高并发低消耗（相对）的Query 对整个系统的影响远比低并发高消耗的Query 大。)
    2. 定位优化对象的性能瓶颈；
    3. 明确的优化目标；
    4. 从Explain 入手；
    5. 多使用profile
    6. 永远用小结果集驱动大的结果集；
    7. 尽可能在索引中完成排序；
    8. 只取出自己需要的Columns； (涉及到排序，有两种排序算法。)
    9. 仅仅使用最有效的过滤条件；
    10. 尽可能避免复杂的Join 和子查询；

3. 充分利用 Explain 和 Profi l ing

    执行EXPLAIN 命令来告诉我们他将使用一个什么样的执行计划来优化我们的Query

    Query Profiler 是一个使用非常方便的Query 诊断分析工具，通过该工具可以获取一条Query 在整个执行过程中多种资源的消耗情况

        > set profiling=1;
        执行Query...
        > show profiles;
        > show profile cpu, block io for query 6;

4. 合理设计并利用索引
    
    在MySQL 中，主要有四种类型的索引，分别为：B-Tree 索引，Hash 索引，Fulltext 索引和RTree索引

    * B-Tree 索引

    * Hash 索引 (Hash 索引在MySQL 中使用的并不是很多，目前主要是Memory 存储引擎使用)
    
    * Full-text 索引 (Full-text 索引也就是我们常说的全文索引，目前在MySQL 中仅有MyISAM 存储引擎支持)

    * R-Tree 索引 (主要用来解决空间数据检索的问题)

    索引的利处: 我们能够在进行排序分组操作中利用好索引，将会极大的降低CPU 资源的消耗

    索引的弊端: 占用的空间; 调整因为更新所带来键值变化后的索引信息. 

    * 较频繁的作为查询条件的字段应该创建索引；

    * 唯一性太差的字段不适合单独创建索引，即使频繁作为查询条件；

    * 更新非常频繁的字段不适合创建索引；

    * 不会出现在WHERE 子句中的字段不该创建索引；

5. Join 语句的优化

    1. 尽可能减少Join 语句中的Nested Loop 的循环总次数；

    2. 优先优化Nested Loop 的内层循环；

    3. 保证Join 语句中被驱动表上Join 条件字段已经被索引；

    4. 当无法保证被驱动表的Join 条件字段被索引且内存资源充足的前提下，不要太吝惜JoinBuffer 的设置

6. ORDER BY，GROUP BY 和 DI STI NCT 优化

    