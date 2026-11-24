# Spring 面试题集（含参考答案）

> 答案按 `prompt/面试回答生成.md` 的六段式生成，选题合并自《Java架构师技术栈锚点.md》2.1 Spring Framework、2.2 Spring Boot、2.3 Spring Cloud、2.4 Spring Security。以 Spring Boot 3.x / Spring Framework 6 / Spring Cloud 2022+ / Spring Security 6 为准，涉及旧版本差异单独标注。口述版可直接读出声，卡壳处即理解缺口。

## 一、Spring Framework

### 1. 【必答】Spring Framework：容器启动 refresh 的关键步骤？

**口述版**

refresh 是容器启动的主线，核心十来步：准备上下文和环境、获取 BeanFactory、给 BeanFactory 填充内置依赖、执行 BeanFactoryPostProcessor 修改 BeanDefinition、注册 BeanPostProcessor、初始化消息源和事件广播器、调用 onRefresh 创建 Web 容器之类的子类组件、注册监听器、实例化所有非懒加载单例、最后完成刷新并发布 ContextRefreshedEvent。要抓住三条主线：BeanDefinition 什么时候齐全，是在 invokeBeanFactoryPostProcessors 之后；Bean 什么时候真正创建，是在 finishBeanFactoryInitialization；AOP 代理什么时候产生，是在 Bean 初始化后的 BeanPostProcessor 里。理解这三条，扩展点、循环依赖、事务失效都能挂上去。

**短版**

refresh 的顺序是准备环境 → 拿 BeanFactory → 执行 BFPP 改 BeanDefinition → 注册 BPP → 初始化消息源与事件广播 → onRefresh → 注册监听器 → 实例化单例 → finishRefresh；记住定义齐全、Bean 创建、代理生成三个阶段。

**展开版**

- 机制链：`prepareRefresh` → `obtainFreshBeanFactory` → `prepareBeanFactory` → `invokeBeanFactoryPostProcessors` → `registerBeanPostProcessors` → `onRefresh` → `registerListeners` → `finishBeanFactoryInitialization` → `finishRefresh`。
- 关键结构：`DefaultListableBeanFactory` 的 `beanDefinitionMap`、`BeanPostProcessor` 列表、`ApplicationEventMulticaster`。
- 触发条件：应用启动、上下文刷新、`@RefreshScope` 局部重建（不走完整 refresh）。
- 例外：父容器与子容器各自 refresh，Spring MVC 场景下 Service 在父容器、Controller 在子容器。
- 对比取舍：一次性全量初始化保证启动即暴露问题，代价是启动慢，Boot 用懒加载和 AOT 缓解。

**追问预判**

- AOP 代理在哪一步产生 → `finishBeanFactoryInitialization` 创建 Bean 时，初始化后的 BPP 里生成。
- BeanDefinition 从哪来 → 注解扫描、`@Bean` 方法、XML、`ImportSelector`、`ImportBeanDefinitionRegistrar`。
- onRefresh 有什么用 → 模板方法，Web 容器在这里创建内嵌 Tomcat。

**常见错误答法**

- 把 refresh 说成"扫描注解并 new 对象"（中间还有 BFPP、BPP、事件等阶段）。
- 认为 AOP 代理在实例化之前生成（在初始化之后由 BPP 包装）。

**记忆锚点**

BFPP 改定义 → BPP 改实例 → onRefresh 建容器 → 最后实例化单例 → finishRefresh。

---

### 2. 【必答】Spring Framework：Bean 的生命周期与销毁顺序？

**口述版**

创建阶段：实例化、属性填充、Aware 回调、BeanPostProcessor 前置处理、初始化方法、BeanPostProcessor 后置处理，之后才进入使用阶段。初始化方法内部顺序是 @PostConstruct 先，然后 afterPropertiesSet，最后自定义 init-method，因为 @PostConstruct 由 CommonAnnotationBeanPostProcessor 处理，位置最靠前。销毁阶段顺序相反：@PreDestroy、DisposableBean 的 destroy、自定义 destroy-method，而且单例按创建的反序销毁，保证依赖方先于被依赖方关闭。要注意 prototype 的 Bean Spring 不管理销毁，只负责创建，销毁要自己处理，这也是原型 Bean 容易泄漏的原因。

**短版**

实例化 → 属性填充 → Aware → BPP 前置 → @PostConstruct → afterPropertiesSet → init-method → BPP 后置 → 使用；销毁是 @PreDestroy → destroy → destroy-method，单例逆序销毁，prototype 不管销毁。

**展开版**

- 机制链：`createBean` → `doCreateBean` → `populateBean` → `initializeBean`（Aware + BPP + init）→ 放入单例池 → 容器关闭时 `destroySingletons`。
- 关键结构：`BeanPostProcessor.postProcessBeforeInitialization/AfterInitialization`、`InitializingBean`、`DisposableBean`、`CommonAnnotationBeanPostProcessor`。
- 触发条件：容器启动、`getBean` 首次获取、容器关闭或 `destroy()`。
- 例外：`@DependsOn` 会影响销毁顺序；`prototype` 的销毁回调完全不触发。
- 对比取舍：回调丰富便于扩展，代价是顺序容易记混，排查初始化问题时必须按顺序对照。

**追问预判**

- @PostConstruct 和 afterPropertiesSet 谁先 → @PostConstruct 先，由注解 BPP 在初始化回调前处理。
- 优雅停机时 Bean 怎么关 → 先停止接收请求，再按依赖逆序销毁，配合 `server.shutdown=graceful`。
- 为什么 prototype 不销毁 → 容器不持有引用，无法追踪生命周期，需要业务自己释放。

**常见错误答法**

- 说 init-method 在 afterPropertiesSet 之前执行（顺序相反）。
- 认为所有 Bean 都会被 Spring 销毁（prototype 不会）。

**记忆锚点**

实例化 → 填充 → Aware → BPP 前 → @PostConstruct → afterPropertiesSet → init → BPP 后；销毁逆序。

---

### 3. 【必答】Spring Framework：循环依赖与三级缓存怎么解决？

**口述版**

三级缓存是 singletonObjects 成品池、earlySingletonObjects 半成品池、singletonFactories 对象工厂池。A 依赖 B、B 依赖 A 时，A 实例化后先把一个能返回早期引用的 ObjectFactory 放进三级缓存，再去填充属性发现需要 B；B 创建时又要 A，就从三级缓存拿到工厂，调用得到 A 的早期引用，放入二级缓存并清掉三级缓存，B 完成创建，A 再继续填充。所以能解决的是 setter 或字段注入，构造器注入解决不了，因为实例都没造出来，没有早期引用可暴露。二级缓存不够的原因是要支持 AOP：工厂可以在需要时提前生成代理对象，保证 B 注入的是代理而不是原始对象。另外 Spring Boot 2.6 起默认禁止循环依赖，需要显式开启。

**短版**

三级缓存：成品 singletonObjects、早期引用 earlySingletonObjects、对象工厂 singletonFactories；A 先暴露工厂再填充属性，B 从工厂拿到 A 的早期引用完成创建；构造器注入无解，二级缓存不够是因为要支持 AOP 提前代理。

**展开版**

- 机制链：`doCreateBean` 实例化 → `addSingletonFactory` 放三级缓存 → `populateBean` 触发 B 创建 → B 从三级缓存取 A → 提升到二级缓存 → B 完成 → A 完成并进一级缓存。
- 关键结构：`DefaultSingletonBeanRegistry` 的三个 Map、`ObjectFactory` 的 `getEarlyBeanReference`（内部触发 AOP 提前代理）。
- 触发条件：单例、setter/字段注入、允许循环依赖。
- 例外：构造器注入、prototype Bean、`@Async` 代理场景都可能失败，Boot 2.6+ 默认 `allow-circular-references=false`。
- 对比取舍：三级缓存解决绝大多数循环依赖，代价是语义隐晦、掩盖设计问题，最佳实践仍是拆依赖或引入事件。

**追问预判**

- 为什么二级缓存不够 → 需要 ObjectFactory 在适当时机生成代理，直接放对象可能放的是未增强的原始 Bean。
- 构造器注入为什么不行 → 实例都未创建，没有可暴露的早期引用。
- 三级缓存会一直增长吗 → 不会，早期引用提升到二级后三级条目被移除，最终都进一级缓存。

**常见错误答法**

- 说三级缓存是为了性能（是为了支持 AOP 提前暴露代理）。
- 认为构造器注入的循环依赖也能被解决（不能）。

**记忆锚点**

三级缓存 → 工厂暴露早期引用 → 支持 AOP 提前代理 → 构造器注入无解。

---

### 4. 【必答】Spring Framework：BeanFactoryPostProcessor 与 BeanPostProcessor 的区别？

**口述版**

BeanFactoryPostProcessor 处理的是 BeanDefinition，也就是"元数据"，它在容器 refresh 的早期执行，此时 Bean 还没实例化，典型用途是修改属性值、注册新的定义，占位符替换的 PropertySourcesPlaceholderConfigurer 就是它的实现。BeanPostProcessor 处理的是已经实例化出来的 Bean 对象，在初始化前后各回调一次，AOP 代理、@Autowired 注入、@PostConstruct 调用都靠它，所以它是"改实例"。一句话区分：BFPP 改图纸，BPP 改成品。顺序上 BFPP 一定先于所有普通 Bean 的创建，BPP 则在每个 Bean 创建时逐个生效，而且 BPP 本身会被提前实例化，所以不要在 BPP 里依赖普通 Bean，容易造成提前初始化。

**短版**

BFPP 处理 BeanDefinition、在 Bean 实例化前执行、用于改元数据和注册定义；BPP 处理 Bean 实例、在初始化前后回调、用于 AOP、注入和注解回调；BFPP 改图纸，BPP 改成品。

**展开版**

- 机制链：`invokeBeanFactoryPostProcessors` 遍历执行 BFPP → 之后创建 Bean → 每个 Bean 在 `initializeBean` 前后触发 BPP。
- 关键结构：`BeanDefinitionRegistryPostProcessor`（BFPP 子接口，可注册新定义）、`InstantiationAwareBeanPostProcessor`、`SmartInstantiationAwareBeanPostProcessor`。
- 触发条件：占位符替换、条件装配、自定义注解处理、代理增强。
- 例外：`BeanDefinitionRegistryPostProcessor` 执行更早，用于动态注册 Bean；BPP 返回 null 会中断后续处理。
- 对比取舍：BFPP 能改结构但拿不到实例，BPP 能改实例但定义已成型，二者配合才能覆盖完整生命周期。

**追问预判**

- 想动态注册一个 Bean 用哪个 → `BeanDefinitionRegistryPostProcessor` 或 `ImportBeanDefinitionRegistrar`。
- BPP 里能注入普通 Bean 吗 → 不建议，会引发提前初始化甚至循环依赖告警。
- @Autowired 是哪个 BPP 处理的 → `AutowiredAnnotationBeanPostProcessor`。

**常见错误答法**

- 认为 BPP 能修改 BeanDefinition（它拿到的是实例）。
- 在 BFPP 里获取 Bean 实例（此时还没创建）。

**记忆锚点**

BFPP 改定义、早于实例化；BPP 改实例、初始化前后回调。

---

### 5. 【必答】Spring Framework：AOP 的代理机制与自调用失效？

**口述版**

Spring AOP 是运行时代理，目标有接口时用 JDK 动态代理，没有接口用 CGLIB 生成子类，Spring Boot 2 之后默认统一用 CGLIB，避免强制转接口的问题。切面逻辑包在 MethodInterceptor 里，代理对象调用时先执行通知链再调用目标方法。自调用失效的根因是：类内部用 this 调另一个方法，走的是原始对象而不是代理对象，所以事务、缓存、异步这些基于代理的注解都不会生效。三种解法：把方法拆到另一个 Bean 里通过注入调用；注入自身代理；或者开启 exposeProxy 用 AopContext.currentProxy 拿当前代理。另外 private 方法、final 方法、final 类也代理不了，因为它们无法被重写或拦截。

**短版**

有接口用 JDK 代理、无接口用 CGLIB，Boot 2 起默认 CGLIB；自调用走 this 不经过代理导致注解失效；解法是拆 Bean、注入自身、或 AopContext 取代理；private、final 方法拦不到。

**展开版**

- 机制链：切点匹配 → 创建代理 → 调用进入 `ReflectiveMethodInvocation` → 依次执行通知 → 反射调用目标方法。
- 关键结构：`Advisor`、`Pointcut`、`MethodInterceptor`、`proxyTargetClass`、`exposeProxy`。
- 触发条件：`@Transactional`、`@Cacheable`、`@Async`、自定义切面、`@PreAuthorize`。
- 例外：AspectJ 编译期织入没有自调用问题，但需要额外编译步骤；`@Async` 自调用同样失效。
- 对比取舍：代理方式无侵入、易用，代价是只能拦截外部调用，且有代理对象与原始对象的身份差异。

**追问预判**

- 为什么 this 调用不走代理 → this 指向原始对象，代理逻辑只存在于代理实例上。
- 注入自身为什么会循环依赖 → 需要 `@Lazy` 或允许循环依赖，Boot 2.6+ 要注意配置。
- 同类方法调用怎么验证失效 → 打断点看栈里有没有代理类帧，或看事务是否真的开了。

**常见错误答法**

- 认为加了 @Transactional 就一定生效（自调用、private、异常吞掉都会失效）。
- 说 Spring AOP 基于字节码增强在编译期织入（默认是运行时代理）。

**记忆锚点**

代理对象才有增强 → this 调用绕过 → 拆 Bean / 注入自身 / AopContext。

---

### 6. 【进阶】Spring Framework：事务传播行为与常见失效场景？

**口述版**

七种传播行为里最常用三个：REQUIRED 是默认，有事务就加入、没有就新建；REQUIRES_NEW 总是新建并挂起当前事务，适合必须独立提交的日志或补偿；NESTED 是嵌套，用保存点实现，内层回滚不影响外层，但外层回滚会带着内层一起回滚。其余是 SUPPORTS、NOT_SUPPORTED、MANDATORY、NEVER，用于明确要不要事务。失效场景要背下来：自调用、private 或 final 方法、异常被 catch 没抛出、抛的是受检异常但没配 rollbackFor、多线程里新线程不受原事务控制、以及数据库引擎本身不支持事务。排查顺序是先确认代理是否生效，再看异常类型和传播行为。

**短版**

REQUIRED 加入或新建、REQUIRES_NEW 挂起当前另起、NESTED 用保存点；失效场景：自调用、非 public/final、异常被吞、受检异常未配 rollbackFor、跨线程、引擎不支持。

**展开版**

- 机制链：进入代理 → `TransactionInterceptor` 读注解 → `PlatformTransactionManager` 按传播行为决定加入或新建 → 提交或回滚。
- 关键结构：`TransactionDefinition` 的传播、隔离、超时、只读、回滚规则；`TransactionSynchronizationManager` 的线程绑定资源。
- 触发条件：跨服务方法调用、批量任务、日志与主业务分离、补偿逻辑。
- 例外：REQUIRES_NEW 会占用额外连接，嵌套过深可能耗尽连接池；NESTED 依赖保存点，部分数据源不支持。
- 对比取舍：细粒度传播提升灵活性，代价是连接占用和锁持有时间增加，传播用错会造成"该回滚的没回滚"。

**追问预判**

- 为什么 catch 住异常就不回滚 → 拦截器只根据逃逸出的异常决定回滚。
- 受检异常默认回滚吗 → 不回滚，只有 RuntimeException 和 Error 默认回滚。
- NESTED 和 REQUIRES_NEW 的本质区别 → 前者同一事务加保存点，后者是独立的新事务。

**常见错误答法**

- 认为所有异常都会触发回滚（受检异常默认不回滚）。
- 用 REQUIRES_NEW 做日志导致连接池被打满而不自知。

**记忆锚点**

REQUIRED 加入 → REQUIRES_NEW 独立 → NESTED 保存点 → 异常必须逃逸出去。

---

### 7. 【进阶】Spring Framework：事务边界怎么治理，长事务和异步怎么办？

**口述版**

事务边界的原则是尽量短、只包住必要的数据库操作。长事务的危害是长时间持有连接和行锁，阻塞其他请求、放大死锁概率、还可能拖慢主从同步。治理手段有四个：一是把远程调用、消息发送、文件 IO 移出事务，先用本地事务落库，再用事务提交后的回调或事件去触发下游；二是拆分大事务，按业务阶段拆成多个短事务，用状态机推进；三是用 afterCommit 回调或 TransactionSynchronization 做提交后动作，避免"消息发了但事务回滚"；四是读写分离场景注意事务内必须走主库，否则刚写就查不到。异步线程不继承事务，跨线程要在提交后再触发，不能直接把事务上下文传过去。

**短版**

事务要短，只包数据库操作；远程调用、MQ、IO 移出事务，用 afterCommit 或事件在提交后触发；大事务拆成状态机推进；异步线程不继承事务，读写分离在事务内走主库。

**展开版**

- 机制链：开启事务 → 业务操作 → 提交 → `TransactionSynchronization.afterCommit` 触发下游 → 异步执行。
- 关键结构：`TransactionSynchronizationManager.registerSynchronization`、`@TransactionalEventListener(phase = AFTER_COMMIT)`、`TransactionTemplate`。
- 触发条件：下单、支付、库存扣减、跨服务调用、消息通知。
- 例外：`REQUIRES_NEW` 里的提交是独立时点，afterCommit 的触发时机要按实际传播行为判断。
- 对比取舍：短事务提升并发和稳定性，代价是业务一致性要靠状态机和补偿来保证，不能只依赖单库事务。

**追问预判**

- 消息发送放在事务里有什么问题 → 事务回滚但消息已发，下游产生脏数据。
- 怎么保证"提交后才发消息" → 用 afterCommit 回调、事务消息或本地消息表加补偿。
- 异步任务能拿到原事务吗 → 不能，事务绑定在线程上，新线程没有事务上下文。

**常见错误答法**

- 在事务方法里同步调 RPC（把远程耗时算进事务，锁持有时间被放大）。
- 认为 afterCommit 里抛异常会回滚事务（此时已经提交）。

**记忆锚点**

事务只包 DB → 远程和 MQ 移出 → afterCommit 触发 → 异步不继承事务。

---

### 8. 【进阶】Spring Framework：一次请求在 Spring MVC 里怎么走完？

**口述版**

请求先到 DispatcherServlet，它查 HandlerMapping 找到对应的 Handler 和拦截器链，再找合适的 HandlerAdapter 来执行。执行前会做参数解析，把请求参数、路径变量、请求体按 HttpMessageConverter 反序列化成方法参数；执行完把返回值交给 ReturnValueHandler 处理，@ResponseBody 就走消息转换器直接写响应，否则解析成 ModelAndView 交给 ViewResolver 渲染。过程中有异常会被 HandlerExceptionResolver 处理，@ControllerAdvice 就是在这个环节生效。另外 Filter 在 Servlet 层、Interceptor 在 MVC 层、AOP 在 Bean 层，三者的顺序是 Filter 最外层，然后 DispatcherServlet 内的 Interceptor，最后才是 Controller 方法上的 AOP 切面。

**短版**

DispatcherServlet → HandlerMapping 找 Handler 与拦截器 → HandlerAdapter 执行并解析参数 → 返回值经 ReturnValueHandler 或 ViewResolver 处理 → 异常交 HandlerExceptionResolver；Filter 在最外层，Interceptor 次之，AOP 最内。

**展开版**

- 机制链：`doDispatch` → `getHandler` → `getHandlerAdapter` → `applyPreHandle` → `handle` → `applyPostHandle` → `processDispatchResult` → 渲染或写响应。
- 关键结构：九大组件（HandlerMapping、HandlerAdapter、ViewResolver、HandlerExceptionResolver 等）、`HttpMessageConverter`、`HandlerInterceptor`。
- 触发条件：HTTP 请求、异常、跨域、参数校验、文件上传。
- 例外：`@ResponseBody` 不走视图解析；异步返回（Callable、DeferredResult）会释放容器线程。
- 对比取舍：分层清晰便于扩展，代价是组件多、链路长，排查问题时要知道每层职责。

**追问预判**

- 参数校验在哪一步 → 参数解析后由 `@Valid` 触发，失败抛 MethodArgumentNotValidException。
- Filter 和 Interceptor 的区别 → Filter 属于 Servlet 规范、能改请求响应；Interceptor 属于 Spring MVC、能拿到 Handler。
- 跨域在哪处理 → Filter 或 `CorsFilter`，`@CrossOrigin` 最终也是走 MVC 的跨域配置。

**常见错误答法**

- 说 Interceptor 在 Filter 之前执行（Filter 在外层）。
- 认为 @ResponseBody 也会走视图解析（直接写响应体）。

**记忆锚点**

DispatcherServlet 分发 → Mapping 找 Handler → Adapter 执行 → 转换器写响应 → 异常解析器兜底。

---

### 9. 【兜底】Spring Framework：同类型多个 Bean 时 @Primary 和 @Qualifier 谁说了算？

**口述版**

@Qualifier 优先。按类型注入时如果找到多个候选，Spring 先看 @Qualifier 指定的名字，精确匹配就选它；没有 @Qualifier 才看 @Primary，选被标记为默认的那个；两个都没有就按字段名或参数名去匹配 Bean 名字；还匹配不上才抛 NoUniqueBeanDefinitionException。反过来，如果 @Qualifier 指定的名字不存在，会直接报错，不会再回退到 @Primary。实践建议是：基础组件用 @Primary 定一个默认实现，业务上需要精确指定的地方用 @Qualifier，不要依赖字段名匹配，因为改名就会静默改变注入结果，这类问题很难排查。

**短版**

@Qualifier 精确指定优先级最高，其次 @Primary，再按字段名匹配，最后报 NoUniqueBean；@Qualifier 名字不存在直接失败，不回退。

**展开版**

- 机制链：`resolveDependency` 按类型找候选 → 有 @Qualifier 则按名字过滤 → 否则取 @Primary → 再按字段名匹配 → 仍多个则抛异常。
- 关键结构：`DefaultListableBeanFactory.doResolveDependency`、`@Primary`、`@Qualifier`、`@Priority`。
- 触发条件：策略模式多实现、多数据源、多消息模板、测试替换实现。
- 例外：集合注入（`List<T>`）会把所有候选都注入，不受 @Primary 影响。
- 对比取舍：@Primary 省事但隐式，@Qualifier 明确但啰嗦，团队应统一约定避免混用。

**追问预判**

- 注入 List 时 @Primary 有用吗 → 没用，所有实现都会被注入，顺序可用 @Order 控制。
- 字段名匹配可靠吗 → 不可靠，改名会导致注入对象变化，不推荐依赖。
- 两个 Bean 都有 @Primary 会怎样 → 抛异常，@Primary 只能有一个生效。

**常见错误答法**

- 认为 @Primary 优先级高于 @Qualifier（@Qualifier 更精确）。
- 依赖字段名自动匹配作为主要手段（改名字就出问题）。

**记忆锚点**

@Qualifier 精确 → @Primary 默认 → 字段名兜底 → 都不行就报错。

---

## 二、Spring Boot

### 10. 【必答】Spring Boot：@SpringBootApplication 与自动装配链路？

**口述版**

@SpringBootApplication 是三个注解的组合：@SpringBootConfiguration 声明配置类，@ComponentScan 扫描当前包及子包，@EnableAutoConfiguration 开启自动装配。自动装配的链路是：@EnableAutoConfiguration 导入 AutoConfigurationImportSelector，它读取类路径下所有 jar 里的自动配置清单，Boot 2.7 之前是 META-INF/spring.factories，之后改成 META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports；拿到候选类后按条件注解决定是否生效，比如 @ConditionalOnClass、@ConditionalOnMissingBean、@ConditionalOnProperty；最后按 @AutoConfigureOrder、@AutoConfigureBefore/After 排序并注册 BeanDefinition。所以"引入依赖就生效、用户配置优先、可以关闭"这三件事，分别对应条件装配、@ConditionalOnMissingBean 和 exclude 属性。

**短版**

@SpringBootApplication = @SpringBootConfiguration + @ComponentScan + @EnableAutoConfiguration；@EnableAutoConfiguration 经 ImportSelector 读自动配置清单文件，再用条件注解过滤、排序后注册；用户 Bean 优先靠 @ConditionalOnMissingBean。

**展开版**

- 机制链：`@EnableAutoConfiguration` → `AutoConfigurationImportSelector` → 读 `AutoConfiguration.imports` → 条件过滤 → 排序 → 注册定义 → 实例化。
- 关键结构：`ImportSelector`、`AutoConfigurationImportFilter`、`@Conditional*` 家族、`spring.autoconfigure.exclude`。
- 触发条件：应用启动、引入 starter、自定义自动配置。
- 例外：Boot 2.7 起 `spring.factories` 的自动配置写法被废弃，改为 imports 文件；AOT 下条件在构建期求值。
- 对比取舍：自动装配减少配置，代价是启动期扫描多、链路隐式，排查要开 `--debug` 看条件评估报告。

**追问预判**

- 为什么用户自己定义的 Bean 能覆盖自动配置 → 自动配置类上的 `@ConditionalOnMissingBean` 在用户 Bean 注册后不成立。
- 怎么排除某个自动配置 → `@SpringBootApplication(exclude = ...)` 或配置 `spring.autoconfigure.exclude`。
- 条件评估不生效怎么查 → 启动加 `--debug` 输出 ConditionEvaluationReport。

**常见错误答法**

- 认为自动装配是扫描所有类找注解（是读清单文件加条件过滤）。
- 说 @SpringBootApplication 只有 @EnableAutoConfiguration 一个注解（是三个组合）。

**记忆锚点**

三注解组合 → ImportSelector 读清单 → 条件过滤 + 排序 → MissingBean 实现用户优先。

---

### 11. 【必答】Spring Boot：配置加载优先级与 @ConfigurationProperties？

**口述版**

配置优先级大致是：命令行参数最高，然后是 Java 系统属性、环境变量、外部配置文件，最后才是 jar 内的 application 配置文件；同一位置下 profile 专属文件优先于通用文件，后加载的覆盖先加载的。所以线上排障要先怀疑启动命令行和环境变量，而不是只改配置文件。@ConfigurationProperties 是把一组前缀相同的配置绑定成对象，支持松散绑定，比如 user-name、user_name、USER_NAME 都能绑到 userName；它还支持嵌套对象、集合、校验和元数据提示。和 @Value 的区别是：@Value 只能单个注入、启动时求值、不支持松散绑定和复杂类型；@ConfigurationProperties 是批量绑定、可校验、可刷新，更适合配置成组出现的情况。

**短版**

命令行 > 系统属性 > 环境变量 > 外部配置 > jar 内配置，profile 专属优先且后加载覆盖；@ConfigurationProperties 支持松散绑定、嵌套、校验和批量绑定，@Value 只能单值且不支持松散绑定。

**展开版**

- 机制链：`Environment` 聚合多个 PropertySource → 按顺序查找 → 命中即返回 → `Binder` 把前缀属性绑定到对象。
- 关键结构：`PropertySource` 顺序、`spring.profiles.active`、`Binder`、`ConfigurationPropertiesBindingPostProcessor`。
- 触发条件：多环境部署、容器化注入、配置中心接管、需要校验的配置。
- 例外：`@PropertySource` 加载的文件优先级低于 application 配置；系统属性与环境变量对松散绑定的匹配规则不同。
- 对比取舍：外部化配置提升灵活性，代价是来源多、覆盖关系复杂，必须用 Actuator 的 env 端点确认最终值。

**追问预判**

- 为什么改了配置文件不生效 → 命令行或环境变量覆盖了它，查 `/actuator/env` 看来源。
- 松散绑定支持哪些写法 → 中划线、下划线、大小写都能匹配，但前缀必须规范。
- @ConfigurationProperties 能热更新吗 → 单例 Bean 不会自动重建，需要 `@RefreshScope` 或监听环境变更事件。

**常见错误答法**

- 认为 jar 内配置优先级最高（外部配置会覆盖）。
- 用 @Value 绑定复杂对象（它做不到）。

**记忆锚点**

命令行最高 → 环境变量次之 → 外部覆盖内部 → 成组配置用 ConfigurationProperties。

---

### 12. 【必答】Spring Boot：SpringApplication.run 的启动流程？

**口述版**

run 大致分六段：准备环境，创建并配置 Environment，加载配置文件和激活 profile；创建 ApplicationContext，根据是否 Web 应用决定类型；准备上下文，把主配置类注册成 BeanDefinition，执行初始化器；refresh，走的就是 Spring 的 refresh 十二步，这一步会创建所有单例、启动内嵌容器；刷新后回调，执行 Runner；最后发布启动完成事件。要理解的是 Boot 没有另起一套容器，它就是包装了 Spring 的 refresh，把自动装配、内嵌容器、配置加载这些能力通过初始化器和监听器插进去。启动慢基本都出在 refresh 阶段，尤其是扫描和单例创建。

**短版**

准备 Environment → 创建 ApplicationContext → prepareContext 注册主类和初始化器 → refresh 创建单例并启动内嵌容器 → afterRefresh → 执行 ApplicationRunner/CommandLineRunner → 发布就绪事件。

**展开版**

- 机制链：`SpringApplication.run` → `prepareEnvironment` → `createApplicationContext` → `prepareContext` → `refreshContext` → `afterRefresh` → `callRunners`。
- 关键结构：`ApplicationContextInitializer`、`ApplicationListener`、`SpringApplicationRunListener`、`Banner`。
- 触发条件：应用启动、测试上下文加载、命令行启动。
- 例外：`WebApplicationType.NONE/REACTIVE` 会创建不同类型的上下文；AOT 模式会跳过部分运行期处理。
- 对比取舍：统一入口便于扩展，代价是启动链路长、问题定位要先分清是环境准备还是 refresh 阶段。

**追问预判**

- Runner 和监听器的区别 → Runner 在启动完成后执行一次，监听器响应生命周期事件。
- 启动卡住怎么排查 → 看日志停在哪个阶段，堆栈里通常是某个 Bean 初始化或外部连接。
- 内嵌容器什么时候启动 → refresh 的 onRefresh 阶段创建，finishRefresh 时启动完成。

**常见错误答法**

- 认为 Boot 有独立的容器实现（复用 Spring 的 refresh）。
- 在构造函数里做耗时初始化导致启动慢却怪框架。

**记忆锚点**

环境 → 上下文 → prepare → refresh → Runners → 就绪事件。

---

### 13. 【进阶】Spring Boot：内嵌容器、优雅停机与 Actuator 怎么配？

**口述版**

内嵌容器让应用打成可执行 jar 直接运行，Boot 根据依赖自动选择 Tomcat、Jetty 或 Undertow，可以通过 WebServerFactoryCustomizer 改端口、线程数、连接数这些参数。优雅停机是设置 server.shutdown=graceful，并配 spring.lifecycle.timeout-per-shutdown-phase，收到停止信号后先停止接收新请求，等正在处理的请求完成或超时，再关闭容器和销毁 Bean，这样滚动发布时不会出现大量 502。Actuator 提供健康检查、指标、环境、线程栈等端点，生产上要只暴露必要端点并加鉴权，health 要能反映数据库、Redis 这些依赖状态，再接 Prometheus 做监控告警。

**短版**

内嵌容器由依赖自动选择，可自定义参数；优雅停机开 server.shutdown=graceful 并设超时，先拒新请求再等在途完成；Actuator 暴露 health/metrics 等端点，必须限制暴露范围并加鉴权。

**展开版**

- 机制链：收到 SIGTERM → 停止接收新请求 → 等待在途请求 → 关闭 Web 容器 → 销毁 Bean → 退出。
- 关键结构：`WebServerFactoryCustomizer`、`GracefulShutdown`、`management.endpoints.web.exposure.include`、`HealthIndicator`。
- 触发条件：K8s 滚动发布、扩容缩容、发布回滚、健康检查失败。
- 例外：优雅停机的等待时间要小于 K8s 的 terminationGracePeriodSeconds，否则会被强杀。
- 对比取舍：优雅停机减少请求失败，代价是发布变慢；Actuator 提升可观测性，代价是暴露面带来安全风险。

**追问预判**

- 优雅停机没生效怎么查 → 确认是否走了 SIGTERM、配置是否在正确的 profile、等待时间是否够。
- health 检查为什么要包含依赖 → 否则依赖挂了实例仍被判健康，流量继续打进来。
- 怎么防止端点被外网访问 → 单独管理端口、Spring Security 鉴权、网关层禁止转发。

**常见错误答法**

- 认为优雅停机只是延迟关闭（核心是先停止接流量）。
- 把 Actuator 全部端点暴露在公网（会泄露环境和配置）。

**记忆锚点**

graceful 先拒新再等在途 → 超时要小于 K8s 宽限 → Actuator 限暴露加鉴权。

---

### 14. 【进阶】Spring Boot：@SpringBootTest 为什么慢，Boot 3 迁移要注意什么？

**口述版**

@SpringBootTest 默认加载完整上下文，包括自动装配、数据源、Web 环境，一次启动就要几百毫秒到几秒，测试多了非常慢。Spring 会用上下文缓存来复用，缓存的 key 包括配置类、profile、属性、MockBean 等，任何一项不同都会重建上下文，所以滥用 @MockBean、@TestPropertySource 会让缓存失效，测试越写越慢。优化手段是尽量用切片测试，比如 @WebMvcTest、@DataJpaTest，只加载相关层；用 @MockBean 要集中管理；能不起容器就不起。Boot 3 迁移的核心变化是基线升到 Java 17、javax 全面换成 jakarta、自动配置注册文件改为 imports、部分配置属性重命名，升级前要先确认依赖库都有 Jakarta 版本。

**短版**

@SpringBootTest 加载完整上下文，慢在启动；上下文缓存 key 受配置、profile、MockBean 影响，滥用会反复重建；用切片测试和集中 Mock 优化；Boot 3 迁移要处理 Java 17 基线、javax→jakarta、imports 文件和属性重命名。

**展开版**

- 机制链：测试启动 → 计算上下文缓存 key → 命中则复用 → 未命中则完整 refresh → 执行测试。
- 关键结构：`MergedContextConfiguration` 的缓存 key、`@WebMvcTest` 等切片注解、`@MockBean`、`@DynamicPropertySource`。
- 触发条件：集成测试、CI 流水线、多模块项目。
- 例外：`@DirtiesContext` 会主动让上下文失效，能不用就不用。
- 对比取舍：完整上下文贴近真实，代价是慢；切片测试快但覆盖有限，通常两层配合。

**追问预判**

- 为什么加了 @MockBean 测试更慢 → 缓存 key 变化导致上下文重建，每个不同组合都新建一个。
- 切片测试能测什么 → 对应层的组件和配置，依赖用 Mock 替代，不启动完整容器。
- Boot 3 最常见的编译错误 → javax.servlet 等包不存在，需要换成 jakarta 或升级依赖版本。

**常见错误答法**

- 认为 @SpringBootTest 每次都会新建上下文（有缓存，但滥用会失效）。
- 升级 Boot 3 只改版本号（javax 到 jakarta 是破坏性变更）。

**记忆锚点**

完整上下文才慢 → 缓存 key 决定复用 → 切片测试优先 → Boot 3 换 jakarta。

---

### 15. 【兜底】Spring Boot：自定义 starter 怎么做才"引入即生效、可覆盖、可关闭"？

**口述版**

一个标准 starter 分两个模块：autoconfigure 放自动配置类和默认配置，starter 只做依赖聚合。自动配置类上用 @AutoConfiguration 并注册到 META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports，这样引入依赖就会被扫描到。要"可覆盖"，就给默认 Bean 加 @ConditionalOnMissingBean，用户自己定义同类型 Bean 时自动配置让位。要"可关闭"，配置类上加 @ConditionalOnProperty，或者让用户通过 exclude 排除；同时把所有可调参数收进 @ConfigurationProperties，配好元数据方便提示。还要注意自动配置不能无条件依赖用户环境，要用 @ConditionalOnClass、@ConditionalOnWebApplication 这类条件保护，避免引入就报错。

**短版**

autoconfigure 模块放 @AutoConfiguration 并注册到 imports 文件，starter 模块只聚合依赖；可覆盖靠 @ConditionalOnMissingBean，可关闭靠 @ConditionalOnProperty 或 exclude，参数收进 @ConfigurationProperties，并用 @ConditionalOnClass 保护。

**展开版**

- 机制链：引入 starter → 聚合依赖 → 读取 imports 清单 → 条件评估 → 注册默认 Bean → 用户 Bean 覆盖。
- 关键结构：`@AutoConfiguration`、`@ConditionalOnMissingBean`、`@ConditionalOnProperty`、`@ConfigurationProperties`、`AutoConfiguration.imports`。
- 触发条件：公司内部中间件封装、SDK 化、统一埋点与日志组件。
- 例外：`@ConditionalOnMissingBean` 只对同一容器内已注册的 Bean 生效，顺序敏感时要用 `@AutoConfigureAfter`。
- 对比取舍：开箱即用降低接入成本，代价是隐式行为多，必须提供开关、日志和文档，否则排查困难。

**追问预判**

- 为什么用户 Bean 能覆盖默认 Bean → 条件注解在用户定义注册之后评估，不满足就不注册。
- starter 里要不要写代码 → 一般不写，只做依赖聚合，代码放 autoconfigure。
- 条件顺序错了会怎样 → 默认 Bean 先注册，用户 Bean 被跳过或冲突，需要调整 AutoConfigure 顺序。

**常见错误答法**

- 把自动配置类写进 spring.factories 就以为 Boot 3 也能用（新版本要 imports 文件）。
- 默认 Bean 不加 @ConditionalOnMissingBean，导致用户无法覆盖。

**记忆锚点**

autoconfigure + imports → MissingBean 可覆盖 → Property 可关闭 → 条件保护防报错。

---

## 三、Spring Cloud

### 16. 【必答】Spring Cloud：Eureka 与 Nacos 的差别，注册中心怎么选？

**口述版**

Eureka 是 AP 设计，节点之间互相复制注册信息，网络分区时优先保证可用，可能出现短暂的不一致；它靠客户端心跳和自我保护机制维持实例列表，自我保护开启时大量实例不剔除，避免网络抖动误杀。Nacos 同时支持 AP 和 CP，临时实例走 AP 加心跳，持久实例走 CP 加 Raft，还集成了配置中心，一个组件解决注册和配置两件事。选型上看生态：Spring Cloud Alibaba 体系用 Nacos，原生 Netflix 体系用 Eureka，但 Netflix 组件早已停止维护，新项目基本都往 Nacos 或 Consul 走。核心要讲清的是 CAP 取舍和对业务的影响：注册中心短暂不一致会导致调用打到已下线实例，所以客户端要有重试和熔断兜底。

**短版**

Eureka 是 AP、节点复制、靠心跳和自我保护；Nacos 支持 AP/CP 双模式、临时实例 AP 持久实例 CP、还带配置中心；新项目多选 Nacos，注册中心不一致要靠客户端重试和熔断兜底。

**展开版**

- 机制链：服务启动注册 → 定时心跳续约 → 注册中心维护实例列表 → 客户端定时拉取或订阅推送 → 负载均衡选实例。
- 关键结构：Eureka 的 `PeerAwareInstanceRegistry`、自我保护阈值；Nacos 的临时/持久实例、`Raft`、命名空间与分组。
- 触发条件：服务上下线、网络分区、扩缩容、灰度发布。
- 例外：Eureka 2.x 已停止开发；Nacos 持久实例在 CP 模式下分区时可能不可写。
- 对比取舍：AP 保证可用但可能读到旧列表，CP 保证一致但分区时可能拒绝写入，取决于业务能容忍哪种失败。

**追问预判**

- 服务下线了为什么还有流量 → 客户端缓存未刷新、心跳未过期，需要重试、熔断和优雅下线配合。
- Eureka 自我保护要不要关 → 一般保留，关掉后网络抖动会误删大量实例。
- Nacos 临时和持久实例的区别 → 临时靠心跳、掉线即删；持久靠服务端探测、可保留。

**常见错误答法**

- 说注册中心必须强一致（Eureka 是 AP，强一致会牺牲可用性）。
- 认为下线后流量立刻消失（客户端有缓存和刷新间隔）。

**记忆锚点**

Eureka AP 加自我保护 → Nacos AP/CP 双模 → 下线有延迟 → 客户端重试熔断兜底。

---

### 17. 【必答】Spring Cloud：OpenFeign 的原理与超时、重试怎么配？

**口述版**

OpenFeign 的核心是动态代理：启动时扫描 @FeignClient 接口，为每个接口生成代理对象，方法调用时把注解里的信息解析成 RequestTemplate，也就是 HTTP 方法、路径、参数，再经过编码器序列化请求体，交给底层客户端发送；同时通过负载均衡从注册中心拿到实例列表并选一个。所以一次 Feign 调用包含代理、编解码、负载均衡、HTTP 发送四段。超时要分层配置：连接超时和读超时必须分别设，而且 Feign 的超时要大于下游实际 RT，又要小于上游给自己的超时，形成递减。重试要谨慎，默认不重试是有道理的，因为多数接口不幂等，盲目重试会放大故障，必须重试时只对幂等接口开，并配退避和次数上限。

**短版**

Feign 用动态代理把接口方法解析成 RequestTemplate，经编码、负载均衡、HTTP 客户端发出；超时分连接和读取，要按调用链递减；重试默认关闭，只对幂等接口开并配退避上限。

**展开版**

- 机制链：扫描接口生成代理 → 调用时解析元数据生成请求 → Encoder 序列化 → LoadBalancer 选实例 → Client 发送 → Decoder 反序列化。
- 关键结构：`Feign.Builder`、`Contract`、`Encoder/Decoder`、`Client`、`Retryer`、`Request.Options`。
- 触发条件：服务间调用、第三方 HTTP 接口封装、统一鉴权与日志。
- 例外：Spring Cloud 2020 之后 Feign 的超时配置从 Ribbon 迁到 `spring.cloud.openfeign.client.config`。
- 对比取舍：声明式调用简洁，代价是隐式行为多（重试、编码、负载均衡），出问题要逐层确认。

**追问预判**

- Feign 的重试和 HTTP 客户端重试会叠加吗 → 会，两处都开等于乘法放大，必须统一在一层。
- 超时链路怎么设 → 外层 > 内层 > 下游 RT，且下游要能感知上游的剩余时间。
- 怎么给 Feign 加统一请求头 → 用 RequestInterceptor，注意别把不相关头透传出去。

**常见错误答法**

- 默认开启重试（非幂等接口会重复下单）。
- 只配一个总超时（连接和读取应该分开，且要分层）。

**记忆锚点**

动态代理 → RequestTemplate → 编码 + 负载均衡 → 超时分层、重试只给幂等。

---

### 18. 【必答】Spring Cloud：Gateway 的断言、过滤器与边界？

**口述版**

Spring Cloud Gateway 基于 WebFlux 和 Reactor Netty，是非阻塞模型。核心概念是 Route，由 id、目标 uri、断言集合和过滤器集合组成。断言决定请求是否匹配这条路由，比如 Path、Method、Header、Query、时间窗口；过滤器分路由级和全局级，用来改请求头、限流、鉴权、重写路径、熔断降级。它的优势是异步非阻塞、能承接高并发连接，而且和注册中心、限流组件集成方便。边界要说清楚：网关只做流量治理，不该写业务逻辑，因为业务逻辑在网关里难以复用、难以测试，而且一旦出问题影响所有服务；另外网关的连接池和超时参数要单独调，不能沿用业务服务的经验值。

**短版**

Route = id + uri + 断言 + 过滤器；断言做匹配（Path/Method/Header/时间），过滤器做改写、鉴权、限流、熔断；基于 WebFlux 非阻塞；网关只做流量治理，不写业务。

**展开版**

- 机制链：请求进入 → RoutePredicateHandlerMapping 匹配断言 → 过滤器链处理 → 转发到目标服务 → 响应回写。
- 关键结构：`RouteDefinition`、`GlobalFilter`、`GatewayFilter`、`RouteLocator`、`Reactor Netty` 连接池。
- 触发条件：统一入口、灰度路由、鉴权、限流、协议转换。
- 例外：全局过滤器对所有路由生效且顺序敏感；请求体和响应体在网关里读取后需要重新包装才能继续传递。
- 对比取舍：统一治理降低重复建设，代价是网关成为单点和性能瓶颈，必须做高可用和容量评估。

**追问预判**

- 网关里改写请求体为什么会出问题 → 请求体是流，读一次就消费掉，必须缓存后重新构造。
- 全局过滤器和路由过滤器谁先执行 → 按 Order 排序，全局过滤器通常在最外层。
- 为什么不该在网关做鉴权以外的事 → 业务逻辑膨胀后难复用、难测试，故障影响面覆盖所有服务。

**常见错误答法**

- 在网关里直接查数据库拼业务数据（职责越界）。
- 认为网关是阻塞模型（基于 WebFlux 非阻塞）。

**记忆锚点**

断言匹配 → 过滤器链 → 转发；网关只做流量治理，不做业务。

---

### 19. 【进阶】Spring Cloud：熔断状态机与常见限流算法？

**口述版**

熔断器有三个状态：CLOSED 正常放行并统计失败率，失败率超过阈值转 OPEN；OPEN 直接快速失败，不再调用下游，经过一段等待时间转 HALF_OPEN；HALF_OPEN 放少量试探请求，成功就回 CLOSED，失败就回 OPEN。它的价值是防止故障扩散和线程堆积，配合降级返回兜底数据。限流算法有四种：固定窗口计数器简单但有临界突刺；滑动窗口把时间切细，平滑一些；漏桶以固定速率流出，能削峰但会引入延迟；令牌桶按速率放令牌，允许一定突发，是实践中最常用的。落地时要注意限流粒度，按接口、按用户、按 IP，阈值要结合压测和下游容量定，不能凭感觉。

**短版**

熔断 CLOSED→OPEN→HALF_OPEN 三态：超阈值开路、等待后半开试探、成功恢复；限流有固定窗口、滑动窗口、漏桶、令牌桶，令牌桶允许突发最常用；阈值要按压测和下游容量定。

**展开版**

- 机制链：统计滑动窗口内失败率 → 超阈值切 OPEN → 等待后放试探请求 → 成功回 CLOSED / 失败回 OPEN。
- 关键结构：`CircuitBreaker` 的滑动窗口、`Resilience4j`、`Sentinel` 的规则与统计、令牌桶的 `RateLimiter`。
- 触发条件：下游超时、错误率上升、流量突增、依赖故障。
- 例外：慢调用比例也能触发熔断，不只是异常率；HALF_OPEN 放行量过大可能再次打垮下游。
- 对比取舍：熔断保护系统但会损失部分功能，降级策略必须提前设计；限流保稳定但会影响用户体验，要有明确错误码。

**追问预判**

- 熔断和限流的区别 → 熔断是对已故障依赖的快速失败，限流是对入口流量的主动控制。
- 为什么固定窗口有突刺 → 窗口边界前后两个窗口的流量可能叠加到两倍。
- 阈值怎么定 → 压测出下游容量上限，取七到八成作为阈值，再按业务优先级分层。

**常见错误答法**

- 认为熔断打开后永久拒绝（会半开试探恢复）。
- 只按 QPS 限流不看下游连接数和 RT（瓶颈可能在连接池）。

**记忆锚点**

三态：关→开→半开 → 滑动窗口统失败率 → 令牌桶限流 → 阈值来自压测。

---

### 20. 【进阶】Spring Cloud：配置中心与 @RefreshScope 的副作用？

**口述版**

配置中心的价值是集中管理、按环境隔离、变更可审计、支持灰度。客户端启动时拉取配置，之后靠长轮询或推送感知变更。@RefreshScope 的原理是给 Bean 套一层作用域代理，配置变更时销毁该作用域的缓存实例，下次访问重新创建，所以能拿到新配置。副作用有三个：一是有状态的 Bean 被重建会丢运行时状态，比如连接池、线程池、本地缓存；二是重建过程中可能出现短暂的并发问题，正在执行的请求可能拿到旧实例；三是频繁刷新会放大启动成本。更稳的做法是把可动态调整的参数收进 @ConfigurationProperties 并配合 EnvironmentChangeEvent 局部更新，而不是整个 Bean 重建。

**短版**

配置中心集中管理、长轮询或推送变更；@RefreshScope 用作用域代理在刷新时销毁并重建 Bean；副作用是丢运行时状态、重建期间并发不一致、频繁刷新成本高；稳定参数用 ConfigurationProperties 局部更新。

**展开版**

- 机制链：配置变更 → 发布事件 → 销毁 refresh scope 缓存 → 下次获取时重新创建 Bean → 重新绑定属性。
- 关键结构：`RefreshScope`、`Scope` 的 `destroy`、`EnvironmentChangeEvent`、`ConfigurationPropertiesRebinder`。
- 触发条件：动态开关、限流阈值调整、灰度参数、应急降级。
- 例外：连接池、线程池这类资源型 Bean 重建会导致连接中断，通常不应该放在 refresh scope 里。
- 对比取舍：动态配置提升应急能力，代价是状态一致性和并发安全需要额外设计，不能滥用。

**追问预判**

- 为什么连接池刷新后报错 → Bean 被重建，旧连接被关闭，正在使用旧连接的请求失败。
- 怎么做到只改参数不重建 Bean → 用 `@ConfigurationProperties` 加监听环境变更，或自己实现局部刷新。
- 刷新是推还是拉 → Nacos 是长轮询加推送，Apollo 是定时拉加通知，本质都是客户端感知。

**常见错误答法**

- 给所有 Bean 加 @RefreshScope（资源型 Bean 重建会引发故障）。
- 认为配置中心改了本地就会立刻生效（有推送延迟和缓存）。

**记忆锚点**

作用域代理 → 刷新时销毁重建 → 状态会丢 → 资源型 Bean 别放进去。

---

### 21. 【兜底】Spring Cloud：TraceId 怎么跨线程和跨服务透传？

**口述版**

TraceId 的目标是让一次请求在所有服务和线程里都能串起来。跨服务靠协议头透传：入口从请求头取 TraceId，没有就生成一个，写进 MDC 让日志带上，调用下游时再放到请求头里传出去，一般还要带 SpanId 和父 SpanId 表达调用层级。跨线程的难点是 MDC 基于 ThreadLocal，线程池复用后拿不到，所以要在线程切换时做快照传递，比如封装 Runnable 或线程池的 TaskDecorator，在提交时捕获、执行前恢复、执行后清理。异步框架如 CompletableFuture、Reactor 需要各自的上下文传播机制，Reactor 用 context 加 Hooks。落地建议是用 Micrometer Tracing 或 SkyWalking 这类成熟方案，自己实现容易漏埋点。

**短版**

入口取或生成 TraceId 写入 MDC，跨服务放请求头透传并带 SpanId；跨线程要在线程切换时快照恢复（TaskDecorator），Reactor 用 context 传播；建议直接上 Micrometer Tracing 或 SkyWalking。

**展开版**

- 机制链：入口拦截 → 取或生成 TraceId → 写入 MDC 与响应头 → 下游调用注入头 → 下游再取出继续传递。
- 关键结构：MDC、`TaskDecorator`、`RequestInterceptor`、Reactor 的 `Hooks.onEachOperator`、OpenTelemetry 上下文。
- 触发条件：微服务调用、异步任务、消息消费、定时任务。
- 例外：MQ 消费要在消息属性里带 TraceId，线程池要在提交时捕获；WebFlux 不能用 MDC 直接透传。
- 对比取舍：全链路追踪提升排障效率，代价是埋点和上下文传递有一致性成本，漏一处就断链。

**追问预判**

- 异步线程里 TraceId 丢了怎么办 → 用 TaskDecorator 或 TTL 在提交时捕获并在执行前恢复。
- 日志里为什么有的有有的没有 → 日志格式没配 MDC，或线程切换时没恢复上下文。
- 采样率怎么定 → 高流量按比例采样，异常和慢请求强制采样。

**常见错误答法**

- 只在入口打 TraceId，不往下游传（链路断掉）。
- 用静态变量存 TraceId（并发下互相覆盖）。

**记忆锚点**

MDC 存 → 请求头传 → 线程切换做快照 → 成熟框架优先。

---

## 四、Spring Security

### 22. 【必答】Spring Security：过滤器链与认证流程？

**口述版**

Spring Security 的核心是一条过滤器链，通过 DelegatingFilterProxy 把 Servlet 容器和 Spring 的 FilterChainProxy 接起来，FilterChainProxy 再按匹配规则选择一条 SecurityFilterChain 执行。链上常见的有 SecurityContextHolderFilter、UsernamePasswordAuthenticationFilter、ExceptionTranslationFilter、AuthorizationFilter。认证流程是：过滤器把用户名密码封装成 Authentication 交给 AuthenticationManager，实际由 ProviderManager 轮询多个 AuthenticationProvider，常用的 DaoAuthenticationProvider 会用 UserDetailsService 加载用户、用 PasswordEncoder 校验密码，成功后把 Authentication 写进 SecurityContext 并绑定到当前线程。所以记住三个角色：入口过滤器、认证管理器、用户与密码服务。

**短版**

DelegatingFilterProxy → FilterChainProxy → 按匹配选 SecurityFilterChain；认证时过滤器把凭证交给 AuthenticationManager/ProviderManager → DaoAuthenticationProvider → UserDetailsService + PasswordEncoder，成功后写入 SecurityContext。

**展开版**

- 机制链：请求进容器 → 过滤器链 → 认证过滤器构造 Authentication → ProviderManager 认证 → 写入 SecurityContext → 授权过滤器校验。
- 关键结构：`SecurityContextHolder`、`Authentication`、`ProviderManager`、`UserDetailsService`、`PasswordEncoder`。
- 触发条件：表单登录、Basic 认证、JWT 无状态认证、OAuth2 登录。
- 例外：无状态场景用 `SessionCreationPolicy.STATELESS`，SecurityContext 不落 session，每个请求重新认证。
- 对比取舍：链式设计扩展性强，代价是过滤器多、顺序敏感，排查要开 debug 日志看当前命中的链。

**追问预判**

- 认证信息存在哪 → 默认存 SecurityContext，再由 SecurityContextHolder 绑定到线程或 session。
- 为什么自定义过滤器要指定位置 → 链顺序决定逻辑先后，加错位置可能绕过鉴权。
- 无状态服务怎么维持登录 → 每次请求带 JWT，由过滤器解析并构造 Authentication。

**常见错误答法**

- 认为 Security 是一个拦截器（是 Servlet 过滤器链）。
- 把认证和授权混为一谈（认证是你是谁，授权是你能做什么）。

**记忆锚点**

FilterChainProxy → ProviderManager → UserDetailsService + Encoder → SecurityContext。

---

### 23. 【必答】Spring Security：OAuth2 的授权模式与 JWT 结构？

**口述版**

OAuth2 有四种授权模式：授权码模式最常用也最安全，适合有后端的 Web 应用，先拿 code 再换 token，避免 token 暴露在前端；简化模式直接把 token 放在回调地址里，已经废弃；密码模式让客户端拿用户名密码换 token，只适合高度信任的第一方应用，也已不推荐；客户端凭证模式用于服务间调用，没有用户参与。现在推荐授权码加 PKCE，移动端和 SPA 也用这个。JWT 由三段组成：头部声明算法和类型，载荷放声明比如 sub、exp、scope，签名是对前两段用密钥或私钥算出来的，用来防篡改，注意载荷只是 Base64 编码不是加密，敏感信息不能放进去。

**短版**

四种模式：授权码（+PKCE，最常用）、简化（已废弃）、密码（不推荐）、客户端凭证（服务间）；JWT 三段为 header.payload.signature，载荷只编码不加密，签名防篡改。

**展开版**

- 机制链：客户端引导用户到授权页 → 用户同意 → 回调带 code → 客户端用 code 换 token → 带 token 访问资源。
- 关键结构：`authorization_code`、`client_credentials`、PKCE 的 code_verifier/challenge、JWT 的 `alg`、`exp`、`iss`、`aud`。
- 触发条件：第三方登录、开放平台、微服务间鉴权、单点登录。
- 例外：客户端凭证模式拿不到用户身份，只能代表应用本身；JWT 不能主动撤销，只能靠短有效期加黑名单。
- 对比取舍：JWT 无状态、易横向扩展，代价是无法即时失效；不透明 token 可随时撤销，但每次要查存储。

**追问预判**

- 为什么授权码模式最安全 → token 不经过浏览器前端，只通过后端通道交换。
- JWT 能存密码吗 → 不能，payload 只是 Base64 编码，任何人都能解开。
- 怎么让 JWT 提前失效 → 缩短有效期、维护黑名单、或用版本号在网关注销。

**常见错误答法**

- 认为 JWT 是加密的（默认只是签名）。
- 在 SPA 里用密码模式存 token（已不推荐，且暴露风险高）。

**记忆锚点**

授权码 + PKCE → 密码/简化废弃 → 客户端凭证给服务；JWT 三段、只签名不加密。

---

### 24. 【必答】Spring Security：JWT 无状态登录怎么处理注销与续期？

**口述版**

JWT 无状态的代价是无法像 session 那样服务端一删就失效，所以注销和续期要另外设计。常见方案有三种：一是短有效期加 refresh token，访问 token 十几分钟过期，refresh token 有效期长且存在服务端或 Redis，注销时删掉 refresh token，访问 token 靠短有效期自然失效；二是黑名单，注销时把未过期的 token 标识写进 Redis 并设置剩余过期时间，网关或过滤器每次校验，代价是多一次查询；三是版本号方案，用户表存一个 token 版本，签发时写进 JWT，注销时版本加一，校验时比对，适合能接受缓存延迟的场景。续期一般用 refresh token 换新的访问 token，不要用旧 token 自动续，否则等于永不过期。

**短版**

三种方案：短访问 token + 服务端 refresh token（注销删 refresh）、Redis 黑名单（按剩余有效期存）、用户版本号（注销版本加一）；续期用 refresh token 换新 token，别用旧 token 自动续。

**展开版**

- 机制链：登录签发访问 token 与 refresh token → 前端带访问 token → 过期前用 refresh 换新 → 注销时删 refresh 或写黑名单。
- 关键结构：`refresh_token` 存储、Redis 黑名单 key 设计、JWT 的 `jti`、网关校验过滤器。
- 触发条件：用户主动退出、改密码、账号被封、权限变更。
- 例外：多端登录要按设备维度存 token，否则一处注销会踢掉所有端。
- 对比取舍：黑名单能即时失效但有存储和查询成本；版本号成本低但有缓存延迟；短 token 最简单但用户体验依赖续期。

**追问预判**

- 改密码后旧 token 还能用吗 → 默认能，需要版本号或黑名单机制强制失效。
- 黑名单会不会无限增长 → 设置 key 的过期时间为 token 剩余有效期，到期自动清理。
- refresh token 被盗怎么办 → 一次性使用加轮换，检测到重复使用就吊销整个会话。

**常见错误答法**

- 说注销就是把前端 token 删掉（只清客户端，服务端仍认这个 token）。
- 用访问 token 自身续期，等于永不过期，丢失后无法回收（方案错）。

**记忆锚点**

短 token + refresh → 注销删 refresh 或写黑名单 → 版本号强制失效 → refresh 要轮换。

---

### 25. 【进阶】Spring Security：水平与垂直越权怎么防？

**口述版**

垂直越权是低权限用户访问高权限接口，比如普通用户调管理接口；水平越权是同权限用户访问别人的数据，比如改一下订单 ID 就查到别人的订单，也叫 IDOR。垂直越权靠接口级鉴权，用 @PreAuthorize 或网关做角色校验，关键是默认拒绝，新接口默认需要权限而不是默认放行。水平越权靠数据级鉴权，每个按 ID 查询和修改的操作都必须校验数据归属，不能只信前端传来的 ID；做法是在查询条件里强制带上当前用户或租户 ID，或者查出来之后比对 owner，不匹配返回 404 而不是 403，避免泄露资源是否存在。另外批量接口、导出接口、文件下载这些容易被忽略，都要逐个过一遍权限矩阵。

**短版**

垂直越权是越级访问接口，靠默认拒绝加 @PreAuthorize 做接口级鉴权；水平越权是越权访问他人数据（IDOR），靠查询强制带用户或租户条件、查后校验归属；批量、导出、下载接口最容易漏。

**展开版**

- 机制链：请求进入 → 认证拿身份 → 接口级鉴权校验角色 → 数据级校验比对 owner 或租户 → 返回结果。
- 关键结构：`@PreAuthorize`、`SecurityContext`、MyBatis 的租户拦截器、数据权限切面。
- 触发条件：订单、账单、消息、文件等按 ID 访问的资源。
- 例外：管理员代操作要有独立审计和二次确认；共享资源要显式建模授权关系，不能靠 ID 猜。
- 对比取舍：默认拒绝最安全但配置成本高；细粒度数据权限灵活但性能有开销，通常用 SQL 条件而非查后过滤。

**追问预判**

- 为什么返回 404 而不是 403 → 避免攻击者通过状态码探测资源是否存在。
- 只在前端隐藏按钮够吗 → 不够，接口必须独立校验，前端只是体验层。
- 怎么系统性地查漏 → 按接口清单做权限矩阵，对每个 ID 参数检查是否校验了归属。

**常见错误答法**

- 只在列表接口做过滤，详情接口直接用 ID 查（典型 IDOR）。
- 认为用了 JWT 就不会越权（认证不等于授权）。

**记忆锚点**

垂直越权靠角色 → 水平越权靠归属 → 默认拒绝 → 详情和导出接口重点查。

---

### 26. 【兜底】Spring Security：Security 6 的配置变化与升级注意点？

**口述版**

Security 6 最大的变化是配置方式：WebSecurityConfigurerAdapter 被移除，改成声明 SecurityFilterChain Bean 并用 lambda DSL 配置，写法更直观也更难配错，比如 authorizeHttpRequests 里用 requestMatchers 而不是 antMatchers。另外授权过滤器从 FilterSecurityInterceptor 换成了 AuthorizationFilter，默认按请求级授权，性能更好。方法级安全要显式加 @EnableMethodSecurity，旧的 @EnableGlobalMethodSecurity 废弃。升级还要注意：默认要求认证的规则更严格、CSRF 默认开启、跨域要显式配置、密码编码器推荐用 DelegatingPasswordEncoder。迁移时先把配置改成组件式，再逐条核对 URL 规则，避免升级后接口突然 403。

**短版**

Security 6 移除 WebSecurityConfigurerAdapter，改用 SecurityFilterChain + lambda DSL；antMatchers 变 requestMatchers；FilterSecurityInterceptor 换 AuthorizationFilter；方法级用 @EnableMethodSecurity；注意 CSRF、CORS 和默认拒绝规则的变化。

**展开版**

- 机制链：定义 SecurityFilterChain Bean → HttpSecurity 链式配置 → 生成过滤器链 → 请求匹配后鉴权。
- 关键结构：`SecurityFilterChain`、`authorizeHttpRequests`、`requestMatchers`、`AuthorizationFilter`、`@EnableMethodSecurity`。
- 触发条件：框架升级、新项目搭建、旧配置迁移。
- 例外：仍需兼容旧写法时可临时保留适配层，但官方已不推荐；WebFlux 用的是另一套 `ServerHttpSecurity`。
- 对比取舍：新 DSL 更清晰安全，代价是迁移工作量，规则顺序和匹配器的细微差异容易导致 403/401 问题。

**追问预判**

- 升级后接口全 403 怎么查 → 检查 requestMatchers 顺序和是否被默认拒绝规则覆盖，开 Security debug 日志。
- authorizeHttpRequests 和 authorizeRequests 的区别 → 前者基于 AuthorizationFilter，是新版本推荐写法。
- 方法级注解不生效怎么办 → 确认加了 @EnableMethodSecurity，且调用经过代理。

**常见错误答法**

- 继续用 WebSecurityConfigurerAdapter（Security 6 已移除）。
- 认为 CSRF 可以无脑关闭（要按是否有状态和浏览器场景判断）。

**记忆锚点**

SecurityFilterChain Bean → lambda DSL → requestMatchers → AuthorizationFilter → EnableMethodSecurity。

---

