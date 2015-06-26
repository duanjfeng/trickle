#DB2: Copy database
#DB2：复制数据库
`copy` `database`

##问题简述
有时我们需要复制/重建一套系统，例如搭建一套与生产一模一样的测试系统。<br/>
系统代码本来就是一样的，只是环境有差异，其中关键是数据库。<br/>
那么，在DB2数据库系统上，如何复制出一套数据库（或者说是schema）呢？<br/>

##问题解决
###导出建表语句
使用db2look命令。<br/>
-d指定数据库名称，-z指定schema名称，-e抽取复制数据库所需的DDL文件，-o指定输出文件名称。更多参数请使用“db2look --help”查看。<br/>
```bash
	
	db2look -d DATABASE_NAME -z SCHEMA_NAME -e -o create.sql
```
<br/>
只需要执行这一条命令，在输出的sql脚步文件中，将会包含创建schema的语句，创建所有table的语句（含表空间），所有comment语句，所有主键、外键依赖语句等内容。<br/>

###导出数据
使用db2 export命令。<br/>
指定目标文件名称、文件格式、输出日志、查询语句等内容。更多参数请使用“db2 ? export”查看。<br/>
与db2look不同的是，这里一个表格需要一行命令。因此，最好将所有命令都放到一个.sh文件中一次性执行。<br/>
```bash
	
	db2 "export to TABLE_NAME.ixf of ixf messages exp_TABLE_NAME.log select * from SCHEMA_NAME.TABLE_NAME";
```
<br/>
关于文件格式：ixf为二进制格式，del为跨平台格式，一般同一个数据库系统的备份/复制，建议使用ixf。<br/>
del格式有一个不足，需要指定列分隔符，如果数据中出现该分隔符，则会导致错误。例如，两列的表格，我们以“|”分隔，但是如果第一列的数据中出现了“|”，导出的数据就会变成三列，这样就无法导入。（我曾经使用del备份数据导致无法恢复，最后是使用数据库日志恢复的数据，印象深刻。）<br/>

###导入数据
使用db2 import命令。<br/>
指定数据文件名称、文件格式、输出日志、插入语句等内容。更多参数请使用“db2 ? import”查看。<br/>
```bash
	
	db2 "import from TABLE_NAME.ixf of ixf messages imp_TABLE_NAME.log insert into SCHEMA_NAME.TABLE_NAME";
```
<br/>
db2 export与db2 import是两个对称命令，可相互参考，不再赘述。<br/>

<br/>
2015-6-26
