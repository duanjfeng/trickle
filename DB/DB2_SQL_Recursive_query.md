#SQL Recursive query
`recursive query` `递归查询`

##问题简述
写一个SQL语句（DB2），递归查询某个机构及其子机构。<br>
机构表其中两列为(ORG_ID,PARENT_ORG_ID)。这里所谓递归查询是指，不仅要查到A的子机构B，而且如果B有子机构C，那么C也应该在结果集中。

##问题解决
###fibonacci数列
为什么要提fibonacci数列？因为这里有我们接触最早的关于递归的概念，甚至很多人小学时候就知道。而作为一个程序员，通常会被要求写一个C语言的程序来计算fibonacci数列第n个元素的值。<br>
关于递归有最基本的三个概念，初始条件，递归算法，结束条件。fibonacci中f0=0，f1=1即为初始条件；fn=fn-1+fn-2为递归算法；某个n值为结束条件。<br>
###递归查询
我们要写一个递归的查询SQL语句，也应该搞清楚对应三个概念。<br>
初始条件：某个机构。<br>
递归算法：这里的递归算法是由机构的上下级关系来体现的，即机构树中父子节点关系。<br>
结束条件：没有新的子节点产生。
###SQL语句
下面写一个满足DB2语法的SQL语句。
```SQL
with VT(ORG_ID,PARENT_ORG_ID) as
(
select root.ORG_ID,root.PARENT_ORG_ID from DM_ORG root where ORG_ID = '0001'
union all
select child.ORG_ID,child.PARENT_ORG_ID from VT parent,DM_ORG child WHERE child.PARENT_ORG_ID = parent.ORG_ID
)
select distinct(ORG_ID) from VT order by ORG_ID
```
第1行，with表达式，在DB2中，含义为虚拟表，所以我这里命名为VT（意为virtual table）。这个虚拟表为查询的最终结果集<br>
第3行，select语句，定义初始条件，这个语句查询的结果将首先进入VT，并参与后面的递归运算。<br>
第4行，union all，递归语句中必须为union all，你如果写union，DB2将会报错。<br>
第5行，select语句，定义递归算法，这个select查询的结果将进入VT，并继续运算。因此最开始VT里面只有机构0001，运算第1次select到0001的子节点，并加入VT，以此类推。直到没有查询到新的子节点为止，即结束。<br>
第7行，使用递归结果集的SQL语句。VT作为一张临时表，充当实体表的角色，可以作为过滤条件。<br>

##小结
小结三点：<br>
1.通过基本原理理清思路。<br>
2.通过实例加强理解。<br>
3.通过手册查询语法，得到结果。<br>

2015-01-06
