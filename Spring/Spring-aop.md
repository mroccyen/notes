### 切面(Aspect)

切面是一个横切关注点的模块化，一个切面能够包含同一个类型的不同增强方法，比如说事务处理和日志处理可以理解为两个切面。切面由切入点和通知组成，它既包含了横切逻辑的定义，也包括了切入点的定义。 Spring AOP就是负责实施切面的框架，它将切面所定义的横切逻辑织入到切面所指定的连接点中。


```java
@Component
@Aspect
public class LogAspect {
}
```

**可以简单地认为, 使用 @Aspect 注解的类就是切面**

### 目标对象(Target)

目标对象指将要被增强的对象，即包含主业务逻辑的类对象。或者说是被一个或者多个切面所通知的对象。

### 连接点(JoinPoint)

程序执行过程中明确的点，如方法的调用或特定的异常被抛出。连接点由两个信息确定：

- 方法(表示程序执行点，即在哪个目标方法)
- 相对点(表示方位，即目标方法的什么位置，比如调用前，后等)

简单来说，连接点就是被拦截到的程序执行点，因为Spring只支持方法类型的连接点，所以在Spring中连接点就是被拦截到的方法。

```java
@Before("pointcut()")
public void log(JoinPoint joinPoint) { //这个JoinPoint参数就是连接点
}
```

### 切入点(PointCut)

切入点是对连接点进行拦截的条件定义。切入点表达式如何和连接点匹配是AOP的核心，Spring缺省使用AspectJ切入点语法。 一般认为，所有的方法都可以认为是连接点，但是我们并不希望在所有的方法上都添加通知，而切入点的作用就是提供一组规则(使用 AspectJ pointcut expression language 来描述) 来匹配连接点，给满足规则的连接点添加通知。

```java
@Pointcut("execution(* com.remcarpediem.test.aop.service..*(..))")
public void pointcut() {
}
```

上边切入点的匹配规则是 `com.remcarpediem.test.aop.service`包下的所有类的所有函数。

### 通知(Advice)

通知是指拦截到连接点之后要执行的代码，包括了“around”、“before”和“after”等不同类型的通知。Spring AOP框架以拦截器来实现通知模型，并维护一个以连接点为中心的拦截器链。

```java
// @Before说明这是一个前置通知，log函数中是要前置执行的代码，JoinPoint是连接点，
@Before("pointcut()")
public void log(JoinPoint joinPoint) { 
}
```

### 织入(Weaving)

织入是将切面和业务逻辑对象连接起来, 并创建通知代理的过程。织入可以在编译时，类加载时和运行时完成。在编译时进行织入就是静态代理，而在运行时进行织入则是动态代理。

### 增强器(Adviser)

Advisor是切面的另外一种实现，能够将通知以更为复杂的方式织入到目标对象中，是将通知包装为更复杂切面的装配器。Advisor由切入点和Advice组成。 Advisor这个概念来自于Spring对AOP的支撑，在AspectJ中是没有等价的概念的。Advisor就像是一个小的自包含的切面，这个切面只有一个通知。切面自身通过一个Bean表示，并且必须实现一个默认接口。

```java
// AbstractPointcutAdvisor是默认接口
public class LogAdvisor extends AbstractPointcutAdvisor {
 private Advice advice; // Advice
 private Pointcut pointcut; // 切入点

 @PostConstruct
 public void init() {
 // AnnotationMatchingPointcut是依据修饰类和方法的注解进行拦截的切入点。
 this.pointcut = new AnnotationMatchingPointcut((Class) null, Log.class);
 // 通知
 this.advice = new LogMethodInterceptor();
 }
}
```

### 深入理解

看完了上面的理论部分知识, 我相信还是会有不少朋友感觉AOP 的概念还是很模糊, 对 AOP 的术语理解的还不是很透彻。现在我们就找一个具体的案例来说明一下。 简单来讲，整个 aspect 可以描述为: **满足 pointcut 规则的 joinpoint 会被添加相应的 advice 操作**。我们来看下边这个例子。

```java
@Component
@Aspect // 切面
public class LogAspect {
 private final static Logger LOGGER = LoggerFactory.getLogger(LogAspect.class.getName());
 // 切入点，表达式是指com.remcarpediem.test.aop.service
 // 包下的所有类的所有方法
 @Pointcut("execution(* com.remcarpediem.test.aop.service..*(..))")
 public void aspect() {}
 // 通知，在符合aspect切入点的方法前插入如下代码，并且将连接点作为参数传递
 @Before("aspect()")
 public void log(JoinPoint joinPoint) { //连接点作为参数传入
 if (LOGGER.isInfoEnabled()) {
 // 获得类名，方法名，参数和参数名称。
 Signature signature = joinPoint.getSignature();
 String className = joinPoint.getTarget().getClass().getName();
 String methodName = joinPoint.getSignature().getName();
 Object[] arguments = joinPoint.getArgs();
 MethodSignature methodSignature = (MethodSignature) joinPoint.getSignature();

 String[] argumentNames = methodSignature.getParameterNames();

 StringBuilder sb = new StringBuilder(className + "." + methodName + "(");

 for (int i = 0; i< arguments.length; i++) {
                Object argument = arguments[i];
                sb.append(argumentNames[i] + "->");
                sb.append(argument != null ? argument.toString() : "null ");
 }
 sb.append(")");
 LOGGER.info(sb.toString());
 }
 }
}
```