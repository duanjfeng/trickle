#Mybatis Update前后Select返回相同结果
`Mybatis` `LocalCache`

##问题简述
在一段数据库操作代码里，针对同一条数据库记录，Update语句前后相同的Select语句返回相同的结果。也就是说Update语句好像没有生效一样。
##问题分析
难道是Update失败了？No，Mybatis没有报错，没有任何异常。
难道是数据可见性问题？你是说事务么？No，同一个事务内，更新操作对于查询操作是可见的。
是Mybatis搞的鬼？Yes，跟踪一下Mybatis的代码，你会发现LocalCache大神，就是它导致了上面那个诡异的问题。
###LocalCache的作用
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
###什么时候用到LocalCache
还是上面那个BaseExecutor，在query时会先去LocalCache看看有没有结果。
```Java
list = resultHandler == null ? (List<E>) localCache.getObject(key) : null;
      if (list != null) {
        handleLocallyCachedOutputParameters(ms, key, parameter, boundSql);
      } else {
        list = queryFromDatabase(ms, parameter, rowBounds, resultHandler, key, boundSql);
      }
```
###什么时候会清理LocalCache
TODO
###LocalCache的作用域
TODO LocalCacheScope


2015-3-14
