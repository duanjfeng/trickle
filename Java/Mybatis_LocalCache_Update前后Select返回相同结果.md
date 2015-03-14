#Mybatis Update前后Select返回相同结果
`Mybatis` `LocalCache`

##问题简述
使用[开源工作流引擎Activiti](http://activiti.org/)开发工作流应用，由于业务需求，有这样一个场景：<br/>
(1)使用Mybatis直接访问数据的形式获取某个流程实例的当前处理人；<br/>
(2)执行Activiti的complete操作完成一个任务(Update操作)；<br/>
(3)再次使用使用Mybatis直接访问数据的形式获取某个流程实例的当前处理人。<br/>
根据上述描述，应该流程实例发生了迁移，当前处理人发生变化。可是实际情况是，第(1)和第(3)步查询结果相同。
##问题分析
(1)难道是Update失败了？No，Mybatis没有报错，没有任何异常。<br/>
(2)难道是数据可见性问题？你是说事务么？No，同一个事务内，更新操作对于查询操作是可见的。<br/>
(3)难道是Mybatis搞的鬼？Yes，跟踪一下Mybatis的代码，你会发现LocalCache大神，就是它导致了上面那个诡异的问题。<br/>
为了控制篇幅，下面着重介绍LocalCache，前面两条可能性的排除过程就不赘述了。
###什么时候产生LocalCache？
为了提高效率，在从数据库查询到结果后，Mybatis会将查询结果放到一个LocalCache里面。代码如下(org.apache.ibatis.executor.BaseExecutor)：
```Java
private <E> List<E> queryFromDatabase(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
    List<E> list;
    localCache.putObject(key, EXECUTION_PLACEHOLDER);
    try {
      list = doQuery(ms, parameter, rowBounds, resultHandler, boundSql);
    } finally {
      localCache.removeObject(key);
    }
    localCache.putObject(key, list);
    if (ms.getStatementType() == StatementType.CALLABLE) {
      localOutputParameterCache.putObject(key, parameter);
    }
    return list;
  }
```
###什么时候使用LocalCache？
还是上面那个BaseExecutor，在query时会先去LocalCache看看有没有结果。
```Java
list = resultHandler == null ? (List<E>) localCache.getObject(key) : null;
      if (list != null) {
        handleLocallyCachedOutputParameters(ms, key, parameter, boundSql);
      } else {
        list = queryFromDatabase(ms, parameter, rowBounds, resultHandler, key, boundSql);
      }
```
###什么时候会清理LocalCache？
BaseExecutor对象清理LocalCache发生在：<br/>
(1)关闭的时候。<br/>
(2)执行update语句的时候。<br/>
(3)提交事务的时候。<br/>
(4)回滚事务的时候。<br/>
###Update语句为什么没有起作用？
答：第二次查询的时候Mybatis返回了缓存内容。<br/>
问：不是说update语句会清理缓存么？<br/>
答：回答这个问题得搞清楚LocalCache的作用域。<br/>
###LocalCache的作用域是什么？
LocalCache的作用域有两种，都定义在LocalCacheScope类中，分别是SESSION和STATEMENT。而默认是SESSION，定义在Configuration中。也就是说默认在一个Session范围内使用。<br/>
而Activiti与我们的查询语句使用的是不同的Session，所以Activiti的complete并没有导致LocalCache的清理。

#问题解决
通过分析，我们知道，如果Mybatis不使用LocalCache就不会导致最开始的问题。那么怎么样关闭呢？<br/>
再次阅读query方法的实现，会发现MappedStatement有个flushCacheRequired属性，通过配置这个属性可以强制Statement清理LocalCache。<br/>
这个属性在xml中attribute为flushCache="true"。


[全文完]
2015-3-14
