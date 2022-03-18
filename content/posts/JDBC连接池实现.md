---
date: 2020-11-14T11:25:05-04:00
featured_image: "https://images.unsplash.com/photo-1627389955646-6596047473d7?ixlib=rb-1.2.1&ixid=MnwxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8&auto=format&fit=crop&w=1974&q=80"
tags: [数据库,JDBC]
title: "自定义数据库连接池的实现"
categories: Java
description: 实现简单的数据库连接池,讨论多种方式归还连接的功能
---

### 背景

数据库连接池用于分配、管理和释放（归还）数据库连接对象 Connection

- 有了数据库连接池，可以允许应用重复使用一个现有的数据库连接
- 从而避免了频繁创建和释放连接，消耗系统资源

最常用的第三方连接池有DBCP、 C3p0、Druid 等，本文实现一个具有基本功能的连接池

### 基本功能

JDBC 中 `java.sql.DataSource` 接口定义了数据库连接池的规范

- 代表一个数据源，可以用于实现连接池
- 获取数据库连接对象的方法：`Connection getConnection();`

覆写这个接口中的方法，可实现基本的连接池功能：

1. 定义一个类，实现DataSource接口
2. 定义一个容器，保存多个Connection连接对象
3. 定义静态代码块，通过JDBC工具类获取多个连接保存到容器中
4. ==重写getConnection方法，从容器获取连接并返回==
5. 定义getSize方法，获取容器大小并返回

```java
public class MyDataSource implements DataSource{
    //1.定义集合容器，用于保存多个数据库连接对象
    //Collections.synchronizedList可以将普通集合转成线程安全集合
    private static List<Connection> pool = Collections.synchronizedList(new ArrayList<Connection>());
    //2.定义静态代码块，生成10个数据库连接保存到集合中
    static {
        for (int i = 0; i < 10; i++) {
            Connection con = JDBCUtils.getConnection();
            pool.add(con);
        }
    }
    //返回连接池的大小
    public int getSize() {
        return pool.size();
    }

    //从池中返回一个数据库连接
    @Override
    public Connection getConnection() {
        if(pool.size() > 0) {
            //从池中获取数据库连接
            return pool.remove(0);
        }else {
            throw new RuntimeException("连接数量已用尽");
        }
    }
 .....其他需要重写的方法
}
```

### 归还连接功能

#### ~~继承~~

通过打印 Connection.getClass() 可以发现 DriverManager 获取的连接对象实现类是 JDBC4Connection，是不是可以自定义一个类来继承JDBC4Connection这个类，重写close() 方法呢？

经过测试是行不通的，DriverManager 还是会去获取JDBC4Connection对象，猜测应该是使用了自定义的类加载的缘故，而这个驱动内部的类我们是无法修改的

#### 装饰者模式

> 定义一个类，实现Connection接口，这样就具备了和JDBC4Connection相同行为， 重写close()方法，完成连接归还，其余功能调用原有实现方法

##### 实现步骤

1. 定义一个类实现Connection接口
2. 定义Connection连接对象和连接池容量对象的成员变量
3. 用有参构造方法给成员变量赋值
4. 重写close()方法，将连接对象添加到池子中
5. 其他方法调用Mysql 驱动包的原方法即可
6. 在自定义连接池中，将获取的连接对象通过自定义连接对象进行包装

```java
public class MyConnection implements Connection {

    //定义Connection连接对象和连接池容器对象的变量
    private Connection con;
    private List<Connection> pool;

    //提供有参构造方法，接收连接对象和连接池对象，对变量赋值
    public MyConnection2(Connection con,List<Connection> pool) {
        this.con = con;
        this.pool = pool;
    }

    //重写close()方法，完成连接的归还
    @Override
    public void close() throws SQLException {
        pool.add(con);
    }
    其他方法....
    return con.xxx
}
```

##### 修改获取连接方法

```java
    //从池中返回一个数据库连接
    @Override
    public Connection getConnection() {
        if(pool.size() > 0) {
            //从池中获取数据库连接
            Connection con = pool.remove(0);
            //通过自定义连接对象进行包装
            MyConnection mycon = new MyConnection(con,pool);
            //返回包装后的连接对象
            return mycon;
        }else {
            throw new RuntimeException("连接数量已用尽");
        }
    }
```

> 问题是：我只想重写close方法，却有其他大量的方法需要在类中重写

#### 适配器模式

> 定义一个适配器类（中间类）实现Connection接口，将其他所有方法进行实现 自定义类只需要继承这个适配器类，重写要改进的那个方法就行

##### 实现步骤

- 定义适配器类，实现 Connection 接口
	- 定义 Connection 连接对象的成员变量
	- 有参构造给成员变量赋值
	- 除了 close 之外的方法，都调用 conn 原本的方法完成
- 定义连接类，继承适配器
	- 定义 Connection 连接对象和连接池容器(List)
	  通过有参构造赋值
	- 重写 close() 方法，完成连接归还
	  ``` java
	  	  public class MyConnection2 extends MyAdapter {
	  	      //2.定义Connection连接对象和连接池容器对象的变量
	  	      private Connection con;
	  	      private List<Connection> pool;
	  	      //3.提供有参构造方法，接收连接对象和连接池对象，对变量赋值
	  	      public MyConnection2(Connection con,List<Connection> pool) {
	  	          super(con);    // 将接收的数据库连接对象给适配器父类传递
	  	          this.con = con;
	  	          this.pool = pool;
	  	      }
	  	      //4.在close()方法中，完成连接的归还
	  	      @Override
	  	      public void close() throws SQLException {
	  	          pool.add(con);
	  	      }
	  	  }
	  ```
- 在自定义连接池中，对连接对象用自定义连接类包装
	  ``` java
	  	  //从池中返回一个数据库连接
	  	  @Override
	  	  public Connection getConnection() {
	  	      if(pool.size() > 0) {
	  	          //从池中获取数据库连接
	  	          Connection con = pool.remove(0);
	  	  
	  	          //通过自定义连接对象进行包装
	  	          //MyConnection mycon = new MyConnection(con,pool);
	  	          MyConnection2 mycon = new MyConnection2(con,pool);
	  	  
	  	          //返回包装后的连接对象
	  	          return mycon;
	  	      }else {
	  	          throw new RuntimeException("连接数量已用尽");
	  	      }
	  	  }
	```

> 但依然还要自己写个适配器类，还是有点繁琐

### 动态代理模式

使用JDK动态代理的方式，重写 invoke 方法，在方法中对原有方法进行增强

```java
//动态代理方式
@Override
public Connection getConnection() {
    if(pool.size() > 0) {
        //从池中获取数据库连接
        Connection con = pool.remove(0);

        Connection proxyCon = (Connection)Proxy.newProxyInstance(con.getClass().getClassLoader(), new Class[]{Connection.class}, new InvocationHandler() {
            /*
                执行Connection实现类所有方法都会经过invoke
                如果是close方法，则将连接还回池中
                如果不是，直接执行实现类的原有方法
             */
            @Override
            public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
                if(method.getName().equals("close")) {
                    pool.add(con);
                    return null;
                }else {
                    return method.invoke(con,args);
                }
            }
        });

        return proxyCon;
    }else {
        throw new RuntimeException("连接数量已用尽");
    }
}
```

