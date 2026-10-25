# Java 基础与集合核心知识点学习文档

Oct 15, 2026 · @meteorzhong

## 导读

本文展开《知识图谱.html》「Java 核心」分类的节点（虚拟线程已在并发文档中覆盖），并对齐《Java开发工程师核心知识点.md》第一、二章。标注沿用图谱：**【核心】** 面试必问；**【重点】** 常被追问；**【了解】** 知道概念即可。默认版本为 JDK 17 / 21。

| 图谱节点 | 级别 | 本文章节 |
| --- | --- | --- |
| Java 语言基础（java-basics） | 核心 | 语言基础 |
| 集合框架（collections） | 核心 | 集合框架与 List、HashMap |
| 泛型与注解（generics） | 重点 | 泛型、注解与函数式 |
| 函数式编程（functional） | 重点 | 泛型、注解与函数式 |
| IO / NIO / AIO（io-nio） | 重点 | 五、IO / NIO（Buffer、Channel、Selector、文件 NIO.2、异步通道） |
| 现代 Java 特性（new-features） | 了解 | 现代 Java 特性 |

**复习顺序**：String 与 equals / hashCode → HashMap → ArrayList → 其余。这部分题目看起来基础，但常被用来「热身」和考察源码功底，尤其是 HashMap，要能讲到字段和方法级别。ConcurrentHashMap 见并发文档第七章。

## 一、语言基础

这一章的题目答案都很短，关键是答准确并能说出背后的原因。

### 1.1 面向对象【核心】

**三大特性**：封装（隐藏内部实现，只暴露必要接口）、继承（复用父类代码）、多态（父类引用指向子类对象，运行时调用子类的方法，靠动态绑定实现）。

**SOLID 原则**：单一职责、开闭原则（对扩展开放、对修改关闭）、里氏替换、接口隔离、依赖倒置（依赖抽象而不是实现）。

| 对比 | 要点 |
| --- | --- |
| 重载 vs 重写【重点】 | 重载：同一类中方法名相同、参数列表不同，编译期确定。重写：子类覆盖父类方法，运行期确定；返回值可以是父类返回值的子类，访问权限不能更严格，不能抛出更宽的受检异常 |
| 抽象类 vs 接口【重点】 | 抽象类是「is-a」，可以有构造器和成员变量，单继承；接口是「can-do」，只有常量，可多实现。JDK 8 起接口可有 default / static 方法，JDK 9 起可有 private 方法 |

**只有值传递【重点】**：Java 只有值传递。传对象时传的是「引用的副本」，所以方法内通过它修改对象的属性会生效，但让它指向新对象不会影响调用方。

**final 与 static【重点】**：final 类不能继承、方法不能重写、变量只能赋值一次（引用不可变，对象内容可变）；static 属于类而非实例，静态方法不能访问实例成员，静态代码块在类初始化时执行一次。

**访问修饰符【了解】**：`private`（本类）→ 默认（同包）→ `protected`（同包 + 子类）→ `public`（全部）。接口方法默认 public，实现时不能降低可见性。

**Object 的常用方法【了解】**：`equals` / `hashCode`（重写必须成对，见 1.3）、`toString`（默认输出类名@哈希，日志里建议重写）、`clone`（浅拷贝，需实现 `Cloneable`）、`finalize`（已废弃，不要使用）、`wait` / `notify`（属于对象锁机制，见并发文档）。

### 1.2 String【核心】

**不可变**：`String` 类是 final 的，内部字符数组 `private final`，且没有提供修改方法。JDK 9 起内部由 `char[]` 改为 `byte[]` + 编码标记（紧凑字符串），纯 Latin-1 字符串只占一半内存。

**为什么要不可变**：可以放入字符串常量池复用；缓存 hashCode，适合做 HashMap 的 key；天然线程安全；作为文件路径、类名等参数时更安全。

| 类 | 可变 | 线程安全 | 场景 |
| --- | --- | --- | --- |
| String | 否 | 是 | 少量、不变的字符串 |
| StringBuilder | 是 | 否 | 单线程大量拼接，最常用 |
| StringBuffer | 是 | 是（方法加 synchronized） | 多线程拼接，很少用 |

**常量池与 intern【重点】**：字面量 `"abc"` 在常量池中；`new String("abc")` 在堆上另建一个对象（常量池中没有时还会先创建字面量，最多 2 个对象）。`intern()` 返回常量池中的引用，没有就把当前字符串的引用放进去（JDK 7 起常量池在堆中）。

```java
String a = "ab";
String b = "a" + "b";        // 编译期常量折叠，等于 "ab"
String c = new String("ab");
String x = "a";
String d = x + "b";           // 含变量，运行期拼接出新对象
System.out.println(a == b);   // true
System.out.println(a == c);   // false
System.out.println(a == d);   // false
System.out.println(a == d.intern()); // true
```

### 1.3 == 与 equals、hashCode【核心】

- `==`：基本类型比值，引用类型比内存地址。
- `equals`：Object 的默认实现就是 `==`，String、包装类等重写为比较内容。
- **约定**：`equals` 相等的两个对象，`hashCode` 必须相等；反过来不要求（哈希冲突）。
- **为什么重写 equals 必须重写 hashCode**：HashMap / HashSet 先用 hashCode 定位桶，再用 equals 比较。只重写 equals 时，两个「相等」的对象会落到不同的桶，HashSet 会存两个、`map.get` 取不到值。
- 用作 Map key 的对象，参与 hashCode 计算的字段放入后不要再修改，否则就「丢」在旧桶里了。JDK 16 的 `record` 会自动生成正确的 equals / hashCode。

### 1.4 包装类与自动装箱【重点】

- 自动装箱调用 `Integer.valueOf()`，拆箱调用 `intValue()`。
- **Integer 缓存**：`valueOf` 对 -128\~127 返回缓存对象，所以 `Integer a = 127, b = 127; a == b` 为 true，128 时为 false。**包装类比较一律用 equals**。
- **拆箱 NPE**：`Integer` 为 null 时自动拆箱会抛空指针，常见于三元表达式和数据库查出的 null 字段。
- 金额不用 double，用 `BigDecimal`，并用 `new BigDecimal("0.1")` 或 `BigDecimal.valueOf(0.1)` 构造，不要用 `new BigDecimal(0.1)`；比较大小用 `compareTo`，`equals` 还会比较精度（`1.0` 不等于 `1.00`）。

### 1.5 异常体系【重点】

`Throwable` 分为 `Error`（JVM 严重问题，如 OOM、StackOverflowError，不应捕获）和 `Exception`。`Exception` 又分为**受检异常**（必须处理，如 IOException、SQLException）和**非受检异常**（`RuntimeException` 及其子类，如 NPE、数组越界）。

- **try-with-resources**（JDK 7）：实现了 `AutoCloseable` 的资源自动关闭，关闭时的异常作为被抑制异常附加到主异常上，不会覆盖主异常。
- **finally 的坑**：finally 中的 return 会覆盖 try 中的 return 和异常；`System.exit()` 或线程被杀时 finally 不执行。
- **规范**：不要用异常做流程控制；不要吞掉异常；包装异常时保留 cause。

### 1.6 反射、拷贝与序列化【重点】

- **反射**：运行时通过 `Class` 对象获取类的信息并操作字段、方法、构造器。获取 Class 的三种方式：`Xxx.class`、`obj.getClass()`、`Class.forName("...")`（会触发类初始化）。是 Spring IOC、AOP、MyBatis 等框架的基础。代价是性能较差（无法内联、有权限检查）和破坏封装。JDK 9 的模块系统限制了对 JDK 内部类的反射访问。
- **深拷贝与浅拷贝**：浅拷贝只复制引用，新旧对象共享内部对象；深拷贝递归复制所有对象。`Object.clone()` 默认是浅拷贝；深拷贝可用序列化或手写拷贝构造器。
- **序列化【了解】**：实现 `Serializable`；显式声明 `serialVersionUID`，否则类结构一变，自动生成的 UID 就变，反序列化失败；`transient` 字段不参与序列化。实际项目中更常用 JSON、Protobuf 等。

### 1.7 内部类与枚举【了解】

**四种内部类**：

| 类型 | 特点 | 典型用途 |
| --- | --- | --- |
| 成员内部类 | 持有外部类实例引用，可直接访问外部成员；`外部类.this` 指代外部对象 | 强耦合的辅助类 |
| 静态内部类 | 不持有外部引用，可独立创建；`Outer.Inner` | 静态内部类单例、Builder |
| 局部内部类 | 定义在方法内，可访问 effectively final 的局部变量 | 极少使用 |
| 匿名内部类 | 没有类名，一次性实现接口或继承类；会生成 `Outer$1.class` | 回调、线程 Runnable |

注意：非静态内部类不能有静态成员（常量除外）；匿名内部类与 Lambda 的区别已在 2.3 说明。

**枚举**：本质是继承 `java.lang.Enum` 的 final 类，每个枚举值都是静态单例。自带 `values()`、`valueOf()`、`name()`、`ordinal()`；可以定义字段、构造器和方法，因此能承载状态与行为（比 int 常量安全，避免非法值）。枚举单例天然线程安全，且能防反射（`newInstance` 抛异常）和反序列化破坏（返回同一实例）；`EnumSet` / `EnumMap` 是专为枚举优化的高性能集合。

### 1.8 SPI 机制【了解】

SPI（Service Provider Interface）是「接口在调用方、实现在提供方」的扩展机制：调用方只依赖接口，运行时从 classpath 发现实现。

- **用法**：在 `META-INF/services/` 下建以接口全限定名命名的文件，内容是实现类全名；`ServiceLoader.load(Xxx.class)` 遍历加载并实例化。
- **为什么需要它**：父加载器（启动类加载器）加载了接口，却看不到子加载器（应用类加载器）里的实现，所以 `ServiceLoader` 用**线程上下文类加载器**来加载实现类，这也是「打破双亲委派」的典型场景。
- **典型应用**：JDBC 的 `DriverManager`（各数据库驱动 jar 内声明 `java.sql.Driver` 实现）、Dubbo 扩展点（`@SPI` / `@Adaptive`）、Spring Boot 3 的 `AutoConfiguration.imports`（思路与 SPI 相同）。
- **局限**：会一次性实例化所有实现、无法按名称或优先级选择；Dubbo SPI 做了改进（按名加载、支持 IOC / AOP、用 Wrapper 包装）。

## 二、泛型、注解与函数式编程

这三部分都是「重点」级别，常以「为什么这样写会报错 / 会出问题」的形式考。

### 2.1 泛型与类型擦除【重点】

**类型擦除**：泛型只存在于编译期，编译后 `List<String>` 和 `List<Integer>` 都是 `List`，类型参数被替换为上界（默认 Object），并在使用处插入强制转换。这是为了兼容 JDK 5 之前的代码。

**由此带来的限制**：不能用基本类型做类型参数（`List<int>`）；不能 `new T()` 或 `new T[]`；不能 `instanceof List<String>`；静态成员不能使用类的类型参数；不能仅靠泛型参数不同来重载方法。用反射可以往 `List<String>` 里放入 Integer。

**通配符与 PECS**（Producer Extends, Consumer Super）：

- `<? extends T>`：只能读取为 T，不能写入（不知道具体是哪个子类），适合作为数据的生产者。
- `<? super T>`：可以写入 T，读出来只能是 Object，适合作为数据的消费者。
- 典型例子：`Collections.copy(List<? super T> dest, List<? extends T> src)`。

### 2.2 注解【重点】

注解本质是继承 `Annotation` 的特殊接口，本身不产生行为，需要有代码去读取它。

- `@Retention`：生命周期。`SOURCE`（仅源码，如 `@Override`、Lombok）、`CLASS`（默认，写入 class 但运行时不可见）、`RUNTIME`（运行时可通过反射读取，Spring 注解都是这种）。
- `@Target`：作用位置（类、方法、字段、参数等）。
- **处理方式**：运行时反射读取（Spring、自定义 AOP 切面）；编译期注解处理器 APT 生成或修改代码（Lombok 修改抽象语法树、MapStruct 生成映射类）。

### 2.3 Lambda 与函数式接口【重点】

Lambda 是函数式接口（只有一个抽象方法的接口，可加 `@FunctionalInterface` 校验）的简写。Lambda 中引用的局部变量必须是 effectively final。与匿名内部类的区别：Lambda 不生成额外的 class 文件，而是通过 `invokedynamic` 在运行时生成；Lambda 中的 `this` 指向外部类。

| 接口 | 方法 | 含义 |
| --- | --- | --- |
| `Function<T, R>` | `R apply(T)` | 转换 |
| `Consumer<T>` | `void accept(T)` | 消费 |
| `Supplier<T>` | `T get()` | 提供 |
| `Predicate<T>` | `boolean test(T)` | 判断 |

### 2.4 Stream【重点】

- **惰性求值**：中间操作（`filter`、`map`、`sorted`）只构建流水线，遇到终端操作（`collect`、`forEach`、`count`）才执行，并且是逐元素穿过整条流水线，而不是逐个操作遍历。
- **Stream 只能消费一次**。
- **`Collectors.toMap` 的坑**：key 重复时抛 `IllegalStateException`，要传入合并函数 `(a, b) -> a`；value 为 null 时抛 NPE。
- **parallelStream 的坑**：默认使用全局共享的 `ForkJoinPool.commonPool()`，放入阻塞 I/O 会拖慢整个 JVM 的并行任务；在并行流中操作非线程安全的集合会出错；数据量小时反而更慢。

### 2.5 Optional【重点】

用于显式表达「方法返回值可能为空」。正确用法：`map`、`filter`、`orElse`、`orElseGet`（默认值计算代价大时用，`orElse` 的参数不管是否为空都会执行）、`orElseThrow`。反模式：先 `isPresent()` 再 `get()`；用 Optional 做字段或方法参数；返回集合时用 Optional（应该返回空集合）。

## 三、集合框架总览与 List

集合分为两大体系：`Collection`（存单个元素，分为 List、Set、Queue）和 `Map`（存键值对）。选择集合时先问三个问题：是否需要键值对、是否允许重复、是否需要有序。

### 3.1 常用集合速查【重点】

| 接口 | 实现 | 底层 | 特点 |
| --- | --- | --- | --- |
| List | ArrayList | 动态数组 | 随机访问 O(1)，最常用 |
| List | LinkedList | 双向链表 | 头尾增删 O(1)，实际很少用 |
| Set | HashSet | HashMap（value 是同一个占位对象） | 无序去重 |
| Set | LinkedHashSet | LinkedHashMap | 按插入顺序 |
| Set | TreeSet | TreeMap（红黑树） | 排序 |
| Map | HashMap | 数组 + 链表 + 红黑树 | 最常用，允许一个 null key |
| Map | LinkedHashMap | HashMap + 双向链表 | 保持插入或访问顺序 |
| Map | TreeMap | 红黑树 | 按 key 排序，支持范围查询 |
| Queue | ArrayDeque | 循环数组 | 栈和队列的首选（替代 Stack、LinkedList） |
| Queue | PriorityQueue | 二叉堆 | 优先级队列、Top K |

线程安全版本（ConcurrentHashMap、CopyOnWriteArrayList、BlockingQueue 等）见并发文档。`Vector`、`Hashtable`、`Stack` 是早期的全方法加锁实现，已不推荐使用。

### 3.2 ArrayList 扩容机制【重点】

- 无参构造时内部数组是空数组，**第一次 add 时才初始化为容量 10**。
- 容量不足时扩容为原来的 **1.5 倍**（`oldCapacity + (oldCapacity >> 1)`），用 `Arrays.copyOf` 复制到新数组。
- 中间插入和删除要用 `System.arraycopy` 移动后面的元素，O(n)。
- **优化**：知道大致数量时指定初始容量，避免多次扩容和复制。

### 3.3 ArrayList 与 LinkedList【核心】

| 维度 | ArrayList | LinkedList |
| --- | --- | --- |
| 底层 | 连续数组 | 双向链表 |
| 随机访问 | O(1) | O(n) |
| 尾部插入 | 均摊 O(1) | O(1) |
| 中间插入 / 删除 | O(n)，要移动元素 | 定位 O(n) + 修改指针 O(1) |
| 内存 | 紧凑，缓存友好 | 每个节点多两个指针，内存不连续 |

**结论**：几乎所有场景都用 ArrayList。LinkedList 的中间插入优势在实际中被「先遍历定位」和「缓存不友好」抵消，实测通常也慢于 ArrayList。需要队列或双端队列时用 ArrayDeque。

### 3.4 fail-fast 与 fail-safe【重点】

- **fail-fast**：`ArrayList`、`HashMap` 的迭代器在遍历时检查 `modCount`，发现集合被迭代器之外的方式修改就抛 `ConcurrentModificationException`。所以**不要在 foreach 循环中调用 `list.remove()`**，应该用 `iterator.remove()` 或 `list.removeIf(...)`。单线程也会触发，它是一种尽力而为的错误检测，不是并发保障。
- **fail-safe**：`CopyOnWriteArrayList`、`ConcurrentHashMap` 等遍历的是快照或弱一致视图，不会抛异常，但可能看不到最新修改。

### 3.5 常见坑【重点】

- `Arrays.asList()` 返回的是固定大小的内部类，调用 add / remove 抛 `UnsupportedOperationException`；传入基本类型数组时整个数组变成一个元素。
- `List.of()`（JDK 9）创建不可变集合，且不允许 null。
- `subList()` 返回的是原列表的视图，修改原列表结构后再操作它会抛异常。
- `list.toArray(new String[0])` 传空数组即可，性能不差。

### 3.6 Iterator 与 Comparable / Comparator【了解】

**Iterator**：`Collection` 继承 `Iterable`，遍历统一走 `hasNext()` / `next()` / `remove()`；增强 for 循环编译后就是用迭代器。`remove()` 是迭代过程中安全删除元素的唯一方式（直接调 `list.remove` 会触发 fail-fast，见 3.4）。`ListIterator` 额外支持双向遍历和 `set` / `add`。

**Comparable 与 Comparator**：

| 维度 | Comparable | Comparator |
| --- | --- | --- |
| 位置 | 实现在元素类内部 | 独立的比较器对象 |
| 方法 | `compareTo(T o)` | `compare(T a, T b)` |
| 排序 | 自然排序，只有一种 | 可定义多种排序 |
| 使用 | `TreeMap` / `TreeSet` 默认使用 | 可传入 `TreeMap` 构造器、`sort`、`Stream.sorted` |

优先用 `Comparator`：`Comparator.comparing(Order::getAmount).reversed().thenComparing(Order::getId())`，还可配合 `nullsFirst` / `nullsLast`。注意 `compareTo` 与 `equals` 的一致性——`BigDecimal` 就是 `compareTo` 忽略精度而 `equals` 比较精度，放进 `TreeSet` 和 `HashSet` 行为会不同。

## 四、HashMap 与其他 Map

HashMap 是集合部分的重中之重，面试常从它一路追问到 ConcurrentHashMap。以下以 JDK 8 为准。

### 4.1 底层结构【核心】

- **数组 + 链表 + 红黑树**：`Node<K,V>[] table`，每个桶是一条链表。
- **树化**：链表长度 **≥ 8** 且数组长度 **≥ 64** 时转为红黑树；数组不到 64 时优先扩容。红黑树节点 **≤ 6** 时退化为链表（留出差距避免反复转换）。
- **为什么阈值是 8**：哈希分布均匀时，桶中元素个数服从泊松分布，长度达到 8 的概率只有千万分之一。树化是为了应对哈希函数很差或被恶意攻击的极端情况，把最坏情况从 O(n) 降到 O(log n)。
- **默认参数**：初始容量 16（第一次 put 时才创建数组），负载因子 0.75，阈值 = 容量 × 0.75。负载因子 0.75 是空间和冲突率之间的折中。

&#91;embedded content: HashMap 结构示意 · 容量 8 仅为便于展示\]

图中容量画成 8 只为方便展示；实际树化要求数组长度至少 64。

### 4.2 hash 计算与容量为 2 的幂【核心】

```java
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);  // 扰动函数
}
// 桶下标：(n - 1) & hash
```

- **扰动函数**：高 16 位异或低 16 位。因为下标只用到 hash 的低几位，让高位也参与运算可以减少冲突。
- **容量为什么是 2 的幂**：① `(n - 1) & hash` 等价于 `hash % n`，位运算更快；② 扩容时元素要么留在原位置、要么移到「原位置 + 旧容量」，不用重新计算 hash。指定容量不是 2 的幂时会被调整为大于它的最小 2 的幂（`tableSizeFor`）。

### 4.3 put 流程【核心】

1. 计算 key 的 hash。
2. table 为空则调用 `resize()` 初始化。
3. 用 `(n - 1) & hash` 定位桶，**桶为空就直接放入新节点**。
4. 桶不为空：首节点 key 相同（hash 相等且 `==` 或 `equals`）则记下待覆盖；是树节点则按红黑树插入；否则遍历链表，找到相同 key 就覆盖，找不到就**尾插**，插入后长度达到 8 则尝试树化。
5. key 已存在则替换 value 并返回旧值。
6. `++size > threshold` 则扩容。

### 4.4 扩容【重点】

扩容为原来的 **2 倍**，然后把每个桶的元素重新分配。利用容量是 2 的幂的特点，判断 `hash & oldCap`：结果为 0 的留在原索引（低位链表），不为 0 的放到「原索引 + oldCap」（高位链表），一个桶拆成两条链表且**保持原有顺序**。

**初始容量怎么设**：预计放 n 个元素时，初始容量设为 `n / 0.75 + 1`，或直接用 JDK 19 的 `HashMap.newHashMap(n)`。

### 4.5 线程不安全【核心】

- **JDK 7 扩容死循环**：JDK 7 用头插法，扩容时链表顺序会反转；两个线程同时扩容时可能形成环形链表，之后 get 会死循环，CPU 100%。JDK 8 改为尾插并保持顺序，解决了这个问题。
- **JDK 8 仍然不安全**：两个线程同时 put 到同一个空桶，后者会覆盖前者，**丢失数据**；`++size` 不是原子操作，size 不准。
- **解决**：用 `ConcurrentHashMap`（见并发文档 7.1 节）。`Collections.synchronizedMap` 和 `Hashtable` 是整表一把锁，性能差。

### 4.6 各种 Map 对比【重点】

| 维度 | HashMap | Hashtable | LinkedHashMap | TreeMap | ConcurrentHashMap |
| --- | --- | --- | --- | --- | --- |
| 顺序 | 无序 | 无序 | 插入顺序或访问顺序 | key 排序 | 无序 |
| null key / value | 允许 / 允许 | 不允许 | 允许 | key 不允许 | 不允许 |
| 线程安全 | 否 | 是（全表锁） | 否 | 否 | 是（CAS + 桶锁） |
| 复杂度 | O(1) | O(1) | O(1) | O(log n) | O(1) |

**WeakHashMap【了解】**：key 是**弱引用**，当 key 不再被外部强引用时，GC 会回收它，对应的 entry 随之被移除（内部用 `ReferenceQueue` 清理）。适合做「附加在对象上的元数据」这类缓存；注意 value 不能强引用 key，否则形成循环引用导致永远无法回收，而且 `size()` 的结果随时可能变化，不能用于需要精确计数的场景。

**ConcurrentHashMap 的版本差异**：JDK 7 是 Segment 分段锁（默认 16 段，锁粒度是段）；JDK 8 改为「CAS + synchronized 锁单个桶头节点」，并发度等于桶数量，并引入 `ForwardingNode` 协助扩容、`sizeCtl` 控制初始化与扩容状态。详见并发文档。

**用 LinkedHashMap 实现 LRU 缓存【重点】**：

```java
class LruCache<K, V> extends LinkedHashMap<K, V> {
    private final int capacity;

    LruCache(int capacity) {
        super(16, 0.75f, true);   // accessOrder = true：按访问顺序，最近访问的移到队尾
        this.capacity = capacity;
    }

    @Override
    protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
        return size() > capacity;  // 超过容量时删除队首（最久未访问）
    }
}
```

手写 LRU 的话用「HashMap + 双向链表」，get 和 put 都是 O(1)，是高频算法题。

## 五、IO / NIO 与现代 Java 特性

IO 模型是理解 Netty、Redis、Kafka 等高性能组件的基础；现代 Java 特性则考察你是否在跟进新版本。

### 5.1 BIO、NIO、AIO【核心】

| 模型 | 特点 | 线程模型 | 场景 |
| --- | --- | --- | --- |
| BIO（同步阻塞） | `read` 没有数据时线程一直阻塞 | 一个连接一个线程，连接多时线程爆炸 | 连接少、简单的场景 |
| NIO（同步非阻塞 + I/O 多路复用） | Selector 一个线程监听多个 Channel，哪个就绪处理哪个 | 少量线程处理大量连接 | 高并发网络服务，Netty 的基础 |
| AIO（异步非阻塞） | 发起读写后立即返回，完成后回调 | 系统完成 I/O 后通知 | Linux 上底层仍是 epoll 模拟，实际使用少 |

**先分清两组概念**：阻塞 / 非阻塞说的是「调用后线程是否被挂起」；同步 / 异步说的是「数据从内核到用户缓冲区这一步由谁完成」。NIO 是同步非阻塞——`read` 立即返回 0，但内核数据就绪后仍要用户线程自己调 `read` 把数据拷进 `ByteBuffer`；AIO 才是真正的异步，内核把数据拷完再回调。所以「NIO 是非阻塞的，但仍属于同步 I/O」。

`SocketChannel.configureBlocking(false)` 才是 NIO 非阻塞模式的开关，默认仍是阻塞的。

### 5.2 Buffer：NIO 的数据载体【核心】

**Channel 只认 ByteBuffer**，其他类型的 Buffer 只用于内存内数据加工。

| 类型 | 说明 |
| --- | --- |
| `ByteBuffer` | 唯一能与 Channel 直接交互的缓冲；支持 `allocateDirect` |
| `CharBuffer` / `ShortBuffer` / `IntBuffer` / `LongBuffer` / `FloatBuffer` / `DoubleBuffer` | 按基本类型读写，常用于编解码、结构化数据拼装 |
| `MappedByteBuffer` | `FileChannel.map` 返回，把文件区域映射为堆外内存 |

**四个核心属性**：`capacity`（容量，创建后不变）、`limit`（可读写上限）、`position`（下一个读写位置）、`mark`（临时记录点）。不变式：`0 <= mark <= position <= limit <= capacity`。

| 方法 | 作用 | 典型时机 |
| --- | --- | --- |
| `flip()` | `limit = position; position = 0`，从「写模式」切到「读模式」 | 写完数据准备读 / 发给 Channel 前 |
| `clear()` | `position = 0; limit = capacity`，丢弃未读数据 | 一次读写完成后复用缓冲区 |
| `rewind()` | `position = 0`，保留 limit，可重读一遍 | 需要重复读取同一段数据 |
| `compact()` | 把未读数据移到开头，`position = 剩余长度`，`limit = capacity` | **读了一半就要继续写**（如 HTTP 响应还没写完又来数据） |
| `mark()` / `reset()` | 记住 / 回到 mark 位置 | 解析报文时试探性读取 |

`slice()` 以当前 position 到 limit 为范围创建一个新 Buffer，与 `duplicate()` 一样**共享底层数组**，只复制索引，属于零拷贝手段；`asReadOnlyBuffer()` 用于防止下游修改数据。

**Scattering Reads / Gathering Writes**：`read(ByteBuffer[])` 按数组顺序把数据依次填满各个 Buffer（如先 8 字节头部、再变长 body）；`write(ByteBuffer[])` 把多个 Buffer 一次写出。两者都是一次系统调用，减少调用次数与数据拷贝。

**直接缓冲区与非直接缓冲区【重点】**：

| 维度 | 堆内 `allocate()` | 堆外 `allocateDirect()` |
| --- | --- | --- |
| 分配 / 回收成本 | 低（JVM 管理，GC 回收） | 高（`malloc` + `Cleaner`，随 Full GC 或显式释放） |
| I/O 拷贝 | 写 Socket 时需先拷到临时直接内存 | 可直接交给内核，少一次拷贝 |
| 适用 | 短生命周期、频繁创建 | 长连接、网络读写、大文件 |

**非直接缓冲（以及直接缓冲）的回收延迟处理**：`allocateDirect` 的回收依赖 `Cleaner`（`sun.misc.Unsafe.invokeCleaner` 或反射调 `clean`，JDK 9+ 推荐 `Unsafe`），只调 `System.gc()` 不可靠且代价高。工程上的正确做法是**复用**：用池化（Netty 的 `PooledByteBufAllocator`）或自己维护 Buffer 池，并把 `-XX:MaxDirectMemorySize` 设为可控上限，监控堆外内存。

**大文件高效读写**：`FileChannel` + 固定大小 `ByteBuffer` 循环读写（配合 `transferTo` 直接通道间传输），或 `MappedByteBuffer` 做随机访问；避免一次 `Files.readAllBytes` 把整个文件读进堆。

### 5.3 Channel 与文件 NIO.2【重点】

| 维度 | Stream | Channel |
| --- | --- | --- |
| 方向 | 单向（输入流 / 输出流） | **双向**，同一通道可读可写 |
| 阻塞 | 只能阻塞 | 可阻塞也可非阻塞（`configureBlocking(false)`） |
| 与 Buffer | 直接读写字节 / 字符 | 必须经过 Buffer |
| 复用与异步 | 不支持 | 支持多路复用，NIO.2 支持异步 |

**四种主要 Channel**：`FileChannel`（文件）、`SocketChannel`（TCP 客户端）、`ServerSocketChannel`（TCP 服务端，只处理 `OP_ACCEPT`）、`DatagramChannel`（UDP）。Channel 与 Buffer 的关系是「通道负责搬运，缓冲区负责存放」，一次读写总是成对出现。

`FileChannel` 的核心能力：`read/write`、`position/size/truncate`、`force`（刷盘）、`transferTo/transferFrom`（通道间直接传输，底层可走 sendfile）、`map`（映射文件）、`lock`（文件锁）。

**`MappedByteBuffer`**：用 `mmap` 把文件映射到进程虚拟地址空间，读写像操作数组，适合大文件随机读写与索引文件（如 Kafka 的索引、RocketMQ 的 CommitLog 写）。注意：映射内存不在堆内、回收时机不确定，文件被映射期间删除 / 截断在 Windows 上会失败，用完最好显式 `unmap`。

**NIO.2 文件 API（`Path` / `Paths` / `Files`）**：`Path` 取代 `File` 表示路径，`Files` 提供 `readAllBytes`、`newBufferedReader`、`lines`、`walk`、`copy`、`move`、`delete`、`createDirectories`、`readAttributes` 等静态方法；`Files.walk` + `try-with-resources` 是遍历目录的标准写法。

**`FileAttributeView`**：批量读写文件属性。常见实现：`BasicFileAttributeView`（大小、时间、类型）、`PosixFileAttributeView`（Linux 权限、owner、group）、`DosFileAttributeView`（隐藏、只读、存档）、`UserDefinedFileAttributeView`（自定义扩展属性，如打标签）。

**`WatchService` 文件监控**：`FileSystems.getDefault().newWatchService()`，目录 `register` 时指定 `ENTRY_CREATE` / `ENTRY_MODIFY` / `ENTRY_DELETE`，`take()` 阻塞获取、`poll()` 非阻塞。Linux 底层是 inotify，macOS 是 FSEvents / kqueue。坑：事件会合并、同一修改可能重复触发、只监控注册的那一层目录；生产上更常用 Filebeat / inotify 封装库。

**`FileLock` 文件锁**：`channel.lock()`（阻塞）/ `tryLock()`（非阻塞）加锁，支持共享锁与排他锁。它是**进程间**锁，不是线程锁——同一 JVM 内对同一文件重复加锁会抛 `OverlappingFileLockException`，线程间互斥要用 `ReentrantLock`。

**`Charset` 编解码**：`Charset.forName("UTF-8")`（或 `StandardCharsets.UTF_8`）→ `newEncoder()/newDecoder()` → `encode(CharBuffer)` 得 `ByteBuffer`、`decode(ByteBuffer)` 得 `CharBuffer`。解码要处理 `MalformedInputException` / `UnmappableCharacterException`，可用 `CodingErrorAction.REPLACE` 容错；网络传输中「半包」导致的半个多字节字符，必须等下一批数据补齐再解码。

**`Pipe` 管道**：单向，一对 `source` / `sink` 两个 Channel，用于**同一 JVM 内线程间**传数据（跨进程用 Socket）。注意 `sink` 写满后会阻塞，读者必须及时消费。

**线程间数据共享的三种方式**：`Pipe`（字节流）、`Exchanger`（成对交换对象）、`BlockingQueue`（最常用）。

### 5.4 Selector 与多路复用【核心】

Selector 让**一个线程管理多个 Channel**：Channel 注册到 Selector 并声明关心的事件，`select()` 阻塞等待，返回后遍历 `selectedKeys()` 处理就绪事件。

```java
Selector selector = Selector.open();
serverChannel.configureBlocking(false);
serverChannel.register(selector, SelectionKey.OP_ACCEPT);

while (true) {
    selector.select();                       // 阻塞直到有事件就绪
    Iterator<SelectionKey> it = selector.selectedKeys().iterator();
    while (it.hasNext()) {
        SelectionKey key = it.next();
        it.remove();                         // 必须先移除，否则重复处理
        if (key.isAcceptable()) { /* accept 新连接并注册 OP_READ */ }
        else if (key.isReadable()) { /* 读数据，注意 read 返回 -1 表示对端关闭 */ }
    }
}
```

**四种事件**：`OP_ACCEPT`（服务端有新连接）、`OP_CONNECT`（客户端连接完成）、`OP_READ`（可读）、`OP_WRITE`（可写）。**`OP_WRITE` 绝大多数时间都是就绪的**，常驻注册会导致 `select()` 一直空转占满 CPU，正确做法是「写不完时才注册，写完立即取消」。

| 方法 | 行为 |
| --- | --- |
| `select()` | 阻塞，直到至少一个事件就绪 |
| `select(long timeout)` | 最多阻塞 timeout 毫秒，超时返回 0 |
| `selectNow()` | 不阻塞，立即返回就绪数量 |
| `wakeup()` | 唤醒阻塞在 `select()` 上的线程（另一个线程退出 / 提交任务时用） |

**`SelectionKey`**：保存 `interestOps`（关心的事件）、`readyOps`（就绪的事件）、`attachment`（可挂业务对象，如会话状态）、对应的 Channel 与 Selector。事件回调里处理完要判断「是否还有兴趣继续」，必要时 `key.cancel()`。

**安全地取消与关闭【重点】**：先 `key.cancel()`，再把 Channel 关闭；被取消的 key 会在下一次 `select()` 时从内部集合清理，**不要在遍历 `selectedKeys()` 的过程中直接 `remove` 以外的结构修改**（用 `Iterator.remove()`）。`channel.close()` 会自动取消该 Channel 的所有 key；关闭后对 key 的操作会抛 `CancelledKeyException`，处理事件前可先 `key.isValid()` 判断。

**Selector 性能优化**：① 不在 Selector 线程里做耗时业务（丢线程池）；② 不常驻注册 `OP_WRITE`；③ 用直接缓冲区，必要时池化；④ 边缘触发下一次读到 `read == 0` 为止；⑤ 合理设置线程数与 `SO_BACKLOG`；⑥ 规避 JDK 空轮询（Netty 通过重建 Selector 解决）。

**SocketChannel / ServerSocketChannel 非阻塞编程**：`accept()` 可能返回 `null`（无新连接），`read()` 返回 0（暂无数据）或 -1（对端关闭）；连接未完成时 `finishConnect()` 返回 false，需注册 `OP_CONNECT` 等下次事件再 finish。

**UDP 通信**：`DatagramChannel.open().bind(port)`，`configureBlocking(false)` 后 `register(selector, OP_READ)`，用 `receive(ByteBuffer)` / `send(ByteBuffer, address)` 收发；UDP 无连接、有消息边界，不存在粘包但可能丢包乱序。

**SSL/TLS 加密通信**：NIO 下用 `SSLEngine` 做非阻塞握手，`wrap()` / `unwrap()` 在 `ByteBuffer` 之间转换，需要处理 `NEED_TASK`（可能要执行委托任务）、`BUFFER_OVERFLOW`（放大缓冲区）、`BUFFER_UNDERFLOW`（数据不够，等下次读）。手写 `SSLEngine` + Selector 状态机非常复杂，生产上直接用 Netty 的 `SslHandler`。

**select / poll / epoll【核心】**：

| 机制 | 连接上限 | 找就绪连接 | 数据拷贝 |
| --- | --- | --- | --- |
| select | 1024（fd\_set 位图） | 遍历全部 O(n) | 每次把整个集合拷贝到内核 |
| poll | 无（链表） | 遍历全部 O(n) | 同上 |
| epoll | 无 | 内核通过回调把就绪的 fd 放入就绪队列，只返回就绪的 | fd 只注册一次（红黑树），不重复拷贝 |

epoll 支持水平触发（LT，默认，只要有数据就一直通知）和边缘触发（ET，只在状态变化时通知一次，必须一次读完，Netty 的 epoll 传输用 ET）。

**为什么 epoll 高效**：fd 只在注册时拷贝一次，用红黑树管理；内核通过回调把就绪 fd 放进就绪队列，`epoll_wait` 只返回就绪的那几个，开销与活跃连接数相关而不是总连接数。

### 5.5 Reactor 模型【重点】

单 Reactor 单线程（Redis 6 之前）→ 单 Reactor 多线程（业务丢线程池，Accept 与读写仍在一个 Reactor）→ 主从 Reactor（Netty：`BossGroup` 负责 accept，`WorkerGroup` 负责读写）。

- **单 Reactor 单线程**：实现最简单，任一慢操作阻塞全链路，无法利用多核。
- **单 Reactor 多线程**：I/O 与业务分离，但一个 Reactor 仍要处理所有连接的读写事件，高并发下是瓶颈。
- **主从 Reactor**：Accept 与读写分离，是最主流的高并发网络模型，详见 Netty 文档第一章。

### 5.6 零拷贝【重点】

传统的「读文件 + 发网络」有 4 次拷贝和 4 次上下文切换。`mmap` 把内核缓冲区映射到用户空间，省一次 CPU 拷贝（RocketMQ 用它读写 CommitLog）；`sendfile` 直接在内核中从文件传到 socket，配合 DMA gather 可以做到零 CPU 拷贝（Kafka 用 `transferTo`）。`DirectByteBuffer` 分配堆外内存，也能少一次堆内到堆外的拷贝。

### 5.7 异步通道与 NIO.2 异步 I/O【了解】

`AsynchronousFileChannel`、`AsynchronousSocketChannel`、`AsynchronousServerSocketChannel` 发起读写后立即返回，结果通过两种方式获取：

| 方式 | 写法 | 特点 |
| --- | --- | --- |
| `Future` | `Future<Integer> f = channel.read(...)`，再 `f.get()` | 写法简单，但 `get()` 会阻塞，容易退化成同步 |
| `CompletionHandler` | `channel.read(buf, pos, attachment, handler)` | 回调式，真正异步；`completed` 成功、`failed` 处理异常 |

注意 Linux 上 JDK 的异步文件通道是**用线程池模拟**的（并非内核 AIO），所以「AIO 比 NIO 快」并不成立；这也是 Netty 选择 NIO 而不是 AIO 的原因之一——AIO 在 Linux 上不成熟、编程复杂、性能优势不明显。

### 5.8 现代 Java 特性【了解】

| 版本 | 重要特性 |
| --- | --- |
| JDK 8（LTS） | Lambda、Stream、Optional、接口 default 方法、新日期 API `java.time`、元空间 |
| JDK 9 | 模块系统、`List.of` 等不可变集合工厂、String 紧凑存储、G1 成为默认 |
| JDK 10 / 11（LTS） | `var` 局部变量类型推断、标准 HTTP Client、String 新方法（`isBlank`、`strip`） |
| JDK 14 \~ 17（LTS） | Switch 表达式、文本块 `"""`、**record**、**sealed 类**、instanceof 模式匹配、NPE 详细提示 |
| JDK 21（LTS） | **虚拟线程**、switch 模式匹配、record 模式、有序集合 `SequencedCollection`、分代 ZGC |
| JDK 25（LTS） | ScopedValue 转正、紧凑对象头、模块导入声明、简化的 main 方法 |

**新日期 API**：`LocalDate`、`LocalDateTime`、`Instant`、`DateTimeFormatter` 都是不可变且线程安全的，取代线程不安全的 `SimpleDateFormat` 和设计混乱的 `Date` / `Calendar`。

```java
// JDK 21 的 record + sealed + switch 模式匹配
sealed interface Shape permits Circle, Square {}
record Circle(double r) implements Shape {}
record Square(double side) implements Shape {}

double area(Shape s) {
    return switch (s) {          // 编译器检查是否穷举所有子类
        case Circle c -> Math.PI * c.r() * c.r();
        case Square q -> q.side() * q.side();
    };
}
```

## 六、高频面试题与自测清单

下面 24 道题覆盖图谱中全部「核心」知识点。这部分题目答案短，加分点在于能说出源码细节和实际踩过的坑。

### 6.1 高频面试题

| # | 题目 | 答案要点 | 章节 |
| --- | --- | --- | --- |
| 1 | 面向对象三大特性？重载和重写的区别？ | 封装、继承、多态；编译期 vs 运行期 | 一 |
| 2 | 抽象类和接口的区别？ | is-a vs can-do；单继承 vs 多实现；default 方法 | 一 |
| 3 | Java 是值传递还是引用传递？ | 只有值传递，对象传的是引用的副本 | 一 |
| 4 | String 为什么不可变？和 StringBuilder、StringBuffer 的区别？ | final + 私有数组；常量池、hashCode 缓存、安全 | 一 |
| 5 | `new String("abc")` 创建了几个对象？ | 1 个或 2 个，取决于常量池中是否已有 | 一 |
| 6 | == 和 equals 的区别？为什么重写 equals 要重写 hashCode？ | 地址 vs 内容；哈希集合先比 hashCode | 一 |
| 7 | Integer 缓存是什么？ | -128\~127；包装类比较用 equals | 一 |
| 8 | 什么是类型擦除？PECS 是什么？ | 编译期检查、运行期无类型参数；extends 读、super 写 | 二 |
| 9 | ArrayList 的扩容机制？和 LinkedList 的区别？ | 懒初始化 10、1.5 倍、copyOf；实际几乎都用 ArrayList | 三 |
| 10 | 怎么在遍历时安全删除元素？ | fail-fast 与 modCount；iterator.remove、removeIf | 三 |
| 11 | HashMap 的底层结构？什么时候树化？ | 数组 + 链表 + 红黑树；8 和 64、退化 6；泊松分布 | 四 |
| 12 | HashMap 的 put 流程？ | hash → 初始化 → 定位桶 → 空桶直接放 / 覆盖 / 尾插 / 树化 → 扩容 | 四 |
| 13 | 为什么容量是 2 的幂？扰动函数的作用？ | 位运算取模、扩容不重算 hash；高位参与运算 | 四 |
| 14 | HashMap 为什么线程不安全？ | 1.7 头插死循环；1.8 覆盖丢数据、size 不准 | 四 |
| 15 | 怎么用 LinkedHashMap 实现 LRU？ | accessOrder + removeEldestEntry | 四 |
| 16 | BIO、NIO、AIO 的区别？epoll 为什么高效？ | 阻塞 vs 多路复用 vs 异步；就绪队列、不重复拷贝 | 五 |
| 17 | Buffer 的 flip、clear、compact 有什么区别？compact 什么时候用？ | 切换读写模式 / 丢弃重写 / 保留未读数据继续写 | 五 |
| 18 | 直接缓冲区和非直接缓冲区的区别？堆外内存回收延迟怎么处理？ | 少一次拷贝但分配回收贵；池化复用、限制 MaxDirectMemorySize | 五 |
| 19 | slice、duplicate 与 Scattering Reads / Gathering Writes 的作用？ | 共享底层内存的视图；一次系统调用读写多个 Buffer | 五 |
| 20 | Selector 的三个 select 方法区别？怎么安全取消 key 并关闭 Channel？ | 阻塞 / 立即返回 / 超时；先 cancel 再 close，遍历时用 Iterator.remove | 五 |
| 21 | 为什么常驻注册 OP_WRITE 会导致 CPU 空转？ | 写缓冲区几乎总是可写；只在写不完时注册、写完取消 | 五 |
| 22 | FileChannel 怎么实现大文件高效读写与通道间传输？ | 固定 ByteBuffer 循环、transferTo、MappedByteBuffer 随机访问 | 五 |
| 23 | MappedByteBuffer、WatchService、FileLock 各有什么坑？ | mmap 回收时机不确定；inotify 事件合并；FileLock 是进程间锁 | 五 |
| 24 | NIO.2 异步通道怎么用？为什么 Netty 选择 NIO 而不是 AIO？ | Future 会阻塞、CompletionHandler 真异步；Linux 上 AIO 由线程池模拟 | 五 |

### 6.2 动手实验

- [ ] 写一个只重写 equals 不重写 hashCode 的类，放入 HashSet 观察重复
- [ ] 用反射读取 HashMap 的 `table` 字段，观察扩容前后容量和元素分布（JDK 17 需加 `--add-opens`）
- [ ] 构造大量 hashCode 相同的 key，对比树化前后的查询耗时
- [ ] 在 foreach 中删除元素复现 `ConcurrentModificationException`，再用 removeIf 修复
- [ ] 不用 LinkedHashMap，用 HashMap + 双向链表手写 LRU
- [ ] 用 NIO 的 Selector 写一个简单的 Echo 服务器
- [ ] 用同一个 ByteBuffer 连续执行 `write → flip → read → compact → write`，打印每一步的 position / limit
- [ ] 用 `FileChannel.transferTo` 与普通流复制一个 1GB 文件，对比耗时
- [ ] 用 `WatchService` 监控一个目录，观察创建 / 修改 / 删除事件的触发次数

### 6.3 自测清单

- [ ] 能解释 1.2 节代码中每一行 == 比较的结果
- [ ] 能写出 HashMap 的 hash 方法和下标计算，并讲清 put 的每一步
- [ ] 能说出 HashMap 扩容时元素如何重新分布，以及 JDK 7 死循环的成因
- [ ] 能对比 5 种 Map 的顺序、null 支持和线程安全性
- [ ] 能说出 3 个以上集合工具方法的坑（Arrays.asList、toMap、subList 等）
- [ ] 能讲清 epoll 比 select 高效的原因，以及它与 Netty、Redis 的关系
- [ ] 能在白板上写出 flip / clear / compact 对 position 与 limit 的变化
- [ ] 能解释为什么「NIO 是非阻塞但仍属于同步 I/O」
- [ ] 能说出 Selector 事件循环里三个容易踩的坑（OP_WRITE 空转、忘记 Iterator.remove、关闭后操作已取消的 key）
