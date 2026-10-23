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
| IO / NIO / AIO（io-nio） | 重点 | IO / NIO |
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

**NIO 三大组件【重点】**：**Channel**（双向通道）、**Buffer**（数据必须经过缓冲区，读写切换要调用 `flip()`）、**Selector**（多路复用器）。

**select / poll / epoll【核心】**：

| 机制 | 连接上限 | 找就绪连接 | 数据拷贝 |
| --- | --- | --- | --- |
| select | 1024（fd\_set 位图） | 遍历全部 O(n) | 每次把整个集合拷贝到内核 |
| poll | 无（链表） | 遍历全部 O(n) | 同上 |
| epoll | 无 | 内核通过回调把就绪的 fd 放入就绪队列，只返回就绪的 | fd 只注册一次（红黑树），不重复拷贝 |

epoll 支持水平触发（LT，默认，只要有数据就一直通知）和边缘触发（ET，只在状态变化时通知一次，必须一次读完，Netty 的 epoll 传输用 ET）。

**Reactor 模型【重点】**：单 Reactor 单线程（Redis 6 之前）→ 单 Reactor 多线程 → 主从 Reactor（Netty：BossGroup 负责 accept，WorkerGroup 负责读写）。

**零拷贝【重点】**：传统的「读文件 + 发网络」有 4 次拷贝和 4 次上下文切换。`mmap` 把内核缓冲区映射到用户空间，省一次 CPU 拷贝（RocketMQ 用它读写 CommitLog）；`sendfile` 直接在内核中从文件传到 socket，配合 DMA gather 可以做到零 CPU 拷贝（Kafka 用 `transferTo`）。`DirectByteBuffer` 分配堆外内存，也能少一次堆内到堆外的拷贝。

### 5.2 现代 Java 特性【了解】

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

下面 16 道题覆盖图谱中全部「核心」知识点。这部分题目答案短，加分点在于能说出源码细节和实际踩过的坑。

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

### 6.2 动手实验

- [ ] 写一个只重写 equals 不重写 hashCode 的类，放入 HashSet 观察重复
- [ ] 用反射读取 HashMap 的 `table` 字段，观察扩容前后容量和元素分布（JDK 17 需加 `--add-opens`）
- [ ] 构造大量 hashCode 相同的 key，对比树化前后的查询耗时
- [ ] 在 foreach 中删除元素复现 `ConcurrentModificationException`，再用 removeIf 修复
- [ ] 不用 LinkedHashMap，用 HashMap + 双向链表手写 LRU
- [ ] 用 NIO 的 Selector 写一个简单的 Echo 服务器

### 6.3 自测清单

- [ ] 能解释 1.2 节代码中每一行 == 比较的结果
- [ ] 能写出 HashMap 的 hash 方法和下标计算，并讲清 put 的每一步
- [ ] 能说出 HashMap 扩容时元素如何重新分布，以及 JDK 7 死循环的成因
- [ ] 能对比 5 种 Map 的顺序、null 支持和线程安全性
- [ ] 能说出 3 个以上集合工具方法的坑（Arrays.asList、toMap、subList 等）
- [ ] 能讲清 epoll 比 select 高效的原因，以及它与 Netty、Redis 的关系
