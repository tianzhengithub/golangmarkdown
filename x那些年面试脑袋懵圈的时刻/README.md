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

![1](image/1.png)

1. Spring 的 bean 的初始化过程中，发现方法有 @Transaction 注解，就需要对相应的 Bean 进行代理，生成代理对象。
2. 然后再方法调用的时候，会执行切面的逻辑，而这里切卖你的逻辑中就包含了开启事务，提交事务或者回滚事务等逻辑。

另外注意一点的是，Spring 本身不实现事务，底层还是依赖于数据库的事务。没有数据库事务的支持，Spring 事务是不会生效的。

#### 1.3 Spring 事务失效场景

##### 1.3.1 抛出检查异常

比如你的事务控制代码如下：

```java
@Transactional
public void transactionTest() throws IOException{
    User user = new User();
    UserService.insert(user);
    throw new IOException();
}
```

如果 @Transactional 没有特别指定，Spring只会在遇到运行时异常RuntimeException或者error时进行回滚，而 IOException 等检查异常不会影响回滚。

```java
public boolean rollbackOn(Throwable ex) {
  	return (ex instanceof RuntimeException || ex instanceof  Error);
}
```

**解决方案：**

知道原因后，解决方法也很简单。配置rollbackFor 属性，例如：

```java
 @Transactional（rollbackFor = Exception.class)
```

##### 1.3.2 业务方法本身捕获了异常

```java
@Transactional(rollbackFor = Exception.class)
public void transactionTest() {
    try {
        User user = new User();
        UserService.insert(user);
        int i = 1 / 0;
    }catch (Exception e) {
        e.printStackTrace();
    }
}
```

这种场景下，事务失败的原因也很简单，Spring 是否进行回滚时根据你是否抛出异常决定的，所以如果你自己捕获了异常，Spring也无能为力。

看了上面的代码，你可能认为这么简单的问题你不可能犯这么愚蠢的错误，但是我想告诉你的是，我身边几乎一半的人都被这一幕困扰过。

写业务代码的时候，代码可能比较复杂，嵌套的方法很多。如果你不小心，很可能会触发此问题。举一个非常简单的例子，假设你有一个审计功能。每个方法执行后，审计结果保存在数据库中，那么代码可能会这样写。

```java
@Service
public class TransactionService {

    @Transactional(rollbackFor = Exception.class)
    public void transactionTest() throws IOException {
        User user = new User();
        UserService.insert(user);
        throw new IOException();

    }
}

@Component
public class AuditAspect {

	@Autowired
	private auditService auditService;

    @Around(value = "execution (* com.alvin.*.*(..))")
    public Object around(ProceedingJoinPoint pjp) {
        try {
            Audit audit = new Audit();
            Signature signature = pjp.getSignature();
            MethodSignature methodSignature = (MethodSignature) signature;
            String[] strings = methodSignature.getParameterNames();
            audit.setMethod(signature.getName());
            audit.setParameters(strings);
            Object proceed = pjp.proceed();
            audit.success(true);
            return proceed;
        } catch (Throwable e) {
            log.error("{}", e);
            audit.success(false);
        }
        
        auditService.save(audit);
        return null;
    }

}
```

上面的示例中，事务将失败。原因是Spring 的事务切卖你优先级最低，所以如果异常倍切面捕获，Spring自然不能正常处理事务，因为事务管理器无法捕获异常。

**解决方案**：

虽然我们知道在处理使唔使业务代码不能自己捕获异常，但是只要代码变得复杂，我们就很可能再次出错，所以我们在处理事务的时候要小心，还是不要使用声明式事务，并使用编程式事务：

```java
transactionTemplate.execute()
```

##### 1.3.3 同一类的方法调用

```java
@Service
public class DefaultTransactionService implement Service {

    public void saveUser() throws Exception {
        //do something
        doInsert();
    }

    @Transactional(rollbackFor = Exception.class)
    public void doInsert() throws IOException {
        User user = new User();
        UserService.insert(user);
        throw new IOException();

    }
}
```

这也是一个容易出错的场景。事务失败的原因也很简单，因为Spring的事务管理功能是通过动态代理实现的，而Spring默认使用 JDK 动态代理，而 JDK 动态代理采用接口实现的方式，通过反射调用目标类。简单理解，就是 saveUser()  方法中调用 this.doInsert() ，这里的this 是被真实对象，所以会直接走 doInsert 的业务逻辑，而不走切面逻辑，所以事务失败。

**解决方案**：

```java
方案一：解决方法可以是直接在启动类中添加 @Transcational 注解 saveUser() 
方案二：@EnableAspectJAutoProxy(exposeProxy = true)在启动类中添加，会由Cglib代理实现。
```

##### 1.3.4 方法使用 final 或 static 关键字

如果Spring 使用了 Cglib 代理实现 （比如你的代理类没有实现接口），而你的业务方法敲好使用了 final  或者 static 关键字，那么事务也会失败。更具体的说，它应该抛出异常，因为 Cglib 使用字节码增强技术生成被代理类的子类并重写代理类的方法来实现代理。如果被代理的方法使用 final 或 static 关键字，则子类不能重写被代理的方法。



如果 Spring 使用 JDK 动态代理实现，JDK动态代理是基于接口实现的，那么 final 和 static 修饰的方法也就无法被代理。

总而言之，方法连代理都没有，那么肯定无法实现事务回滚了。

**解决方案**：

```java
想办法去掉 final 或者 static 关键字
```

##### 1.3.5 方法不是 public

如果方法不是 public，Spring 事务也会失败，因为 spring  的事务管理源码 `AbstractFallbackTransactionAttributeSource`中有判断`computeTransactionAttribute()。`如果目标方法不是公共的，则`TransactionAttribute`返回`null`。

```java
// Don't allow no-public methods as required.
if (allowPublicMethodsOnly() && !Modifier.isPublic(method.getModifiers())) {
  return null;
}
```

**解决方案：**

```java
将当前方法访问级别更改为 public 
```

##### 1.3.6 错误使用传播机制

Spring 事务的传播机制是指在多个事务方法互相调用时，确定事务应该如何传播的策略。Spring 提供了 7  种食物传播机制：

1. REQUIRED
2. SUPPORT
3. MANDATORY
4. REQUIRES_NEW
5. NOT_SUPPORTED
6. NEVER
7. NESTED

如果不知道这些传播策略的原理，很可能会导致交易失败。

```java
@Service
public class TransactionService {


    @Autowired
    private UserMapper userMapper;

    @Autowired
    private AddressMapper addressMapper;


    @Transactional(propagation = Propagation.REQUIRES_NEW,rollbackFor = Exception.class)
    public  void doInsert(User user,Address address) throws Exception {
        //do something
        userMapper.insert(user);
        saveAddress(address);
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public  void saveAddress(Address address) {
        //do something
        addressMapper.insert(address);
    }
}
```

在上面的例子中，如果用户插入失败，不会导致 saveAddress() 回滚，因为这里使用的传播是 `REQUIRES_NEW`，传播机制 `REQUIRES_NEW` 的原理是如果当前方法中没有事务，就会创建一个新的事物。如果一个事物已经存在，则当前事务将被挂起，并创建一个新事物。在当前事务完成之前，不会提交父事务。如果父事务发生异常，则不影响子事务的提交。

事务的传播机制说明如下：

- REQUIRED：如果当前上下文中存在事务，那么加入该事务，如果不存在事务，创建一个事务，这是默认的传播属性值。
- SUPPORT：如果当前上下文存在事务，则支持事务加入事务，如果不存在事务，则使用非事务的方式执行。
- MANDATORY：如果当前上下文中存在事务，并且同时将上下文中的事务挂起，执行当前新建事务完成以后，上下文事务回复在执行。
- NOT_SUPPORTED  如果当前上下文存在事务，则挂起当前事务，然后新的方法在没有事务的环境中执行。
- NEVER 如果当前上下文中存在书屋，则抛出异常，否则在无事务环境上执行代码。
- NESTED 如果当前上下文中存在是我，则嵌套是我执行，如果不存在事务，则新建事务。

**解决方案**：

```tex
将事务传播策略更改为默认值 REQUIRED, REQUIRED 原理是如果当前有一个事务被添加到一个事务中，如果没有，则创建一个新事物，父事务和被调用的事务在同一个事务中。即时被调用的异常被捕获，整个事务仍然会被回滚。
```

##### **1.3.7 没有被Spring管理**

```java
// @Service
public class OrderServiceImpl implements OrderService {
    @Transactional
    public void updateOrder(Order order) {
        // update order
    }
}
```

如果此时把 @Server 注解注释掉，这个类就不会被加载成一个 Bean ，那这个类就不会被 Spring 管理了，事务自然就失效了。

**解决方案**：

```
需要保证每个事物注解的每个Bean被Spring管理。
```

##### 1.3.8 多线程

```java
@Service
public class UserService {

    @Autowired
    private UserMapper userMapper;
    @Autowired
    private RoleService roleService;

    @Transactional
    public void add(UserModel userModel) throws Exception {

        userMapper.insertUser(userModel);
        new Thread(() -> {
             try {
                 test();
             } catch (Exception e) {
                roleService.doOtherThing();
             }
        }).start();
    }
}

@Service
public class RoleService {

    @Transactional
    public void doOtherThing() {
         try {
             int i = 1/0;
             System.out.println("保存role表数据");
         }catch (Exception e) {
            throw new RuntimeException();
        }
    }
}
```

我们可以看到实物方法add中，调用了实物方法 doOtherThing ，但是实物方法 doOtherThing 是在另外一个线程中被调用的。

这样会导致两个方法不再同一个线程中，获取到的数据库连接不一样，从而使两个不同的事务。如果想 doOtherThing 方法中抛出异常， add 方法也回滚时不可能的。

我们说的同一个事物，其实是指同一个数据库连接，只有拥有同一个数据库连接才能同时提交和回滚。如果在不同的线程，拿到的数据库连接肯定时不一样的，所以是不同的事务。

**解决方案**：

```tex
这里就有点分布式事务的感觉了，尽量还是保证在同一个事物中处理。
```

##### 1.3.9 总结

以上总结了 Spring 中事务实现的原理，同时列举了 8  重 Spring 事务失败的场景，相信很多人都遇到过，失败的原理也有详细说明。希望大家对 Spring 事务有一个新的认识。







