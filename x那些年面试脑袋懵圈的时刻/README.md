### 一、Spring事务失效场景

#### 1.1 前言

身为Java开发工程师，相信大家对Spring种事务的使用并不陌生。但是你可能只停留在基础的使用层面上，在遇到一些比较特殊的场景，事务可能没有生效，直接在生产上暴露了，这可能就会导致比较严重的生产事故。今天，我们就简单来说一下Spring事务的原理，然后总结出对应的解决方案。

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

但是 Spring 不乐意这么干了，这样对业务代码侵入性太大了，所有就用一个事务注解 @Transaction 来控制事务，底层实现是基于切面编程 AOP 实现的，而Spring 中实现 AOP 机制采用的动态代理，具体分为 JDK 动态代理和 CGLib 动态代理两种模式。

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

上面的示例中，事务将失败。原因是Spring 的事务切面 优先级最低，所以如果异常被切面捕获，Spring自然不能正常处理事务，因为事务管理器无法捕获异常。

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

Spring 事务的传播机制是指在多个事务方法互相调用时，确定事务应该如何传播的策略。Spring 提供了 7  种事务传播机制：

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

在上面的例子中，如果用户插入失败，不会导致 saveAddress() 回滚，因为这里使用的传播是 `REQUIRES_NEW`，传播机制 `REQUIRES_NEW` 的原理是如果当前方法中没有事务，就会创建一个新的事物。如果一个事物已经存在，则当前事务将被挂起，并创建一个新事务。在当前事务完成之前，不会提交父事务。如果父事务发生异常，则不影响子事务的提交。

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



### 二、JDK动态代理和CGLIB动态代理

#### 2.1 什么是代理模式

代理模式（Proxy Pattern）给某一个对象提供一个代理，并有代理对象控制原对象的引用。代理对象再客户端和目标对象之间起到中介作用。

代理模式是常用的结构型设计模式之一，当直接访问某些对象存在问题时可以通过一个代理对象来间接访问，为了保证客户端使用的透明性，所访问的真实对象与代理对象需要实现相同的接口。代理模式属于结构型设计模式，属于GOF23设计模式

代理模式可以分为静态代理和动态代理两种类型，而动态代理中又分为 JDK 动态代理和 CGLIB 代理两种。

![2](image/2.png)

**代理模式包含如下角色**：

1. **Subject**（抽象主题角色）抽象主题角色生命了真实主题和代理主题的共同接口，这样依赖在任何使用真实主题的地方都可以使用代理主题。客户端需要针对抽象主题角色进行编程。
2. **Proxy**（代理主题角色）代理主题角色内部包含对真实主题的引用，从而可以在任何时候操作真是主题对象。在代理主题角色中提供一个与真实主题角色相同的接口，以便在任何时候都可以代替真实主体。代理主题角色还可以控制对真实主题的使用，负责在需要的时候创建和删除真是主题对象，并对真实主题对象的使用加以约束。代理角色通常在客户端调用所引用的真实主题操作之前或之后还需要执行其他操作，而不仅仅是单纯的调用真是主题对象中的操作。
3. **RealSubject**（真是主题角色）真是主题角色定义了代理角色所代表的真实对象，在真实主题角色中实现了真实的业务操作，客户端可以通过代理主题角色间接调用真是主题角色重定义的方法。

#### 2.2 代理模式的优点

- 代理模式能将代理对象与真实被调用的目标对象分离。
- 一定程度上降低了系统柜的耦合度，扩展性好。
- 可以起到保护目标对象的作用。
- 可以对目标对象的功能增强

#### 2.3 代理模式的缺点

- 代理模式会造成系统设计中类的数量增加。
- 在客户端和目标对象增加一个代理对象，会造成请求处理速度变慢。

#### 2.4 JDK动态代理

在java的动态代理机制中，有两个重要的类或接口，一个是 InvocationHandler(Interface)、另外一个则是Prox（Class），这个类和接口是实现我们动态代理所必须用到的。

##### 2.4.1 **InvocationHandler**

每一个动态代理类都必须要实现 InvocationHandler 这个接口，并且每个代理类的实例都关联了一个handler，当我们通过代理对象调用一个方法的时候，这个方法的调用就会被转发为由 InvocationHandler 这个接口的 invoke 方法来进行调用。

InvocationHandler 这个接口的唯一一个方法 invoke 方法：

```java
Object invoke(Object proxy, Method method, Object[] args) throws Throwable
```

这个方法一共接受三个参数，那么这三个参数分别代表如下：

- **proxy** ：指代 JDK 动态生成的最终代理对象。
- **method** ：指代的是我们所要调用真是对象的某个方法的 Method 对象。
- **args** ： 指代的是调用真实对象某个方法时接受的参数。

##### 2.4.2 Proxy

Proxy 这个类的作用就是来动态创建一个代理对象的类，它提供了许多的方法，但是我们用的最多的就是 newProxyInstance 这个方法：

```java
public static Object newProxyInstance(ClassLoader loader, Class<?>[] interfaces,  InvocationHandler handler)  throws IllegalArgumentException
```

这个方法的作用就是得到一个动态的代理对象，其接收三个参数，我们来看看这三个参数所代表的含义：

- **loader** ：ClassLoader对象，定义了由那些 ClassLoader 来对生成的代理对象进行加载，即代理类的类加载器。
- **interfaces** ： Interface 对象的数组，表示的是我将要给我需要代理的对象提供一组什么接口，如果我提供了一组接口给它，那么这个代理对象就宣称实现了该接口（多态），这样我就能调用这组接口中的方法了。
- **Handler** ：InvocationHandler 对象，表示的是当我这个动态代理对象在调用方法的时候，会关联到哪一个 InvocationHandler 对象上。

所以我们所说的 DynamicProxy （动态代理类）是这样一种class ： 它是在运行时生成的 class ，在生成它时你必须提供一组 interface 给它，然后改 class 就宣称它实现了这些 interface。这个 DynamicProxy 其实就是一个 Proxy，它不会做实质性的工作，在生成它的实例时你必须提供一个 handler，由它接管实际的工作。

##### 2.4.3 JDK 动态代理实例

1. 创建接口类

```java
public interface HelloInterface {
    void sayHello();
}
```

2. 创建被代理类，实现接口

```java
/**
 * 被代理类
 */
public class HelloImpl implements HelloInterface{
    @Override
    public void sayHello() {
        System.out.println("hello");
    }
}
```

3. 创建InvocationHandler实现类

```java
/**
 * 每次生成动态代理类对象时都需要指定一个实现了InvocationHandler接口的调用处理器对象
 */
public class ProxyHandler implements InvocationHandler{
    private Object subject; // 这个就是我们要代理的真实对象，也就是真正执行业务逻辑的类
    public ProxyHandler(Object subject) {// 通过构造方法传入这个被代理对象
        this.subject = subject;
    }
    /**
     *当代理对象调用真实对象的方法时，其会自动的跳转到代理对象关联的handler对象的invoke方法来进行调用
     */
    @Override
    public Object invoke(Object obj, Method method, Object[] objs)
            throws Throwable {
        Object result = null;
        System.out.println("可以在调用实际方法前做一些事情");
        System.out.println("当前调用的方法是" + method.getName());
        result = method.invoke(subject, objs);// 需要指定被代理对象和传入参数
        System.out.println(method.getName() + "方法的返回值是" + result);
        System.out.println("可以在调用实际方法后做一些事情");
        System.out.println("------------------------");
        return result;// 返回method方法执行后的返回值
    }
}
```

4. 测试

```java
public class Mytest {
    public static void main(String[] args) {
        //第一步：创建被代理对象
        HelloImpl hello = new HelloImpl();
        //第二步：创建handler,传入真实对象
        ProxyHandler handler = new ProxyHandler(hello);
        //第三步：创建代理对象，传入类加载器、接口、handler
        HelloInterface helloProxy = (HelloInterface) Proxy.newProxyInstance(
                HelloInterface.class.getClassLoader(), 
                new Class[]{HelloInterface.class}, handler);
        //第四步：调用方法
        helloProxy.sayHello();
    }
}
```

5. 结果

```java
可以在调用实际方法前做一些事情
当前调用的方法是sayHello
hello
sayHello方法的返回值是null
可以在调用实际方法后做一些事情
------------------------
```

##### 2.4.4 JDK 动态代理步骤

JDK 动态代理分为以下几步：

1. 拿到被代理对象的引用，并且通过反射获取到它的所有的接口。
2. 通过 JDK Proxy 类重新生成一个新的类，同时新的类要实现被代理类所实现的所有的接口。
3. 动态生成 Java 代码，把新加的业务逻辑方法由一定的逻辑代码去调用。
4. 编译新生成的 Java 代码 .class。
5. 将新生成的Class文件重新加载到JVM中运行。

所以说 JDK 动态代理的核心是通过重写被代理对象所实现的接口中的方法重新生成代理类来实现的，那么加入被代理对象没有实现接口呢？那么这时候就需要 CGLIB 动态代理了。

#### 2.5 CGLIB 动态代理

JDK 动态代理是通过重写被代理对象实现的接口中的方法来实现，而CGLIB是通过集成部诶代理对象来实现和 JDK 动态代理需要实现指定接口一样，CGLIB 也要求代理对象必须要实现 MethodInterceptor 接口，并重写其唯一的方法 intercept 。

CGLib 采用了非常底层的字节码技术，其原理是通过字节码技术为一个类创建子类，并在子类中采用方法拦截的技术拦截所有父类方法的调用，顺势植入横切逻辑。（利用ASM开源包，对代理对象类的 class 文件加载进来，通过修改其字节码生成子类来处理）。

**注意** ：因为CGLIB 是童工集成目标类来重写其方法来实现的，故而如果是 final  和 private  方法则无法被重写，也就无法被代理。

```xml
<dependency>
  <groupId>cglib</groupId>
	<artifactId>cglib-nodep</artifactId>
	<version>2.2</version>
</dependency>
```

##### 2.5.1 CGLIB 核心类

1. **net.sf.cglib.proxy.Enhancer** ：主要增强类，通过字节码技术动态创建委托类的子类实例；

   Enhancer 可能是 CGLIB 中最常用的一个类，和Java1.3 动态代理中引入的Proxy类差不多。和Proxy不同的是，Enhancer 技能够代理普通的 class，也能够代理接口。Enhancer 创建一个被代理对象的子类并且拦截所有的方法调用（包括从 Object 中继承的 toString 和 hashCode 方法）。Enhancer 不能够拦截 final 方法，例如 Object.getClass() 方法，这是由于 Java final  方法语义决定的。基于同样的道理，Enhancer 也不能对 final 类进行 代理操作。这也是 Hibernate 为什么不能持久化 final class 的原因。

2. **net.sf.cglib.proxy.MethodInterceptor** ：常用的方法拦截器接口，需要实现 intercept 方法，实现具体拦截处理：

```java
public java.lang.Object intercept(java.lang.Object obj,
                                   java.lang.reflect.Method method,
                                   java.lang.Object[] args,
                                   MethodProxy proxy) throws java.lang.Throwable{}

```

- obj ： 动态生成的代理对象。
- method ： 实际调用的方法。
- args ： 调用方法入参。

- **net.sf.cglib.proxy.MethodProxy** ：java Method类的代理类，可以实现委托类对象的方法的调用；常用方法：methodProxy.invokeSuper(proxy, args)；在拦截方法内可以调用多次。

##### 2.5.2 CGLIB 代理实例

1. 创建被代理类

```java
public class SayHello {
  public void say() {
    System.out.println("hello")
  }
}
```

2. 创建代理类

```java
/**
 *代理类 
 */
public class ProxyCglib implements MethodInterceptor{
     private Enhancer enhancer = new Enhancer();  
     public Object getProxy(Class clazz){  
          //设置需要创建子类的类  
          enhancer.setSuperclass(clazz);  
          enhancer.setCallback(this);  
          //通过字节码技术动态创建子类实例  
          return enhancer.create();  
     }  

     //实现MethodInterceptor接口方法  
     public Object intercept(Object obj, Method method, Object[] args, MethodProxy proxy) throws Throwable {  
          System.out.println("可以在调用实际方法前做一些事情");  
          //通过代理类调用父类中的方法  
          Object result = proxy.invokeSuper(obj, args);  
          System.out.println("可以在调用实际方法后做一些事情");  
          return result;  
     } 
}
```

3. **测试**

```java
public class Mytest {

    public static void main(String[] args) {
          ProxyCglib proxy = new ProxyCglib();  
          //通过生成子类的方式创建代理类  
          SayHello proxyImp = (SayHello)proxy.getProxy(SayHello.class);  
          proxyImp.say();  
    }
}
```

4. 结果

```java
可以在调用实际方法前做一些事情
hello
可以在调用实际方法后做一些事情
```

##### 2.5.3 CGLIB 动态代理实现分析

CGLIB 动态代理采用了 FastClass 机制，其分别为代理类和被代理类各生成一个FastClass，这个 FastClass 类会为代理类或被代理类的方法分配一个 index（int类型）。这个index 当作一个入参，FastClass 就可以直接定位要调用的方法直接进行调用，这样省去了反射调用，所以调用效率比 JDK 动态代理通过反射调用更高。

但是我们看上面的源码也可以明显看到，JDK 动态代理只生成一个文件，而 CGLIB 生成了三个文件，所以生成代理对象的过程会更复杂。

#### 2.6 JDK 和 CGLIB 动态代理对比

JDK 动态代理是实现了呗代理对象啊所实现的接口，CGLIB 是继承了被代理对象。JDK 和 CGLIB 都是在运行期生成字节码，JDK 是直接写 Class 字节码，CGLib 使用 ASM 框架写 Class 字节码，Cglib 代理实现更为复杂，生成代理类的效率比 JDK 代理低。

JDK 调用代理方法，是通过反射机制调用，CGLIB 是通过 FastClass 机制直接调用方法，CGLIB 执行效率更高。

##### 2.6.1 原理区别：

java 动态代理是利用反射机制生成一个实现代理接口的匿名类，在调用具体方法前调用 InvocationHandler 来处理。核心是实现 InvocationHandler 接口，使用 invoke() 方法进行面向切面的处理，调用相应的通知。

1. 如果目标对象实现了接口，默认情况下会采用 JDK 的动态代理实现 AOP。
2. 如果目标对象实现了接口，可以强制使用 CGLIB 实现 AOP。
3. 如果目标对象没有实现了接口，必须采用 CGLIB  库，spring 会自动在 JDK 动态代理和 CGLIB 之间转换

##### 2.6.2 性能区别：

1. CGLib底层采用ASM字节码生成框架，使用字节码技术生成代理类，在jdk6之前比使用Java反射效率要高。唯一需要注意的是，CGLib不能对声明为final的方法进行代理，因为CGLib原理是动态生成被代理类的子类。

2. 在jdk6、jdk7、jdk8逐步对JDK动态代理优化之后，在调用次数较少的情况下，JDK代理效率高于CGLIB代理效率，只有当进行大量调用的时候，jdk6和jdk7比CGLIB代理效率低一点，但是到jdk8的时候，jdk代理效率高于CGLIB代理。

##### 2.6.3 各自局限

1. JDK的动态代理机制只能代理实现了接口的类，而不能实现接口的类就不能实现JDK的动态代理。

2. cglib是针对类来实现代理的，他的原理是对指定的目标类生成一个子类，并覆盖其中方法实现增强，但因为采用的是继承，所以不能对final修饰的类进行代理。

|     类型      |                             机制                             | 回调方式                  | 适用场景                         | 效率                                                         |
| :-----------: | :----------------------------------------------------------: | ------------------------- | -------------------------------- | ------------------------------------------------------------ |
|  JDK动态代理  | 委托机制，代理类和目标类都实现了同样的接口，InvocationHandler持有目标类，代理类委托InvocationHandler去调用目标类的原始方法 | 反射                      | 目标类是接口类                   | 效率瓶颈在反射调用稍慢                                       |
| CGLIB动态代理 | 继承机制，代理类继承了目标类并重写了目标方法，通过回调函数MethodInterceptor调用父类方法执行原始逻辑 | 通过FastClass方法索引调用 | 非接口类、非final类，非final方法 | 第一次调用因为要生成多个Class对象，比JDK方式慢。多次调用因为有方法索引比反射快，如果方法过多，switch case过多其效率还需测试 |

#### 2.7 静态代理和动态代理的区别

静态代理只能通过手动完成代理操作，如果被代理类增加新的方法，代理类需要同步新增，违背开闭原则。

动态代理采用在运行时动态生成代码的方式，取消了对被代理类的扩展限制，遵循开闭原则。

若动态代理要对目标类的增强逻辑扩展，结合策略模式，只需要新增策略类便可完成，无需修改代理类的代码。