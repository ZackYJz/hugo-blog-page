---
disableToC: true
title: "利用 Spring AOP 打印接口日志"
date: 2021-03-03T12:43:23+08:00
draft: false
featured_image: "封面"
tags: [SpringBoot]
categories: Java
description: 前文介绍了过滤器和拦截器打印接口日志的方式
---

### AOP 使用

SpringBoot 项目中，只需要引入SpringBoot 集成的 AOP 依赖

```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-aop</artifactId>
</dependency>
```

包含了用于解析切点表达式，实现切面类和方法匹配的 AspectJ 中的 Weaver 组件

如果同时配置了拦截器和过滤器，执行顺序是：过滤器 ->拦截器 -> AOP

想要使用 AOP，步骤如下：

1. 明确 Target (被 AOP 增强的对象和方法)，这里指的就是 Controller 层方法
2. 编写 AOP 类，把公共的增强方法（日志打印）制作成通知方法，并传入切点 JoinPoint 作为参数
3. 定义切点 ，并在通知方法的注解属性中指明切点，声明切点和通知的对应关系 (切面)



### 打印接口日志

接口日志由前置方法打印

- 定义切点：

  ```java
   @Pointcut("execution(public * xyz.uuuvw.yoaswxapi.controller..*Controller.*(..))")
      public void controllerPointcut() {
      }
  ```

- 编写前置通知方法

  ```java
  @Before("controllerPointcut()")
  public void doBefore(JoinPoint joinPoint) throws Throwable {
  
    // 增加日志流水号
    //MDC.put("LOG_ID", String.valueOf(snowFlake.nextId()));
  
    // 开始打印请求日志
    ServletRequestAttributes attributes = (ServletRequestAttributes) RequestContextHolder.getRequestAttributes();
    HttpServletRequest request = attributes.getRequest();
    Signature signature = joinPoint.getSignature();
    String name = signature.getName();
  
    // 打印请求信息
    log.info("------------- 开始 -------------");
    log.info("请求地址: {} {}", request.getRequestURL().toString(), request.getMethod());
    log.info("类名方法: {}.{}", signature.getDeclaringTypeName(), name);
    log.info("远程地址: {}", request.getRemoteAddr());
  }
  ```

  - 这里在使用 nginx 作为反向代理时，无法获取到真实的远程 IP 地址，需要从 HttpServletRequest 的请求头中获得

    ```java
        /**
         * 使用nginx做反向代理时，获取到真实的远程IP
         *
         * @param request
         * @return
         */
        public String getRemoteIp(HttpServletRequest request) {
            String ip = request.getHeader("x-forwarded-for");
            if (ip == null || ip.length() == 0 || "unknown".equalsIgnoreCase(ip)) {
                ip = request.getHeader("Proxy-Client-IP");
            }
            if (ip == null || ip.length() == 0 || "unknown".equalsIgnoreCase(ip)) {
                ip = request.getHeader("WL-Proxy-Client-IP");
            }
            if (ip == null || ip.length() == 0 || "unknown".equalsIgnoreCase(ip)) {
                ip = request.getRemoteAddr();
            }
            return ip;
        }
    ```

- 打印请求参数，并排除敏感字段

  ```java
   // 打印请求参数
  Object[] args = joinPoint.getArgs();
  //log.info("请求参数: {}", JSONObject.toJSONString(args));
  
  Object[] arguments = new Object[args.length];
  for (int i = 0; i < args.length; i++) {
    if (args[i] instanceof ServletRequest
        || args[i] instanceof ServletResponse
        || args[i] instanceof MultipartFile) {
      continue;
    }
    arguments[i] = args[i];
  }
  // 排除字段，敏感字段或太长的字段不显示
  String[] excludeProperties = {"password", "file", "resource", "photo"};
  PropertyPreFilters filters = new PropertyPreFilters();
  PropertyPreFilters.MySimplePropertyPreFilter excludeFilter = filters.addFilter();
  excludeFilter.addExcludes(excludeProperties);
  log.info("请求参数: {}", JSONObject.toJSONString(arguments, excludeFilter));
  ```

  

##  打印接口耗时

接口耗时使用 环绕通知实现，并在返回的结果中排除敏感字段

```java
    @Around("controllerPointcut()")
    public Object doAround(ProceedingJoinPoint proceedingJoinPoint) throws Throwable {
        long startTime = System.currentTimeMillis();
        Object result = proceedingJoinPoint.proceed();
        // 排除字段，敏感字段或太长的字段不显示
        String[] excludeProperties = {"password", "file"};
        PropertyPreFilters filters = new PropertyPreFilters();
        PropertyPreFilters.MySimplePropertyPreFilter excludefilter = filters.addFilter();
        excludefilter.addExcludes(excludeProperties);
        log.info("返回结果: {}", JSONObject.toJSONString(result, excludefilter));
        log.info("-------- 结束 耗时：{} ms ---------", System.currentTimeMillis() - startTime);
        return result;
    }
```

