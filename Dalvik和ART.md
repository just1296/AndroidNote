# Dalvik和ART
## 1、Dalvik
### 1.1 Dalvik和JVM的区别
#### 1.1.1 基于的架构不同
**JVM基于栈**，需要去栈中读取数据，所需的指令更多，导致速度变慢。

**DVM基于寄存器**，指令更紧凑，更简洁。由于显式的指定了操作数，所以基于寄存器的指令会比基于栈的指令更大，但是由于指令数量的减少，总的代码数不会增加多少。
#### 1.1.2 执行的字节码不同
JVM的执行文件是class文件，DVM的执行文件是dex文件。

每个class文件包含了该类的常量池、类信息、属性等，当JVM加载jar文件时，会加载jar文件里面所有的class文件，JVM这种加载方式很慢，对于内存有限的移动设备并不合适。

而dex文件将所有的class文件里面包含的信息全部整合在一起，去除了class文件中的冗余信息，减少了IO操作，加快了类的查找速度。

#### 1.1.3 DVM允许在有限的内存中同时存在多个进程
Android中的每一个应用都运行在一个DVM实例中，每一个DVM实例都运行在一个独立的进程空间中，独立的进程可以防止虚拟机崩溃的时候所有程序都关闭。

#### 1.1.4 DVM由Zygote创建和初始化
Zygote是一个DVM进程，同时也是用来创建和初始化DVM实例。每当系统需要创建一个应用程序时，Zygote会fork自身，快速创建和初始化一个DVM实例，用于应用程序的运行。对于一些只读的系统库，所有的DVM实例都会和Zygote共享一块内存区域，节省了内存开销。

#### 1.1.5 DVM有共享机制
DVM拥有预加载共享的机制，不同应用之间在运行时可以共享相同的类。JVM机制没有这种机制，打包以后的程序都是彼此独立的，即便它们在包里面使用了相同的类，运行时也是单独加载和运行的。

### 1.2 DVM架构
首先Java编译期编译的class文件经过DX工具转换为dex文件，dex文件由类加载器处理，接着解释器根据指令集对Dalvik字节码进行解释、执行，最后交于Linux处理。

### 1.3 DVM的运行时堆
DVM的运行时堆采用标记-清除算法进行GC，它由两个Space以及多个辅助数据结构组成，两个Space分别是**Zygote Space（Zygote Heap）**和**Allocation Space（Active Heap）**。

Zygote Space用来管理Zygote进程在启动过程中预加载和创建的各种对象，Zygote Space不会触发GC，在Zygote进程和应用程序进程之间会共享Zygote Space。在Zygote进程fork第一个子进程之前，会把Zygote Space分为两个部分，原来已经被使用的堆仍称为Zygote Space，未使用的那部分堆叫做Allocation Space，以后的对象都会在Allocation Space上进行分配和释放，Allocation Space不是进程间共享的。

除了以上两个Space，还有以下数据结构：

- Card Table：用于DVM Concurrent GC，当第一次进行垃圾标记后，记录垃圾信息。
- Heap Bitmap：有两个Heap Bitmap，一个用来记录上次GC存活的对象，另一个用来记录这次GC存活的对象。
- Mark Stack：DVM的运行时堆使用标记-清除算法进行GC，Mark Stack就是在GC的标记阶段使用的，用来遍历存活的对象。

### 1.4 DVM引起GC的原因
- GC_CONCURRENT：当堆开始填充时，并发GC可以释放内存。
- GC_FOR_MALLOC：当堆内存已满时，App尝试分配内存而引起的GC
- GC_HPROF_DUMP_HEAP：当请求创建HPROF文件来分析堆内存时出现的GC
- GC_EXPLICIT：显示的GC

## 2、ART虚拟机
### 2.1 ART和DVM的区别
- DVM中的应用每次运行时，字节码都需要通过JIT编译器编译为机器码，这使得应用程序的运行效率降低。在ART中，系统在安装应用程序时会进行一次AOT（ahead of time compilation，预编译），将字节码预先编译成机器码并存储在本地，这样应用程序每次运行时就不需要执行编译了，运行效率提示，耗电量降低。

	AOT缺点：
	
	- 应用程序安装时间变长
	- 字节码预先编译成机器码，机器码需要的存储空间会多一些。

	为解决以上问题，Android 7.0版本的ART中加入了即时编译器JIT，在应用程序安装时并不会将字节码全部编译成机器码，而是在运行中将热点代码编译成机器码，从而缩短应用程序的安装时间并节省了存储空间。
	
- DVM是为32位CPU设计的，而ART支持64位并兼容32位的CPU。
- ART在垃圾回收机制进行了优化，比如更频繁地执行并行垃圾收集，将GC暂停由2次减少为1次等。
- ART的运行时堆空间划分和DVM不同。

### 2.2 ART的运行时堆
ART采用了多种垃圾收集方案，每个方案会运行不同的垃圾收集器，默认采用了CMS（Concurrent Mark-Sweep）方案，该方案主要使用了sticky-CMS和partial-CMS。

ART默认是由4个Space和多个辅助数据结构组成，4个Space分别是**Zygote Space**、**Allocation Space**、**Image Space**和**Large Object Space**。

Zygote Space、Allocation Space和DVM中的作用一样。

Image Space：用于存放一些预加载类。

Large Object Space：用于分配一些大对象。

### 2.3 ART的GC
#### 2.3.1 引起GC的原因
- Concurrent：并发GC，在后台线程运行，不会阻止内存分配
- Alloc：当堆内存已满时，App尝试分配内存引起的GC，这个GC发生在正在分配内存的线程中
- Explicit：显示的请求GC，显示请求GC会阻止分配线程，并且非必要的浪费CPU周期
- NativeAlloc：Native内存分配时，比如Bitmap分配对象，会导致Native内存压力，触发GC
- HomogeneousSpaceCompat：齐性空间压缩是指空闲列表到压缩的空闲列表空间，通常发生在当App已经移动到可察觉的暂停进程状态。这样做的主要原因是减少了内存使用并对堆内存进行碎片整理。

#### 2.3.2 垃圾收集器名称
- CSM：CMS收集器是一种以获取最短收集暂停时间为目标的收集器，采用了标记-清除算法。它是完整的堆垃圾收集器，能释放除了Image Space外的所有空间。
- Concurrent Partial Mark Sweep：部分完整的堆垃圾收集器，能释放除了Image Space和Zygote Space以外的所有空间。
- Concurrent Sticky Mark Sweep：粘性收集器，基于分代的垃圾收集思想，只能释放自上次GC依赖分配的对象。这个垃圾收集器比一个完整的或部分完整的垃圾收集器扫描更频繁，因为它更快并且有更短的暂停时间。
- Marksweep+Semispace：非并发的GC，复制GC用于堆转换以及堆碎片整理。