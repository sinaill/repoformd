title: MyBatis大致运行流程
categories: 框架
tags: 
	- Mybaits

---

### 原生调用方式

```
//第一种
SqlSession session = sqlSessionFactory.openSession();
try {
  Blog blog = (Blog) session.selectOne("org.mybatis.example.BlogMapper.selectBlog", 101);
} finally {
  session.close();
}

//第二种，使用到了代理(getMapper方法)

SqlSession session = sqlSessionFactory.openSession();
try {
  BlogMapper mapper = session.getMapper(BlogMapper.class);
  Blog blog = mapper.selectBlog(101);
} finally {
  session.close();
}
```

### 集成框架后使用代理

1. 启动服务器后，自动创建数据库服务接口的代理类

```
//MapperRegistry类的addMapper方法
public <T> void addMapper(Class<T> type) {
if (type.isInterface()) {
  if (hasMapper(type)) {
    throw new BindingException("Type " + type + " is already known to the MapperRegistry.");
  }
  boolean loadCompleted = false;
  try {
	//每个接口对应一个代理工厂实例
    knownMappers.put(type, new MapperProxyFactory<T>(type));
    // It's important that the type is added before the parser is run
    // otherwise the binding may automatically be attempted by the
    // mapper parser. If the type is already known, it won't try.
    MapperAnnotationBuilder parser = new MapperAnnotationBuilder(config, type);
    parser.parse();
    loadCompleted = true;
  } finally {
    if (!loadCompleted) {
      knownMappers.remove(type);
    }
  }
}
}
```

2. 触发Spring的autowire自动向service注入接口的代理类，通过sqlsession的getMappder的方法

```
public <T> T getMapper(Class<T> type) {
return getConfiguration().getMapper(type, this);
}

//跟踪源码
public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
	return mapperRegistry.getMapper(type, sqlSession);
}

//mapperRegistry就是初始化容器时，通过addMapper方法，给每个接口确定了对应的代理工厂实例
public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
	return mapperRegistry.getMapper(type, sqlSession);
}

//mapperRegistry类
public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
//取出之前添加的代理工厂类
	final MapperProxyFactory<T> mapperProxyFactory = (MapperProxyFactory<T>) knownMappers.get(type);
	if (mapperProxyFactory == null) {
	  throw new BindingException("Type " + type + " is not known to the MapperRegistry.");
	}
	try {
	  //使用代理工厂类创建代理类实例
	  return mapperProxyFactory.newInstance(sqlSession);
	} catch (Exception e) {
	  throw new BindingException("Error getting mapper instance. Cause: " + e, e);
	}
}

//转进mapperProxyFactory类
protected T newInstance(MapperProxy<T> mapperProxy) {
	//这行代码生成代理
	return (T) Proxy.newProxyInstance(mapperInterface.getClassLoader(), new Class[] { mapperInterface }, mapperProxy);
}

public T newInstance(SqlSession sqlSession) {
	final MapperProxy<T> mapperProxy = new MapperProxy<T>(sqlSession, mapperInterface, methodCache);
	return newInstance(mapperProxy);
}
  
```

3. 代理类的生成

我们知道代理类的生成需要一个实现invocationHandler接口的类，在MyBatis中这个类为MapperProxy，查看invoke方法

```
  public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    if (Object.class.equals(method.getDeclaringClass())) {
      try {
        return method.invoke(this, args);
      } catch (Throwable t) {
        throw ExceptionUtil.unwrapThrowable(t);
      }
    }
	查看method方法对用的MapperMehtod是否存在，不存在则创建
    final MapperMethod mapperMethod = cachedMapperMethod(method);
    return mapperMethod.execute(sqlSession, args);
  }

  private MapperMethod cachedMapperMethod(Method method) {
    MapperMethod mapperMethod = methodCache.get(method);
    if (mapperMethod == null) {
      mapperMethod = new MapperMethod(mapperInterface, method, sqlSession.getConfiguration());
      methodCache.put(method, mapperMethod);
    }
    return mapperMethod;
  }
```

4. 通过代理调用原生调用方式

通过MapperMehtod这个类来实现，先看下它的成员变量

```
  private final SqlCommand command;
  private final MethodSignature method;

  public MapperMethod(Class<?> mapperInterface, Method method, Configuration config) {
    this.command = new SqlCommand(config, mapperInterface, method);
    this.method = new MethodSignature(config, method);
  }



```

查看sqlcommand是什么，大概就是关于sql语句

```
public SqlCommand(Configuration configuration, Class<?> mapperInterface, Method method) {
  //接口名加方法名，正好等于我们配置文件的命名空间namespace加上语句的id
  String statementName = mapperInterface.getName() + "." + method.getName();
  MappedStatement ms = null;
  if (configuration.hasStatement(statementName)) {
	//大概是通过命名空间和id从配置文件查找语句
    ms = configuration.getMappedStatement(statementName);
  } else if (!mapperInterface.equals(method.getDeclaringClass())) { // issue #35
    String parentStatementName = method.getDeclaringClass().getName() + "." + method.getName();
    if (configuration.hasStatement(parentStatementName)) {
      ms = configuration.getMappedStatement(parentStatementName);
    }
  }
  if (ms == null) {
    if(method.getAnnotation(Flush.class) != null){
      name = null;
      type = SqlCommandType.FLUSH;
    } else {
      throw new BindingException("Invalid bound statement (not found): " + statementName);
    }
  } else {
    name = ms.getId();
    type = ms.getSqlCommandType();
    if (type == SqlCommandType.UNKNOWN) {
      throw new BindingException("Unknown execution method for: " + name);
    }
  }
}
```

在代理类中生成MethodMapper类后，调用execute方法来指定sqlcommand中的sql语句

```
  public Object execute(SqlSession sqlSession, Object[] args) {
	//args是由代理类invoke方法传来的方法参数
    Object result;
    if (SqlCommandType.INSERT == command.getType()) {
	
      Object param = method.convertArgsToSqlCommandParam(args);
      result = rowCountResult(sqlSession.insert(command.getName(), param));
    } else if (SqlCommandType.UPDATE == command.getType()) {
      Object param = method.convertArgsToSqlCommandParam(args);
      result = rowCountResult(sqlSession.update(command.getName(), param));
    } else if (SqlCommandType.DELETE == command.getType()) {
      Object param = method.convertArgsToSqlCommandParam(args);
      result = rowCountResult(sqlSession.delete(command.getName(), param));
    } else if (SqlCommandType.SELECT == command.getType()) {
      if (method.returnsVoid() && method.hasResultHandler()) {
        executeWithResultHandler(sqlSession, args);
        result = null;
      } else if (method.returnsMany()) {
        result = executeForMany(sqlSession, args);
      } else if (method.returnsMap()) {
        result = executeForMap(sqlSession, args);
      } else {
        Object param = method.convertArgsToSqlCommandParam(args);
        result = sqlSession.selectOne(command.getName(), param);
      }
    } else if (SqlCommandType.FLUSH == command.getType()) {
        result = sqlSession.flushStatements();
    } else {
      throw new BindingException("Unknown execution method for: " + command.getName());
    }
    if (result == null && method.getReturnType().isPrimitive() && !method.returnsVoid()) {
      throw new BindingException("Mapper method '" + command.getName() 
          + " attempted to return null from a method with a primitive return type (" + method.getReturnType() + ").");
    }
	//返回结果
    return result;
  }
	
  //对返回结果result进行类型转换
  private Object rowCountResult(int rowCount) {
    final Object result;
    if (method.returnsVoid()) {
      result = null;
    } else if (Integer.class.equals(method.getReturnType()) || Integer.TYPE.equals(method.getReturnType())) {
      result = Integer.valueOf(rowCount);
    } else if (Long.class.equals(method.getReturnType()) || Long.TYPE.equals(method.getReturnType())) {
      result = Long.valueOf(rowCount);
    } else if (Boolean.class.equals(method.getReturnType()) || Boolean.TYPE.equals(method.getReturnType())) {
      result = Boolean.valueOf(rowCount > 0);
    } else {
      throw new BindingException("Mapper method '" + command.getName() + "' has an unsupported return type: " + method.getReturnType());
    }
    return result;
  }
```