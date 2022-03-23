---
disableToC: true
date: 2021-01-02T18:31:23+08:00
featured_image: ""
tags: [缓存]
title: "Redis缓存设计案例"
categories: Redis
description: 实现简单的 Redis 缓存和回写操作
---

## 查询

> 查询先查 Reids
- 先从 Redis 里查询，如果有数据直接返回，没有再去查询 Mysql
- 从 MySQL 查询到数据后，需要将数据写回 Redis 以保障下一次缓存命中
	```java
	      User user = null;
        String key = CACHE_KEY_USER+id;
        //先从redis里面查询
        user = (User) redisTemplate.opsForValue().get(key);
        if(user == null)
        {
            // redis里面无，继续查询mysql
            user = userMapper.selectByPrimaryKey(id);
            if(user == null)
            {
                //都无数据,返回空数据
                return user;
            }else{
                //数据写回 redis
                redisTemplate.opsForValue().set(key,user);
            }
        }
        return user;
	```

---

实际生产环境中，通常 redis 的 key 都会设置过期时间，如果正好遇到过期时间到期，某个热点 key 突然失效，会导致缓存击穿问题

> 击穿：缓存先击中(一开始 Redis 中有数据），但热点 key 突然失效
>
> 在高并发下，出现大量的查询请求流向 MySQL （打穿）

### 改进

为了避免缓存击穿，要保证只有一个请求操作，其他请求等待

借鉴单例模式的双锁检测（ Double Check）思想

```java
       user = (User)redisTemplate.opsForValue().get(key);
				//第一重检测
        if(user == null)
        {
           //进来就加锁，保证一个请求操作
            synchronized (UserService.class){
                //加锁后再查一次 redis
                user = (User) redisTemplate.opsForValue().get(key);
                //第二重检测，如果二次 Redis 还是 null，就可以去查MySQL 了
                if (user == null) {
                    user = userMapper.selectByPrimaryKey(id);
                    if (user == null) { //如果还是 null ，刚才被人删了
                        return null;
                    }else{ //回写 redis 完成一致性的同步工作
                     /redisTemplate.opsForValue().setIfAbsent(key,user,7L,TimeUnit.DAYS);
                    }
                }
            }
        }
        return user;
```



## 增删改

> 修改先操作 MySQL

插入用户数据时做第二次判断

- 不仅判断返回的 int 行数 > 1，还要重新查出一次确保正确的插入

- 确定成功后再存进 redis，以保证数据一致性

  ```java
  int i = userMapper.insertSelective(user);
  
  if(i > 0)
  {
    //再次查询 mysql 数据捞回来
    user =userMapper.selectByPrimaryKey(user.getId());
    //保证数据一致性
    String key = CACHE_KEY_USER+user.getId();
    redisTemplate.opsForValue().set(key,user);
  }
  ```

- 删除：保证先操作数据库再操作 Redis
	- 生产环境下使用逻辑删除居多，方便演示直接使用物理删除
  ```java
  int i = userMapper.deleteByPrimaryKey(id);
  if(i > 0)
  {
    String key = CACHE_KEY_USER+id;
    redisTemplate.delete(key);
  }
  ```
- 修改：同样使用修改操作后查询出的数据修改 Redis
	```java
	  int i = userMapper.updateByPrimaryKeySelective(user);
    if(i > 0)
    {
      user = userMapper.selectByPrimaryKey(user.getId());
      String key = CACHE_KEY_USER+user.getId();
      redisTemplate.opsForValue().set(key,user);
    }
	```
