# Java 核心面试题集（含参考答案）

> 答案按 `prompt/面试回答生成.md` 的六段式生成，选题来自《Java架构师技术栈锚点.md》1.1 Java 语言核心（P0）。岗位默认「高级 Java 后端 / 架构师」，默认以 JDK 17 / 21 为准，涉及 JDK 8 差异的地方单独标注。口述版可直接读出声，卡壳处即理解缺口。

## 一、泛型与类型擦除

### 1. 【必答】泛型与类型擦除：泛型擦除擦掉了什么，运行时还剩什么？

**口述版**

Java 泛型是编译期特性。编译后类型参数全部被擦除成它的上界，没有上界就擦成 Object，然后编译器在需要取值的地方插入强制类型转换，并为多态生成桥方法。所以运行时 `List<String>` 和 `List<Integer>` 是同一个 Class，反射拿不到元素的真实类型，也就不能 `new T[]`、不能对泛型做 instanceof、不能靠泛型参数做方法重载。擦除的动机是兼容 JDK 5 之前的字节码和已有类库，代价是运行时类型信息丢失、类型安全只到编译期。要拿回类型只有两条路：把类型令牌 `Class<T>` 显式传进去，或者把泛型写在类、字段、方法的签名上，通过 `getGenericSuperclass`、`getGenericParameterTypes` 反射读出来。

**短版**

泛型只在编译期存在，类型参数擦到上界（无界为 Object），编译器插强转、生成桥方法；运行时 `List<String>` 与 `List<Integer>` 同一个 Class，所以不能 `new T[]`、不能 instanceof、不能按泛型重载。

**展开版**

- 机制链：源码带类型参数 → javac 做类型检查 → 擦除到上界 → 取值处插入 checkcast → 需要时生成桥方法 → class 文件里只剩原始类型。
- 关键结构：泛型签名仍保留在 class 文件的 `Signature` 属性里，`getGenericType`、`getGenericSuperclass` 能读；局部变量的泛型信息编译期就丢了。
- 触发条件：所有泛型类、泛型方法、通配符都走擦除；`List<? extends Number>` 读出来就是 Number。
- 例外：`new TypeReference<List<String>>(){}` 这类匿名子类把类型写进了父类签名，属于借壳留存，不是泛型本身活着。
- 对比取舍：擦除换来二进制兼容和低迁移成本，代价是运行时无法校验、不能重载、容器只能靠调用方自觉。

**追问预判**

- 为什么不能 `new T[]` → 擦除后 T 变成 Object，数组必须有具体组件类型做运行时检查，编译器无法保证类型安全。
- `List<String>` 和 `List<Integer>` 能重载吗 → 不能，擦除后签名相同，编译报 name clash。
- 泛型信息真的一点不剩吗 → 类、字段、方法上的签名保留在 Signature 属性里，可以反射读取。

**常见错误答法**

- 说运行时完全没有任何泛型信息（忽略 Signature 属性）。
- 认为 `List<String>` 和 `List<Integer>` 是两个不同的 Class（概念错）。

**记忆锚点**

泛型编译期存在 → 擦到上界 → 插强转 + 桥方法 → 运行时同一 Class，靠签名或令牌取回。

---

### 2. 【必答】泛型与类型擦除：为什么不能 `new T[]`，工程上怎么绕过？

**口述版**

因为擦除后 T 没有运行时类型，而数组是带组件类型的，JVM 创建数组时必须知道元素类型，还要靠它做 ArrayStoreException 检查。`new T[]` 编译不过，本质是编译器没法生成一个运行期能校验元素类型的数组。常见绕过有三种：第一，传 `Class<T>`，用 `Array.newInstance(clazz, n)` 造，类型安全，代价是调用方多传一个参数；第二，先建 `Object[]` 再强转成 `T[]`，写法简单，但等于把风险推到运行期，写入错误类型不会立刻报错；第三，直接用集合代替数组，这是最推荐的，因为集合内部虽然也是 Object 数组，但每次 get 都做 checkcast，类型错误会尽早暴露。

**短版**

擦除后 T 没有运行时类型，数组必须有组件类型才能创建和校验，所以 `new T[]` 不合法；用 `Class<T>` 加 `Array.newInstance`、`Object[]` 强转（不安全）或直接用集合。

**展开版**

- 机制链：数组在 JVM 里带组件类型 → 创建走 anewarray，必须给具体类型 → T 已被擦除 → javac 直接拒绝。
- 关键结构：`Array.newInstance(Class, int)` 可在运行期按 Class 造数组；可变参数泛型数组会触发堆污染告警，用 `@SafeVarargs` 抑制。
- 触发条件：写泛型容器、泛型工厂方法、泛型可变参数 `T...`。
- 例外：`(T[]) new Object[n]` 能编译通过，但当 T 有具体上界且结果被赋给具体数组类型时，强转处会抛 ClassCastException。
- 对比取舍：Class 令牌方式类型安全但 API 变重；Object 数组强转写法轻但把错误推迟；集合最稳妥但有装箱和额外对象开销。

**追问预判**

- `(T[]) new Object[10]` 为什么有时会炸 → T 有上界时，强转目标变成具体类型数组，运行期检查不通过就抛 ClassCastException。
- `ArrayList` 内部怎么存 → 用 `Object[] elementData`，get 时强转，写入错误类型在编译期就被拦住。
- 泛型可变参数有什么坑 → 编译器会生成泛型数组，可能发生堆污染，需要在确认安全时加 `@SafeVarargs`。

**常见错误答法**

- 说 `new T[]` 只是加个 `@SuppressWarnings` 就能用（编译期就过不去）。
- 说数组和集合一样都不做元素类型检查（数组是协变且运行期检查的）。

**记忆锚点**

T 无运行时类型 → 数组必须有 → 令牌造数组 / Object 兜底 / 集合最优。

---

### 3. 【进阶】泛型与类型擦除：桥方法是什么，为什么会产生方法重载冲突？

**口述版**

桥方法是擦除的补偿机制。以 Comparable 为例，接口里的方法是 `compareTo(Object)`，实现类写的是 `compareTo(T)`，擦除后子类的方法签名变成 `compareTo(MyType)`，跟接口要求的对不上，多态就断了。编译器于是在子类里生成一个 `compareTo(Object)` 的桥方法，内部强转后调用真正的实现，这样通过接口引用调用才能正确分发。桥方法带 ACC_BRIDGE 和 ACC_SYNTHETIC 标志，源码里看不到，反射能看到。重载冲突是同一原因的另一个面：两个方法擦除后参数类型相同，比如 `f(List<String>)` 和 `f(List<Integer>)` 都变成 `f(List)`，编译器直接报 name clash，因为 JVM 的方法签名不含泛型参数，无法区分。

**短版**

桥方法是编译器为保持多态生成的转发方法，带 ACC_BRIDGE，内部强转后调用真实实现；重载冲突是因为擦除后两个方法的参数类型相同，JVM 无法区分，编译直接报错。

**展开版**

- 机制链：子类重写泛型方法 → 擦除后签名与父类或接口不一致 → 编译器生成桥方法转发 → 调用时按桥方法分发。
- 关键结构：桥方法标记 ACC_BRIDGE 与 ACC_SYNTHETIC，`Method.isBridge()` 可判断，`getDeclaredMethods` 会返回它。
- 触发条件：实现泛型接口、继承泛型父类并重写方法、协变返回值重写。
- 例外：返回值协变也会生成桥方法，不限于参数类型；桥方法本身不能再被重写。
- 对比取舍：桥方法让擦除后的多态成立，代价是方法表里多出转发层，框架做方法匹配、统计、AOP 时容易重复命中。

**追问预判**

- 桥方法会干扰框架吗 → 会，反射扫描方法时可能重复注册，通常要过滤 `isBridge()` 或 `isSynthetic()`。
- 为什么不能靠返回值区分重载 → Java 方法签名不含返回值，JVM 也不允许只靠返回值重载。
- 桥方法里抛的异常怎么排查 → 栈里会看到 synthetic 方法帧，往上看真实方法的调用链。

**常见错误答法**

- 把桥方法说成是运行时 JVM 动态生成的（是编译器生成的）。
- 认为泛型参数不同就能重载（擦除后签名相同）。

**记忆锚点**

擦除导致签名不匹配 → 编译器造桥方法转发 → ACC_BRIDGE → 重载冲突同源。

---

### 4. 【兜底】泛型与类型擦除：PECS 是什么，通配符上下界怎么选？

**口述版**

PECS 是 Producer Extends、Consumer Super。如果这个泛型容器对你是生产者，你只从里面读数据，就用 `? extends T`，比如 `List<? extends Number>` 只能 get 出 Number，不能 add；如果它是消费者，你只往里写数据，就用 `? super T`，比如 `List<? super Integer>` 能 add Integer，取出来只能是 Object。记一句：读用 extends，写用 super，既要读又要写就别用通配符，直接写具体类型。JDK 里的典型例子是 `Collections.copy(List<? super T> dest, List<? extends T> src)`，源是生产者、目标是消费者。方向用反了编译就过不去，这其实是编译器在帮你保证类型安全，代价是 API 签名变复杂、内部实现受限。

**短版**

PECS：生产者用 `? extends T`，只能读不能写；消费者用 `? super T`，只能写、读出来是 Object；读写都要就不用通配符。例如 `Collections.copy(dest super, src extends)`。

**展开版**

- 机制链：通配符决定编译器允许的操作集合 → extends 上界把 get 的类型收窄、把 add 堵死 → super 下界反过来。
- 关键结构：`? extends T` 的 add 参数类型是捕获类型，编译器无法确定，所以除 null 外都拒绝。
- 触发条件：写公共 API、工具方法、需要接收多种参数化类型的容器时。
- 例外：`?` 无界通配符等价于 `? extends Object`，只能读成 Object；返回类型上不建议用通配符，会把复杂度转嫁给调用方。
- 对比取舍：通配符提升 API 的接收面，代价是实现内部不能写、签名可读性下降，过度使用反而难维护。

**追问预判**

- `List<? extends Number>` 为什么不能 add → 编译器不知道实际类型是 Integer 还是 Double，写任何一个都可能破坏容器。
- `List<? super Integer>` 为什么能 add → 下界保证容器至少能装 Integer，写入是安全的。
- 什么时候干脆不用通配符 → 方法内部既要读又要写，或者返回类型，直接用具体类型更清晰。

**常见错误答法**

- 把 extends 和 super 的读写能力说反（方向混淆）。
- 认为通配符能提升运行时性能（只是编译期约束）。

**记忆锚点**

生产者 extends 只读 → 消费者 super 只写 → 读写都要不用通配符。

---

## 二、集合

### 5. 【必答】集合：HashMap 的 put 流程、扩容与树化条件？

**口述版**

put 先算 key 的 hash，做一次高低位异或扰动，再用 `hash & (n-1)` 定位桶。桶为空就放进去；不为空先比 hash 再比 equals，相同就覆盖 value；是树节点就走红黑树插入；否则遍历链表，JDK 8 是尾插。插入后如果链表长度到 8 且数组容量至少 64 就树化，容量不到 64 则优先扩容，因为小表扩容比树化更划算；最后 size 超过阈值，也就是容量乘 0.75，就扩容成两倍。相比 JDK 7 最大的变化是头插改尾插，避免了并发扩容时链表成环导致 CPU 打满，同时引入红黑树，把极端冲突下的查询从 O(n) 降到 O(log n)。但 HashMap 本身仍然不是线程安全的，并发写会丢数据。

**短版**

算 hash 扰动 → `hash & (n-1)` 定位 → 空桶直放、命中覆盖、树节点走红黑树、链表尾插 → 链表到 8 且容量到 64 才树化 → size 超 0.75 阈值扩容两倍；JDK 8 尾插防成环。

**展开版**

- 机制链：`hash(key)` 高位异或低位 → 定位桶 → 判空/覆盖/树插/尾插 → `++size` 超阈值 `resize()`。
- 关键结构：数组 `table`、链表 `Node`、红黑树 `TreeNode`；树化阈值 8、退化阈值 6、最小树化容量 64、负载因子 0.75。
- 触发条件：链表长度到 8 且容量不足 64 时先扩容；删除到 6 个节点退化成链表。
- 例外：hash 冲突严重且 key 实现 Comparable 时，树内按 compareTo 排序，否则按类名和 identityHashCode 兜底。
- 对比取舍：0.75 是空间和冲突的折中；树化防极端退化，但 TreeNode 内存是 Node 的两倍，所以阈值不能太低。

**追问预判**

- 为什么阈值是 8 和 6 → 泊松分布下链表到 8 的概率极低，留 6 到 8 的缓冲避免频繁树化退化抖动。
- 为什么容量不到 64 先扩容 → 小表扩容后冲突自然缓解，树化收益不如扩容，且树化成本高。
- 多线程 put 会怎样 → JDK 8 尾插不会成环，但会丢更新、size 不准，仍然必须用 ConcurrentHashMap。

**常见错误答法**

- 说链表长度一到 8 就一定树化（漏掉容量 64 的前提）。
- 说 JDK 8 用头插法（那是 JDK 7）。

**记忆锚点**

扰动定位 → 空放/覆盖/树插/尾插 → 8 且 64 才树化 → 0.75 扩容两倍。

---

### 6. 【必答】集合：HashMap 容量为什么是 2 的幂，扩容时元素怎么迁移？

**口述版**

容量取 2 的幂是为了把取模换成位运算。`hash & (n-1)` 等价于 `hash % n`，但快得多，而且 n-1 的低位全是 1，能让哈希值分布更均匀；如果容量不是 2 的幂，低位会有 0，某些桶永远用不到。构造时传的容量会被 `tableSizeFor` 向上取整到最近的 2 的幂。扩容时容量翻倍，元素不需要重新算 hash：因为 n 翻倍后，元素的新下标只取决于 hash 里新增的那一位，这一位是 0 就留在原索引，是 1 就落在原索引加旧容量。JDK 8 用 loHead 和 hiHead 两条链一次拆完，避免了每个节点重新计算下标，这也是 JDK 8 扩容更快的原因。JDK 7 是每个元素重新 hash 再头插，并发下还会成环。

**短版**

2 的幂让 `hash & (n-1)` 等价取模且低位全 1 分布均匀，`tableSizeFor` 负责向上取整；扩容翻倍后新下标只看 hash 新增那一位，0 留原位、1 移到原位加旧容量，JDK 8 用高低位两条链拆。

**展开版**

- 机制链：容量 n 为 2 的幂 → `n-1` 是低位全 1 的掩码 → 与运算即取模 → 扩容翻倍只多一位掩码。
- 关键结构：`tableSizeFor` 用五次右移或运算把任意值填成 2 的幂减一；`resize` 里 `e.hash & oldCap` 判断高低位。
- 触发条件：首次 put 初始化容量、size 超阈值、树化前容量不足 64。
- 例外：指定初始容量时若传入 7 会变成 8，传入 9 会变成 16；加载因子不是 2 的幂也没关系，只有容量要求。
- 对比取舍：位运算换来性能与均匀分布，代价是容量不能任意指定、扩容必须翻倍，内存增长呈阶梯式。

**追问预判**

- 为什么不用 `%` → 取模是除法指令，比位与慢，热点路径上差距明显。
- 高低位拆分的原理 → 旧容量这一位在翻倍后成为新下标的最高位，所以只需看它。
- 初始容量怎么给 → 按预估元素数除以 0.75 再向上取整，避免反复扩容。

**常见错误答法**

- 说扩容时每个元素都要重新计算 hash（JDK 8 已优化）。
- 说容量可以是任意值，只是性能差一点（实际会被强制取整）。

**记忆锚点**

2 的幂 → n-1 是掩码 → 位与代替取模 → 扩容只看新增一位。

---

### 7. 【必答】集合：ConcurrentHashMap 的并发粒度，get 为什么不用加锁？

**口述版**

JDK 8 的 ConcurrentHashMap 放弃了 JDK 7 的分段锁，改成 CAS 加 synchronized 锁单个桶的头节点，锁粒度细化到一个桶，并发度约等于数组长度。put 时桶为空用 CAS 直接放，不为空就 synchronized 锁住头节点再操作；扩容时用 ForwardingNode 标记已迁移的桶，并把迁移任务拆成多个区间，让多个线程协同搬迁，所以扩容是并行的。get 完全不加锁，因为 Node 的 val 和 next 都用 volatile 修饰，保证可见性，最坏读到旧值，这符合它弱一致性的语义。size 用 baseCount 加 CounterCell 分散计数，避免所有线程抢同一个计数器。它不允许 null 键和 null 值，因为并发下无法区分"不存在"和"值为 null"。

**短版**

JDK 8 用 CAS 加 synchronized 锁单个桶，并发度等于数组长度；扩容用 ForwardingNode 加多线程协同迁移；get 靠 volatile 的 val/next 无锁读，属弱一致；size 用 CounterCell 分散计数，且不允许 null。

**展开版**

- 机制链：定位桶 → 空桶 CAS 插入 → 非空桶 synchronized 锁头节点 → 链表或红黑树插入 → 检查是否需要协助扩容。
- 关键结构：`table`、`Node.val` 与 `next` 的 volatile、`ForwardingNode`、`CounterCell`、`sizeCtl` 控制初始化和扩容状态。
- 触发条件：链表长度到 8 且容量到 64 树化；扩容时线程发现 `sizeCtl` 为负会参与迁移。
- 例外：get 可能读到迁移中的旧值或新值，但不阻塞；`size()` 是估算值，并发下不精确。
- 对比取舍：细粒度锁换来高并发，代价是弱一致性、复合操作要靠 `computeIfAbsent` 等原子方法，不能用它做跨 key 的原子事务。

**追问预判**

- 为什么 get 不加锁还安全 → volatile 保证节点字段可见，链表或树的链接关系在发布时也是安全的，最坏读到过期值。
- 为什么不允许 null → 并发下 `get` 返回 null 无法判断是没映射还是映射到 null，`containsKey` 也无法补救。
- 复合操作用什么 → 用 `computeIfAbsent`、`merge`、`putIfAbsent`，它们内部对桶加锁保证原子。

**常见错误答法**

- 说 JDK 8 还是分段锁（那是 JDK 7）。
- 说 get 加了读锁所以线程安全（实际是无锁弱一致）。

**记忆锚点**

CAS 空桶 → synchronized 锁桶头 → ForwardingNode 协同扩容 → volatile 无锁读。

---

### 8. 【进阶】集合：fail-fast 与 ConcurrentModificationException 怎么产生的，怎么避免？

**口述版**

fail-fast 是集合在遍历过程中被结构性修改就尽快抛 ConcurrentModificationException。实现是集合维护一个 modCount，每次增删改加一，迭代器创建时把当时的 modCount 记进 expectedModCount，每次 next 都比对，不相等就抛。它是尽力而为的检测，不保证一定发现，更不代表线程安全，只是避免在状态不确定时继续遍历。避免方式有三种：单线程里用迭代器自己的 remove，它会同步更新 expectedModCount；多线程改用 ConcurrentHashMap、CopyOnWriteArrayList 这类并发容器；或者遍历前复制一份快照。最常见的触发场景就是 for-each 里直接调 `list.remove`，因为 for-each 底层就是迭代器。

**短版**

集合用 modCount，迭代器用 expectedModCount 比对，结构性修改后 next 就抛 CME；它只是尽力检测、不代表线程安全。避免：迭代器 remove、并发容器、复制快照。

**展开版**

- 机制链：增删改 `modCount++` → 迭代器持有 expectedModCount → 每次 next 校验 → 不等抛 CME。
- 关键结构：`AbstractList.Itr`、`HashMap.HashIterator` 的 modCount 检查；`remove` 后同步 expectedModCount。
- 触发条件：for-each 中增删、多线程同时改同一集合、`subList` 视图被外部修改。
- 例外：`set` 修改值不算结构性修改，不会触发；并发容器的迭代器是弱一致的，不抛 CME。
- 对比取舍：快速失败能尽早暴露 bug，代价是并发场景不可用；弱一致迭代器不抛但可能读到旧数据。

**追问预判**

- 为什么 for-each 里 remove 会炸 → for-each 编译成迭代器遍历，直接改集合绕过了迭代器，expectedModCount 对不上。
- 一定抛吗 → 不一定，删除后恰好不再调 next 就不会抛，所以它是"尽力而为"。
- CopyOnWriteArrayList 为什么不抛 → 写时复制新数组，迭代器持有旧数组快照，读写互不影响。

**常见错误答法**

- 说 fail-fast 是线程安全机制（它只是错误检测）。
- 说只要用 for 循环按下标删除就一定安全（会漏元素，需倒序或修正下标）。

**记忆锚点**

modCount 对不上 → next 就抛 CME → 迭代器删 / 并发容器 / 快照遍历。

---

### 9. 【兜底】集合：ArrayList 的扩容机制，和 LinkedList 怎么选？

**口述版**

ArrayList 底层是 Object 数组，默认容量 10 但不是构造时分配，而是第一次 add 时才分配。满了就扩容到 1.5 倍，用 Arrays.copyOf 把老数组复制过去，所以扩容有拷贝成本。它的随机访问是 O(1)，中间插入删除要搬移后面所有元素是 O(n)。LinkedList 是双向链表，头尾增删 O(1)，但按下标访问要从头或尾走是 O(n)，每个节点还要额外存前后指针，内存开销大且不连续，缓存不友好。实践里几乎总是选 ArrayList，因为连续内存对 CPU 缓存友好，数据量不大时中间插入也常常比链表快。只有确实频繁在头部增删才考虑 LinkedList，而当作队列用时现在更推荐 ArrayDeque。

**短版**

ArrayList 是数组，首次 add 分配 10，满则 1.5 倍复制扩容，随机访问 O(1)、中间增删 O(n)；LinkedList 是双向链表，头尾增删 O(1)、随机访问 O(n) 且内存开销大；默认选 ArrayList，队列用 ArrayDeque。

**展开版**

- 机制链：add 前检查 `size+1 > elementData.length` → `grow` 到 1.5 倍 → `Arrays.copyOf` 复制 → 再写入。
- 关键结构：`elementData` 数组、`size`、`DEFAULT_CAPACITY=10`、`MAX_ARRAY_SIZE` 上限保护。
- 触发条件：元素数超过当前容量；`addAll` 会一次扩到够用为止。
- 例外：`trimToSize` 可缩容；指定初始容量能避免反复扩容，大批量写入前应预估。
- 对比取舍：数组换随机访问和缓存友好，代价是插入删除搬移；链表换局部增删，代价是随机访问和内存。

**追问预判**

- 1.5 倍为什么不是 2 倍 → 1.5 倍在空间浪费和扩容次数之间更平衡，也能复用旧数组释放的空间。
- 中间删除怎么优化 → 用 `removeIf` 批量删，或者从尾部倒序删，避免每次搬移。
- `Arrays.asList` 的坑 → 返回固定长度列表，add/remove 抛 UnsupportedOperationException。

**常见错误答法**

- 说 ArrayList 构造时就分配 10 个元素（实际首次 add 才分配）。
- 说 LinkedList 插入删除一定比 ArrayList 快（先要定位，定位是 O(n)）。

**记忆锚点**

数组 1.5 倍复制 → 随机访问快 → 链表头尾快但定位慢 → 默认 ArrayList。

---

## 三、String 与基础契约

### 10. 【必答】String 与常量池：`==` 和 `equals` 有什么区别，字符串常量池在哪？

**口述版**

`==` 比的是值本身，基本类型比数值，引用类型比地址；`equals` 是 Object 的方法，默认实现也是比地址，String 等类重写成了比内容。字符串常量池在 JDK 7 之后从方法区搬到了堆里。字面量 `"abc"` 会先查池，有就复用，没有就创建并放入。所以 `"a" + "b" == "ab"` 是 true，因为编译期就折叠成了常量；而 `new String("ab") == "ab"` 是 false，new 一定在堆上新建对象；用变量拼接走的是 StringBuilder，结果也不在池里。要主动入池用 intern，它返回池中对象的引用，JDK 7 起如果池里没有，是把堆上这个对象的引用登记进去，而不是复制一份。

**短版**

`==` 比地址（基本类型比数值），`equals` 默认也比地址、String 重写成比内容；常量池 JDK 7 起在堆里；字面量复用、new 必新建、变量拼接不进池，intern 负责入池。

**展开版**

- 机制链：编译期字面量进常量池 → 运行期 new 在堆上新建 → intern 查池并返回池中引用。
- 关键结构：`StringTable` 是 native 哈希表，可用 `-XX:StringTableSize` 调整；JDK 7 起池内容指向堆对象。
- 触发条件：字面量赋值、常量表达式拼接、intern 调用；`new String(...)`、变量拼接、`substring` 等不自动入池。
- 例外：编译期常量拼接会被折叠；`final` 修饰的局部变量参与拼接也可能被当成常量。
- 对比取舍：常量池省内存并支持引用比较，代价是池本身占用内存、intern 有哈希查找开销。

**追问预判**

- 为什么 JDK 7 要把池移到堆 → 方法区空间有限，池容易 OOM，移到堆可随 GC 回收并统一管理。
- intern 适合什么场景 → 大量重复字符串去重，或需要按引用快速比较的枚举式取值。
- `String s = new String("a")` 创建几个对象 → 字面量 "a" 已在池里就只新建一个堆对象，否则两个。

**常见错误答法**

- 说常量池一直在方法区（JDK 7 起已在堆）。
- 认为 `equals` 默认比内容（Object 默认比地址）。

**记忆锚点**

`==` 比地址 → equals 比内容 → 字面量入池 → new 必新建 → intern 取池引用。

---

### 11. 【必答】String 与基础契约：为什么重写 equals 必须重写 hashCode？

**口述版**

因为哈希容器是两段定位：先按 hashCode 找桶，再在桶里用 equals 逐个比对。如果只重写 equals 不重写 hashCode，两个逻辑相等的对象哈希值可能不同，会被分到不同桶，HashMap 里就会出现 put 进去却 get 不到、containsKey 返回 false，HashSet 里也会出现两个"相等"的元素。契约是三条：equals 相等则 hashCode 必须相等；hashCode 相等 equals 不一定相等；equals 不变时 hashCode 也不能变。实现时用 Objects.hash 或 IDE 生成，字段要选参与 equals 判断的同一组，而且这些字段不能变，否则对象放进 HashSet 后一改就再也找不到了。String 就是因为两个都重写了才适合做 key。

**短版**

哈希容器先按 hashCode 找桶再用 equals 比对；只重写 equals 会让相等对象落到不同桶，出现 get 不到、Set 重复。契约：equals 相等则 hashCode 必相等，参与计算的字段要与 equals 一致且不可变。

**展开版**

- 机制链：put 算 hashCode → 定位桶 → equals 比对 → 只重写 equals 时桶位置不同，永远比不到。
- 关键结构：HashMap 的 `hash` 扰动、桶内链表或红黑树、`equals` 与 `hashCode` 的契约由 Object 文档定义。
- 触发条件：任何作为 HashMap key、HashSet 元素、进入哈希结构的自定义类。
- 例外：只用 TreeMap 或 List 时不依赖 hashCode，但这是脆弱前提，不该作为不重写的理由。
- 对比取舍：重写两者保证语义一致，代价是可变字段作为 key 时进容器后修改会导致对象"丢失"。

**追问预判**

- hashCode 冲突了怎么办 → 落到同一桶，再用 equals 区分，冲突多时链表转红黑树。
- 用可变对象做 key 有什么问题 → 字段变更后 hashCode 变了，取不到原来的映射，等于内存泄漏。
- Lombok 的 `@EqualsAndHashCode` 要注意什么 → 只选业务唯一字段，避免把可变字段或关联对象整串带进去。

**常见错误答法**

- 说 hashCode 相同对象就相等（碰撞允许不等）。
- 只重写 hashCode 不重写 equals（同样破坏契约）。

**记忆锚点**

先 hash 后 equals → 两者必须同源 → 字段不可变 → 否则容器里"消失"。

---

### 12. 【必答】String 与常量池：`intern()` 做了什么，什么时候该用？

**口述版**

intern 的作用是把字符串放进常量池并返回池中的引用。调用时如果池里已有内容相同的字符串，直接返回池里的；没有的话，JDK 7 之前是在池里复制一份，JDK 7 起池移到堆里，直接把堆上这个对象的引用登记进池，避免复制。典型用途是大量重复字符串做去重省内存，比如解析日志里的状态码、地区码，或者需要按引用快速比较的场景。但滥用有风险：池本身有大小限制，大量唯一字符串会把内存撑大，而且 intern 要做哈希查找，比普通 equals 慢。所以一般优先用枚举或规范化字典，不要拿它当万能缓存。

**短版**

intern 查常量池，有就返回池中引用，没有就把当前对象的引用登记进池（JDK 7 起不复制）；适合大量重复字符串去重和按引用比较，但不适合大量唯一字符串，优先用枚举。

**展开版**

- 机制链：调用 intern → 查 StringTable → 命中返回池引用 → 未命中把当前堆对象引用登记入池。
- 关键结构：StringTable 哈希表，JDK 7 起存放在堆，可用 `-XX:StringTableSize` 调桶数减少冲突。
- 触发条件：同一内容字符串高频重复、需要 `==` 快速判定、解析外部数据做规范化。
- 例外：编译期字面量本来就已入池，对字面量调 intern 基本无额外收益。
- 对比取舍：换内存和比较速度，代价是查表开销与池占用；唯一字符串越多，收益越小甚至为负。

**追问预判**

- JDK 6 和 JDK 7 的 intern 差别 → 6 是复制到永久代，7 起改为登记堆中引用，减少一次复制。
- intern 会导致什么线上问题 → 大量唯一字符串入池，StringTable 膨胀、GC 压力上升。
- 和 `String.intern()` 替代方案 → 枚举、字典映射、`-XX:+UseStringDeduplication` 做底层去重。

**常见错误答法**

- 说 intern 总是复制一份字符串到池里（JDK 7 起不复制）。
- 认为 intern 是零成本的，可以无脑用（有哈希查找和池膨胀风险）。

**记忆锚点**

查池 → 命中复用 → 未命中登记引用 → 只适合重复字符串。

---

### 13. 【进阶】String 与基础契约：Integer 缓存区间与自动装箱有哪些坑？

**口述版**

Integer 默认缓存 -128 到 127，这个区间内自动装箱直接返回缓存对象，所以两个值为 100 的 Integer 用 `==` 是 true，而 200 就是 false，因为超出范围会 new 新对象。缓存上界可以用 `-XX:AutoBoxCacheMax` 调大。坑主要在包装类型的比较：Integer 和 int 比较会自动拆箱比数值，两个 Integer 用 `==` 比的是引用。Boolean、Byte、Short、Character 也有类似缓存，Long 同样是 -128 到 127，Float 和 Double 没有缓存。更隐蔽的是三元表达式里混用包装和基本类型，会触发整体拆箱，null 就直接 NPE。实践上包装类型一律用 equals 比较，或者先拆成基本类型。

**短版**

Integer 缓存 -128 到 127，区间内 `==` 为 true、区间外为 false；包装类型比较用 equals，`==` 比引用；三元表达式混用会触发拆箱，null 会 NPE，可用 AutoBoxCacheMax 调上界。

**展开版**

- 机制链：自动装箱调 `Integer.valueOf` → 在缓存区间返回缓存数组元素 → 区间外 `new Integer`。
- 关键结构：`IntegerCache.low=-128`、`high` 默认 127 可由 `AutoBoxCacheMax` 调整；Long、Short、Byte、Character 各有缓存。
- 触发条件：`==` 比较两个包装对象、集合中取出的包装值比较、三元表达式类型推断。
- 例外：Float 和 Double 没有缓存；`new Integer(100) == new Integer(100)` 一定是 false，因为绕过了 valueOf。
- 对比取舍：缓存节省小整数对象，代价是 `==` 的语义随数值变化，容易写出"测试环境对、生产环境错"的代码。

**追问预判**

- 为什么 100 相等 200 不等 → 100 在缓存区间走 valueOf 复用，200 超出范围每次新建。
- `Integer a = 127; a == 127` 一定 true 吗 → 与基本类型比较会自动拆箱，比数值，恒为 true（a 不为 null 时）。
- 缓存能关吗 → 下界固定 -128，上界可通过参数调整，不能完全关闭。

**常见错误答法**

- 说 Integer 缓存是 -128 到 128（上界是 127）。
- 认为包装类型用 `==` 比较数值是安全的（区间外就不成立）。

**记忆锚点**

valueOf 查缓存 → -128 到 127 → 区间外新建 → 包装比较用 equals。

---

## 四、Lambda、Stream 与 Optional

### 14. 【必答】函数式：Lambda 编译后是什么，为什么只能访问 effectively final 的变量？

**口述版**

Lambda 编译后不是匿名内部类。javac 用 invokedynamic 加 LambdaMetafactory，在运行时首次调用时生成实现类，之后复用，所以不像匿名类那样每次编译出一个独立的 class 文件，启动和体积都更省。它捕获外部局部变量时要求变量是 effectively final，本质是把变量的值复制进生成的实现类字段。如果不限制 final，你在外部改了变量而 lambda 里还是旧副本，两边语义就分裂了，所以编译器直接禁止，只允许捕获不再变化的变量。这也意味着 lambda 里改不了外部局部变量，要改只能用数组、AtomicInteger 或者干脆把逻辑改成返回值。另外 this 的语义也不同：lambda 里的 this 指向外部类实例，匿名内部类的 this 指向匿名类自己。

**短版**

Lambda 编译成 invokedynamic 加 LambdaMetafactory，运行时生成实现类并复用；捕获局部变量是把值复制进去，所以要求 effectively final，否则内外副本不一致；lambda 的 this 指向外部类。

**展开版**

- 机制链：javac 生成 invokedynamic 指令 → 首次执行时 LambdaMetafactory 引导生成实现类 → 后续调用直接复用该 CallSite。
- 关键结构：`LambdaMetafactory.metafactory`、生成的实现类实现函数式接口、捕获值作为构造参数或字段。
- 触发条件：任何 lambda 或方法引用；捕获实例字段时其实是捕获 this，不受 final 限制。
- 例外：捕获成员变量合法，因为字段可变且通过 this 访问；捕获的局部变量若是数组，可以改元素但不能改引用。
- 对比取舍：invokedynamic 换来启动性能和实现灵活性，代价是首次调用有引导开销、栈里多出合成方法帧。

**追问预判**

- 为什么不用匿名内部类实现 → 每个 lambda 一个 class 文件，类数量膨胀，加载和元空间成本高。
- 怎么在 lambda 里累加 → 用 `AtomicInteger`、单元素数组，或把 stream 改成返回值的规约操作。
- lambda 和匿名类的 this 区别 → lambda 无独立作用域，this 是外部实例；匿名类 this 是自己。

**常见错误答法**

- 说 lambda 就是匿名内部类的语法糖（实现机制不同）。
- 说捕获的变量必须显式加 final（effectively final 即可）。

**记忆锚点**

invokedynamic → 运行时生成实现类 → 捕获是值复制 → 必须 effectively final。

---

### 15. 【必答】函数式：Stream 的惰性求值与短路是怎么实现的？

**口述版**

Stream 分中间操作和终止操作。中间操作像 filter、map、sorted 都是惰性的，只是把操作记录成一条流水线，不会立即执行；只有遇到终止操作如 collect、forEach、count、findFirst 才一次性触发，把元素逐个送进流水线。好处是多个操作能合并成一次遍历，避免多次循环；短路操作还能提前结束，比如 findFirst 配 limit 在无限流上也能停下来，anyMatch 找到就返回。要注意三点：流只能消费一次，重复用会抛 IllegalStateException；forEach 不保证顺序，需要顺序用 forEachOrdered；有状态的中间操作如 sorted、distinct 会缓冲全部元素，破坏短路和流式特性。

**短版**

中间操作只记录流水线、惰性执行，终止操作才触发一次遍历；多个操作合并成一次循环，findFirst、limit、anyMatch 支持短路；流只能消费一次，sorted 等有状态操作会缓冲全量元素。

**展开版**

- 机制链：中间操作返回新的 Stream 并挂上 Sink → 终止操作触发 `evaluate` → 元素逐个经过 Sink 链 → 短路信号沿链回传。
- 关键结构：`AbstractPipeline` 的 stage 链、`Sink` 接口、`Spliterator` 负责数据源切分与遍历。
- 触发条件：终止操作执行；短路操作遇到满足条件或 limit 达到上限即停止拉取。
- 例外：`sorted`、`distinct`、`limit` 在并行下需要缓冲或协调，并非纯逐元素流式。
- 对比取舍：惰性求值减少遍历次数并支持短路，代价是调试困难、异常栈里全是 lambda 帧、状态操作内存占用高。

**追问预判**

- 为什么流不能重复消费 → 流水线内部有 linkedOrConsumed 标记，消费后置位，再次终止操作直接抛异常。
- filter 里打印日志为什么看不到 → 没有终止操作，流水线根本不会执行。
- 并行流还有短路吗 → 有，但只是近似，其他分片可能已经处理了一部分数据。

**常见错误答法**

- 说中间操作会立即执行（惰性，只有终止操作才触发）。
- 认为 stream 可以反复遍历（一次性，用完要重新创建）。

**记忆锚点**

中间操作记流水线 → 终止操作才跑 → 短路靠 Sink 回传 → 流只能用一次。

---

### 16. 【进阶】函数式：并行流的线程池与常见坑？

**口述版**

并行流默认用 ForkJoinPool.commonPool，线程数是 CPU 核数减一。坑有三个：一是 commonPool 是全局共享的，如果在里面做阻塞 IO，会把其他并行流甚至 CompletableFuture 默认用的线程全堵死；二是它只对可拆分、计算密集、数据量足够大的任务有效，数据量小时拆分合并开销比计算还大；三是用了非线程安全的操作，比如往普通 ArrayList 里 add、用 forEach 累加，结果就错。真要用，应该保证纯函数、无共享可变状态，并对数据量做基准测试；阻塞场景应该自定义 ForkJoinPool 提交任务，而不是直接 parallelStream。默认建议是：先写串行，压测证明是瓶颈再考虑并行。

**短版**

并行流默认跑在 ForkJoinPool.commonPool（核数减一），全局共享，阻塞会拖垮其他任务；只适合计算密集、数据量大、无共享可变状态；累加要用 reduce 或 collector，阻塞场景自定义线程池。

**展开版**

- 机制链：`parallelStream` 用 ForkJoin 拆分 Spliterator → 各分片在 commonPool 执行 → 结果按 encounter order 合并。
- 关键结构：`ForkJoinPool.commonPool()`、`parallelism` 默认 `availableProcessors-1`、`Spliterator.trySplit` 决定拆分质量。
- 触发条件：数据源可高效拆分（ArrayList、数组好，LinkedList、迭代器差）、单元素计算成本高。
- 例外：`forEach` 在并行下不保证顺序且会丢更新；`forEachOrdered` 有顺序但损失并行收益。
- 对比取舍：并行换吞吐，代价是共享池争用、调试困难和结果不确定性，多数业务场景收益有限。

**追问预判**

- 并行流里调 RPC 会怎样 → 阻塞占满 commonPool，其他并行任务和默认 CompletableFuture 一起排队。
- 怎么自定义线程池 → 把流收集成 List 后用自定义 ForkJoinPool 提交，或改用 CompletableFuture 编排。
- 为什么 LinkedList 不适合并行 → 拆分效率低，`trySplit` 只能对半切，遍历本身是 O(n)。

**常见错误答法**

- 说并行流一定比串行快（小数据量下更慢）。
- 在并行流里用普通集合 add 收集结果（线程不安全，会丢数据）。

**记忆锚点**

commonPool 全局共享 → 只适合计算密集 → 别做阻塞 IO → 收集用 collector。

---

### 17. 【兜底】函数式：Optional 的常见误用有哪些？

**口述版**

Optional 的本意是表达"可能没有返回值"，设计上就是给方法返回值用的。常见误用有四类：一是当字段或方法参数用，Optional 不可序列化，做 DTO 和实体字段会出问题，参数用 Optional 也只会让调用方多包一层；二是用 isPresent 加 get 的写法，等于把 null 检查换个皮，还更容易忘判空；三是集合里套 Optional，应该直接用空集合表示没有；四是无脑用 orElse，注意 orElse 的参数不管有没有值都会被求值，构造开销大时要用 orElseGet。正确用法是 map、flatMap、filter 链式处理，或者 orElseThrow 明确失败语义，不要在 Optional 上再返回 null。它解决的是可读性和显式性，不是性能。

**短版**

Optional 只用于返回值：别做字段和参数（不可序列化）、别用 isPresent 加 get 代替判空、别在集合里套、有开销的默认值用 orElseGet；用 map/flatMap/orElseThrow 链式表达。

**展开版**

- 机制链：`ofNullable` 包值 → `map` 在存在时转换 → `orElse` 立即求值 / `orElseGet` 惰性求值 → `orElseThrow` 明确失败。
- 关键结构：`Optional` 是 final 值对象，内部一个 value 字段，未实现 Serializable。
- 触发条件：可能返回空值的查询方法、级联取值、需要显式区分"无值"的 API。
- 例外：基本类型有 OptionalInt、OptionalLong、OptionalDouble，避免装箱；集合返回空集合比返回 Optional 更好。
- 对比取舍：显式空值提升可读性，代价是多一层包装、不能序列化、链式过长反而难读。

**追问预判**

- orElse 和 orElseGet 的区别 → orElse 的参数总会求值，orElseGet 只在为空时调用 Supplier。
- Optional 能解决 NPE 吗 → 只能约束返回值这一层，链式里仍要小心 map 返回 null 的情况。
- 为什么不建议做字段 → 不可序列化、JPA 等框架不支持，且会让对象模型变复杂。

**常见错误答法**

- 用 `optional.get()` 前不判断，等于换了个方式 NPE。
- 在 Optional 上返回 null（应该返回 `Optional.empty()`）。

**记忆锚点**

只用于返回值 → map 链式 → 惰性用 orElseGet → 别当字段和参数。

---

## 五、异常与资源管理

### 18. 【必答】异常：异常体系怎么分层，checked 和 unchecked 怎么选？

**口述版**

Throwable 下面分 Error 和 Exception。Error 是 JVM 层面的严重问题，比如 OOM、StackOverflowError，业务代码不该捕获，捕获了也处理不了。Exception 再分 checked 和 unchecked：checked 继承 Exception 但不是 RuntimeException，编译器强制处理；unchecked 继承 RuntimeException，不强制。选型原则是：调用方能合理恢复的用 checked，比如文件不存在可以重试或降级；编程错误、参数非法、状态不合法用 unchecked，比如 IllegalArgumentException、IllegalStateException。实践上业务系统更倾向于统一用 unchecked 加全局异常处理器，因为 checked 会污染每一层的方法签名，在 lambda 和 Stream 里也很难用，但前提是不能把异常吞掉，要保留 cause 和上下文。

**短版**

Throwable 分 Error 与 Exception，Error 不该捕获；Exception 分 checked（编译器强制处理）和 unchecked（RuntimeException）；能恢复用 checked，编程错误用 unchecked，业务系统多用 unchecked 加全局处理器。

**展开版**

- 机制链：异常抛出 → 沿调用栈查找匹配的 catch → 未捕获则由线程的 UncaughtExceptionHandler 处理 → 线程终止。
- 关键结构：`Throwable` 的 message、cause、suppressed、stackTrace 四个要素；`try-catch-finally` 与 try-with-resources 的编译产物。
- 触发条件：checked 由编译器在方法签名上强制声明，unchecked 无编译检查。
- 例外：`finally` 中的 return 或抛异常会覆盖原异常；`Error` 中的 `AssertionError` 有时被断言库捕获。
- 对比取舍：checked 强制调用方直面失败，代价是签名污染和 lambda 不友好；unchecked 简洁，代价是容易漏处理。

**追问预判**

- 为什么 lambda 里用 checked 异常很难受 → 函数式接口没声明 throws，只能在 lambda 内部包成 RuntimeException。
- 自定义异常继承哪个 → 业务可恢复的继承 Exception，参数和状态错误继承 RuntimeException。
- 捕获 Exception 合适吗 → 顶层兜底可以，但要记录并区分，不能吞掉后返回默认值。

**常见错误答法**

- 捕获 Throwable 或 Exception 后只打印不处理，等于吞异常。
- 把 OOM 也包成业务异常重试（Error 级别无法恢复）。

**记忆锚点**

Error 不捕 → checked 强制处理 → unchecked 表编程错误 → 顶层兜底但别吞。

---

### 19. 【必答】异常：try-with-resources 的关闭顺序与抑制异常？

**口述版**

try-with-resources 要求资源实现 AutoCloseable，编译器会生成 finally 块，按声明的逆序调用 close，后声明的先关。它的关键价值在于抑制异常：如果 try 块里抛了异常，close 又抛一个，原来的异常会被保留，close 的异常作为 suppressed 挂在它上面，用 getSuppressed 能拿到，不会像手写 finally 那样把主异常覆盖掉。JDK 7 起每个资源单独声明，JDK 9 起可以把已声明的 effectively final 变量直接放进去。要注意关闭顺序要和资源依赖方向一致，先开连接再开语句，关闭时就是语句先于连接；还有 close 里不要抛新异常，虽然会被抑制，但会掩盖真实的失败原因。

**短版**

资源实现 AutoCloseable，编译器生成 finally 按声明逆序 close；主异常保留，close 的异常作为 suppressed 挂上去，可用 getSuppressed 取；JDK 9 支持直接传已声明变量。

**展开版**

- 机制链：try 块执行 → 异常暂存 → 逆序 close → close 异常 addSuppressed 到主异常 → 抛出主异常。
- 关键结构：`AutoCloseable.close()` 声明 throws Exception；`Throwable.addSuppressed/getSuppressed`；`Closeable` 是其子接口。
- 触发条件：任何实现了 AutoCloseable 的资源，包括流、连接、语句、锁包装类。
- 例外：资源为 null 时不会调 close；JDK 9 之前必须在 try 括号里新建，不能传已有变量。
- 对比取舍：自动关闭并保留主异常，代价是 close 的失败容易被忽略，排查时要主动打印 suppressed。

**追问预判**

- 不加 try-with-resources 会怎样 → 手写 finally 时 close 抛异常会覆盖主异常，丢失根因。
- 多个资源 close 抛异常怎么排 → 按逆序关闭，后续 close 异常继续追加为 suppressed。
- close 抛异常会影响返回值吗 → 不会改变 try 的返回值，但异常会传播出去。

**常见错误答法**

- 说 close 异常会覆盖主异常（try-with-resources 是抑制）。
- 认为资源声明顺序就是关闭顺序（是逆序）。

**记忆锚点**

逆序 close → 主异常保留 → close 异常 suppressed → 依赖方向决定声明顺序。

---

### 20. 【进阶】异常：异常的性能代价在哪，finally 里 return 有什么坑？

**口述版**

抛异常的开销主要在构造时填充栈轨迹：fillInStackTrace 要遍历整个调用栈，越深越慢，所以异常不该用来做流程控制，比如用异常判断用户是否存在。优化手段是复用无栈异常单例，或者重写 fillInStackTrace 只保留必要信息，但要注意别丢掉排查线索。另一个高频坑是 finally 里 return：它会覆盖 try 里的返回值和异常，让异常凭空消失，语义非常反直觉。finally 里修改返回值对基本类型无效，因为返回值已经算好压栈了；对引用类型改字段才会生效。规范做法是 finally 只做资源清理，不写 return 也不抛异常。

**短版**

异常开销在 fillInStackTrace 遍历调用栈，所以不能拿异常做流程控制；finally 里 return 会覆盖返回值和异常，基本类型改不动、引用类型改字段才生效；finally 只做清理。

**展开版**

- 机制链：throw → 分配异常对象 → fillInStackTrace 记录栈帧 → 栈展开查找 catch → 未命中继续向上。
- 关键结构：`Throwable.fillInStackTrace`、`stackTrace` 数组、JIT 对隐式异常（如 NPE）的优化。
- 触发条件：高频循环里抛异常、把异常当分支判断、在异常构造里拼大量字符串。
- 例外：JIT 可能省略部分栈信息；`-XX:-OmitStackTraceInFastThrow` 可避免复用无栈异常导致看不到堆栈。
- 对比取舍：保留完整栈利于排查但慢，去栈异常快但丢失现场，通常只对已知的高频控制流异常做裁剪。

**追问预判**

- 为什么线上有时看不到异常堆栈 → JIT 对反复抛出的同类异常用无栈单例，需要加 `-XX:-OmitStackTraceInFastThrow`。
- finally 里 return 会吞异常吗 → 会，try 中的异常被丢弃，只剩 finally 的返回值。
- 怎么避免异常做流程控制 → 用返回值或 Optional 表达"没有"，异常只留给真正的异常情况。

**常见错误答法**

- 说异常性能开销可以忽略，随便用（高频场景影响明显）。
- 在 finally 里写 return 做兜底返回（会吞掉真实异常）。

**记忆锚点**

开销在 fillInStackTrace → 别做流程控制 → finally 别 return → 只做资源清理。

---

## 六、反射、动态代理与 SPI

### 21. 【必答】反射与代理：反射的调用链路与性能代价，`setAccessible` 做了什么？

**口述版**

反射调用分三步：拿到 Class，再 getMethod 拿到 Method 对象，最后 invoke。invoke 时 JVM 要做访问检查、参数装箱拆箱、把参数打包成 Object 数组，早期版本每次都要走 native 调用，所以比直接调用慢一个数量级。JDK 后面加了 inflation 机制，同一个方法反射调用超过阈值后会生成 GeneratedMethodAccessor 字节码来加速，默认阈值 15，可以用 `-Dsun.reflect.inflationThreshold` 调整。`setAccessible(true)` 的作用是关闭访问检查，让私有成员也能访问，代价是破坏封装；JDK 17 强封装之后，对 JDK 内部类这么做会抛 InaccessibleObjectException，需要加 `--add-opens`。框架里通常把 Method 缓存起来，摊薄查找成本。

**短版**

反射链路是取 Class → 取 Method → invoke，invoke 有访问检查、装箱和参数数组开销；inflation 超阈值后生成 MethodAccessor 加速；`setAccessible` 关闭访问检查、破坏封装，JDK 17 起对内部类需 `--add-opens`。

**展开版**

- 机制链：`getDeclaredMethod` 扫描方法表 → `invoke` 做访问校验 → 参数封装为 Object 数组 → MethodAccessor 执行 → 返回值装箱。
- 关键结构：`MethodAccessor` 的 native 与 generated 两种实现、`inflationThreshold`、`Reflection.getCallerClass`。
- 触发条件：框架做依赖注入、ORM 映射、序列化、AOP 调用、注解解析。
- 例外：方法句柄 `MethodHandle` 和 `VarHandle` 可以做到接近直接调用的性能；直接调用在 JIT 内联后远快于反射。
- 对比取舍：反射换来通用性和动态性，代价是性能和封装破坏；热点路径应缓存 Method 或改用 MethodHandle、代码生成。

**追问预判**

- 反射为什么慢 → 访问检查、参数数组分配与装箱、无法内联、早期走 native 边界。
- 怎么优化反射 → 缓存 Method、`setAccessible(true)` 跳过检查、用 MethodHandle 或 LambdaMetafactory 转成函数式接口。
- JDK 17 之后框架为什么报 InaccessibleObjectException → 强封装默认禁止反射 JDK 内部 API，需要显式 `--add-opens`。

**常见错误答法**

- 说反射每次都比直接调用慢十倍以上（inflation 后差距已缩小，但仍不可内联）。
- 认为 `setAccessible` 能突破模块系统的所有限制（强封装下会被拒绝）。

**记忆锚点**

取 Class → 取 Method → invoke 三重开销 → 缓存 + inflation → JDK 17 强封装。

---

### 22. 【必答】反射与代理：JDK 动态代理和 CGLIB 有什么区别，怎么选？

**口述版**

JDK 动态代理基于接口，用 `Proxy.newProxyInstance` 在运行时生成一个实现类，所有方法调用都进 InvocationHandler 统一拦截，所以它只能代理接口方法，目标类必须有接口。CGLIB 基于继承，用 ASM 生成目标类的子类，重写方法做拦截，所以能代理普通类，但 final 类和 final 方法代理不了，private 方法也拦不到。选型上，Spring AOP 早期是有接口用 JDK、没接口用 CGLIB，Spring Boot 2 之后默认统一用 CGLIB，因为不用强制转接口、行为也更一致。要注意自调用问题：不管哪种代理，方法内部用 this 调用另一个方法都不经过代理，事务和缓存注解会失效，得靠注入自身或 AopContext 解决。

**短版**

JDK 代理基于接口、生成实现类、InvocationHandler 拦截，只能代理接口方法；CGLIB 基于继承、生成子类，能代理普通类但 final 和 private 拦不到；Boot 2 起默认 CGLIB。两者都有自调用失效问题。

**展开版**

- 机制链：JDK 代理运行时生成 `$Proxy` 类实现接口 → 方法调用转给 InvocationHandler → 反射调用目标方法。
- 关键结构：`Proxy`、`InvocationHandler`、`Enhancer`、`MethodInterceptor`；CGLIB 生成类的命名通常带 `$$`。
- 触发条件：Spring AOP 切面、RPC 客户端桩、事务和缓存、MyBatis Mapper 接口。
- 例外：JDK 16 起 `Proxy` 也支持部分类代理的限制；final 类无法被 CGLIB 继承，只能改用接口。
- 对比取舍：JDK 代理无三方依赖、生成类少，但要求接口；CGLIB 通用但生成类更重、首次创建慢、可能触发 Metaspace 增长。

**追问预判**

- 为什么自调用会失效 → 内部调用用的是原始 this，不经过代理对象，切面逻辑不执行。
- 怎么判断当前用的是哪种代理 → 打印 `AopUtils.isJdkDynamicProxy` 或看类名是否含 `$Proxy`、`$$`。
- CGLIB 代理类多了会怎样 → 每次增强都生成新类，大量动态类会推高 Metaspace，需要缓存或复用。

**常见错误答法**

- 说 JDK 动态代理能代理任意类（必须基于接口）。
- 认为 CGLIB 能拦截 private 和 final 方法（做不到）。

**记忆锚点**

JDK 靠接口生成实现类 → CGLIB 靠继承生成子类 → 自调用都失效 → Boot 2 默认 CGLIB。

---

### 23. 【进阶】反射与代理：注解的保留策略与生效方式，注解处理器和运行期反射怎么分工？

**口述版**

注解按保留策略分三种：SOURCE 只在源码存在，编译后就丢，比如 @Override；CLASS 保留在 class 文件但运行期读不到，是默认值；RUNTIME 才能被反射读到，Spring 的注解基本都是这种。注解本身没有行为，它只是元数据，逻辑靠读取方实现。运行期注解由框架用 `getAnnotation` 或 AnnotatedElement 读取，再决定怎么处理，比如看到 @Transactional 就生成代理。编译期注解则由注解处理器 APT 在编译时扫描，用来生成代码或做校验，比如 Lombok、MapStruct、MapStruct 的 Mapper 实现。所以自定义注解必须配处理器或者配拦截逻辑才有用，光定义注解什么都不会发生。

**短版**

注解保留策略分 SOURCE、CLASS（默认）、RUNTIME，只有 RUNTIME 能反射读到；注解只是元数据，运行期靠框架反射读取，编译期靠 APT 生成代码或校验；自定义注解要配处理器或拦截逻辑才生效。

**展开版**

- 机制链：javac 解析注解 → 按 Retention 决定写入位置 → RUNTIME 写进 RuntimeVisibleAnnotations → 运行期反射读取。
- 关键结构：`@Retention`、`@Target`、`@Inherited`、`@Repeatable`；APT 通过 `Processor` 和 `RoundEnvironment` 工作。
- 触发条件：框架启动扫描、编译期代码生成、切点匹配、参数校验。
- 例外：`@Inherited` 只对类生效且只影响类继承，不影响接口和字段；组合注解要用 `@AliasFor` 才能传递属性。
- 对比取舍：运行期反射灵活但启动慢、无法做编译期检查；APT 生成代码性能好、能提前报错，但构建链复杂、调试难。

**追问预判**

- 为什么自定义注解不生效 → 保留策略没设成 RUNTIME，或者根本没有读取它的代码。
- 注解能继承吗 → 类上的 @Inherited 可以让子类"看到"，方法注解不会继承。
- Lombok 是注解处理器吗 → 是，它通过 APT 在编译期修改 AST 生成代码，所以 IDE 需要装插件。

**常见错误答法**

- 以为定义了注解就自动有行为（注解只是标记）。
- 说注解默认保留到运行期（默认是 CLASS）。

**记忆锚点**

SOURCE / CLASS / RUNTIME → 默认 CLASS → 运行期靠反射读 → 编译期靠 APT。

---

### 24. 【兜底】反射与代理：SPI 是什么，ServiceLoader 怎么和类加载配合？

**口述版**

SPI 是服务提供者接口，调用方定义接口，第三方在 jar 的 META-INF/services 下按接口全限定名建一个文件，内容是实现类的全限定名，ServiceLoader 负责加载并实例化。它解决的是"框架定义接口、实现由外部提供"的扩展问题，JDBC 驱动、SLF4J 的日志实现都是这么做的。它和双亲委派有冲突：加载接口的是上层加载器，而实现类在应用的 classpath 里，父加载器看不到子加载器的类，所以 ServiceLoader 改用线程上下文类加载器去加载实现。SPI 的缺点是一次性实例化所有实现、不能按需获取，也不能注入依赖，所以 Dubbo 自己实现了一套带 key 的 ExtensionLoader，Spring 也有自己的扩展机制。

**短版**

SPI 是接口由框架定义、实现在 `META-INF/services` 声明、ServiceLoader 加载；因父加载器看不到子类，改用线程上下文类加载器；缺点是全量实例化、不可按需、不能注入，Dubbo 用 ExtensionLoader 增强。

**展开版**

- 机制链：`ServiceLoader.load(接口)` → 用线程上下文类加载器找 `META-INF/services/接口全名` → 逐行读实现类名 → 反射实例化。
- 关键结构：`ServiceLoader`、`META-INF/services` 目录、线程上下文类加载器、JDK 9 的 `module-info` 中 `provides ... with ...`。
- 触发条件：JDBC 4.0 自动发现驱动、日志门面绑定实现、插件式扩展点。
- 例外：JDK 9 模块化后可以用 module 声明替代 services 文件；同一接口可以声明多个实现，按迭代顺序全部实例化。
- 对比取舍：SPI 换来零配置扩展，代价是全量加载、无法按名称选择、异常在遍历时才暴露、实现类无法依赖注入。

**追问预判**

- 为什么必须用线程上下文类加载器 → 接口由父加载器加载，实现类在子加载器路径上，父加载器无法直接加载子类。
- 怎么按 key 取实现 → 原生 SPI 做不到，需要自己解析文件内容或改用 Dubbo 的 ExtensionLoader。
- 实现类实例化了却没用上会怎样 → 浪费启动时间和内存，实现类构造里有副作用时还可能出问题。

**常见错误答法**

- 说 SPI 是通过反射扫描所有类找实现（是读声明文件）。
- 认为 ServiceLoader 遵守双亲委派加载实现（用的是上下文类加载器）。

**记忆锚点**

接口在外、实现在内 → services 文件声明 → 上下文加载器加载 → 全量实例化不可选。

---

## 七、版本特性

### 25. 【必答】版本特性：record 和普通类有什么区别，适用边界在哪？

**口述版**

record 是 JDK 16 正式的类型，用来声明不可变的数据载体。编译器自动生成全参构造、访问器、equals、hashCode、toString，字段都是 private final，类默认 final，既不能继承别人也不能被别人继承。它适合 DTO、查询结果、事件、坐标这类纯数据，能省掉大量样板代码，语义上也更明确。边界是：需要可变状态、需要无参构造给框架反射用、需要继承体系时就不合适，JPA 实体一般不能用 record。可以加紧凑构造器做参数校验，也可以实现接口、加自定义方法，但不能声明额外的实例字段，静态字段可以。

**短版**

record 自动生成全参构造、访问器、equals、hashCode、toString，字段 private final、类 final 不可继承；适合不可变数据载体，不适合需要可变、无参构造或继承体系的场景；可加紧凑构造器校验。

**展开版**

- 机制链：javac 为 record 生成规范构造器、访问器、equals/hashCode/toString，并标记为 final 与 java.lang.Record 子类。
- 关键结构：`java.lang.Record` 基类、RecordComponents 反射信息、紧凑构造器 `public Foo { ... }`。
- 触发条件：方法返回多字段结果、消息事件、配置快照、Map 的复合 key。
- 例外：record 可以实现接口、可以定义静态字段和方法、可以有实例方法，但不能声明实例字段也不能被继承。
- 对比取舍：省样板并保证不可变，代价是与依赖 setter 和无参构造的框架不兼容，序列化也要注意自定义逻辑。

**追问预判**

- record 能序列化吗 → 可以，但反序列化走规范构造器，不能像普通类那样绕过构造器赋值。
- 能做 JPA 实体吗 → 一般不行，JPA 需要无参构造和可变字段，更适合用普通类加 Lombok。
- 紧凑构造器能改参数吗 → 可以，在里面重新赋值参数，字段会按新值初始化，适合做校验和规范化。

**常见错误答法**

- 说 record 可以继承普通类（只能实现接口）。
- 认为 record 的字段能被反射修改（final，且运行时受保护）。

**记忆锚点**

自动六件套 → 字段全 final → 类 final → 数据载体限定，不兼容 setter 框架。

---

### 26. 【进阶】版本特性：密封类和 switch 模式匹配解决什么问题？

**口述版**

密封类用 `sealed` 加 `permits` 限定哪些类可以继承它，子类必须是 final、sealed 或 non-sealed，这样就形成了封闭的继承体系。它解决的是"类型集合已知且有限"的建模问题，比如支付状态、表达式节点、消息类型。配合 switch 模式匹配之后，编译器能判断所有子类型是否都被覆盖，不需要 default 分支，漏写一个会编译报错，这就是穷尽性检查。实际收益是把运行期的 if-else 类型判断变成编译期可校验的分支，将来加新子类时所有 switch 会立刻报错提醒你补逻辑，比开放继承安全得多。代价是扩展性受限，外部不能再新增子类型。

**短版**

sealed 加 permits 限定子类范围，子类必须 final、sealed 或 non-sealed；配合 switch 模式匹配能做穷尽性检查，不用 default，漏分支编译报错；适合状态机、表达式、消息等有限封闭类型，代价是外部无法扩展。

**展开版**

- 机制链：sealed 声明 → permits 列出允许的子类 → 子类标注 final/sealed/non-sealed → switch 按类型分支 → 编译器验证穷尽性。
- 关键结构：`sealed`、`permits`、`non-sealed`、`PermittedSubclasses` 属性、模式匹配中的类型模式和 record 解构。
- 触发条件：领域模型类型集合固定、需要按类型分支处理、替代带 type 字段的枚举式设计。
- 例外：permits 可以省略，由同一编译单元中的子类推断；用 `non-sealed` 可以重新开放某个分支的继承。
- 对比取舍：封闭换来穷尽性和安全演进，代价是模型不能跨模块扩展；开放继承灵活但只能靠 default 兜底，容易漏分支。

**追问预判**

- 为什么能省掉 default → 编译器知道全部允许的子类型，覆盖完就穷尽，`default` 反而是死代码。
- 加新子类会怎样 → 所有相关 switch 编译失败，强制你补齐分支，这是它最大的价值。
- 和枚举怎么选 → 枚举适合无数据的常量集合，密封类适合每个分支带不同数据的场景。

**常见错误答法**

- 说密封类就是 final 类的别名（允许有限个已知子类）。
- 认为 switch 模式匹配必须写 default（穷尽时不需要）。

**记忆锚点**

sealed + permits → 子类 final/sealed/non-sealed → switch 穷尽检查 → 加子类即编译报错。

---

### 27. 【兜底】版本特性：虚拟线程的适用边界，和现有线程池模型怎么共存？

**口述版**

虚拟线程是 JDK 21 正式的特性，由 JVM 调度而不是操作系统调度，阻塞时自动从载体线程上卸载，所以创建成本极低，可以一个请求一个线程。它适合 IO 密集、阻塞时间长的场景，比如调下游 HTTP、查数据库，能把原来靠线程池加异步回调的复杂度降下来，代码又回到同步写法。边界是：CPU 密集任务没有收益，因为核还是那几个；JDK 21 里 synchronized 块内阻塞会钉住载体线程，要改用 ReentrantLock，JDK 24 的 JEP 491 已解决这个问题；大量使用 ThreadLocal 会带来内存压力；池化虚拟线程没有意义，直接 new 即可。共存策略是：IO 并发用虚拟线程，计算型任务仍用固定线程池，用信号量控制对下游的并发度。

**短版**

虚拟线程由 JVM 调度、阻塞时卸载，创建成本极低，适合 IO 密集、一个请求一个线程；CPU 密集无收益，JDK 21 的 synchronized 阻塞会钉住载体线程（JDK 24 已修复），别池化、别滥用 ThreadLocal。

**展开版**

- 机制链：虚拟线程挂载到载体线程（ForkJoinPool）→ 遇到阻塞 IO 卸载 → 状态存到堆 → IO 就绪后重新挂载继续执行。
- 关键结构：载体线程池、Continuation、`Thread.ofVirtual()`、`Executors.newVirtualThreadPerTaskExecutor()`、结构化并发预览 API。
- 触发条件：HTTP 调用、数据库访问、消息等待等高阻塞低计算场景；并发量远大于核数的场景。
- 例外：CPU 密集、`synchronized` 长阻塞（JDK 21）、JNI 调用会钉住载体线程；虚拟线程不是更快的线程，只是更省资源。
- 对比取舍：同步代码换高并发，代价是失去线程池的排队与隔离能力、调试和监控工具适配滞后、下游要能承受真实并发。

**追问预判**

- 为什么不能池化虚拟线程 → 创建成本极低，池化反而引入排队和状态复用问题，直接每个任务一个更简单。
- 用什么限制下游并发 → 用 Semaphore 或固定大小的信号量控制，而不是虚拟线程池大小。
- 迁移要改什么 → 替换 Executor、去掉 ThreadLocal 滥用、把 synchronized 长阻塞改 ReentrantLock、检查依赖库的阻塞点。

**常见错误答法**

- 说虚拟线程比平台线程快（是更省内存、更高并发，不是更快）。
- 把虚拟线程放进固定大小线程池使用（池化失去意义）。

**记忆锚点**

JVM 调度 → 阻塞即卸载 → IO 密集专用 → 不池化、限流用信号量。

---

