#DB2 BatchUpdateException ERRORCODE -4229

`DB2` `BatchUpdateException` `ERRORCODE -4229`

##问题现象
* Java MyBatis程序，执行批量SQL时（BATCH），程序抛出异常，如下：<br>
  com.ibm.db2.jcc.am.BatchUpdateException: [jcc][t4][102][10040][4.14.88] 批处理出现故障。虽然已经提交了批处理，但是该批处理的某个成员至少发生了一个异常。
使用 getNextException() 来检索已经过批处理的特定元素的异常。 ERRORCODE=-4229, SQLSTATE=null
* 使用db2 ? sql-4229命令没有查询到该ERRORCODE对应的解释。
* 通过搜索引擎搜索到一篇文档：[Retrieving information from a BatchUpdateException](http://publib.boulder.ibm.com/infocenter/idshelp/v111/topic/com.ibm.jccids.doc/com.ibm.db2.luw.apdv.java.doc/doc/tjvjdbue.htm "Retrieving information from a BatchUpdateException")，进而很快解决该问题。

##解决过程

* 由于程序直接抛出的是org.apache.ibatis.exceptions.PersistenceException，所以首先需要捕获该对象，并分析结构。结构如下：

 >PersistenceException
 >>BatchExecutorException
 >>>BatchUpdateException
 >>>>next

* 参考Retrieving information from a BatchUpdateException文档，修改代码以获取BatchUpdateException的更多信息。代码如下：
``` java
 catch (PersistenceException pe) {
            logger.error("PersistenceException");
            Throwable cause = pe.getCause();
            if (cause instanceof BatchExecutorException) {
                BatchExecutorException bee = (BatchExecutorException) cause;
                BatchUpdateException bue = bee.getBatchUpdateException();
                logger.error("Contents of BatchUpdateException:");
                logger.error("Update counts:");
                int[] updateCounts = bue.getUpdateCounts();
                for (int i = 0; i &lt; updateCounts.length; i++) {
                    int affected = updateCounts[i];
                    logger.error("Statement " + i + ":" + affected);
                    if (affected &lt;= 0) {
                        //LOGGER BUSINESS INFO IF NEEDED
                    }
                }
                logger.error("Message" + bue.getMessage());
                logger.error("SQLSTATE: " + bue.getSQLState());
                logger.error("Error code: " + bue.getErrorCode());
                SQLException nextException = bue.getNextException();
                while (nextException != null) {
                    logger.error("SQLException:");
                    logger.error("Message: " + nextException.getMessage());
                    logger.error("SQLSTATE: " + nextException.getSQLState());
                    logger.error("Error code: " + nextException.getErrorCode());
                    nextException = nextException.getNextException();
                }
            }
            else {
                logger.error("Cause not instanceof BatchExecutorException. Message: " + cause.getMessage());
            }
            if (session != null) {
                session.rollback();
            }
        }
```
* 在测试环境运行同样的数据，根据日志中更为详尽的信息，分析导致BatchUpdateException的根本原因。我碰到的就是因为业务人员人工维护的数据错误，导致主键冲突。由于数据量较大，4万行，如果不通过上述方法，将很难定位到哪一行。

<br>2014-09-24
