---
date: 2021-01-03T08:31:23+08:00
featured_image: "https://images.unsplash.com/photo-1616077167555-51f6bc516dfa?ixlib=rb-1.2.1&ixid=MnwxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8&auto=format&fit=crop&w=2071&q=80"
tags: [spring,事务]
title: "转账案例-引入事务和AOP改造"
categories: Java
description: 通过简单的转账业务案例，从最初无事务的基础功能版本，通过代理和AOP逐步进行改造和优化
---

### 基础功能

---

spring 整合 DBUtils ，实现简单的用户转账业务

- Account 实体类

  ```java
  public class Account {
    private Integer id;
    private String name;
    private Double money;
    // setter getter....
  }
  ```

- AccountDao 接口和实现类

  - ```java
    public interface AccountDao {
      // 转出操作
      void out(String outUser, Double money);
      // 转入操作
      void in(String inUser, Double money);
    }
    ```

  - ```java
    @Repository
    public class AccountDaoImpl implements AccountDao {
        @Autowired
        private QueryRunner queryRunner;
        @Override
        public void out(String outUser, Double money) {
          try {
              queryRunner.update("update account set money=money-? where name=?",
              money, outUser);
          } catch (SQLException e) {
              e.printStackTrace();
          }
        }
        
        @Override
        public void in(String inUser, Double money) {
          try {
            queryRunner.update("update account set money=money+? where name=?",
            money, inUser);
          } catch (SQLException e) {
            e.printStackTrace();
          }
    		}
    }
    ```

- AccountService 接口和实现类

  - ```java
      public interface AccountService {
          void transfer(String outUser, String inUser, Double money);
      }
      ```

  - ```java
    @Service
    public class AccountServiceImpl implements AccountService {
        @Autowired
        private AccountDao accountDao;
      
        @Override
        public void transfer(String outUser, String inUser, Double money) {
          accountDao.out(outUser, money);
          accountDao.in(inUser, money);
    		}
    }
    ```

> 至此实现了基本的业务转账逻辑，但以上代码中，传出和转入操作在 Dao 层实现，分别都是独立的事务，如果在转出后发生异常，转入操作会得不到原子性的保证，出现数据不一致的情况

改进：把业务逻辑控制在一个事务中，将事务挪到service层

### 传统的事务操作

---

#### 连接线程绑定工具类

为了保证在 Dao 层调用 in,out 方法的是同一个线程，编写一个连接工具类，从数据库获取连接对象，利用 ThreadLocal  实现和线程的绑定

```java
public class ConnectionUtils {
    @Autowired
    private DataSource dataSource;
    private ThreadLocal<Connection> threadLocal = new ThreadLocal<>();

    /*
        获取当前线程上绑定连接：如果获取到的连接为空，那么就要从数据源中获取连接，并且放到ThreadLocal中（绑定到当前线程）
     */
    public  Connection getThreadConnection(){
        // 1. 先从ThreadLocal上获取连接
        Connection connection = threadLocal.get();

        // 2. 判断当前线程中是否是有Connection
        if(connection == null){
            // 3. 从数据源中获取一个连接，并且存入ThreadLocal中
            try {
                // 不为null
                connection = dataSource.getConnection();

                threadLocal.set(connection);

            } catch (SQLException e) {
                e.printStackTrace();
            }
        }
        return  connection;
    }
    /*
        解除当前线程的连接绑定
     */
    public void  removeThreadConnection(){
        threadLocal.remove();
    }
}
```

#### 事务管理器工具类

封装事务操作，通过上面的线程绑定工具类获取 Connection

在这个工具类中，进行开启事务、提交事务、回滚事务、释放资源的统一控制

```java
@Component("transactionManager")
public class TransactionManager {

    @Autowired
    private ConnectionUtils connectionUtils;

    /*
      开启事务
    */
    public void beginTransaction() {

        // 获取connection对象
        Connection connection = connectionUtils.getThreadConnection();
        try {
            // 开启了一个手动事务
            connection.setAutoCommit(false);
        } catch (SQLException e) {
            e.printStackTrace();
        }

    }

    /*
        提交事务
     */
    public void commit() {
        Connection connection = connectionUtils.getThreadConnection();
        try {
            connection.commit();
        } catch (SQLException e) {
            e.printStackTrace();
        }

    }

    /*
        回滚事务
     */
    public void rollback() {
        Connection connection = connectionUtils.getThreadConnection();
        try {
            connection.rollback();
        } catch (SQLException e) {
            e.printStackTrace();
        }
    }

    /*
        释放资源
     */
    public void release() {
        // 将手动事务改回成自动提交事务
        Connection connection = connectionUtils.getThreadConnection();
        try {
            connection.setAutoCommit(true);
            // 将连接归还到连接池
            connectionUtils.getThreadConnection().close();
            // 解除线程绑定
            connectionUtils.removeThreadConnection();
        } catch (SQLException e) {
            e.printStackTrace();
        }
    }
}
```

#### 改造 Servie 和 Dao

- Service 层中进行手动开启事务，无误则手动提交，异常则回滚

  ```java
      public void transfer(String outUser, String inUser, Double money) {
          //手动开启事务
          transactionManager.beginTransaction();
          try {
              accountDao.out(outUser,money);
              accountDao.in(inUser, money);
              //手动提交事务
              transactionManager.commit();
          } catch (Exception e) {
              transactionManager.rollback();
              e.printStackTrace();
          } finally {
              transactionManager.release();
          }
  ```

  

- Dao 方法中使用 DBUtils 时，传入从 ConnectionManager 中获取的 Connection

  ```java
   public void out(String outAccount, Double money) {
          String sql = "update account set money = money - ? where name = ?";
          try {
              queryRunner.update(connectionUtils.getThreadConnection(), sql,money,outAccount);
          } catch (SQLException e) {
              e.printStackTrace();
          }
      }
  
      @Override
      public void in(String inAccount, Double money) {
          String sql = "update account set money=money+? where name=?";
          try {
           queryRunner.update(connectionUtils.getThreadConnection(),sql,money,inAccount);
          } catch (SQLException e) {
              e.printStackTrace();
          }
      }
  ```

> 通过以上的改造，已经实现了业务控制，但也引入了新的问题：

业务层方法变得臃肿了，业务逻辑和事务控制方法耦合，如果业务层方法越来越多，将充斥着重复的代码，违背了面向对象低耦合的开发思想

### Proxy 动态代理优化

---

思路：将业务代码和事务控制代码分离，通过动态代理的方式对业务方法进行事务增强，以降低耦合从而不对业务层产生影响

#### JDK 动态代理

基于接口的动态代理，被代理的类（目标类）必须实现一个接口

![](https://logseq.oss-cn-chengdu.aliyuncs.com/noteImg/202203122002207.png)

利用实现 InvocationHandler 接口的拦截器和反射机制，生成代理接口的匿名类（代理对象），在调用目标方法前调用 InvokeHander 来实现方法增强

- JDKProxyFactory：动态代理工厂类，在返回的动态代理对象中进行事务控制

  ```java
  @Component("proxyFactory")
  public class JDKProxyFactory {
      /**
       * 目标类（被代理对象）
       * 利用接口来接收
       */
      @Autowired
      private AccountService accountService;
  
      @Autowired
      private TransactionManager transactionManager;
  
      public AccountService createAccountServiceJDKProxy() {
          AccountService accountServiceProxy = null;
          accountServiceProxy = (AccountService) Proxy.newProxyInstance(accountService.getClass().getClassLoader(),
                  accountService.getClass().getInterfaces(),new InvocationHandler(){
  
                      @Override
                      public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
                          Object result = null;
                          try {
                              transactionManager.beginTransaction();
                              result = method.invoke(accountService,args);
                              transactionManager.commit();
                          }catch(InvocationTargetException e) {
                              e.printStackTrace();
                              transactionManager.rollback();
                          }finally {
                              transactionManager.release();
                          }
                          return result;
                      }
                  });
          return accountServiceProxy;
      }
  }
  ```

#### CGLIB 动态代理

基于父类的动态代理技术，无需实现接口

在运行时，为代理类动态生成子类，重写被代理类所有非 final 方法。在子类中使用方法拦截的技术拦截所有方法调用，植入横切逻辑对方法进行增强

```java
public class CglibProxyFactory {
    @Autowired
    private AccountService accountService;
    @Autowired
    private TransactionManager transactionManager;

    public AccountService createAccountServiceCglibProxy() {
        AccountService accountServiceProxy = null;
        accountServiceProxy = (AccountService)
          Enhancer.create(accountService.getClass(), new MethodInterceptor() {
              @Override
              public Object intercept(Object o, Method method, Object[] objects, MethodProxy methodProxy) throws Throwable {
                  Object result = null;
                  try {
                      // 1.开启事务
                      transactionManager.beginTransaction();
                      // 2.业务操作
                      result = method.invoke(accountService, objects);
                      // 3.提交事务
                      transactionManager.commit();
                  } catch (Exception e) {
                      e.printStackTrace();
                      // 4.回滚事务
                      transactionManager.rollback();
                  } finally {
                      // 5.释放资源
                      transactionManager.release();
                  }
                  return result;
              }
          });
        return accountServiceProxy;
    }
}
```

### AOP 优化

配置切点和切面 

- XML 格式的基本通知类型

```xml
    <aop:config>
<!--        切点表达式-->
        <aop:pointcut id="myPt" expression="execution(* xyz.wvuuvw.service.impl.AccountServiceImpl.*(..))"/>
<!--        切面配置-->
        <aop:aspect ref="transactionManager">
            <aop:before method="beginTransaction" pointcut-ref = "myPt"/>
            <aop:after-returning method="commit" pointcut-ref = "myPt"/>
            <aop:after-throwing method="rollback" pointcut-ref = "myPt"/>
            <aop:after method="release" pointcut-ref = "myPt"/>
        </aop:aspect>
    </aop:config>
```

- 注解形式的环绕通知

```java
@Component("transactionManager")
@Aspect //表明该类为切面类
public class TransactionManager {

    @Autowired
    private ConnectionUtils connectionUtils;


    @Around("execution(* xyz.wvuuvw.service.impl.AccountServiceImpl.*(..))")
    public Object around(ProceedingJoinPoint pjp) throws SQLException {

        Object proceed = null;
        try {
            // 开启手动事务
            connectionUtils.getThreadConnection().setAutoCommit(false);
            // 切入点方法执行
            proceed = pjp.proceed();

            // 手动提交事务
            connectionUtils.getThreadConnection().commit();

        } catch (Throwable throwable) {
            throwable.printStackTrace();
            // 手动回滚事务
            connectionUtils.getThreadConnection().rollback();

        } finally {
            // 将手动事务恢复成自动事务
            connectionUtils.getThreadConnection().setAutoCommit(true);
            // 将连接归还到连接池
            connectionUtils.getThreadConnection().close();
            // 解除线程绑定
            connectionUtils.removeThreadConnection();
        }
        return  proceed;
    }
  ....
```



