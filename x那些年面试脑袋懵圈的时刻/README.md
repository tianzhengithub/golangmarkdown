### 一、Spring事务失效场景

#### 1.1 前言

身为Java开发工程师，相信大家对Spring种食物的使用并不陌生。但是你可能只停留在基础的使用层面上，在遇到一些比较特殊的场景，事务可能没有生效，直接在生产上暴露了，这可能就会导致比较严重的生产事故。今天，我们就简单来说一下Spring事务的原理，然后总结出对应的解决方案。

- 声明式事务是Spring 功能中最爽之一，可是有些时候，我们在使用声明式事务并为生效，这是为什么呢？
- 再次就聊聊声明式事务的几种失效场景。本文将会从以下两个方面来说一下事务为什么会失效？

#### 1.2 Spring 事务原理

大家还记得在JDBC中是如何操作事务的吗？伪代码可能如下：

```java
//Get database connection
Connection connection = DriverManager.getConnection();
//Set autoCommit is false
connection.setAutoCommit(false);
//use sql to operate database
.........
//Commit or rollback
connection.commit()/connection.rollback

connection.close();
```

需要再各个业务中编写代码如 commit() 、close() 来控制事务。

但是Spring 不乐意这么干了，这样对业务代码侵入性太大了，所有就用一个事务注解 @Transaction 来控制事务，底层实现是基于切面编程 AOP 实现的，而Spring 中实现 AOP 机制采用的动态代理，具体分为 JDK 动态代理和 CGLib 动态代理两种模式。



























