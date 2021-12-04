### 1、创建SqlSession


```java
//在DefaultSqlSessionFactory中创建SqlSession
@Override
public SqlSession openSession() {
    //Configuration中的默认值
    //protected ExecutorType defaultExecutorType = ExecutorType.SIMPLE;
    return openSessionFromDataSource(configuration.getDefaultExecutorType(), null, false);
}
```

```java
//创建SqlSession
private SqlSession openSessionFromDataSource(ExecutorType execType, TransactionIsolationLevel level, boolean autoCommit) {
    Transaction tx = null;
    try {
      final Environment environment = configuration.getEnvironment();
      final TransactionFactory transactionFactory = getTransactionFactoryFromEnvironment(environment);
      tx = transactionFactory.newTransaction(environment.getDataSource(), level, autoCommit);
      //创建Executor对象
      final Executor executor = configuration.newExecutor(tx, execType);
      //返回DefaultSqlSession对象
      return new DefaultSqlSession(configuration, executor, autoCommit);
    } catch (Exception e) {
      closeTransaction(tx); // may have fetched a connection so lets call close()
      throw ExceptionFactory.wrapException("Error opening session.  Cause: " + e, e);
    } finally {
      ErrorContext.instance().reset();
    }
}
```

#### 1.1、创建Executor过程

调用configuration.newExecutor方法，根据executorType创建不同Executor对象。

```java
public Executor newExecutor(Transaction transaction, ExecutorType executorType) {
    executorType = executorType == null ? defaultExecutorType : executorType;
    executorType = executorType == null ? ExecutorType.SIMPLE : executorType;
    Executor executor;
   	//批量执行器
    if (ExecutorType.BATCH == executorType) {
      executor = new BatchExecutor(this, transaction);
    //可重复使用执行器
    } else if (ExecutorType.REUSE == executorType) {
      executor = new ReuseExecutor(this, transaction);
    } else {
      //简单执行器
      executor = new SimpleExecutor(this, transaction);
    }
    if (cacheEnabled) {
      executor = new CachingExecutor(executor);
    }
    executor = (Executor) interceptorChain.pluginAll(executor);
    return executor;
}
```

```java
public enum ExecutorType {
  	SIMPLE,//为每个语句创建一个PreparedStatement
    REUSE, //重复使用PreparedStatements
    BATCH  //批量执行所有的更新语句
}
```

#### 1.2、创建DefaultSqlSession

这里创建DefaultSqlSession，默认使用SimpleExecutor进行sql执行。



### 2、执行DefaultSqlSession的selectList方法

假设调用selectList

```java
@Override
public <E> List<E> selectList(String statement, Object parameter, RowBounds rowBounds) {
    try {
      //获取MappedStatement对象
      MappedStatement ms = configuration.getMappedStatement(statement);
      //使用执行器执行查询
      return executor.query(ms, wrapCollection(parameter), rowBounds, Executor.NO_RESULT_HANDLER);
    } catch (Exception e) {
      throw ExceptionFactory.wrapException("Error querying database.  Cause: " + e, e);
    } finally {
      ErrorContext.instance().reset();
    }
}
```



### 3、执行BaseExecutor的query方法

```java
@SuppressWarnings("unchecked")
@Override
public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
    ErrorContext.instance().resource(ms.getResource()).activity("executing a query").object(ms.getId());
    if (closed) {
      throw new ExecutorException("Executor was closed.");
    }
    if (queryStack == 0 && ms.isFlushCacheRequired()) {
      clearLocalCache();
    }
    List<E> list;
    try {
      queryStack++;
      list = resultHandler == null ? (List<E>) localCache.getObject(key) : null;
      if (list != null) {
        handleLocallyCachedOutputParameters(ms, key, parameter, boundSql);
      } else {
        //从数据库中进行查询
        list = queryFromDatabase(ms, parameter, rowBounds, resultHandler, key, boundSql);
      }
    } finally {
      queryStack--;
    }
    if (queryStack == 0) {
      for (DeferredLoad deferredLoad : deferredLoads) {
        deferredLoad.load();
      }
      // issue #601
      deferredLoads.clear();
      if (configuration.getLocalCacheScope() == LocalCacheScope.STATEMENT) {
        // issue #482
        clearLocalCache();
      }
    }
    return list;
}
```



### 4、执行BaseExecutor的queryFromDatabase方法

```java
private <E> List<E> queryFromDatabase(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
    List<E> list;
    localCache.putObject(key, EXECUTION_PLACEHOLDER);
    try {
      //这里会调用doQuery方法，由于doQuery是抽象方法，所以调用子类实现
      //这里默认调用SimpleExecutor的doQuery方法
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



### 5、调用SimpleExecutor的doQuery方法

下面是doQuery的内部执行逻辑：

```java
@Override
public <E> List<E> doQuery(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) throws SQLException {
    Statement stmt = null;
    try {
      //获取Configuration对象
      Configuration configuration = ms.getConfiguration();
      //获取StatementHandler
      StatementHandler handler = configuration.newStatementHandler(wrapper, ms, parameter, rowBounds, resultHandler, boundSql);
      //获取Statement
      stmt = prepareStatement(handler, ms.getStatementLog());
      //执行StatementHandler的query方法获取查询结果
      return handler.query(stmt, resultHandler);
    } finally {
      //关闭Statement
      closeStatement(stmt);
    }
}
```



#### 5.1、执行configuration.newStatementHandler

```java
public StatementHandler newStatementHandler(Executor executor, MappedStatement mappedStatement, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) {
    //创建RoutingStatementHandler对象
    StatementHandler statementHandler = new RoutingStatementHandler(executor, mappedStatement, parameterObject, rowBounds, resultHandler, boundSql);
    statementHandler = (StatementHandler) interceptorChain.pluginAll(statementHandler);
    return statementHandler;
}
```

RoutingStatementHandler对象内部包装真正的用于执行的StatementHandler。



下面代码是RoutingStatementHandler初始化时所得事情：

```java
public RoutingStatementHandler(Executor executor, MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) {
	//根据StatementType创建不同的StatementHandler对象
    switch (ms.getStatementType()) {
      case STATEMENT://简单类型
        delegate = new SimpleStatementHandler(executor, ms, parameter, rowBounds, resultHandler, boundSql);
        break;
      case PREPARED://预编译类型
        delegate = new PreparedStatementHandler(executor, ms, parameter, rowBounds, resultHandler, boundSql);
        break;
      case CALLABLE://存储过程类型
        delegate = new CallableStatementHandler(executor, ms, parameter, rowBounds, resultHandler, boundSql);
        break;
      default:
        throw new ExecutorException("Unknown statement type: " + ms.getStatementType());
    }

}
```



#### 5.2、执行prepareStatement

```java
private Statement prepareStatement(StatementHandler handler, Log statementLog) throws SQLException {
    Statement stmt;
    //获取数据库连接Connection
    Connection connection = getConnection(statementLog);
    //获取Statement，这里会调用BaseStatementHandler的prepare方法
    stmt = handler.prepare(connection, transaction.getTimeout());
    //设置Statement相关参数
    handler.parameterize(stmt);
    return stmt;
}
```

##### 5.2.1、BaseStatementHandler.prepare方法

```java
@Override
public Statement prepare(Connection connection, Integer transactionTimeout) throws SQLException {
    ErrorContext.instance().sql(boundSql.getSql());
    Statement statement = null;
    try {
      //初始化Statement
      statement = instantiateStatement(connection);
      //设置超时时间
      setStatementTimeout(statement, transactionTimeout);
      //设置FetchSize大小，用于大数据量的返回结果的处理
      setFetchSize(statement);
      return statement;
    } catch (SQLException e) {
      //异常关闭Statement
      closeStatement(statement);
      throw e;
    } catch (Exception e) {
      //异常关闭Statement
      closeStatement(statement);
      throw new ExecutorException("Error preparing statement.  Cause: " + e, e);
    }
}
```

instantiateStatement是一个抽象方法，会调用子类的实现，这里默认调用PreparedStatementHandler的实现：

```java
@Override
protected Statement instantiateStatement(Connection connection) throws SQLException {
    String sql = boundSql.getSql();
    if (mappedStatement.getKeyGenerator() instanceof Jdbc3KeyGenerator) {
      String[] keyColumnNames = mappedStatement.getKeyColumns();
      if (keyColumnNames == null) {
        //调用connection.prepareStatement进行预编译处理
        return connection.prepareStatement(sql, PreparedStatement.RETURN_GENERATED_KEYS);
      } else {
        //调用connection.prepareStatement进行预编译处理
        return connection.prepareStatement(sql, keyColumnNames);
      }
    } else if (mappedStatement.getResultSetType() == ResultSetType.DEFAULT) {
      //调用connection.prepareStatement进行预编译处理
      return connection.prepareStatement(sql);
    } else {
      //调用connection.prepareStatement进行预编译处理
      return connection.prepareStatement(sql, mappedStatement.getResultSetType().getValue(), ResultSet.CONCUR_READ_ONLY);
    }
}
```

#### 5.3、调用PreparedStatementHandler.query方法

```java
@Override
public <E> List<E> query(Statement statement, ResultHandler resultHandler) throws SQLException {
    PreparedStatement ps = (PreparedStatement) statement;
    //执行操作
    ps.execute();
    //结果解析
    return resultSetHandler.handleResultSets(ps);
}
```

