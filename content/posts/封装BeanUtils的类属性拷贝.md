---
disableToC: true
title: "封装BeanUtils的属性拷贝工具类"
date: 2021-02-21T11:31:23+08:00
draft: false
featured_image: "封面"
tags: [Spring]
categories: Java
description: Spring 自带的 BeanUtils 工具类封装
---

BeanUtils 的 copyProperties 方法常在领域模型 pojo 和 VO BO 的转换中

为了便于使用和简化重复代码，封装一个工具类实现：

- 复制单个对象时，自动新建对象返回
- 对象集合的复制

```java
public class CopyUtil{	
		/**
     * 对象复制
     */
    public static <T> T copy(Object source, Class<T> clazz) {
        if (source == null) {
            return null;
        }
        T obj = null;
        try {
            obj = clazz.newInstance();
        } catch (Exception e) {
            e.printStackTrace();
            return null;
        }
        BeanUtils.copyProperties(source, obj);
        return obj;
    }

    /**
     * 对象集合复制
     */
    public static <T> List<T> copyList(List source, Class<T> clazz) {
        List<T> target = new ArrayList<>();
        if (!CollectionUtils.isEmpty(source)){
            for (Object c: source) {
                T obj = copy(c, clazz);
                target.add(obj);
            }
        }
        return target;
    }
}
```

