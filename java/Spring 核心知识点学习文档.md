# Spring 核心知识点学习文档

Oct 15, 2026 · @meteorzhong

## 导读

本文展开《知识图谱.html》「Web 与框架」分类中的 5 个节点，并对齐《Java开发工程师核心知识点.md》第五章。标注沿用图谱：**【核心】** 面试必问；**【重点】** 常被追问；**【了解】** 知道概念即可。默认版本为 Spring 6 / Spring Boot 3（要求 JDK 17，`javax.*` 迁移到 `jakarta.*`）。Spring Cloud 相关内容放在后续的分布式文档中。

| 图谱节点 | 级别 | 本文章节 |
| --- | --- | --- |
| Spring IOC（spring-ioc） | 核心 | IOC 容器、Bean 生命周期与循环依赖、设计模式 |
| Spring AOP（spring-aop） | 核心 | AOP、事务 |
| Spring Boot（spring-boot） | 核心 | Spring Boot |
| MyBatis / MyBatis-Plus（mybatis） | 核心 | MyBatis |
| Spring MVC（spring-mvc） | 重点 | Spring MVC |

**复习顺序**：IOC → Bean 生命周期 → 循环依赖 → AOP → 事务 → MVC → Boot → MyBatis。一条主线串起来：容器创建 Bean，在生命周期的后置处理阶段生成 AOP 代理，事务就是一种 AOP，Boot 的自动配置又往容器里注册了大量 Bean。面试最常见的三道题是：Bean 生命周期、三级缓存、事务失效。

## 一、IOC 容器

IOC（控制反转）是把对象的创建和依赖关系交给容器管理的思想，DI（依赖注入）是它的实现方式。好处是解耦：类只声明需要什么，不关心对方如何创建，便于替换实现和单元测试。

### 1.1 BeanFactory 与 ApplicationContext【核心】

| 维度 | BeanFactory | ApplicationContext |
| --- | --- | --- |
| 定位 | 最底层的容器接口，提供 getBean 等基本能力 | BeanFactory 的子接口，面向应用的高级容器 |
| Bean 加载 | 延迟加载，第一次 getBean 才创建 | 启动时创建所有非懒加载的单例 Bean，配置错误能尽早暴露 |
| 额外能力 | 无 | 国际化、事件发布、资源加载、自动注册 BeanPostProcessor 等 |

**FactoryBean 与 BeanFactory【重点】**：BeanFactory 是容器；FactoryBean 是一个特殊的 Bean，用来生产复杂对象，`getBean("xxx")` 拿到的是它 `getObject()` 的返回值，`getBean("&xxx")` 才拿到 FactoryBean 本身。MyBatis 的 `MapperFactoryBean`、`SqlSessionFactoryBean` 就是典型。

### 1.2 依赖注入方式【核心】

| 方式 | 特点 |
| --- | --- |
| 构造器注入 | **Spring 官方推荐**：依赖可用 final、保证对象创建后完整可用、便于测试；参数太多说明类职责过多；无法解决循环依赖 |
| Setter 注入 | 适合可选依赖 |
| 字段注入 `@Autowired` | 写法最简单，但依赖隐藏、脱离容器无法测试，IDEA 会提示不推荐 |

**@Autowired 与 @Resource【核心】**：

| 维度 | @Autowired | @Resource |
| --- | --- | --- |
| 来源 | Spring | JSR-250（Jakarta） |
| 匹配顺序 | 先按**类型**，多个同类型时再按名称，可配合 `@Qualifier` / `@Primary` | 先按**名称**，找不到再按类型 |
| 支持位置 | 构造器、方法、字段、参数 | 字段、setter |
| 可选 | `required = false` | 不支持 |

### 1.3 容器启动流程 refresh()【重点】

`AbstractApplicationContext.refresh()` 是容器启动的核心，记住关键的几步即可：

1. `obtainFreshBeanFactory()`：创建 BeanFactory，加载所有 **BeanDefinition**（Bean 的「图纸」：类名、作用域、依赖、是否懒加载等）。
2. `invokeBeanFactoryPostProcessors()`：执行 **BeanFactoryPostProcessor**，可以在 Bean 实例化前修改 BeanDefinition。`@Configuration`、`@ComponentScan`、`@Import` 的解析也在这里（`ConfigurationClassPostProcessor`），Boot 的自动配置也由此导入。
3. `registerBeanPostProcessors()`：注册 **BeanPostProcessor**，它们会在每个 Bean 初始化前后被调用。
4. `initMessageSource()`、`initApplicationEventMulticaster()`、`onRefresh()`（Spring Boot 在这里创建内嵌 Web 服务器）、`registerListeners()`。
5. `finishBeanFactoryInitialization()`：**实例化所有非懒加载的单例 Bean**（生命周期见下一章）。
6. `finishRefresh()`：发布 `ContextRefreshedEvent`，启动 Web 服务器开始接收请求。

**两类后置处理器的区别**：BeanFactoryPostProcessor 处理的是 BeanDefinition（图纸），在实例化之前；BeanPostProcessor 处理的是 Bean 实例，在初始化前后。

## 二、Bean 生命周期、作用域与循环依赖

Bean 的生命周期可以压缩成四步：**实例化 → 属性填充 → 初始化 → 销毁**，其余都是插在这四步之间的扩展点。循环依赖也要放在这条流程中理解。

### 2.1 Bean 生命周期【核心】

1. **实例化**：通过构造器或工厂方法创建对象（`createBeanInstance`）。
2. **属性填充**：注入 `@Autowired`、`@Value` 等依赖（`populateBean`）。
3. **Aware 回调**：`BeanNameAware`、`BeanFactoryAware`、`ApplicationContextAware` 等，把容器信息交给 Bean。
4. **BeanPostProcessor 前置处理**：`postProcessBeforeInitialization`，`@PostConstruct` 就是在这里由 `CommonAnnotationBeanPostProcessor` 执行的。
5. **初始化**：`InitializingBean.afterPropertiesSet()` → 自定义 `init-method`。
6. **BeanPostProcessor 后置处理**：`postProcessAfterInitialization`，**AOP 代理对象在这里生成**，容器里存的是代理。
7. **使用**：放入单例池供调用。
8. **销毁**：容器关闭时依次执行 `@PreDestroy` → `DisposableBean.destroy()` → 自定义 `destroy-method`。

**初始化方法的执行顺序**：`@PostConstruct` → `afterPropertiesSet()` → `init-method`。

### 2.2 作用域与线程安全【重点】

| 作用域 | 含义 |
| --- | --- |
| singleton（默认） | 整个容器一个实例 |
| prototype | 每次 getBean 创建新实例，容器不管理其销毁 |
| request / session / application | Web 环境下每个请求 / 会话 / ServletContext 一个实例 |

**单例 Bean 是线程安全的吗**：Spring 不保证。大多数 Service、Controller 是**无状态**的（只有注入的依赖，没有可变的成员变量），所以是安全的。如果有可变成员变量，应改为局部变量、使用 ThreadLocal 或并发容器，而不是简单地改成 prototype。

**单例中注入 prototype 的坑**：单例只创建一次，注入进来的 prototype 也就只有一个。需要每次新实例时用 `ObjectProvider<T>.getObject()` 或 `@Lookup`。

### 2.3 循环依赖与三级缓存【核心】

A 依赖 B、B 又依赖 A。Spring 通过**提前暴露半成品**来解决，这就是三级缓存（都在 `DefaultSingletonBeanRegistry` 中）：

| 缓存 | 字段 | 存什么 |
| --- | --- | --- |
| 一级 | `singletonObjects` | 完全初始化好的 Bean（成品） |
| 二级 | `earlySingletonObjects` | 提前暴露的半成品（已实例化、未填充属性），可能已是代理 |
| 三级 | `singletonFactories` | 生成半成品的工厂 `ObjectFactory`，调用时如果需要 AOP 就提前创建代理 |

**解决过程**（见列表后的图）：

1. 创建 A：实例化 A，把「生成 A 早期引用的工厂」放入三级缓存。
2. 填充 A 的属性，发现依赖 B，开始创建 B。
3. 实例化 B，填充 B 的属性，发现依赖 A：一级、二级缓存都没有 A，从三级缓存拿到工厂，生成 A 的早期引用（需要代理则此时生成代理），放入二级缓存并删除三级缓存。
4. B 拿到 A 的早期引用，完成初始化，放入一级缓存。
5. 回到 A，注入 B，完成初始化，从二级缓存取出早期引用作为最终对象放入一级缓存。

&#91;embedded content: 三级缓存解决循环依赖 · 6 步\]

关键在第 ④ 步：B 不等 A 创建完成，而是拿一个「已实例化、未填充」的 A 先用上；因为引用指向同一个对象，A 后续填充完成后 B 持有的也是完整的 A。

**为什么要三级而不是二级**：只是解决循环依赖的话二级就够了。第三级的意义在于 **AOP**：正常情况下代理在初始化最后一步才生成，如果没有循环依赖就不需要提前生成。三级缓存放的是工厂，**只有发生循环依赖时才提前创建代理**，既保证注入的是代理对象，又不打乱正常的生命周期。

**无法解决的情况【核心】**：

- **构造器注入的循环依赖**：实例化阶段就需要对方，此时自己还没实例化，无法提前暴露。可以在参数上加 `@Lazy`，注入一个代理延迟解析。
- **prototype 作用域**：不进入缓存。
- **`@Async` 等在初始化后生成代理的情况**：可能报「注入的不是最终版本」的错误。

**Spring Boot 2.6 起默认禁止循环依赖**（`spring.main.allow-circular-references=false`），官方立场是循环依赖代表设计问题，应重构拆分。

## 三、AOP

AOP 把日志、事务、鉴权、缓存等横切逻辑从业务代码中抽离出来。Spring AOP 基于**运行时动态代理**实现：容器里放的是代理对象，调用先经过代理再到目标方法。

### 3.1 核心概念【核心】

| 概念 | 含义 |
| --- | --- |
| 切面 Aspect | 横切逻辑的封装，即 `@Aspect` 类，= 切点 + 通知 |
| 连接点 JoinPoint | 可以插入逻辑的位置，Spring AOP 中只有方法执行 |
| 切点 Pointcut | 匹配哪些连接点，如 `execution(* com.demo.service..*(..))`、`@annotation(...)` |
| 通知 Advice | 在切点做什么，五种：@Before、@After、@AfterReturning、@AfterThrowing、@Around |
| 织入 Weaving | 把切面应用到目标对象生成代理的过程，Spring 在运行时织入 |

**通知执行顺序【重点】**（Spring 5.2.7+）：@Around 前半部分 → @Before → 目标方法 → @AfterReturning / @AfterThrowing → @After → @Around 后半部分。多个切面用 `@Order` 控制，数字越小越在外层。

### 3.2 JDK 动态代理与 CGLIB【核心】

| 维度 | JDK 动态代理 | CGLIB |
| --- | --- | --- |
| 原理 | `Proxy.newProxyInstance` 生成实现同一接口的代理类，调用转给 `InvocationHandler` | 用 ASM 生成目标类的**子类**，重写方法，调用转给 `MethodInterceptor` |
| 要求 | 目标类必须实现接口 | 不需要接口；但 final 类、final / private / static 方法无法代理 |
| 选用规则 | Spring Framework 默认：有接口用 JDK，没有用 CGLIB | **Spring Boot 2.0 起默认一律用 CGLIB**（`proxyTargetClass=true`），避免按实现类注入时报错 |

**Spring AOP 与 AspectJ【重点】**：Spring AOP 是运行时代理，只能拦截 Spring Bean 的方法调用；AspectJ 是编译期或类加载期修改字节码，功能更强（可拦截字段、构造器、自调用），性能更好但使用更复杂。Spring 只是借用了 AspectJ 的注解和切点表达式。

### 3.3 代理对象的创建时机【重点】

`@EnableAspectJAutoProxy`（Boot 自动开启）会注册 `AnnotationAwareAspectJAutoProxyCreator`，它是一个 BeanPostProcessor。在 Bean 初始化完成后的 `postProcessAfterInitialization` 中，它查找匹配当前 Bean 的所有通知（Advisor），有就创建代理并替换原对象。发生循环依赖时则提前在三级缓存的工厂里创建（`getEarlyBeanReference`）。

### 3.4 自调用失效【核心】

```java
@Service
public class OrderService {
    public void create() {
        this.saveLog();   // this 是目标对象，不是代理，saveLog 上的切面 / 事务不生效
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void saveLog() { }
}
```

代理只拦截「从外部经过代理对象」的调用，类内部用 `this` 直接调用不会经过代理。解决方法：

1. **拆到另一个 Bean** 中调用（推荐）。
2. 注入自己：`@Lazy @Autowired private OrderService self;` 然后 `self.saveLog()`。
3. `@EnableAspectJAutoProxy(exposeProxy = true)` + `((OrderService) AopContext.currentProxy()).saveLog()`。

### 3.5 实践：用 AOP 做操作日志【了解】

```java
@Aspect
@Component
public class LogAspect {
    @Around("@annotation(opLog)")
    public Object around(ProceedingJoinPoint pjp, OpLog opLog) throws Throwable {
        long start = System.currentTimeMillis();
        try {
            return pjp.proceed();
        } finally {
            log.info("{} 耗时 {} ms", opLog.value(), System.currentTimeMillis() - start);
        }
    }
}
```

## 四、事务

声明式事务 `@Transactional` 是 AOP 最典型的应用，所以「代理失效」的场景就是「事务失效」的主要来源。

### 4.1 @Transactional 原理【重点】

代理调用 `TransactionInterceptor.invoke()`，内部按以下顺序执行：

1. 读取注解属性（传播行为、隔离级别、超时、只读、回滚规则）。
2. 按传播行为决定开启新事务还是加入当前事务；开启时从数据源拿连接、`setAutoCommit(false)`，并把连接**绑定到 ThreadLocal**（`TransactionSynchronizationManager`），之后同一线程内的 MyBatis / JdbcTemplate 都用这个连接。
3. 执行目标方法。
4. 抛出异常且符合回滚规则 → rollback；否则 commit。
5. 解绑并归还连接。

因为事务连接绑在 ThreadLocal 上，**在事务方法内开新线程操作数据库，不在同一个事务里**。

### 4.2 七种传播行为【核心】

| 传播行为 | 当前有事务 | 当前无事务 |
| --- | --- | --- |
| **REQUIRED（默认）** | 加入 | 新建 |
| **REQUIRES\_NEW** | 挂起当前事务，新建一个独立事务 | 新建 |
| **NESTED** | 在当前事务中创建保存点（savepoint），子事务回滚只回到保存点 | 新建 |
| SUPPORTS | 加入 | 非事务执行 |
| NOT\_SUPPORTED | 挂起当前事务，非事务执行 | 非事务执行 |
| MANDATORY | 加入 | 抛异常 |
| NEVER | 抛异常 | 非事务执行 |

**REQUIRES\_NEW 与 NESTED 的区别【重点】**：REQUIRES\_NEW 是完全独立的事务，内层已提交的不受外层回滚影响（适合「无论业务是否成功都要记录日志」）；NESTED 是外层事务的一部分，外层回滚内层也回滚，但内层回滚不影响外层。

**经典坑 `UnexpectedRollbackException`**：外层 REQUIRED 调用内层 REQUIRED，内层抛异常并把事务标记为 rollback-only，外层 catch 住异常后继续提交，就会报这个错。因为两者是同一个事务，不可能一半提交一半回滚。如果希望内层失败不影响外层，内层应用 REQUIRES\_NEW 或 NESTED。

### 4.3 事务失效场景【核心】

| 场景 | 原因 |
| --- | --- |
| 同类内部自调用 | 没经过代理（见 3.4 节） |
| 方法不是 public | Spring 只对 public 方法生成事务（Spring 6 起 CGLIB 代理也支持 protected 和包可见方法） |
| 方法用 final / static 修饰 | CGLIB 无法重写 |
| 类没有被 Spring 管理 | 自己 new 出来的对象没有代理 |
| 异常被 catch 后没有抛出 | 代理感知不到异常，会正常提交 |
| 抛出的是受检异常 | **默认只回滚 RuntimeException 和 Error**，应写 `rollbackFor = Exception.class` |
| 传播行为配置为 NOT\_SUPPORTED / NEVER 等 | 本来就不在事务中运行 |
| 在事务内开新线程 | 新线程拿不到 ThreadLocal 中的连接 |
| 数据库引擎不支持事务 | 如 MyISAM |
| 多数据源时事务管理器指定错误 | 事务管理的不是实际操作的数据源 |

### 4.4 大事务问题与编程式事务【重点】

`@Transactional` 加在整个方法上，容易把远程调用、文件处理、大量查询也包进事务，导致连接长时间被占用（连接池耗尽）和长事务（见 MySQL 文档）。可以用 `TransactionTemplate` 只包裹真正需要事务的几行代码：

```java
Order order = buildOrder(req);          // 非事务：参数处理、远程查询
transactionTemplate.executeWithoutResult(status -> {
    orderMapper.insert(order);
    stockMapper.deduct(order.getSkuId(), order.getQty());
});
mqProducer.send(order);                // 事务提交后再发消息
```

若必须在事务提交后触发某些动作，可用 `@TransactionalEventListener(phase = AFTER_COMMIT)`。

## 五、Spring MVC

Spring MVC 的核心是前端控制器 `DispatcherServlet`：所有请求先到它，再由它分发给对应的 Controller。

### 5.1 请求处理流程【核心】

1. 请求先经过 Servlet 容器的 **Filter 链**，再到达 `DispatcherServlet.doDispatch()`。
2. **HandlerMapping** 根据 URL 找到处理器（Controller 方法），返回包含拦截器的 `HandlerExecutionChain`。
3. 找到能执行该处理器的 **HandlerAdapter**（注解方式为 `RequestMappingHandlerAdapter`）。
4. 依次执行拦截器的 `preHandle()`，任一返回 false 则中断。
5. HandlerAdapter 用**参数解析器**绑定参数（`@RequestParam`、`@RequestBody` 由 HttpMessageConverter 把 JSON 转为对象），调用 Controller 方法。
6. 执行拦截器的 `postHandle()`。
7. 处理返回值：`@ResponseBody` / `@RestController` 由 HttpMessageConverter 直接写成 JSON；否则返回 ModelAndView，由 **ViewResolver** 解析视图并渲染。
8. 执行拦截器的 `afterCompletion()`（无论是否异常都会执行，适合清理 ThreadLocal）。

中间任何一步抛出异常，由 `HandlerExceptionResolver` 处理，`@ControllerAdvice` + `@ExceptionHandler` 的全局异常处理就是在这里生效。

### 5.2 过滤器与拦截器【核心】

| 维度 | Filter | Interceptor |
| --- | --- | --- |
| 规范 | Servlet 规范，由 Servlet 容器管理 | Spring MVC 的组件，由 Spring 容器管理 |
| 执行位置 | DispatcherServlet 之前 | DispatcherServlet 之后、Controller 前后 |
| 拦截范围 | 所有请求，包括静态资源 | 只拦截进入 Spring MVC 的请求 |
| 能拿到的信息 | request / response | 还能拿到将要执行的 Handler（哪个方法、有什么注解）和 ModelAndView |
| 典型用途 | 编码设置、跨域、XSS 过滤、Spring Security | 登录校验、权限、日志、接口耗时 |

**三者的层次**：Filter → Interceptor → AOP（切面在方法级别，可以作用于任何 Bean，不限于 Web 层）。

### 5.3 常用特性【重点】

- **参数绑定**：`@PathVariable`（路径）、`@RequestParam`（查询参数 / 表单）、`@RequestBody`（JSON 请求体，一个方法只能有一个）、`@RequestHeader`。
- **参数校验**：`@Valid`（Jakarta 标准，支持嵌套校验）与 `@Validated`（Spring 扩展，支持分组，加在类上可校验方法的单个参数）；校验失败抛 `MethodArgumentNotValidException`，在全局异常处理中统一返回。
- **全局异常处理**：`@RestControllerAdvice` + `@ExceptionHandler`，统一返回格式，避免堆栈信息泄露到前端。

## 六、Spring Boot

Spring Boot 的本质是「约定大于配置」：通过 Starter 管理依赖，通过自动配置按条件注册 Bean，通过内嵌 Web 服务器让应用直接 `java -jar` 运行。

### 6.1 @SpringBootApplication 拆解【核心】

- `@SpringBootConfiguration`：本质是 `@Configuration`，标记为配置类。
- `@ComponentScan`：扫描启动类所在包及子包，所以启动类要放在根包。
- `@EnableAutoConfiguration`：开启自动配置，是核心。

### 6.2 自动配置原理【核心】

1. `@EnableAutoConfiguration` 通过 `@Import(AutoConfigurationImportSelector.class)` 导入选择器。
2. 选择器读取所有 jar 包中的 `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` 文件，得到候选自动配置类列表。Spring Boot 2.7 之前读取的是 `META-INF/spring.factories`，3.0 起只支持新文件。
3. 去重、排除 `exclude` 指定的类。
4. 每个自动配置类上有 **@Conditional 系列注解**，条件满足才生效：
   - `@ConditionalOnClass`：classpath 上有某个类（引入了对应 Starter）
   - `@ConditionalOnMissingBean`：用户没有自己定义这个 Bean（所以用户配置优先）
   - `@ConditionalOnProperty`：配置文件中有某个属性
5. 生效的配置类通过 `@EnableConfigurationProperties` 绑定 `application.yml` 中的属性，向容器注册 Bean。

**一句话总结**：读取配置文件拿到候选配置类 → 用条件注解筛选 → 注册 Bean，用户自己定义的 Bean 优先。排查哪些配置生效可以加 `--debug` 启动看条件评估报告。

### 6.3 Starter 与自定义 Starter【重点】

Starter 是一组依赖的集合加上对应的自动配置。自定义步骤：

1. 创建 `xxx-spring-boot-starter` 模块（官方 Starter 命名为 `spring-boot-starter-xxx`）。
2. 写 `XxxProperties`（`@ConfigurationProperties(prefix = "xxx")`）。
3. 写 `XxxAutoConfiguration`（`@AutoConfiguration` + `@ConditionalOnXxx` + `@Bean`）。
4. 在 `AutoConfiguration.imports` 文件中登记自动配置类。
5. 可选：引入 `spring-boot-configuration-processor` 生成配置提示元数据。

### 6.4 启动流程【重点】

`SpringApplication.run()` 的主要步骤：

1. 推断应用类型（Servlet / Reactive / 普通），加载初始化器和监听器。
2. 准备 Environment：加载各种来源的配置（命令行、环境变量、application.yml、profile）。
3. 打印 Banner，创建 ApplicationContext。
4. 准备上下文：加载启动类等 BeanDefinition。
5. **刷新上下文**：调用 Spring 的 `refresh()`（见 1.3 节），其中 `onRefresh()` 创建并启动内嵌 Tomcat。
6. 调用 `ApplicationRunner` 和 `CommandLineRunner`，适合做启动后的预热工作。

### 6.5 配置加载优先级【重点】

从高到低（高优先级覆盖低优先级）：命令行参数 > Java 系统属性（`-D`）> 操作系统环境变量 > jar 包外的 `application-{profile}.yml` > jar 包外的 `application.yml` > jar 包内的 `application-{profile}.yml` > jar 包内的 `application.yml` > `@PropertySource` > 默认值。规律是：外部优先于内部，同一位置带 profile 的优先于不带的（Spring Boot 2.4 起的规则）。

**@ConfigurationProperties 与 @Value**：前者批量绑定一组属性、支持松散绑定和校验，适合成组配置；后者注入单个值、支持 SpEL。

### 6.6 Actuator【了解】

提供 `/actuator/health`、`/metrics`、`/info` 等监控端点，配合 Micrometer + Prometheus + Grafana 做监控，配合 K8s 的存活 / 就绪探针。生产环境只暴露必要端点，并做好鉴权。

## 七、MyBatis / MyBatis-Plus

MyBatis 是半自动的 ORM：SQL 由开发者自己写，框架负责参数绑定和结果映射，在国内的使用广泛程度远高于 JPA。

### 7.1 #{} 与 ${}【核心】

|  | `#{}` | `${}` |
| --- | --- | --- |
| 处理方式 | 预编译，替换为 `?` 占位符，由 PreparedStatement 设置参数 | 直接字符串拼接到 SQL 中 |
| SQL 注入 | 能防止 | **有注入风险** |
| 适用 | 绝大多数参数 | 动态表名、列名、`ORDER BY` 字段等无法用占位符的地方 |

必须用 `${}` 时，要在代码中用**白名单**校验参数（如排序字段只允许 `create_time`、`price`）。模糊查询应写 `LIKE CONCAT('%', #{kw}, '%')`，不要写 `LIKE '%${kw}%'`。

### 7.2 Mapper 接口为什么不需要实现类【核心】

MyBatis 用 **JDK 动态代理**为 Mapper 接口生成代理对象（`MapperProxy`）。调用接口方法时，代理用「接口全限定名 + 方法名」作为 statementId，找到 XML 或注解中对应的 `MappedStatement`，再交给 SqlSession 执行。这也是 Mapper 方法不能重载的原因（statementId 会冲突）。与 Spring 整合时，`MapperScannerConfigurer` 把每个 Mapper 注册为 `MapperFactoryBean`，由它产生代理对象。

### 7.3 SQL 执行流程【重点】

`MapperProxy` → `SqlSession` → **Executor**（执行器，负责缓存和事务）→ **StatementHandler**（创建 Statement）→ **ParameterHandler**（设置参数）→ JDBC 执行 → **ResultSetHandler**（把结果集映射成对象）。

Executor 有三种：SIMPLE（默认，每次新建 Statement）、REUSE（复用 Statement）、BATCH（批量执行，适合大批量写入）。

### 7.4 一级缓存与二级缓存【核心】

|  | 一级缓存 | 二级缓存 |
| --- | --- | --- |
| 范围 | SqlSession 级别 | Mapper（namespace）级别，跨 SqlSession 共享 |
| 默认 | 开启 | 关闭，需配置 `<cache/>` |
| 失效 | 执行增删改、commit、close | 本 namespace 执行增删改 |

**坑**：

- 与 Spring 整合后，**不在事务中时每次调用都是新的 SqlSession**，一级缓存基本不生效；在同一个事务内才生效。
- 二级缓存在多表关联、多实例部署时容易读到脏数据（A namespace 的缓存不知道 B namespace 更新了关联表）。**生产一般不开二级缓存，统一用 Redis**。

### 7.5 插件机制与分页【重点】

MyBatis 允许用 `Interceptor` 拦截四大对象：**Executor、StatementHandler、ParameterHandler、ResultSetHandler**。原理是用 JDK 动态代理对这些对象层层包装（责任链）。典型应用：分页、慢 SQL 日志、数据权限、字段加密、多租户。

**PageHelper 原理**：`PageHelper.startPage()` 把分页参数放进 **ThreadLocal**；插件拦截紧接着的第一条查询，先执行 count 语句，再改写原 SQL 加上 `LIMIT`，执行完清除 ThreadLocal。所以 `startPage()` 后必须**紧跟**查询，否则分页参数可能残留到同线程的下一条查询。

**逻辑分页与物理分页**：MyBatis 自带的 `RowBounds` 是逻辑分页（查出全部再在内存中截取），数据量大时会 OOM；PageHelper 和 MyBatis-Plus 分页插件都是物理分页。

### 7.6 动态 SQL 与 MyBatis-Plus【重点】

- **动态 SQL 标签**：`<if>`、`<choose>/<when>/<otherwise>`、`<where>`（自动去掉多余的 AND）、`<set>`、`<foreach>`（IN 查询、批量插入）、`<trim>`。
- **MyBatis-Plus**：在 MyBatis 基础上提供通用 CRUD（`BaseMapper`、`IService`）、条件构造器（`LambdaQueryWrapper`，用方法引用避免字段名写错）、分页插件、逻辑删除（`@TableLogic`）、乐观锁插件（`@Version`）、自动填充创建时间等。
- **批量插入优化【了解】**：循环单条 insert 最慢；`<foreach>` 拼成一条多值 INSERT 较快，但 SQL 不宜超过 `max_allowed_packet`；BATCH Executor + JDBC URL 加 `rewriteBatchedStatements=true` 通常最快。MyBatis-Plus 的 `saveBatch` 用的就是 BATCH 模式。

## 八、设计模式与扩展点

「Spring 用到了哪些设计模式」是常见的收尾题；扩展点则是项目中写中间件、做框架封装时最常用的入口。

### 8.1 Spring 中的设计模式【重点】

| 模式 | 体现 |
| --- | --- |
| 工厂 | `BeanFactory`、`FactoryBean` |
| 单例 | 默认单例 Bean，由单例注册表（一级缓存）保证 |
| 代理 | AOP、事务、MyBatis Mapper |
| 模板方法 | `JdbcTemplate`、`RestTemplate`、`TransactionTemplate`、`AbstractApplicationContext.refresh()` |
| 观察者 | `ApplicationEvent` + `ApplicationListener` / `@EventListener` |
| 适配器 | `HandlerAdapter`；事务、AOP 中的 `AdvisorAdapter` |
| 策略 | 代理的选择（JDK / CGLIB）、`Resource` 加载、各种 `*Resolver` |
| 责任链 | 拦截器链 `HandlerExecutionChain`、AOP 通知链、Filter 链 |
| 装饰器 | `BeanWrapper`、`TransactionAwareCacheDecorator` |

### 8.2 常用扩展点【重点】

| 扩展点 | 时机 | 典型用途 |
| --- | --- | --- |
| `BeanFactoryPostProcessor` / `BeanDefinitionRegistryPostProcessor` | Bean 实例化前 | 修改或动态注册 BeanDefinition（MyBatis 扫描 Mapper） |
| `BeanPostProcessor` | Bean 初始化前后 | 生成代理、处理自定义注解 |
| `ImportSelector` / `ImportBeanDefinitionRegistrar` | 解析 `@Import` 时 | `@EnableXxx` 注解的实现方式（如 `@EnableFeignClients`） |
| `Aware` 接口 | 属性填充后 | 拿到 ApplicationContext、BeanName 等 |
| `@EventListener` | 事件发布时 | 业务解耦，如下单后发券、发通知 |
| `ApplicationRunner` / `CommandLineRunner` | 应用启动完成后 | 缓存预热、加载字典 |

### 8.3 @Async 与 @Scheduled【了解】

- `@Async` 同样基于代理，自调用失效；**务必指定自定义线程池**。Spring Boot 2.1 起默认使用自动配置的 `ThreadPoolTaskExecutor`（核心线程 8、无界队列），高并发下队列会无限堆积。
- `@Scheduled` 默认是单线程执行所有定时任务，一个任务卡住会影响其他任务，需要配置 `spring.task.scheduling.pool.size`。多实例部署时会重复执行，需要分布式锁（如 ShedLock）或改用 XXL-JOB 等调度平台。

## 九、高频面试题与自测清单

下面 18 道题覆盖图谱中全部「核心」知识点。Spring 题目的加分方式是能说出关键类名和方法名（如 `refresh`、`singletonFactories`、`TransactionInterceptor`），说明看过源码。

### 9.1 高频面试题

| # | 题目 | 答案要点 | 章节 |
| --- | --- | --- | --- |
| 1 | 谈谈你对 IOC 和 DI 的理解 | 控制权交给容器、解耦；DI 是实现方式；三种注入方式 | 一 |
| 2 | BeanFactory 和 ApplicationContext 的区别？FactoryBean 呢？ | 懒加载 vs 预加载、高级功能；getObject 与 & 前缀 | 一 |
| 3 | @Autowired 和 @Resource 的区别？ | 按类型 vs 按名称、来源、支持位置 | 一 |
| 4 | Spring 容器启动流程？ | refresh：加载 BeanDefinition → BFPP → 注册 BPP → 实例化单例 → 发布事件 | 一 |
| 5 | Bean 的生命周期？ | 实例化 → 填充 → Aware → BPP 前 → 初始化 → BPP 后（AOP）→ 销毁 | 二 |
| 6 | 单例 Bean 是线程安全的吗？ | 不保证；无状态即安全；ThreadLocal | 二 |
| 7 | Spring 怎么解决循环依赖？为什么要三级缓存？ | 提前暴露半成品；第三级是工厂，只在循环依赖时提前创建代理 | 二 |
| 8 | 哪些循环依赖解决不了？ | 构造器注入（用 @Lazy）、prototype；Boot 2.6 默认禁止 | 二 |
| 9 | AOP 的实现原理？JDK 代理和 CGLIB 的区别？ | 动态代理；接口 vs 子类；Boot 2.0 起默认 CGLIB | 三 |
| 10 | 代理对象是什么时候创建的？ | BPP 的 postProcessAfterInitialization；循环依赖时提前 | 三 |
| 11 | 为什么同类方法调用 AOP 会失效？ | this 不是代理；拆 Bean / 注入自己 / AopContext | 三 |
| 12 | @Transactional 的原理？ | TransactionInterceptor；连接绑定 ThreadLocal | 四 |
| 13 | 事务传播行为有哪些？REQUIRES\_NEW 和 NESTED 的区别？ | 7 种；独立事务 vs 保存点 | 四 |
| 14 | 事务失效有哪些场景？ | 自调用、非 public、异常被吞、受检异常、多线程等 | 四 |
| 15 | Spring MVC 的请求处理流程？ | DispatcherServlet → HandlerMapping → HandlerAdapter → 拦截器 → 消息转换 | 五 |
| 16 | 拦截器和过滤器的区别？ | 规范、位置、范围、能拿到的信息 | 五 |
| 17 | Spring Boot 自动配置原理？怎么写一个 Starter？ | AutoConfiguration.imports + @Conditional；五步 | 六 |
| 18 | MyBatis #{} 和 ${} 的区别？Mapper 接口怎么工作？缓存有什么坑？ | 预编译 vs 拼接；JDK 代理；一级缓存在 Spring 中的生效范围、二级缓存脏数据 | 七 |

### 9.2 动手实验

- [ ] 写一个实现了所有生命周期接口的 Bean，在每个回调中打印日志，验证顺序
- [ ] 构造 A、B 字段注入的循环依赖，在 `DefaultSingletonBeanRegistry.getSingleton` 打断点观察三级缓存变化；再改成构造器注入看报错
- [ ] 逐一复现 4.3 节的事务失效场景，并验证 `UnexpectedRollbackException`
- [ ] 写一个自定义 Starter（如统一的操作日志或接口限流），引入后无需任何配置即生效
- [ ] 写一个 MyBatis 插件，打印每条 SQL 的执行耗时
- [ ] 用 `--debug` 启动一个 Boot 应用，读懂自动配置的条件评估报告

### 9.3 自测清单

- [ ] 能把「容器启动 → Bean 生命周期 → AOP 代理 → 事务」串成一条线讲清楚
- [ ] 能画出三级缓存解决循环依赖的流程，并说清为什么需要第三级
- [ ] 能说出至少 8 种事务失效场景及原因
- [ ] 能解释自动配置为什么「用户自己定义的 Bean 优先」
- [ ] 能说清 PageHelper 的原理以及为什么 startPage 必须紧跟查询
- [ ] 能列出 Spring 中用到的 6 种以上设计模式及对应的类
