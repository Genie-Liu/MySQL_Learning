# MySql 安全管理

1. 数据库系统安全相关因素
    
    1. 外围网络 

        尽量使用局域网来实现对外界入侵者拦截

    2. 主机 

        对主机的相关用户权限进行设置，避免入侵者对主机上的数据窃取和破坏

    3. 数据库 

        前两层都大同小异，该层则因数据库而异。 MySQL主要由两块组成： 用户管理模块及访问授权控制模块

    4. 代码

        常见的有SQL注入攻击， 通过强壮的代码来验证用户的输入。如PHP中的mysqli 
        或者对用户的验证漏洞, 对用户的信息保密(如对用户密码的加密保存)

2. MySQL 权限系统介绍

    1. 权限系统

        保存在grant table的系统表中，即： mysql.User, mysql.db, mysql.Host, mysql.table_priv, mysql.column_priv

        权限信息数据量较小， 访问频繁， 故启动时直接载入到内存中。若对权限表进行更改，需要使用FLUSH PRIVILEGES命令。而使用GRANT DROP REVOKE 等命令进行的更改， 则不需要重新FLUSH PRIVILEGES.
    
    2. 权限授予和去除

        使用GRANT REVOKE, 若仅指定用户名 则默认为对'username'@'%'

        查看权限 SHOW GRANTS FOR 'username'@'host'
    
    3. 权限级别

        1. Global Level

            全局权限会覆盖其他相同权限， 使用 *.* 来指定Golbal范围

                eg: mysql> GRANT SELECT, UPDATE, INSERT, DELETE ON *.* TO 'username'@'host';

        2. Database Level

            在Global level之下，同样会覆盖低级的权限， 用 database.* 来指定范围 或 先USE database; 然后使用 * 来指定该数据库范围。

                eg： mysql> GRANT ALTER ON test.* TO 'user'@'host';

                eg： mysql> USE test;

                eg： mysql> GRANT ALTER ON * TO 'user'@'host';

        2. Table Level

            同理， 使用 database.table 指定范围

                mysql> GRANT INDEX ON test.t1 TO 'abc'@'localhost';

        2. Column Level
            
            相应权限后面用括号加入列范围

                mysql> GRANT SELECT(id,value) ON test.t2 TO 'abc'@'%.jianzhaoyang.com';
            
            当某个用户在向某个表插入（INSERT）数据的时候，如果该用户在该表中某列上面没有INSERT 权限，则该列的数据将以默认值填充

        2. Routine Level

            Routine Level 的权限主要只有EXECUTE 和ALTER ROUTINE 两种，主要针对的对象是
procedure 和function 这两种对象，在授予Routine Level 权限的时候，需要指定数据库
和相关对象，

                mysql> GRANT EXECUTE ON test.p1 TO 'abc'@'localhost';


        除了上面几类权限之外，还有一个非常特殊的权限GRANT，拥有GRANT 权限的用户可以
将自身所拥有的任何权限全部授予其他任何用户，所以GRANT 权限是一个非常特殊也非常重
要的权限。GRANT 权限的授予方式也和其他任何权限都不太一样，通常都是通过在执行GRANT
授权语句的时候在最后添加WITH GRANT OPTION 子句达到授予GRANT 权限的目的。
        
        此外，我们还可以通过GRANT ALL 语句授予某个Level 的所有可用权限给某个用户

            mysql> GRANT ALL ON test.t5 TO 'abc';

3. 访问控制实现原理

    1. 用户管理

        mysql.user保存所有用户的信息， Host， User等信息。

        但是这里有一个比较特殊的访问限制，如果要通过localhost 访问的话，必须要有一条
专门针对localhost 的授权信息，即使不对任何主机做限制也不行。

        先了解何时MySQL 存放于内存结构中的权限信息被更新：FLUSH PRIVILEGES 会强
行让MySQL 更新Load 到内存中的权限信息；GRANT、REVOKE 或者CREATE USER 和DROP USER
操作会直接更新内存中俄权限信息；重启MySQL 会让MySQL 完全从grant tables 中读取权
限信息。

        * 对于Global Level 的权限信息的修改，仅仅只有更改之后新建连接才会用到，对于已
经连接上的session 并不会受到影响。

        * 对于Database Level 的权限信息的修改，只有当
客户端请求执行了“USE database_name”命令之后，才会在重新校验中使用到新的权限信
息。

        所以有些时候如果在做了比较紧急的Global 和Database 这两个Level 的权限变更之后，
可能需要通过“KILL”命令将已经连接在MySQL 中的session 杀掉强迫他们重新连接以使
用更新后的权限

        对于Table Level 和Column Level 的权限，，这两个Level 的权限，更新之
后立刻就生效了，而不会需要执行“KILL”命令。


4. 访问授权策略

    * 需要了解来访主机

    * 了解用户需求

    * 要为工作分类

    * 最后，确保只有绝对必要者拥有GRANT OPTION 权限。


5. 安全设置注意事项

    * 第一层防线的网络方面的安全

        * 仅提供本地访问: --skip-networking

        * 使用私有局域网络

        * 使用SSL 加密通道

        * 访问授权限定来访主机信息。

    * 第二层防线主机

        * OS 安全方面。关闭MySQL Server 主机上面任何不需要的服务; 严格控制OS 帐号的管理; 

        * 用非root 用户运行MySQL。

        * 文件和进程安全。合理设置文件的权限属性. 


    * 第三道防线MySQL 自身方面

        * 用户设置。 

            访问数据库的用户都有一个比较复杂的内容作为密码. 有超级权限的MySQLroot 帐号，尽量确保只能通过localhost 访问

        * 安全参数. 

            不需要使用的功能模块尽量都不要启用。

            如果不需要使用用户自定义函数，就不要在启动的时候使用“--allow-suspicious-udfs”参数选

            不需要从本地文件中Load数据到数据库中，就使用“--local-infile=0”禁用掉可以从客户端机器上Load 文件到数据库中；

            使用新的密码规则和校验规则（不要使用“--old-passwords”启动数据库），这项功能是为了兼容旧版本的密码校验方式的，如无额数必要，不要使用该功能，旧版本的密码加密方式要比新的方式在安全方面弱很多

    除了以上这三道防线，我们还应该让连接MySQL 数据库的应用程序足够安全，以防止入侵者通过应用程序中的漏洞而入侵到应用服务器，最终通过应用程序中的数据库相关关配置而获取数据库的登录口令
