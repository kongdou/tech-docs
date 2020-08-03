# 描述

Java虚拟机主要由字节码指令集、寄存器、栈、垃圾回收堆和存储方法域等构成，JVM屏蔽了与具体操作系统平台相关的信息，使Java程序只需生成在Java虚拟机上运行的目标代码（字节码）,就可以在多种平台上不加修改地运行。JVM在执行字节码时，实际上最终还是把字节码解释成具体平台上的机器指令执行。

# JVM生命周期 
JVM伴随着Java程序的开始而开始，程序的结束而结束。一个Java程序会开启一个JVM进程，一台计算机上可以运行多个程序，也就是可以运行多个JVM进程。

JVM线程分为两种，守护线程和普通线程。守护线程是JVM自己使用的线程，比如GC就是一个守护线程，普通线程就是一般的Java程序线程，只要JVM中有普通的线程在运行，Java虚拟机就不会停止

# JVM执行引擎
JVM的执行引擎中，有三个部分：**解释器**，**JIT编译器**和**垃圾回收器**。

## 解释器

解释器会将前面编译生成的字节码翻译成机器语言，因为每次都要翻译，相当于比直接编译成机器码要多了一步，所以java执行起来会比较慢，为了解决这个问题，JVM引入了JIT(Just-in-Time)编译器，将热点代码编译成为机器码。

## 即时编译（JIT）
什么是JIT?

日常所谓的Java编译是指前端编译，也就是通过javac编译，将.java文件编译成.class文件，这个过程中**javac并不做性能优化，只会做语法检查**。  
而后端编译指的是将**字节码**转换成操作系统可以执行的**机器码**，这个过程采用混合的方法，包括**解释执行**和**编译执行**，其中编译执行就是所谓的JIT。

### 分层编译模式
HotSpot虚拟机有多个即时编译器，C1（Client Compiler）和C2（Server Compiler），最新的还有Graal即时编译器。  
Java7 之前，对于**启动性能要求高的，选择C1启动编译器，对应参数-client**，对于**性能要求高的，选择C2启动编译器，对应参数-server**  
Java7 开始，引入了分层编译：对应参数-XX:+tieredCompilation，它综合了C1的启动优势和C2巅峰性能优势，也可以通过参数-server和-client强制指定虚拟的即时编译模式  

分层编译将jvm的执行状态分为五个层次：
- 第0层：程序解释执行，默认开启性能监控（profiling），如果不开启，可触发第二层编译
- 第1层：可称为C1编译，将字节码转译为本地代码，进行简单、可靠的优化，不开启profiling
- 第2层：也称为C1编译，开启profiling，仅执行带有方法调用次数和循环回编执行次数的profiling的C1编译
- 第3层：也称为C1编译，执行所有带profiling的C1编译
- 第4层：可称为C2编译，也是将字节码转译为机器码（本地代码），但是会启用一些编译耗时较长的优化，甚至会根据性能监控信息进行一些不可靠的激进优化

对于C1的三种状态，按照执行效率从高到低：第1层、第2层、第3层  
通常情况下，C2的执行效率比C1高30%以上  

在java8中，默认使用了分层编译，-client和-server的设置都已经是无效的，如果只想开启C2，关闭分层编译即可，-XX+TieredCompilation，如果只想使用C1，可以在打开分层编译的同时，使用参数+XX:TieredStoreAtLevel=1  

查看系统的编译模式：  
`
java -version
`  
```
java version "1.8.0_131"  
Java(TM) SE Runtime Environment (build 1.8.0_131-b11)  
Java HotSpot(TM) 64-Bit Server VM (build 25.131-b11, mixed mode)  
```
mixed mode代表是默认的混合编辑模式，除了这种模式外，我们还可以使用-Xint参数强制关闭JIT，只使用解释器的编译模式，也可以使用参数-Xcomp强制虚拟机运行只有JIT的编译模式。  
`
java -Xint -version
`
```
java version "1.8.0_131"
Java(TM) SE Runtime Environment (build 1.8.0_131-b11)
Java HotSpot(TM) 64-Bit Server VM (build 25.131-b11, interpreted mode)
```
`
java -Xcomp -version
`
```
java version "1.8.0_131"
Java(TM) SE Runtime Environment (build 1.8.0_131-b11)
Java HotSpot(TM) 64-Bit Server VM (build 25.131-b11, compiled mode)
```

### 触发标准
在HoSpot虚拟机中，**热点探测**是JIT的触发标准。  
> 热点探测是基于**计数器**的热点探测，采用这种方法的虚拟机会为每个方法建立计数器统计方法的执行次数，如果执行次数超过一定的阀值就认为它是热点方法

虚拟机为每个方法准备了两类计数器，方法调用计数器（Invocation Counter）和循环回边计数器（Back Edge Counter）。在确定虚拟机运行参数的前提下，这两个计数器都有一定的阀值，当计时器超过阀值溢出时，触发JIT编译

#### 方法调用计数器（Invocation Counter）
方法调用计数器用于统计方法被调用的次数，默认阈值在 C1 模式下是 1500 次，在 C2 模式在是 10000 次，可通过-XX: CompileThreashold来设定；而在分层编译模式下设定的参数将失效。此时将会根据当前待编译的方法数和线程数动态调整，当方法计数器和回边计数器之和超过方法计数器的阀值时，触发JIT编译器

#### 回边计数器（Back Edge Counter）
回边计数器用于统计一个方法中循环体代码执行的次数，在字节码中遇到控制流向后跳转的指令称为“回边”（Back Edge），该值用于计算是否触发C1编译的阀值，在不开启分层编译的前提下，C1 默认为 13995，C2 默认为 10700，可通过-XX: OnStackRelacePercentage=N来设置，而在分层的情况下参数设置将失效，此时根据当前待编译的方法数以及编译线程数动态调整。  

OSR是一种在运行时替换正在运行的函数/方法的栈帧的技术，虚拟机一旦发现一个函数执行很频繁的时候，就会采用JIT，提供函数执行性能，按时如果以函数（方法）为单位进行JIT编译，那么无法应对main函数中包含循环体的情况的，OSR就能起到作用，与编译整个方法不同，当虚拟机发现某个方法内部有循环很热的时候，选择只编译某个方法的某个循环，当循环计数器达到触发OSR编译的阀值，等编译完成后就可以执行编译后的代码

```
List<Integer> list0 = new ArrayList<Integer>();
long start0 = System.currentTimeMillis();
for (int i = 0; i < 10000000; i++) {
    list0.add(i);
}
System.out.println(System.currentTimeMillis() - start0);

long start1 = System.currentTimeMillis();
List<Integer> list1 = new ArrayList<Integer>();
for (int i = 10000000; i < 20000000; i++) {
    list1.add(i);
}

System.out.println(System.currentTimeMillis() - start1);
```
第二个1000万数据的增加时间，比第一个1000万的时间短很多，原因是当循环达到回边计数器阈值时，JVM 会认为这段是热点代码，JIT 编译器就会将这段代码编译成机器语言并缓存，在第二个循环内，会直接将执行代码替换，执行缓存的机器语言。

> 分层编译情况下，设定的阀值失效

### JIT优化技术

JIT编译运用了一些经典的编译技术来实现代码的优化，即通过一些例行检查优化，可以智能地编译出运行时的最优性能代码，主要的方法：**方法内联**，**逃逸内联**

#### 方法内联
调用一个方法通常会经过压栈和出栈，调用方式是将程序执行顺序转移到存储方法的内存地址，将方法内容执行完，再返回到执行该方法前的位置。这种方法操作要求执行前保护现场并记忆执行的地址，执行后要恢复现场，并按原来保存的地址，并按原来保存的地址继续执行，因此方法的调用会产生定的时间和空间方面的开销（类似上下文切换）。  
方法内联优化就是把目标方法复制到发起调用的方法中，避免发生真实的方法调用。  
JVM自动识别热点方法，并对它进行内联优化，但是热点方法不一定会被Jvm做内联优化，如：被调用的方法体太大了，Jvm将不执行内联，方法体的大小可以通过参数进行优化：  
1.经常执行的方法，默认情况下，方法体大小<**325**字节的都会进行内联，可以通过-XX:MaxFreqInlineSize=N来设置大小。  
2.不是经常执行的方法，默认情况下，方法大小<35字节才会进行内联，可以通过-XX:MaxInlineSize=N来重置大小值  

热点方法的优化可以有效提高系统性能，一般我们可以通过以下方式提高方法内联：  
1.通过设置JVM参数来**减小热点阀值（CompileThreshold）**或**增加方法体阀值**，以便更多的方法可以进行内联，但是也就意味着消耗更多的内存  
2.在编程中，避免在一个方法中写大量的代码，使用小方法提
3.尽量使用final、private、static关键字修饰方法，编码方法因为继承，需要进行额外的类型检查。

#### 逃逸分析
逃逸分析（Escape Analysis）是判断一个对象是否被外部方法引用或外部线程访问的技术，编译器会根据逃逸分析的结果对代码进行优化。

JVM参数设置：
```
-XX+DoEscapeAnalysis 开启逃逸分析，JDK 1.8后默认开启
-XX-DoEscapeAnalysis 关闭逃逸分析
```
具体优化方法：**栈上分配**、**锁消除**、**标量替换**
##### 栈上分配
Java类中创建的对象、数组是在堆中分配内存的，当堆中的对象不在被使用的时候，GC进行回收。但是在方法中创建的对象分配是分配到栈中，方法运行结束，对象自动销毁，GC不参与回收
> GC只回收堆、方法区中的对象，也就是共享区的对象

逃逸分析如果发现一个对象只在方法中使用，就会将对象分配到栈上（HotSpot目前是否实现？哪个版本）  
```
public class EscapeAnalysis {

     public static Object object;
     
     public void globalVariableEscape(){//全局变量赋值逃逸  
         object =new Object();  
      }  
     
     public Object methodEscape(){  //方法返回值逃逸
         return new Object();
     }
     
     public void instancePassEscape(){ //实例引用发生逃逸
        this.speak(this);
     }
     
     public void speak(EscapeAnalysis escapeAnalysis){
         System.out.println("Escape Hello");
     }
}
```
##### 锁消除
单线程环境下，没有必要使用线程安全的容器，因为即使使用也不会存在线程竞争，此时JIT编译器会对使用该对象的方法进行锁解除，例如：
```
	public static String getString(String s1, String s2) {
		StringBuffer sb = new StringBuffer();
		sb.append(s1);
		sb.append(s2);
		return sb.toString();
	}
```
可以通过JVM参数进行设置：
```
-XX:+EliminateLocks 开启锁消除
-XX:-EliminateLocks 关闭锁消除
```

##### 标量替换
标量即不可被进一步分解的量，而JAVA的基本数据类型就是标量（int、long等基本类型），标量的对立就是可以被进一步分解的量，而这种量称为聚合量，而在JAVA中对象就是可以被进一步分解的量

通过逃逸分析确定该对象不会被外部访问，并且对象可以被进一步分解时，JVM不会创建该对象，而会将该对象成员变量分解若干个被这个方法使用的成员变量所代替。这些代替的成员变量在栈帧或寄存器上分配空间。

```
public void foo() {
	TestInfo info = new TestInfo();
		info.id = 1;
		info.count = 99;
		// to do something
}
```
逃逸分析后，代码会被优化为：
```
public void foo() {
	id = 1;
	count = 99;
	// to do something
}
```
可以通过JVM参数进行设置：
```
-XX+EliminateAllocations 开启标量替换（jdk1.8 默认开启）
-XX-EliminateAllocations 关闭
```

> 结论总结：能在方法内创建对象，就不要再方法外创建对象。毕竟这是为了GC好，也是为了提高性能。方法内部行数和逻辑需要尽可能简单

## 垃圾回收
GC触发的条件：
- 程序调用System.gc()（系统建议执行Full GC，但是不必然执行）
- 系统自身来决定GC触发的时机


GC运行时，除GC所需要的线程外的其他所有线程处于等待状态，直到GG任务完成，GC优化的目的就是减少其他线程等待的时间。  
GC回收**堆**和**方法区**的内的对象。  
当前主流虚拟机（Hotspot VM）的垃圾回收都采用“分代回收”的算法。“分代回收”是基于这样一个事实：对象的生命周期不同，所以针对不同生命周期的对象可以采取不同的回收方式，以便提高回收效率。

![GC](https://github.com/kongdou/tech-docs/blob/master/images/GC.png)

堆=新生代+老年代，不包括永久代（方法区）

### 新生代
由1个Eden区域和2个Survivor区域，Survivor区域由FromSpace和ToSpace组成。  
新创建的对象都会被分配到Eden区(一些大对象特殊处理),Eden空间不足会把存活的对象转移到Survivor中，新生代可以通过**-Xmn**控制，也可以通过调节SurvivorRatio控制Eden与Survivor的比例。  

> 默认的，Eden : from : to = 8 :1 : 1 ( 可以通过参数–XX:SurvivorRatio 来设定 )，即： Eden = 8/10 的新生代空间大小，from = to = 1/10 的新生代空间大小。

MinorGC过程：采用**复制算法**，首先把Eden和SurvivorFrom区域中存活的对象复制到SurvivorTo区域（如果对象的年龄达到老年代标准，则放到老年区），同时把这些对象的年龄+1（如果SurvivorTo不够了就放到老年区），达到“晋升年龄阈值”后，被放到老年代，这个过程也称为“晋升”。然后清空Eden和SurvivorFrom中的对象，最后，SurvivorTo和SurvivorFrom互换，原SurvivorTo成为下一次GC时的SurvivorFrom区。  

MinorGC触发条件：当新生代无法为新生对象分配内存空间的时候，会触发Minor GC

> “晋升年龄阈值”通过参数MaxTenuringThreshold设定，默认值为15
### 老年代
用于存放新生代中经过多次垃圾回收仍然存活的对象。老年代的对象比较稳定，所以MajorGC不会频繁执行，在进行MajorGC前一般都先进行了一次MinorGC，使得有新生代的对象晋身入老年代，导致老年代空间不够用时才触发。  
当无法找到足够大的连续空间分配给新创建的较大对象时也会提前触发一次MajorGC进行垃圾回收腾出空间。


MajorGC过程：采用**标记—清除算法**，首先扫描一次所有老年代，标记出存活的对象，然后回收没有标记的对象。MajorGC的耗时比较长，因为要扫描再回收。MajorGC会产生内存碎片，为了减少内存损耗，我们一般需要进行合并或者标记出来方便下次直接分配。  

当老年代也满了装不下的时候，就会抛出OOM（Out of Memory）异常。

Major GC触发条件：回收老年代，通常至少经历过一次Minor GC

### Full GC
Full GC定义是相对明确的，针对整个新生代、老生代、元空间（metaspace，java8以上版本取代永久代）的全局范围的GC；
- 调用System.gc时，系统建议执行Full GC，但是不必然执行
- 老年代空间不足
- 方法区空间不足
- 通过Minor GC后进入老年代的平均大小大于老年代的可用内存
- 由Eden区、From Space区向To Space区复制时，对象大小大于To Space可用内存，则把该对象转存到老年代，且老年代的可用内存小于该对象大小

### 垃圾回收算法

![GC-algorithm](https://github.com/kongdou/tech-docs/blob/master/images/gc-algorithm.png)

#### 引用计数法
JAVA在运行时，当有一个地方引用该对象实例，会将这个对象实例加1，引用失效时就减1，JVM在扫描内存时，发现引用计数值为0的则是垃圾对象，计数值大于0的则为活跃对象。  
目前垃圾回收算法，没有采用引用计数算法，原因是在对象互相引用的情况下，无法判定两者是否为垃圾对象。  

#### 可达性分析法（根搜索算法）
这个算法的基本思想是通过一系列称为“GC Roots”的对象作为起始点，从这些节点向下搜索，搜索所走过的路径称为引用链，当一个对象到GC Roots没有任何引用链（即GC Roots到对象不可达）时，则证明此对象是不可用的。

如果对象在进行可行性分析后发现没有与GC Roots相连的引用链，也不会理解死亡，GC并不一定会回收该对象。要完全回收一个对象，至少需要经过两次标记的过程。

第一次标记：对于一个没有其他引用的对象，筛选该对象是否有必要执行finalize()方法，如果没有执行必要，则意味可直接回收。（筛选依据：是否覆写或执行过finalize()方法；因为finalize方法只能被执行一次）。

第二次标记：如果被筛选判定位有必要执行，则会放入FQueue队列，并自动创建一个低优先级的finalize线程来执行释放操作。如果在一个对象释放前被其他对象引用，则该对象会被移除FQueue队列。


![gc-root](https://github.com/kongdou/tech-docs/blob/master/images/gc-root.png)

如何选择GC Roots：  
- 虚拟机栈（栈帧中局部变量表）中应用的对象
- 方法区中的类静态属性引用的对象
- 方法区中常量应用的对象
- 本地方法栈中JNI引用的对象

#### 标记-清除法（Mark-Sweep）
![mark-sweep](https://github.com/kongdou/tech-docs/blob/master/images/mark-sweep.png)

JVM会扫描所有的对象实例，通过根搜索算法，将活跃对象进行标记，JVM再一次扫描所有对象，将未标记的对象进行清除，只有清除动作，不作任何的处理，这样导致的结果会存在很多的内存碎片。空间碎片太多可能会导致以后再程序运行过程中需要分配较大对象时，无法找到足够的连续内存二不得不提前触发另一次垃圾收集动作。

#### 复制（copying）
![copying](https://github.com/kongdou/tech-docs/blob/master/images/copying.png)
JVM扫描所有对象，通过根搜索算法标记被引用的对象，之后会申请新的内存空间，将标记的对象复制到新的内存空间里，存活的对象复制完，会清空原来的内存空间，将新的内存作为JVM的对象存储空间。这样虽然解决了内存碎片问题，但是如果对象很多，重新申请新的内存空间会很大，在内存不足的场景下，会对JVM运行造成很大的影响

#### 标记-整理（Mark-compact）
![mark-compact）](https://github.com/kongdou/tech-docs/blob/master/images/mark-compact.png)
标记整理实际上是在标记清除算法上的优化，执行完标记清除全过程之后，再一次对内存进行整理，将所有存活对象统一向一端移动，这样解决了内存碎片问题。

#### 分代回收
![fen-dai）](https://github.com/kongdou/tech-docs/blob/master/images/fen-dai.png)
目前JVM常用回收算法就是分代回收，年轻代以复制算法为主，老年代以标记整理算法为主。原因是年轻代对象比较多，每次垃圾回收都有很多的垃圾对象回收，而且要尽可能快的减少生命周期短的对象，存活的对象较少，这时候复制算法比较适合，只要将有标记的对象复制到另一个内存区域，其余全部清除，并且复制的数量较少，效率较高；而老年代是年轻代筛选出来的对象，被标记比较高，需要删除的对象比较少，显然采用标记整理效率较高。

### 垃圾收集器

![hot-spot）](https://github.com/kongdou/tech-docs/blob/master/images/hotspot.png)

图中展示了7种作用于不同分代的收集器，如果两个收集器之间存在连线，则说明它们可以搭配使用。虚拟机所处的区域则表示它是属于新生代还是老年代收集器。

新生代收集器：serial、ParNew、Parallel scavenge  
老年代收集器：CMS（并发标记清除）、Serial Old、Parallel Old  
整体收集器：G1（garbage first）  

> 并行收集：指多条垃圾收集线程并行工作，但此时用户线程仍处于等待状态  
> 并发收集：指用户线程与垃圾收集线程同时工作（不一定是并行的可能会交替执行）。用户程序在继续运行，而垃圾收集程序运行在另一个CPU上。  
> 并发和并行是针对GC线程和用户线程的  
#### serial
Serial收集器是最基本的、发展历史最悠久的收集器。  
**特点：** 单线程、简单高效（与其他收集器的单线程相比），对于限定单个CPU的环境来说，Serial收集器由于没有线程交互的开销，专心做垃圾收集自然可以获得最高的单线程手机效率。收集器进行垃圾回收时，必须暂停其他所有的工作线程，直到它结束（Stop The World）。  
**适用场景：** 适用于Client模式下的虚拟机。  
**示意图：**  
![serial-gc](https://github.com/kongdou/tech-docs/blob/master/images/serial-gc.png)

#### ParNew
ParNew收集器其实就是Serial收集器的多线程版本。  
除了使用多线程外其余行为均和Serial收集器一模一样（参数控制、收集算法、Stop The World、对象分配规则、回收策略等）  
**特点：** 多线程、ParNew收集器默认开启的收集线程数与CPU的数量相同，在CPU非常多的环境中，可以使用-XX:ParallelGCThreads参数来限制垃圾收集的线程数。和Serial收集器一样存在Stop The World问题  

**应用场景：** ParNew收集器是许多运行在Server模式下的虚拟机中首选的新生代收集器，因为它是除了Serial收集器外，唯一一个能与CMS收集器配合工作的。

**示意图：**

![parnew-gc](https://github.com/kongdou/tech-docs/blob/master/images/parnew-gc.png)

#### parallel Scavenge
与吞吐量关系密切，故也称为吞吐量优先收集器。
> 即CPU用于运行用户代码的时间与CPU总消耗时间的比值（吞吐量 = 运行用户代码时间 / ( 运行用户代码时间 + 垃圾收集时间 )）。例如：虚拟机共运行100分钟，垃圾收集器花掉1分钟，那么吞吐量就是99%

**特点：** 属于新生代收集器也是采用复制算法的收集器，又是并行的多线程收集器（与ParNew收集器类似）。  
该收集器的目标是达到一个可控制的吞吐量。还有一个值得关注的点是：GC自适应调节策略（与ParNew收集器最重要的一个区别）

**GC自适应调节策略：** Parallel Scavenge收集器可设置-XX:+UseAdptiveSizePolicy参数。当开关打开时不需要手动指定新生代的大小（-Xmn）、Eden与Survivor区的比例（-XX:SurvivorRation）、晋升老年代的对象年龄（-XX:PretenureSizeThreshold）等，虚拟机会根据系统的运行状况收集性能监控信息，动态设置这些参数以提供最优的停顿时间和最高的吞吐量，这种调节方式称为GC的自适应调节策略。

Parallel Scavenge收集器使用两个参数控制吞吐量：
- XX:MaxGCPauseMillis 控制最大的垃圾收集停顿时间
- XX:GCRatio 直接设置吞吐量的大小

#### Serial Old 
Serial Old是Serial收集器的老年代版本。  
**特点：** 同样是单线程收集器，采用标记-整理算法。
**应用场景：** 主要也是使用在Client模式下的虚拟机中。也可在Server模式下使用。  
Server模式下主要的两大用途：  
- 在JDK1.5以及以前的版本中与Parallel Scavenge收集器搭配使用
- 作为CMS收集器的后备方案，在并发收集Concurent Mode Failure时使用

![serial-old](https://github.com/kongdou/tech-docs/blob/master/images/serial-old.png)

#### Parallel Old
是Parallel Scavenge收集器的老年代版本。  
**特点：** 多线程，采用标记-整理算法
**应用场景：** 注重高吞吐量以及CPU资源敏感的场合，都可以优先考虑Parallel Scavenge+Parallel Old 收集器。  
**示意图：**
![parallel-old](https://github.com/kongdou/tech-docs/blob/master/images/parallel-old.png)

#### CMS收集器
一种以获取**最短回收停顿时间**为目标的收集器，CMS收集器的内存回收过程是与用户线程一起并发执行的。  
**特点：** 基于标记-清除算法实现。并发收集、低停顿。  
**应用场景：** 适用于注重服务的响应速度，希望系统停顿时间最短，给用户带来更好的体验等场景下。如web程序、b/s服务。  
**CMS收集器的运行过程分为下列4步：**  
1. 初始标记：标记GC Roots能直接关联到的对象。速度很快但是仍存在Stop The World问题。
2. 并发标记：进行GC Roots Tracing 的过程，找出存活对象且用户线程可并发执行。
3. 重新标记：为了修正并发标记期间因用户程序继续运行而导致标记产生变动的那一部分对象的标记记录。仍然存在Stop The World问题。
4. 并发清除：对标记的对象进行清除回收。
![cms](https://github.com/kongdou/tech-docs/blob/master/images/cms
.png)

**CMS缺点：** 
- 对CPU资源非常敏感
- 无法处理浮动垃圾，可能出现Concurrent Model Failure失败而导致另一次Full GC的产生
- 因为采用标记-清除算法所以会存在空间碎片的问题，导致大对象无法分配空间，不得不提前触发一次Full GC
> 浮动垃圾：在初始阶段标记为活着，在并发过程标记为死亡，重新标记只会标记活着的对象，不会标记死亡的对象，死亡的对象就成为了浮动垃圾



#### G1收集器
一款面向服务端应用的垃圾收集器  
详细：https://blog.csdn.net/coderlius/article/details/79272773  

**特点：**  
**并行与并发：** G1能充分利用多CPU、多核环境下的硬件优势，使用多个CPU来缩短Stop-The-World停顿时间。部分收集器原本需要停顿Java线程来执行GC动作，G1收集器仍然可以通过并发的方式让Java程序继续运行。  
**分代收集：** G1能够独自管理整个Java堆，并且采用不同的方式去处理新创建的对象和已经存活了一段时间、熬过多次GC的旧对象以获取更好的收集效果。  
**空间整合：** G1运作期间不会产生空间碎片，收集后能提供规整的可用内存  
**可预测的停顿：** G1除了追求低停顿外，还能建立可预测的停顿时间模型。能让使用者明确指定在一个长度为M毫秒的时间段内，消耗在垃圾收集上的时间不得超过N毫秒。  

**G1为什么建立可预测的停顿时间模型？**  
因为它有计划的避免在整个Java堆中进行全区域的垃圾收集。G1跟踪各个Region里面的垃圾堆积的大小，在后台维护一个优先列表，每次根据允许的收集时间，优先回收价值最大的Region。这样就保证了在有限的时间内可以获取尽可能高的收集效率。  

**G1与其他收集器的区别：**  
其他收集器的工作范围是整个新生代或者老年代、G1收集器的工作范围是整个Java堆。在使用G1收集器时，它将整个Java堆划分为多个大小相等的独立区域（Region）。虽然也保留了新生代、老年代的概念，但新生代和老年代不再是相互隔离的，他们都是一部分Region（不需要连续）的集合。  

**G1收集器存在的问题：**  
Region不可能是孤立的，分配在Region中的对象可以与Java堆中的任意对象发生引用关系。在采用可达性分析算法来判断对象是否存活时，得扫描整个Java堆才能保证准确性。其他收集器也存在这种问题（G1更加突出而已）。会导致Minor GC效率下降。  

**G1收集器是如何解决上述问题的？**  
采用Remembered Set来避免整堆扫描。G1中每个Region都有一个与之对应的Remembered Set，虚拟机发现程序在对Reference类型进行写操作时，会产生一个Write Barrier暂时中断写操作，检查Reference引用对象是否处于多个Region中（即检查老年代中是否引用了新生代中的对象），如果是，便通过CardTable把相关引用信息记录到被引用对象所属的Region的Remembered Set中。当进行内存回收时，在GC根节点的枚举范围中加入Remembered Set即可保证不对全堆进行扫描也不会有遗漏。  

**如果不计算维护 Remembered Set 的操作，G1收集器大致可分为如下步骤：**  
1. 初始标记： 仅标记GC Roots能直接到的对象，并且修改TAMS（Next Top at Mark Start）的值，让下一阶段用户程序并发运行时，能在正确可用的Region中创建新对象。（需要线程停顿，但耗时很短。）  
2. 并发标记： 从GC Roots开始对堆中对象进行可达性分析，找出存活对象。（耗时较长，但可与用户程序并发执行）  
3. 最终标记： 为了修正在并发标记期间因用户程序执行而导致标记产生变化的那一部分标记记录。且对象的变化记录在线程Remembered Set  Logs里面，把Remembered Set  Logs里面的数据合并到Remembered Set中。（需要线程停顿，但可并行执行。）  
4. 筛选回收： 对各个Region的回收价值和成本进行排序，根据用户所期望的GC停顿时间来制定回收计划。（可并发执行）  

# JVM内存模型
主要由**堆内存**、**方法区**、**程序计数器**、**虚拟机栈**和**本地方法栈**组成。其组成结构如下：
![内存模型](https://github.com/kongdou/tech-docs/blob/master/images/JVM虚拟机内存模型.png)

其中**堆**和**方法区**是所有线程共有的，而**虚拟机栈**、**本地方法栈**和**程序计数器**是线程私有的

## 堆内存
堆内存是所有线程共有的，可以分为两部分：年轻代和老年代，另外注意**永久代并不属于堆内存的一部分**，并且在1.8版本以后被移除
![堆内存](https://github.com/kongdou/tech-docs/blob/master/images/heap.png)

堆内存是我们在生产环境中进行内存性能调优中的一个重要的内容，内存回收的一些机制和算法。


## 方法区
方法区与Java堆一样，是各个线程共享的区域，它用于存储已被虚拟机加载的类信息，如**常量**、**静态变量**、**即时编译（JIT）的代码**等数据。  
方法区中包含的都是在整个程序中永远唯一的元素，如class，static变量  
由于程序中所有的线程共享一个方法区，所以访问方法区的信息必须确保线程是安全的。如果两个线程同时去加载一个类，那么只能有一个线程被允许去加载这个类，另一个必须等待。  
在程序运行时，方法区的大小是可以改变的，程序在运行时可以扩展，同时，方法区中的对象也可以被垃圾回收，但条件苛刻，必须在该类没有引用的情况下才能被GC回收。  

## 程序计数器
程序计数器是一个记录着当前线程所执行的字节码的行号指示器。  
字节码解释器就是通过改变程序计数器的值来得到下一条要执行的字节码指令的。比如说程序中的分支、循环、异常处理、跳转等命令都是通过依赖程序计数器来实现的。  
特点：
- 线程隔离性，每个线程工作时都有属于自己的独立计数器。
- 执行java方法时，程序计数器是有值的，且记录的是正在执行的字节码指令的地址
- 执行native本地方法时，程序计数器的值为空（Undefined），因为native方法是java通过JNI直接调用本地的C/C++库
- 程序计数器占用的内存空间很少，也是唯一一个在JVM规范中没有规定任何OutOfMemoryError的区域。  
## java虚拟机栈（JVM Stack）
与程序计数器一样，JVM Stack也是线程私有的，用通俗的话将它就是我们常常听说到堆栈中的那个“栈内存”。一个线程对应一个JVM Stack，JVM Stack包含一组栈帧（Stack Frame）。 

JVM Stack描述的是Java方法执行的内存模型，每个方法在执行的同时都会创建一个**栈帧(Stack Frame)**。线程每调用一个方法就对应着 JVM Stack 中 Stack Frame 的入栈，方法执行完毕或者异常终止对应着出栈（销毁）。  

在活动线程中，只有位于栈顶的栈帧才是有效的，称为当前栈帧，与这个栈帧相关联的方法称为当前方法
### 栈帧
栈帧存储了**局部变量表**、**操作数栈**、**动态链接**、**方法返回地址**、**帧数据区**  
一个栈帧需要分配多少内存，不会受到程序运行期变量数据的影响，而仅仅取决于具体的虚拟机实现。

栈帧结构：  
![stack frame](https://github.com/kongdou/tech-docs/blob/master/images/Stack%20Frame.png)

#### 局部变量表
局部变量表是一组**变量值存储空间**，用于存放**方法参数**和方法内部定义的**局部变量**，并在Java编译为Class时，就已经确定了该方法所需要分配的局部变量表的最大容量。  

局部变量表存放了编译期可知的各种基本数据类型（boolean、byte、char、short、int、float、long、double）、对象引用(reference类型) 、 returnAddress类型。  

局部变量表的容量以 Variable Slot（变量槽）为最小单位，每个变量槽都可以存储 32 位长度的内存空间。基本类型数据以及引用和 returnAddress（返回地址）占用一个变量槽，long 和 double 需要两个。  

#### 操作数栈（Operand Stack）
操作数栈就是JVM执行引擎的一个工作区，当一个方法被调用的时候，一个新的栈帧也会随之被创建出来，但这个时候栈帧中的操作数栈却是空的，只有方法在执行的过程中，才会有各种各样的字节码指令往操作数栈中执行入栈和出栈操作。比如在一个方法内部需要执行一个简单的加法运算时，首先需要从操作数栈中将需要执行运算的两个数值出栈，待运算执行完成后，再将运算结果入栈。

#### 动态链表（Dynamic Linking）
被调用的目标方法在编译期无法被确定下来，只能够在程序运行期将方法的符号引用转换为直接引用，这种引用转换的过程具备动态性，称为动态链接。
![动态链接](https://github.com/kongdou/tech-docs/blob/master/images/动态链接.png)

> 一个字节码文件被装载进JVM内部时，如果被调用的目标方法在编译期可知，且运行期保持不变。将调用方法的符号引用转换为直接引用的过程称为静态链接。

> 方法的绑定机制分为早期绑定（Early Binding）和晚期绑定（Late Bingind）。绑定是一个字段、方法或类在符号引用被替换为直接引用的过程  
早期绑定：被调用的目标方法在编译期可知，且运行保持不变。  
晚期绑定：被调用方法在编译期无法被确定下来，只能够在程序运行期根据实际类型绑定相关的方法。
        
#### 返回地址（Return Address）
方法开始执行后，只有 2 种方式可以退出 ：方法返回指令，异常退出

#### 帧数据区
帧数据区的大小依赖于 JVM 的具体实现


## 本地方法栈
本地方法栈与JVM Stack一样，主要用于存储本地方法的局部变量表，本地方法的操作数栈等信息。当栈内的数据在超出其作用域后，会被自动释放掉。

本地方法栈是在程序调用或JVM调用本地方法接口（Native）时候启用


# 虚拟机参数

参数 | 说明
---|---
-Xms | 初始堆大小。如：-Xms256m
-Xmx | 最大堆大小。如：-Xmx512m
-Xmn | 新生代大小。通常为 Xmx 的 1/3 或 1/4。新生代 = Eden + 2 个 Survivor 空间。实际可用空间为=Eden + 1 个 Survivor，即 90% 
-Xss | JDK1.5+ 每个线程堆栈大小为 1M，一般来说如果栈不是很深的话， 1M 是绝对够用了的。
-XX:NewRatio | 新生代与老年代的比例，如 –XX:NewRatio=2，则新生代占整个堆空间的1/3，老年代占2/3
-XX:SurvivorRatio | 新生代中 Eden 与 Survivor 的比值。默认值为 8。即 Eden 占新生代空间的 8/10，另外两个 Survivor 各占 1/10 
-XX:PermSize | 永久代(方法区)的初始大小（1.8+取消）
-XX:MaxPermSize | 永久代(方法区)的最大值（1.8+取消）
-XX:+PrintGCDetails| 打印 GC 信息
-XX:+PrintGCDateStamps| 输出GC的时间戳
-XX:+PrintCompilation|在控制台打印编译过程信息
-XX:+UnlockDiagnosticVMOptions|解锁对 JVM 进行诊断的选项参数。默认是关闭的，开启后支持一些特定参数对 JVM 进行诊断
-XX:+PrintInlining|将内联方法打印出来
