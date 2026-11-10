# Java 后端八股文

## 一、Java 基础

### 1. `==` 和 `equals()` 的区别？
- `==`：基本类型比较值；引用类型比较内存地址。
- `equals()`：Object 默认实现就是 `==`，String、Integer 等重写后比较内容。
- 重写 `equals()` 必须同时重写 `hashCode()`，否则在 HashMap / HashSet 中会出现"相等的对象放进了不同桶"的问题。

### 2. String、StringBuilder、StringBuffer 区别？
- String 不可变（`final` 修饰的字节数组），每次修改都会生成新对象；天然线程安全，可放入字符串常量池。
- StringBuilder 可变、非线程安全、性能最好。
- StringBuffer 可变、方法加了 `synchronized`，线程安全但较慢。

### 3. 为什么 String 设计成不可变？
- 支持字符串常量池复用；hashCode 可以缓存，适合作为 HashMap 的 key；
- 线程安全；安全性（类加载、网络地址、文件路径等参数不会被篡改）。

### 4. 重载和重写的区别？
- 重载（Overload）：同一个类中方法名相同、参数列表不同，编译期决定。
- 重写（Override）：子类重新实现父类方法，方法签名相同，运行期动态绑定；返回值类型 ≤ 父类，抛出异常 ≤ 父类，访问权限 ≥ 父类。

### 5. 接口和抽象类的区别？
- 抽象类可以有构造器、成员变量和普通方法，单继承；表达 "is-a"。
- 接口只能有常量，Java 8 起可有 `default` / `static` 方法，Java 9 起可有 `private` 方法；类可实现多个接口；表达 "has-a / 能力"。

### 6. 自动装箱拆箱与 Integer 缓存？
- `Integer a = 127` 调用 `Integer.valueOf()`，-128~127 会使用缓存，所以 `127 == 127` 为 true，`128 == 128` 为 false。
- 包装类比较一律用 `equals()`；拆箱时 null 会抛 NPE。

### 7. 深拷贝和浅拷贝？
- 浅拷贝：只复制对象本身，内部引用字段仍指向同一对象（`Object.clone()` 默认行为）。
- 深拷贝：递归复制引用的对象。实现方式：手写 clone、序列化 / 反序列化、JSON 转换等。

### 8. 异常体系？
- `Throwable` → `Error`（OOM、StackOverflow，程序一般无法处理）和 `Exception`。
- `Exception` → 受检异常（IOException，必须 try/catch 或 throws）和运行时异常（NPE、数组越界）。
- `finally` 一般一定执行，除非 `System.exit()` 或线程被杀；不要在 finally 中 return。

### 9. 反射是什么？优缺点？
- 运行时获取类信息、调用方法、修改字段。Spring IoC、动态代理、注解处理都依赖反射。
- 缺点：性能开销、破坏封装、绕过编译期检查。

### 10. JDK 动态代理和 CGLIB 区别？
- JDK 动态代理：基于接口，`Proxy` + `InvocationHandler`。
- CGLIB：基于继承生成子类，不能代理 `final` 类 / 方法。
- Spring AOP：目标类实现了接口默认用 JDK 代理，否则用 CGLIB（Spring Boot 2.x 起默认 CGLIB）。

## 二、集合

### 1. ArrayList 和 LinkedList 区别？
- ArrayList 基于动态数组，随机访问 O(1)，尾插均摊 O(1)，中间插入删除需搬移元素；扩容为 1.5 倍。
- LinkedList 基于双向链表，插入删除 O(1)（找到节点后），随机访问 O(n)，内存占用更大。
- 实际开发几乎都用 ArrayList。

### 2. HashMap 底层原理（JDK 1.8）？
- 数组 + 链表 + 红黑树。链表长度 ≥ 8 且数组长度 ≥ 64 时转红黑树，节点 ≤ 6 时退化为链表。
- hash 计算：`(h = key.hashCode()) ^ (h >>> 16)`，让高位参与运算减少冲突；下标 `(n - 1) & hash`。
- 默认容量 16，负载因子 0.75，超过阈值扩容 2 倍；扩容时节点要么在原位置，要么在"原位置 + 旧容量"。
- 容量为 2 的幂：用位运算代替取模，且扩容迁移简单。

### 3. HashMap 为什么线程不安全？
- JDK 1.7 头插法在并发扩容时可能形成环形链表导致死循环。
- JDK 1.8 改为尾插法，但并发 put 仍可能数据覆盖、size 不准确。

### 4. ConcurrentHashMap 如何保证线程安全？
- JDK 1.7：Segment 分段锁（继承 ReentrantLock）。
- JDK 1.8：Node 数组 + CAS + `synchronized` 锁住桶的头节点，粒度更细；支持多线程协助扩容。
- key 和 value 都不允许为 null（避免并发下的二义性）。

### 5. HashMap 和 Hashtable 区别？
- Hashtable 方法全加 `synchronized`，不允许 null 键值，已过时；HashMap 非线程安全，允许一个 null key。

### 6. fail-fast 和 fail-safe？
- fail-fast：迭代时检测到 `modCount` 变化抛 `ConcurrentModificationException`（ArrayList、HashMap）。
- fail-safe：迭代副本或弱一致性视图（CopyOnWriteArrayList、ConcurrentHashMap）。
- 遍历中删除元素要用 `Iterator.remove()` 或 `removeIf()`。

## 三、JVM

### 1. JVM 内存区域？
- 线程私有：程序计数器、虚拟机栈（栈帧：局部变量表、操作数栈、动态链接、返回地址）、本地方法栈。
- 线程共享：堆（对象实例，GC 主要区域）、方法区（JDK 8 起由元空间实现，使用本地内存）。
- 字符串常量池在 JDK 7 起移到堆中。

### 2. 如何判断对象可以回收？
- 引用计数法：无法解决循环引用，JVM 不用。
- 可达性分析：从 GC Roots（栈中局部变量、静态变量、常量、JNI 引用、锁持有的对象等）出发，不可达即可回收。
- 四种引用：强、软（内存不足回收，适合缓存）、弱（下次 GC 回收，ThreadLocalMap 的 key）、虚（跟踪回收）。

### 3. 垃圾回收算法？
- 标记-清除：会产生碎片。
- 标记-复制：无碎片但浪费一半空间，适合新生代（Eden : S0 : S1 = 8 : 1 : 1）。
- 标记-整理：无碎片，移动对象成本高，适合老年代。
- 分代收集：新生代用复制，老年代用标记-清除 / 整理。

### 4. 常见垃圾收集器？
- Serial / ParNew / Parallel Scavenge（新生代），Serial Old / Parallel Old / CMS（老年代）。
- CMS：以最短停顿为目标，初始标记 → 并发标记 → 重新标记 → 并发清除；缺点是碎片、浮动垃圾，JDK 14 已移除。
- G1：JDK 9 起默认，堆划分为多个 Region，可预测停顿时间，整体标记-整理、局部复制。
- ZGC：亚毫秒级停顿，染色指针 + 读屏障，JDK 21 起支持分代。

### 5. 对象什么时候进入老年代？
- 年龄达到阈值（默认 15）；大对象直接进入；Survivor 中相同年龄对象大小总和超过一半时，大于等于该年龄的直接晋升；Minor GC 后 Survivor 放不下。

### 6. 类加载过程？
- 加载 → 验证 → 准备（静态变量赋零值）→ 解析 → 初始化（执行 `<clinit>`）→ 使用 → 卸载。

### 7. 双亲委派模型？
- 类加载器收到请求先委托父加载器（Bootstrap → Platform/Extension → Application），父加载器无法加载才自己加载。
- 好处：避免类重复加载，保护核心类（无法自定义 `java.lang.String`）。
- 打破：Tomcat 为隔离 Web 应用、SPI 使用线程上下文类加载器。

### 8. OOM 排查思路？
- 加 `-XX:+HeapDumpOnOutOfMemoryError` 导出 dump，用 MAT / JProfiler 分析大对象和引用链。
- 常用命令：`jps`、`jstat -gcutil`、`jmap -histo`、`jstack`（排查死锁、CPU 飙高线程）。

## 四、并发编程

### 1. 线程的创建方式？
- 继承 Thread、实现 Runnable、实现 Callable + FutureTask、线程池。本质上都是 `new Thread().start()`。
- Java 21 引入虚拟线程：`Thread.ofVirtual().start()` / `Executors.newVirtualThreadPerTaskExecutor()`，适合大量 IO 密集型任务。

### 2. 线程有哪些状态？
- NEW、RUNNABLE、BLOCKED（等待 monitor 锁）、WAITING、TIMED_WAITING、TERMINATED。

### 3. `sleep()` 和 `wait()` 区别？
- `sleep()` 是 Thread 的静态方法，不释放锁；`wait()` 是 Object 方法，必须在 synchronized 中调用，会释放锁，需要 `notify()` 唤醒或超时。

### 4. synchronized 原理？
- 修饰代码块：`monitorenter` / `monitorexit` 指令；修饰方法：`ACC_SYNCHRONIZED` 标志。底层依赖对象头 Mark Word 和 Monitor。
- 锁升级：无锁 → 偏向锁 → 轻量级锁（CAS 自旋）→ 重量级锁（JDK 15 起默认禁用偏向锁）。

### 5. synchronized 和 ReentrantLock 区别？
- 都是可重入锁。synchronized 是 JVM 关键字，自动释放；ReentrantLock 是 API 层面，需要手动 `unlock()`（放在 finally）。
- ReentrantLock 额外支持：公平锁、可中断 `lockInterruptibly()`、超时 `tryLock()`、多个 Condition。

### 6. volatile 的作用？
- 保证可见性（写后立即刷回主存，读时从主存读）和禁止指令重排（内存屏障）。
- 不保证原子性，`i++` 仍不安全。经典应用：DCL 单例中修饰实例变量。

### 7. JMM 的三大特性？
- 原子性、可见性、有序性。happens-before 规则：程序顺序、锁、volatile、线程 start/join、传递性等。

### 8. CAS 是什么？有什么问题？
- Compare And Swap，基于 CPU 原子指令的乐观锁，Atomic 类基于它实现。
- 问题：ABA（用 `AtomicStampedReference` 加版本号解决）、自旋开销大、只能保证单个变量原子性。

### 9. AQS 原理？
- AbstractQueuedSynchronizer：一个 `volatile int state` + CLH 双向等待队列。
- 获取失败的线程入队并 park，释放时唤醒后继节点。ReentrantLock、Semaphore、CountDownLatch 都基于 AQS。

### 10. 线程池核心参数？执行流程？
- 参数：corePoolSize、maximumPoolSize、keepAliveTime、unit、workQueue、threadFactory、handler。
- 流程：核心线程未满 → 创建核心线程；否则入队；队列满 → 创建非核心线程直到最大线程数；仍满 → 执行拒绝策略。
- 拒绝策略：AbortPolicy（抛异常，默认）、CallerRunsPolicy、DiscardPolicy、DiscardOldestPolicy。
- 不推荐 `Executors` 创建：FixedThreadPool 无界队列、CachedThreadPool 无限线程都可能 OOM。
- 线程数经验值：CPU 密集 N+1，IO 密集 2N（以压测为准）。

### 11. ThreadLocal 原理与内存泄漏？
- 每个 Thread 持有一个 ThreadLocalMap，key 是 ThreadLocal 的弱引用，value 是强引用。
- key 被回收后 value 仍被强引用，线程池场景下线程不销毁会造成泄漏。用完必须 `remove()`。

### 12. 死锁的四个条件？如何避免？
- 互斥、请求与保持、不可剥夺、循环等待。
- 避免：按固定顺序加锁、使用 `tryLock` 超时、一次性申请所有资源。

## 五、Spring

### 1. IoC 和 DI 是什么？
- IoC 控制反转：对象的创建和依赖关系交给容器管理；DI 依赖注入是 IoC 的实现方式。
- 注入方式：构造器注入（推荐，便于不可变和测试）、Setter 注入、字段注入（`@Autowired`）。

### 2. AOP 原理？
- 面向切面编程，基于动态代理（JDK / CGLIB）在方法前后织入通知。用于事务、日志、权限、监控。
- 概念：切面 Aspect、切点 Pointcut、通知 Advice（Before / After / AfterReturning / AfterThrowing / Around）、连接点 JoinPoint。

### 3. Bean 的生命周期？
- 实例化 → 属性填充 → Aware 接口回调 → `BeanPostProcessor` 前置 → 初始化（`@PostConstruct`、`afterPropertiesSet`、init-method）→ `BeanPostProcessor` 后置（AOP 代理在此生成）→ 使用 → 销毁（`@PreDestroy`、destroy）。

### 4. Spring 如何解决循环依赖？
- 三级缓存：一级 `singletonObjects`（完整 Bean）、二级 `earlySingletonObjects`（早期引用）、三级 `singletonFactories`（ObjectFactory，可生成代理）。
- 只能解决单例 + Setter / 字段注入的循环依赖，构造器注入和原型 Bean 无法解决。Spring Boot 2.6 起默认禁止循环依赖。

### 5. Bean 的作用域？
- singleton（默认）、prototype、request、session、application。单例 Bean 有状态时存在线程安全问题。

### 6. @Transactional 失效场景？
- 方法非 public；同类内部方法调用（不经过代理）；异常被 catch 吞掉；抛出受检异常但未配置 `rollbackFor`；数据库引擎不支持事务；Bean 未被 Spring 管理。

### 7. 事务传播行为？
- REQUIRED（默认，有则加入，无则新建）、REQUIRES_NEW（挂起当前，新建）、NESTED（嵌套，savepoint）、SUPPORTS、NOT_SUPPORTED、MANDATORY、NEVER。

### 8. Spring Boot 自动配置原理？
- `@SpringBootApplication` 包含 `@EnableAutoConfiguration`，通过 `AutoConfigurationImportSelector` 加载 `META-INF/spring/...AutoConfiguration.imports` 中的配置类，再根据 `@ConditionalOnClass`、`@ConditionalOnMissingBean` 等条件决定是否生效。

### 9. Spring MVC 请求处理流程？
- DispatcherServlet → HandlerMapping 找到 Handler → HandlerAdapter 执行 Controller → 返回 ModelAndView / 通过 HttpMessageConverter 写 JSON → ViewResolver 渲染。
