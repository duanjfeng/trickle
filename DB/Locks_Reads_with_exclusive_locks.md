---
layout: default
title: Reads with exclusive locks
---
<h2>{{ page.title }}</h2>

<h3>分类</h3>
<p>DB;Locks;数据库;锁;</p>
<h3>关键词</h3>
<p>exclusive locks;排它锁;</p>

<h3>问题简述</h3>
<p>在一个事务内读取一行数据时防止其他事务执行相同的操作。例如，在transaction1中排他的读取row1时，不允许transaction2中也排他的读取row2。</p>
<p>举个例子：使用程序生成流水号，为了实现多实例，在数据库中保存流水号当前值、步长（每个实例缓存几个流水号，减少数据库访问次数）、流水号最大值。当一个实例缓存的流水号用完时，需要读取并更新当前值。一个实例执行这个读取操作时，其他实例是不允许这个读取操作的。否则就会出现两个实例使用相同流水号的问题。</p>

<h3>解决方法</h3>
<p>
1. DB2数据库：select currentValue from SeqTable for update with rs。
</p>
<p>
2. Oracle数据库：select currentValue from SeqTable for update。
</p>
<p>
3. MySQL数据库：select currentValue from SeqTable for update。注意：需要使用innodb引擎。
</p>
<p>
4. 数据库锁机制远比上述内容要复杂，更多内容请参考官方文档。
DB2：访问http://www-01.ibm.com/support/knowledgecenter/ 搜索“锁定管理”相关内容。
MySQL：http://dev.mysql.com/doc/refman/5.7/en/innodb-locking-reads.html。
</p>


<h3>操作实例</h3>
<p>以DB2为例说明读锁的效果。</p>
<p>1. 开启两个DB2命令行窗口模拟两个事务。</p>
<p>2. 在命令行窗口连接数据库。命令类似：db2 connect to SAMPLE user userxxx using passwordxxx。输入db2进入db2命令模式，效果类似“db2 =>”。</p>
<p>3. 在两个窗口输入以下命令关闭db2自动提交（autocommit）选项：update command options using c off。查看是否设置成功使用命令：list command options。选项-c应该为off。
</p>
<p>4. 在窗口1（事务1）中输入SQL语句：select * from EMPLOYEE where EMPNO='000010' for update with rs。查询结果输出，返回一行。</p>
<p>5. 在窗口2（事务2）中输入SQL语句：select * from EMPLOYEE where EMPNO='000010' for update with rs。查询界面挂起。原因是第1个事务尚未提交，该数据行被第1个事务锁定。</p>
<p>6. 在窗口1（事务1）中输入命令：commit。事务1提交完毕。同时这个时候窗口2中查询结果正常显示。</p>
<p>7. 在窗口2（事务2）中输入命令：commit。事务2提交完毕。</p>
<p>8. 那么如果第5步查询不是for update with rs，查询会挂起么？答案是：不一定，跟锁级别有关系。可以将第5步的for update with rs修改为for update，或者将for update也去掉分别测试一下。</p>

