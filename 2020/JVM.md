## java内存区域

### 运行时数据区域

#### 程序计数器

##### 特点

可以看作是当前线程执行的字节码的行号指示器

每个线程都有一个独立的程序计数器，各个线程之间计数器互不影响，是线程私有的。

如果线程执行的是java方法，计数器记录的是正在执行的虚拟机字节码指令的地址；如果执行的是Native方法，计数器值为空（Undefined）

##### 异常

此内存区域是唯一一个没有OOM异常的区域

#### Java虚拟机栈

##### 特点

**线程私有**，声明周期和线程同步

每个方法执行时都会创建一个**栈帧**，用于存储局部变量表、操作数栈、动态链接、方法出口等信息。每一个方法从调用到执行的过程就是一个栈帧在虚拟机栈中入栈和出栈的过程。

一般的“栈”就指的是虚拟机栈中的**局部变量表**

局部变量表存放了编译期可知的各种基本数据类型。它所需的内存空间在编译期完成分配，当进入一个方法时，这个方法在栈帧中需要的局部变量空间是完全确定的，运行期间不会改变。

##### 异常

如果线程请求的栈深度大于虚拟机允许的深度，抛出StackOverflowError

如果虚拟机栈可以动态扩展（一般都可以），扩展时无法申请到足够的内存时，就抛出OOM

#### 本地方法栈

和虚拟机栈类似，区别是：虚拟机栈为虚拟机执行Java方法服务，本地方法栈为虚拟机执行Native方法服务

也会抛出SOE和OOM

#### Java堆

##### 特点

Java堆是被所有线程共享的一块区域，在虚拟机启动时创建

唯一目的就是存放对象实例，几乎所有的对象实例都在这里分配内存

Java堆可处于物理上不连续的内存空间中，只要逻辑上连续即可

通过 `-Xmx` 和 `-Xms` 控制堆扩展大小

##### 异常

在堆中没有内存完成实例分配，并且堆无法再扩展时，抛出OOM

#### 方法区

##### 特点

线程共享的区域

存储已经被虚拟机加载的类信息、常量、静态变量等

##### 异常

也会抛出OOM

##### 运行时常量池

是方法区的一部分，存放编译期生成的各种字面量和符号引用

#### 直接内存

在NIO中，可以通过channel和buffer进行IO，可以使用Native函数库分配堆外内存，通过一个存储在Java堆中的DirectByteBuffer对象作为这块内存的引用进行操作。避免了在Java堆和Native堆之间复制数据，影响性能

## 垃圾收集器

### 基本概念

垃圾收集（Garbage Collection，GC）的任务

- 哪些内存需要回收
- 什么时候回收
- 如何回收

java内存运行时，程序计数器、虚拟机栈、本地方法栈三个区域是随着线程而生而灭的：栈中的栈帧分配的内存基本是在类结构确定下来时就已知的，因此这几个区域的内存分配和回收都具有确定性，方法结束时或者线程结束时，内存自然跟着回收了。

java堆和方法区则不一样，一个接口中的多个实现类需要的内存可能不一样，一个方法中的多个分支需要的内存也不一样，只有在程序运行期才会知道要创建哪些对象，这部分内存的分配和回收都是动态的，GC关注的也是这部分内存。

### 对象引用问题

堆中存放着几乎所有的对象实例，GC在对堆进行回收时，第一件事就是去判断哪些对象还“活着”，哪些对象“死亡”。

#### 引用计数算法

给对象中添加一个引用计数器，每当有一个地方引用它时，计数器就加1；引用失效时就减1；当计数器为0时就表示不能再被使用。

该算法实现简单，判定效率也高，但在Java虚拟机中没有采用这种算法，其最主要的原因在于该算法无法解决“循环引用问题”。

#### 可达性分析算法

主流的商用程序语言都是通过可达性分析来判定对象是否存活。该算法的基本思路就是通过一系列被称为“GC Roots”的对象为起点，开始向下搜索，搜索路径称为引用链。当一个对象与GC Root之间没有任何引用链时，该对象就是不可用的。

###### Java中，可作为GC Root的对象如下

- 虚拟机栈（栈帧中的本地变量表）中引用的对象
- 方法区中类静态属性引用的对象
- 方法区中常量引用的对象
- 本地方法栈中JNI（即Native方法）引用的对象

#### 引用类型

**强引用**：new出来的对象引用，只要强引用还存在，垃圾收集器就永远不会回收被引用的对象

**软引用**：还有用但非必需的对象。对于软引用关联的对象，在系统即将内存溢出之前，会对这些对象列进回收列表中进行第二次回收。如果这次回收还没有足够的内存，就抛出内存溢出异常。

**弱引用**：非必需对象，其强度比软引用更弱。被弱引用关联的对象只能活到下一次GC发生之前。当GC发生时，无论当前内存是否足够，都会回收弱引用的对象。

**虚引用**：最弱的一种引用关系。一个对象是否有虚引用的存在不会对其生存时间有任何影响，也无法通过虚引用来获取一个对象实例。它存在的唯一目的就是该对象在GC时收到一个系统通知。

### GC算法

#### 标记-清除（Mark-Sweep）

- 算法分为两步，第一步标记所有需要回收的对象；第二步统一回收这些被标记的对象

- 类似于打扫屋子的“定点清除”

- 不足

- - 效率问题：标记和清除的效率都不高
  - 空间问题：清除后会产生内存碎片，不利于大对象的分配，可能会引起多次GC

#### 复制（Copying）

- 为了解决效率问题

- 将内存区域划分为两部分，一次只使用其中的一块，当这块内存用完之后，标记出其中还存活的对象，将其复制到另一块内存中，最后将这块内存完全清除

- 商业的JVM都采用这种方法，但在区域划分时的比例不是这样。

- - 在新生代中，98%的对象都是“朝生夕死”的，所以将内存分为一块Eden区和两块较小的Survive区，每次使用Eden和一块Servive区，当发生GC时，将存活的对象都移动到那块没有使用的Survive区，清理Eden和刚才使用的Survive区。
  - 一般Eden和Survive的比例为8：1
  - 当那块小的Survive不够用时，需要依赖老年代进行分配担保（handle       promotion）

- 类似于打扫屋子时将两块地板上的垃圾先统一扫到其中一块上，然后再清扫那一块脏的。

#### 标记-整理（Mark-Compact）

- 或者是标记-压缩？
- 当对象存活率很高时，会执行较多的复制操作，效率会降低。所以在老年代中不能使用复制算法
- 该算法与标记-清除算法类似，不同的是在标记过后不执行清除，而是将所有存活的对象都向一端移动，然后直接清理之外的内存。



### GC算法的实现（HotSpot）

#### 枚举根节点

- 在可达性分析中需要确定GC root节点，可以作为GC root的节点主要存在于全局性的引用和执行上下文中。但应用很大，不可能一一检查里面的引用，会消耗很多时间。
- 可达性分析还会因为GC停顿而受影响，这些工作需要在一个确保一致性的快照中，也就是说在整个分析过程中应用的执行过程仿佛被冻结在某一时间点上。**这也是为什么GC时需要STW的一个原因。**
- STW时，JVM需要知道在哪些地方存放着对象引用，但不能去一个个的扫描检查。
- 在HotSpot的实现中，使用一组称为OopMap的数据结构来实现。在类加载完成时就记录下对象中什么地方存放了什么计算出来，JIT编译时也会在特定的位置记录下栈和寄存器中哪些位置存放的是引用。这样一来，GC时就可以直接得到这些信息。

#### 安全点（Safe point）

- 有了OopMap，可以快速完成GC根节点枚举，但是如果对所有指令都生成对应的OopMap，会需要大量的额外空间，GC的空间成本太高。

- 实际上HotSpot只在特定位置生成OopMap，这些位置称为安全点（Safepoint），即程序只有在到达安全点时才能暂停。

- 安全点的数量不能多也不能少，其选定是以程序“是否具有长时间执行的特征”为标准。比如指令序列复用、方法调用、循环跳转、异常跳转等。这些地方才会产生安全点。

- 如何让正在运行的线程在发生GC时都位于安全点？

- - 抢先式中断

  - - 发生GC时，先让所有线程中断，然后检查其是否在安全点，如果不在，则恢复线程，让它跑到安全点再中断。
    - **几乎没人用这种方式**

  - 主动式中断

  - - 当需要GC时设置一个标志，所有的线程都需要去主动轮询，发现该标志位true时就自己中断。
    - 轮询的位置一般和安全点重合。

#### 安全区域（Safe Region）

- 安全点只是针对正在运行的线程，但是当线程处于blocked或者sleep时，就无法响应JVM的中断请求。也不能对其分配CPU时间让它运行到安全点，所以需要安全区域
- 安全区域是指在一段代码中，引用关系不会发生变化。在这个区域中任意地方进行GC都是安全的，可以将安全区域看作是扩展了的安全点。

#### 垃圾收集器

##### Serial

- 单线程新生代收集器，工作时必须暂停其他所有工作线程，直到其工作结束
- 简单高效，适用于运行在Client模式下的虚拟机

##### ParNew

- 是Serial收集器的多线程版本，和Serial基本一致
- 是许多运行在Server模式下的虚拟机中首选的新生代收集器
- 除了Serial以外，只有它可以和CMS收集器配合工作 

##### Parallel Scavenge

- 新生代收集器，使用复制算法，并行的多线程收集器
- 目的是达到一个可控制的吞吐量，即吞吐量=运行用户代码时间 /（运行用户代码时间 + 垃圾收集时间）
- 提供两个参数用于精确控制吞吐量：控制最大垃圾收集停顿时间的 -XX:MaxGCPauseMillis、直接设置吞吐量大小的      -XX:GCTimeRatio
- GC自适应的调节策略：虚拟机会根据当前系统的运行情况收集性能监控信息，动态调整这些参数以提供最合适的停顿时间或最大吞吐量

##### Serial Old

- 是Serial的老年代版本，单线程，使用标记-整理算法
- 主要是用于Client模式下的虚拟机 

##### Parallel Old

- 是Parallel Scavenge的老年代版本，使用多线程和标记-整理算法

##### CMS

- Concurrent Mark Sweep 

- 以获取最短回收停顿时间为目标的收集器

- 基于标记-清除算法

- 流程

  - 初始标记

  - 并发标记

  - 重新标记

  - 并发清除

初始标记和重新标记需要STW。

初始标记仅标记GC Roots能直接关联到的对象，速度很快

并发标记是进行GC Roots Tracing

重新标记是为了修正在并发标记期间因用户程序继续工作而导致标记产生变动的那一部分对象的标记记录。

- 缺点

  - 对CPU资源非常敏感

  - 无法处理“浮动垃圾”，可能出现“Concurrent Mode Failure”失败而导致另一次Full GC。

  - - 因为在并发清理阶段用户程序还在运行，所以就不断有新的垃圾产生，这些就是浮动垃圾

  - 因为是基于标记-清除算法，所以会有空间碎片，当空间碎片过多时，会触发Full       GC

##### G1

- 并行与并发

- - 利用多个CPU来缩短STW的时间，在GC时通过并发的方式让Java程序继续执行

- 分代收集

- - 对不同状态的对象有不同的处理方法

- 空间整合

  - 与CMS的标记-清理算法不同，G1整体上看是基于标记-整理的，从局部（Region之间）看是基于复制算法的。
  - 优点就是不会产生空间碎片，收集后能提供规整的可用内存

- 可预测的停顿

- 流程

  - 初始标记
  - 并发标记
  - 最终标记
  - 筛选回收

初始标记仅标记GC Roots能直接关联到的对象，需要停顿线程，但耗时很短

并发标记是从GC Roots开始可达性分析，耗时较长，但可以和用户线程并发执行

最终标记是为了修正在并发标记期间因用户线程继续执行而导致标记产生变动的那一部分标记记录

筛选回收阶段首选对各个Region的回收价值和成本进行排序，根据用户所期望的GC停顿时间来指定回收计划



## 类加载

#### 类的生命周期

> 加载，验证，准备，解析，初始化，使用，卸载

##### 立即对类进行初始化的时机

- 使用new、`getstatic`、putstatic、invokestatic这四条字节码指令时，如果类还没有进行过初始化，则需要对其进行初始化。对应的场景分别是：使用new关键字实例化对象时；读取或设置一个类的静态字段（**被final修饰、已在编译期把结果放入常量池的静态字段除外**）；调用一个类的静态方法

- 使用java.lang.reflect的方法对类进行反射调用时，如果没有初始化，则先初始化

- 初始化子类时，若父类还未初始化，则先初始化父类

- 这些场景中的行为被称为对一个类进行主动引用，除此之外，其他所有引用类的方式都会不触发初始化，称为被动引用

- 被动引用例子

- - 通过子类引用父类的静态字段，子类不会初始化
  - 通过数组定义来引用类，不会触发此类的初始化
  - static final修饰的常量在编译阶段会存入调用类的常量池，本质上没有引用到定义该常量的类，所以不会触发该类的初始化

##### 准备

在准备阶段会为类变量分配内存并且设置类变量初始值，这时候进行内存分配的只有被static修饰的变量，不包括实例变量，实例变量在对象实例化时才随着变量分配在java堆中，这里的初始值指的是数据类型的零值（Integer：0）

##### 初始化

- 初始化阶段是执行类构造器`<clinit>()`方法的过程
- clinit方法是编译期自动收集类中的所有类变量（也就是static变量）和静态语句块（static{}）中的语句块合并而成的，**收集的顺序由语句在源文件中的顺序决定**。
  - 对于静态语句来说，只能访问其之前定义的；其之后定义的，只能赋值，不能访问
- 子类的clinit方法执行之前，父类的clinit方法已经执行完毕。由此可得，父类的静态语句块优于子类的静态语句块
- 在接口中，执行接口的clinit方法不需要先执行父接口的clinit方法。接口的实现类在初始化时也不会执行接口的clinit方法
- 虚拟机保证了多线程环境下一个类的clinit方法的独立性

#### 类加载器

即使两个类来自于同一个class文件，如果加载他们的类加载器不一样，则他们不相等

##### 类加载器种类

- 启动类加载器（Bootstrap      ClassLoader），负责加载`JAVAHOME`下lib目录的、且被虚拟机识别的类库，启动类加载器无法被Java程序引用
- 扩展类加载器（Extension      ClassLoader），负责加载`JAVAHOME`下`/lib/ext`目录中的类库，开发者可以直接使用
- 应用程序类加载器（Application      ClassLoader），加载用户类路径（classpath）上所指定的类库

##### 双亲委派模型

- 类加载器之间的关系不是继承的关系，而是使用组合关系来复用父类加载器的代码
- 工作过程：如果一个类加载器收到了类加载的请求，它不会自己先去尝试加载这个类，而是把这个请求委派给父类加载器去完成，每一层次的类加载器都是如此，所以所有的加载请求最终都应该传送到顶层的启动类加载器中，只有当父加载器反馈自己无法完成这个加载请求时，子加载器才会自己尝试去完成。



## 重载与重写

#### 重载（overload）

在面向接口编程或者存在类继承关系时，一般会有这样的实例化方式`List list = new ArrayList<>()`。左边的List称为变量的**静态类型**，右边的ArrayList称为变量的**实际类型**。其中静态类型是编译期可知的，而实际类型则在运行期才可确定。

![](<https://user-images.githubusercontent.com/34979747/69525016-30703500-0fa2-11ea-9a31-16fc93c02b78.png>)

上图最终打印结果都是`human hello`

方法在重载时具体使用哪一个方法完全取决于传入参数的数量和数据类型，编译期在重载时根据参数的静态类型而不是实际类型作为判定依据。如果查看对应的字节码，在编译完成之后say方法的参数类型都是Human。

依赖静态类型来定位方法执行版本的分派动作称为静态分派，**静态分派的典型使用是“方法重载”**。

但静态分派是选择一个**更合适的版本**，而不是唯一的版本。

更合适的版本意思是：如果对一个字符c来说，say方法有char参数类型的、int、long、Character、Serializable、Object的重载方法。静态分派时会先选择char类型的方法，如果注释掉char的方法，就选择int类型的方法。其规则是按照char->int->long->float->double的顺序依次匹配，如果都没有，就匹配其包装类型Character，接着是其包装类型实现的接口Serializable，最后是Objec

#### 重写（override）

其本质是动态分派

在编译完成之后，执行具体的方法时，invokevirtual指令会去寻找方法接受者的实际类型，就是在运行期确定接受者的实际类型