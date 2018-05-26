# <center>Java Virtual Machine

## Java语言规范
主要定义了什么是Java语言
## JVM规范
主要定义了二进制class文件以及JVM指令 
### 整数的表达
    - 原码 第一位为符号位(1表示负数，0表示正数)
    - 反码 符号位不动，原码取反
    - 补码
        - 负数补码 符号位不动，反码加1
        - 正数补码 和原码相同

    eg.  -6
    原码：10000110
    反码：11111001
    补码：11111010
    

`为什么用补码？
    0 表示有歧义，使用补码来表示0就是唯一
    使用补码来做运算，比如减法运算就是直接使用补码来进行相加，符号位也参与运算，得到的结果就是运算的结果
`

### JVM启动流程
1. java xxx
2. 装载配置，根据当前路径和系统版本查找jvm.cfg
3. 根据配置寻找jvm.dll,jvm.dll为JVM的主要实现
4. 初始化JVM，获得JNIEnv接口，JNIEnv为JVM接口，findClass等操作都是由他来实现
5. 找到main方法并运行

### JVM基本结构
* PC寄存器
  -  每个线程拥有一个PC寄存器
  -  在线程创建时创建
  -  指向下一条指令的地址
  -  执行本地方法是，PC的值为undefined
  
* 方法区
  -  保存装载的类信息
        - 类型的常量池(JDK1.7移到了堆中)
        - 字段，方法信息
        - 方法字节码
   - 通常和永久区(Perm)联系在一起

* Java堆
   - 和程序开发密切相关
   - 应用系统对象都保存在Java堆中
   - 所有线程共享Java堆
   - 对分代GC来说，堆也是分代的
   - 垃圾回收器GC的主要工作区间
   
* Java栈
   - 线程私有
   - 栈由一系列帧组成，因此Java栈也叫帧栈
   - 帧保存一个方法的局部变量，操作数栈，常量池指针
   - 每一次方法调用创建一个的帧，并压栈
   
* 内存模型
   - 每个线程有一个工作内存和主存独立
   - 工作内存存放主存中变量的值的拷贝
   
* 有序性
   - 在本线程内，操作都是有序的
   - 在线程外观察，操作都是无序的（指令重排或者主内存同步延迟）
* 指令重排
   - 线程内串行语义
        -   写后读  a=1,b=a 写入一个变量之后，再读这个位置
        -   写后写  a=1,a=2 写一个变量之后，在写这个变量
        -   读后写  a=b,b=1 读一个变量之后，再写这个变量
        
         `以上语句都不可重排，编译器不考虑多线程的语义，可重排的情况是这样子的：a=1,b=1`
   - 破坏线程间的有序性

    ```
        Class OrderExample{
            int a = 0;
            boolean flag = false;
            public void write() {
                a = 1;
                flag = true;
            }      
            public void read(){
                if(flag){
                    int i = a + 1;
                }
            }
        }
    ``` 
    ` 线程A首先执行write方法，线程B接着执行read方法，线程B在int i = a +1 是不一定能看到a已经被赋值为1的，因为指令重排，write方法中两条语句可能会被打乱`
    
    - 保证有序性的方法
        
        对write方法和read方法改成同步方法
        
    - 指令重排的基本规则
        - 程序顺序规则：一个线程内保证语义的串行性
        - volatile规则：volatile变量的写，先发生于读
        - 锁规则：解锁(unlock)必然发生在随后的加锁(lock)前
        - 传递性：A先于B，B先于C，那么A必然先于C
        - 线程的start方法优先于他的每一个动作
        - 线程的所有操作先于线程的终结（Thread.join()）
        - 线程的中断（interrupt（））先于被中断线程的代码
        - 对象的构造函数执行结束优先于finalize()方法

### JVM常用配置参数
 
* Trace跟踪参数
   
   `打印GC信息`
    -  ```-verbose:gc```
    -  ```-XX:+printGC```
    -  ```-XX:+printGCDetails```
    -  ```-XX:printGCTimeStamps```
    -  ```-Xloggc:log/gc.log```
    -  ```-XX:PrintHeapAtGC```
    -  ```-XX:TraceClassLoading```
    -  ```-XX:PrintClassHistogram```
    
* 堆的分配参数
    - ```-Xmx   指定最大堆大小```
    - ```-Xms   指定最小堆大小```
    - ```-Xmn   指定新生代大小```
    - ```-XX:NewRatio   新生代和老年代的比值```
    - ```-XX:SurvivorRatio  设置两个Survivor区和eden区的比例```
    - ```-XX:HeapDumpOnOutOfMemoryErro  OOM时导出堆信息到文件```
    - ```-XX:HeapDumpPath   导出OOM的路径```
    - ```-XX:OnOutOfMemoryError 在发生OOM时触发```
        -   在OOM时执行一个脚本文件，“-XX:OnOutOfMemoryError=D:/jdk/bin/pritstack.bat %p”  [printstack.bat脚本：  D:/jdk/bin/jstack -F %1 >D:/a.txt] ，当程序发生OOM时，会在a.txt中生成线程的dump，甚至还可以在脚本中加入邮件发送或者重启程序。

    ```
    堆内存分分配总结：
        根据实际情况调整新生代和幸存代的大小，官方推荐新生代占堆内存的3/8,幸存代占新生代的1/10,在发生OOM时，记得dump出堆信息，方便排查问题。
    ```
    
* 永久区分配参数
    - `-XX:PermSize 设置永久区初始空间`
    - `-XX:MaxPermSize  设置永久区最大空间`

* 栈大小分配
    - `-Xss 通常只有几百k，决定了函数调用的深度，每个线程都有独立的栈空间,局部变量，参数分配在栈上`


### GC算法和种类
    GC的对象是堆空间和永久区

* 引用计数法
    
    `老牌垃圾回收算法，通过引用计算来回收垃圾` 
    - 引用计数器实现原理，对于一个对象A，只要有任何一个对象引用了A，则A的引用计数就加1，当引用失效时，引用计数器就减1。只要对象A的引用计数器的值为0，则对象A就不可能在被使用。
    
    - 引用计数法的问题
        - 引用和去引用伴随加法和减法，影响性能
        - 很难处理垃圾对象的循环引用
    

* 标记-清除
 
    `标记清除算法是现代垃圾回收算法的思想基础。标记清除算法将垃圾回收分为两个阶段：标记阶段和清除阶段。`
    
    - 一种可行的实现是，在标记阶段，首先通过根节点，标记所有从根节点开始的可达对象。因此，未被标记的对象就是未被引用的垃圾对象。然后在清除阶段，清除所有的未被标记的对象。  
    
* 标记-压缩

    `标记-压缩算法适合用于存活对象较多的场合，如老年代，它在标记-清除的算法上做了优化。`
    
    - 和标记-清除算法一样，标记-压缩算法也是首先从根节点开始，对所有的可达对象做一次标记。但之后，它并不简单的清理未标记的对象，而是将所有的存活对象压缩到内存的另一端，之后，清理边界外所有的空间

* 复制算法
    `与标记-清除算法相比，复制算法是一种相对高效的回收算法，不适用与存活对象较多的场合，如老年代`
    - 将原有的内存空间分为两块，每次只使用其中的一块，在垃圾回收时，将正在使用的内存中的存活对象复制到未使用的内存块中，之后，清除正在使用的内存块中的所有对象，交换两个内存的角色，完成垃圾回收。
    
    - 复制算法的问题：空间浪费，整合标记-清理算法
    
* 分代思想
    - 依据对象的存活周期进行分类，短命对象归为新生代，长命对象归为老年代
    
    - 根据不同代的特点，选取适合的收集算法
        -   少量对象存活，适合复制算法
        -   大量对象存活，适合标记清理或者标记压缩算法

```
所有的垃圾回收算法都需要识别垃圾对象，如何识别？一般使用可触及性这个概念来表示
    可触及的： 从根节点出发，通过引用链条可以到达这个对象
    可复活的： 所有的引用被释放，当下状态没有被引用到，就是处于可复活状态，在finalize()方法可能复活该对象
    不可触及的： 在finalize()之后，可能会进入不可触及状态，，不可触及的对象不能复活，可以回收。

那什么是根节点呢？
    1. 栈中的对象
    2. 方法区中静态成员或者常量引用的对象(全局对象)
    3. JNI方法栈中引用对象


由于GC引起的一个比较关键的问题：
Stop-The-World
    1. Java中一种全局暂停的现象，
    2. 全局停顿，所有Java代码停止，native代码可以执行，但是不能和JVM交互
    3. 多半由于GC引起
        - dump线程
        - 死锁检查
        - 堆dump

GC时为什么会有全局停顿对象？
    举个例子，比如朋友之间在一个朋友家里边聚会，聚会时很乱，又有新的垃圾产生，房间永远打扫不干净，这个时候只有让大家停止活动了，才能将房间打扫干净。
    全局停顿现象是有危害，可能会长时间的服务停止，没有响应；如果遇到了HA系统，可能会引起主备切换，严重危害生产环境。
- - - - - -    
e.g.
public class CanReliveObj {
	public static CanReliveObj obj;
	@Override
	protected void finalize() throws Throwable {
	    super.finalize();
	    System.out.println("CanReliveObj finalize called");
	    obj=this;
	}
	@Override
	public String toString(){
	    return "I am CanReliveObj";
	}

	public static void main(String[] args) throws
    InterruptedException{
		obj=new CanReliveObj();
		obj=null;   //可复活
		System.gc(); //会执行fialize方法
		Thread.sleep(1000);
		if(obj==null){
		      System.out.println("obj 是 null");
		}else{
		      System.out.println("obj 可用");
		}
		System.out.println("第二次gc");
		obj=null;    //不可复活
		System.gc(); //finalize方法只会执行一次，这次gc不会执行finalize方法
		Thread.sleep(1000);
		if(obj==null){
		      System.out.println("obj 是 null");
		}else{
		      System.out.println("obj 可用");
		}
	}
}

运行结果：
   CanReliveObj finalize called
   obj 可用
   第二次gc
   obj 是 null
   
结论：
    如果第二次gc前没有把obj设为null，那么垃圾回收器就不会回收这个对象，造成内存泄露，容易导致所错，因此得出结论：
        1. 避免使用finalize()方法，操作不慎有可能会导致错误
        2. 优先级低，何时被调用不确定，因为不确定GC在什么时候发生
        3. 可以使用try{}catch(){}来替代它

- - - - - - - - 
```

### GC参数
* 串行收集器
    
    `-XX:+UseSerialGC`
   
   最古老，最稳定，效率高，可能会产生较长的停顿。
   新生代，老年代使用串行回收，新生代复制算法，老年代标记压缩
   
* 并行收集器
    `-XX:+UseParNewGC`
    
    新生代并行，老年代串行，复制算法，多线程，需要多核支持
    
* CMS收集器
   `-XX:+UseConcMarkSweepGC`
   
   Concurrent Mark Sweep 并发标记-清除算法，和用户线程绑定在一起，老年代收集器

### 类装载器

* 类装载验证流程
    - 加载
            
            装载类的第一阶段，取得类的二进制流，转为方法区数据结构，在Java堆中生成java.lang.Class对象

    - 链接
        - 验证
        
                  目的：保证Class流的格式是正确的
                     - 文件格式的验证
                        - 是否以0xCAFEBABE开头
                        - 版本号是否合理
                     - 元数据验证
                        - 是否有父类
                        - 继承了final类？
                        - 非抽象类实现了所有的抽象方法
                     - 字节码验证
                        - 运行检查
                        - 栈数据类型和操作码数据参数吻合
                        - 跳转指令指定到合理位置
                    - 符号引用验证
                        - 常量池中描述类是否存在
                        - 访问的方法或字段是否存在且有足够的权限
                        
        - 准备
            
                - 分配内存，并为类设置初始值（方法区中）
                
        - 解析
            
                - 符号引用替换为直接引用
                
    - 初始化
    
            - 执行类构造器<clinit>
                - static变量  赋值语句
                - static代码块
                
            - 子类的<clinit>调用前保证父类的<clinit>被调用
            - <clinit>是线程安全的

```
    什么是类装载器ClassLoader?
    ClassLoader 是一个抽象类，ClassLoader的实例将读入Java字节码将类装载到JVM中，ClassLoader可以定制，满足不同字节码流获取方式，负责类装载过程中的加载阶段
```
* ClassLoader分类
    -   BootStrap ClassLoader   `启动ClassLoader，默认是jdk包的rt.jar,也可以通过-Xbootclasspath 指定`
    -   Extension ClassLoader  `扩展ClassLoader，默认是%JAVA_HOME%/lib/ext/*.jar%`
    -   App ClassLoader `系统应用ClassLoader,设置的classpath下`
    -   Custom ClassLoader  `自定义ClassLoader`
    
    ` 协同工作模式，自底向上检查类是否已经加载，如果检查到Bootstrap都没有检查到，那么就自顶向下尝试加载类,这就是双亲模式，双亲模式带来了一个问题：顶层ClassLoader,无法加载底层ClassLoader的类。那么如何解决？上下文加载器，基本思想：在顶层ClassLoader中，传入底层ClassLoader的实例` 
    
    ```
        双亲模式是是一种默认的模式，但不是必须的，破坏双亲模式的例子：
            1. Tomcat的WebAppClassLoader 就会先加载自己的Class，找不到再委托parent
            2. OSGI的ClassLoader形成网状结构，根据需要自由加载class
    ```
            
* JVM性能问题定位

    - 工具
   
    ```
     系统常用命令 
            1. vmstat  
            2. top 
            3. pidstat
            4. uptime
    JDK自带命令
            1. jps  
                列出java进程，类似ps, e.g. jps -lvm
                -q : 只输出PID 
                -m : 输出主函数的参数
                -l : 输出主函数的完整路径
                -v ：输出传递给JVM的参数
            2. jmap 
                -  生成Java应用程序的堆快照和对象的统计信息， e.g.  jmap -histo [PID] > jmap.txt
                -  dump堆信息，e.g. jmap -dump:format=b,file=heap.hprof [PID]
            3. jstack
                打印线程dump，e.g. jstack [PID] > jstack.txt
                -l : 打印锁信息
                -m : 打印java和native的帧信息
                -F : 强制dump，当jstack没有响应的时候用
            4. jconsole 
                - 图形化界面
            5. jvisualvm
                - 功能强大的多合一故障诊断和性能监控的可视化工具
        
    ```
    - 堆分析
    
    ```
        OOM 常见原因：
        1. 堆溢出
            
            public static void main(String[] args){
                ArrayList <byte[]> list = new ArrayList<>();
                for(int i = 0; i < 1024; i++ ){
                    list.add(new byte[1024*1024])
                }
                
            }
            
        占用大量堆空间，直接溢出，抛出异常:java.lang.OutOfMemoryError
        解决办法：增大堆内存，或者及时释放堆内存
        
        2. 永久区 生成了大量的类导致
            
            public static void main(String[] args) {
                for(int i=0;i<100000;i++){
                    CglibBean bean = new CglibBean("geym.jvm.ch3.perm.bean"+i,new HashMap());
                }
            }
            
            抛出异常：
            Caused by: java.lang.OutOfMemoryError: PermGen space
                [Full GC[Tenured: 2523K->2523K(10944K), 0.0125610 secs] 2523K->2523K(15936K), 
                [Perm : 4095K->4095K(4096K)], 0.0125868 secs] [Times: user=0.02 sys=0.00, real=0.01 secs] 
                Heap
                    def new generation   total 4992K, used 89K [0x28280000, 0x287e0000, 0x2d7d0000)
                    eden space 4480K,   2% used [0x28280000, 0x282966d0, 0x286e0000)
                    from space 512K,   0% used [0x286e0000, 0x286e0000, 0x28760000)
                    to   space 512K,   0% used [0x28760000, 0x28760000, 0x287e0000)
                     tenured generation   total 10944K, used 2523K [0x2d7d0000, 0x2e280000, 0x38280000)
                    the space 10944K,  23% used [0x2d7d0000, 0x2da46cf0, 0x2da46e00, 0x2e280000)
                    compacting perm gen  total 4096K, used 4095K [0x38280000, 0x38680000, 0x38680000)
                    the space 4096K,  99% used [0x38280000, 0x3867fff0, 0x38680000, 0x38680000)
                    ro space 10240K,  44% used [0x38680000, 0x38af73f0, 0x38af7400, 0x39080000)
                    rw space 12288K,  52% used [0x39080000, 0x396cdd28, 0x396cde00, 0x39c80000)
        
            解决方法：增大Perm区，允许Class回收
    
        3.栈溢出 
            在创建线程的时候，需要为线程分配栈空间，这个栈空间是向操作系统请求的，如果操作系统无法给出足够的空间，就会抛出OOM,线程栈空间和堆空间总和大小不能超过操作系统可分配
           
                public static class SleepThread implements Runnable{
                    public void run(){
                            try {
                                Thread.sleep(10000000);
                            } catch (InterruptedException e) {
                                e.printStackTrace();
                            }
                        }
                 }

                public static void main(String args[]){
                    for(int i=0;i<1000;i++){
                        new Thread(new SleepThread(),"Thread"+i).start();
                        System.out.println("Thread"+i+" created");
                    }
                }

        设置堆栈分配信息：-Xmx 1g -Xss 1m
        
        抛出异常：
        Exception in thread "main" java.lang.OutOfMemoryError: 
unable to create new native thread
        
        解决方法：
         - 减少堆内存
         - 减少线程栈内存

        4. 直接内存溢出
            ByteBuffer.allocateDirect()无法从操作系统获取足够的空间
            
            for(int i=0;i<1024;i++){
            ByteBuffer.allocateDirect(1024*1024);
                System.out.println(i);
                System.gc();
            }
            
            设置JVM参数 ：-Xmx 1g -XX:+PrintGCDetails
            
            发现堆空间剩余还有很多，但是也报了java.lang.OutOfMemoryError,这是因为    堆空间+线程栈空间+直接内存 > 操作系统可分配，直接内存需要GC回收，但是直接内存无法引起GC,即使塞满了直接内存，也无法触发GC,可以借助堆内存满了触发GC来回收直接内存。
            解决方法：
                - 减少堆内存空间
                - 有意触发GC
                

        通过以上这几点可以知道了OOM发生的几种原因，遇到OOM时，如何思考以及解决这类问题？
        
        1. MAT（Memory Analyzer)分析
            支配树：在对象引用图中，所有只想对象B的路径都经过A，则认为对象A支配对象B，如果对象A是离对象B最近的一个支配对象，则认为对象A是对象B的直接支配者，支配者被回收，被支配对象也会被回收
            浅堆： 一个对象结构所占用的内存大小
            深堆： 一个对象被GC回收之后，可以释放的真实内存大小，只能通过对象访问到的（直接或者间接）所有对象（支配树）的浅堆之和
            
        2. visual VM
    ```

### 锁

* 线程安全
    
        常见的比如多线程访问ArrayList.add()，这个时候就可能会发生线程安全问题，解决线程安全问题常用的就是加锁机制

* 对象头Mark
    
        Mark World 对象头的标记，32位
        描述对象的hash，锁信息，垃圾回收标记
* 偏向锁、轻量锁、自旋锁
    
    ```
        偏向锁：
            即锁会偏向于当前已经占有锁的线程，将对象头Mark的标记设置为偏向，并将线程ID写入对象头Mark，只要没有竞争，获得偏向锁的线程，在将来进入同步块，不需要做同步，当其他线程请求相同的锁时，偏向模式结束。因此，在大部分没有竞争的情况下，可以通过偏向锁来提高性能，在竞争激烈的场合，偏向锁会增加系统的负担。
        轻量级锁：
            普通锁性能不够理想，轻量级锁是一种快速锁定的方法，如果一个对象没有被锁定，那就将对象头Mark指针保存到锁对象中，将对象头设置为指向锁的指针(在线程栈空间中)。因此，判断一个线程是否持有轻量级锁，只要判断对象头的指针，是否在线程的栈空间范围内。如果轻量级锁失败了，表示存在竞争，升级为重量级锁，即常规锁，在没有锁竞争的前提下，轻量级锁可以减少传统锁使用OS互斥量产生的性能损耗，在竞争激烈时，轻量级锁会多做很多额外的操作，导致性能下降
        自旋锁：
            当竞争存在时，如果线程可以很快获得锁，那么可以不在OS层挂起线程，让线程多做几个空操作（自旋），在jdk1.7之后内置实现了。如果同步块很长，自旋失败，会降低系统性能，如果同步块很短，自旋成功，节省线程挂起切换时间，提升系统性能。
        
    ```
     - 不是Java语言层面的锁优化方法
     - 内置于JVM中的获取锁的优化方法和获取锁的步骤：
        1. 偏向锁可用会先尝试用偏向锁，在竞争不激烈的情况下，偏向锁可以提高一部分性能
        2. 轻量级锁可用，尝试用轻量级锁
        3. 以上都失败了，尝试自旋锁
        4. 再失败，尝试普通锁，使用OS互斥量在操作系统层挂起
    

* 代码方面优化
    - 减少锁持有时间
        `只同步存在竞争关系的代码块`
            
                public void method(){
                    method1()  
                    synchronized(this){
                        mutextMethod()
                    }
                    method2()
                }
    - 减小锁粒度
        `将大对象拆成小对象，大大增加并行度，降低锁竞争，提高偏向锁，轻量级锁的成功率`
        
                HashMap的同步实现：
                    SynchronizedMap syncMap = Collections.synchronizedMap(Map<K,V> map);
                    
                    Class SynchronizedMap{
                         public V get(Object key){
                                Synchronized(mutex){
                                    return  m.get(key)
                                }
                            }
                        
                          public V put(K key,V value){
                                Synchronized(mutex){
                                    return  m.put(key,value)
                                }
                            }
                    }
                    在返回对象SynchronizedMap中进行put和get操作都是锁的同一个对象，即使是同时操作get和get方法也会竞争锁，这个是没必要的，ConcurrentHashMap就做了减小锁粒度的优化。
                  
            
                ConCurrentHashMap jdk1.7实现：
                    - 将HashMap分为若干个Segment: Segment<K,V>[] segments,Segment中维护HashEntry<K,V>,在进行put操作时，先定位到Segment,锁定Segment，然后执行put操作。
                    - 减小锁粒度之后，ConcurrentHashMap允许若干个线程同时进入
                
                Q:那么问题来了，减小锁粒度之后，可能会带来什么负面影响？以ConcurrentHashMap为例，分割多个segment之后，在什么情况下回性能低下？
                A:segment数量太多的时候是会导致性能低下的，官方推荐的是 hardware thread count,用cat /proc/cpuinfo得到hardware thread count。jdk1.8之后不用segment了，使用的是CAS来实现，后续会专门分析Java8的新特性
                    
 
    - 锁分离
            
        `根据功能进行锁分离`
        
                1. 常用的就是读写锁ReadWriteLock，读-读不互斥，读-写互斥，写-写互斥，在读多写少的情况下可以提高性能
                2. 读写分离思想延伸，只要操作互不影响，锁就可以分离，比如LinkedBlockingQueue（队列，链表数据结构），take操作在链头，put操作在链尾，这是互不影响的操作就可以分离锁。
    
    - 锁粗化
        
            通常情况下，为了保证多线程间的有效并发，会要求每个线程持有锁的时间尽量短，即在使用完共享资源之后，应该立即释放锁，只有这样，等待在这个锁上的其他线程才能尽早的获得执行线程任务。但是，如果对一个锁不停的进行请求、同步和释放，其本身也会消耗系统宝贵的资源，反而不利于性能优化。
            e.g.
            public void method(){
                synchronized(lock){
                    doSomething()
                }
                /** 做一些不需要同步的逻辑，但是执行很快，不需要很长时间的话，可以整个方法都同步起来，避免两次请求同步 */
                doSomethingNotSyncButExecShortly()
                synchronized(lock){
                    doSomething1()
                }
            }      
            
            极端情况：
            for(int  i = 0; i < length; i++){
                synchronized(lock){
                    //每次循环都要请求获取锁，不应该这么做，应该锁整个循环
                    doSomething()
                }
            }      
            
            synchronized(lock){
                for(int  i = 0; i < length; i++){
                    doSomething()
                }
            } 
            
    - 锁消除
        
            在即时编译器时，如果发现了一些不可能被共享的对象，则可以消除这些对象的锁操作，比如final修饰的String，类型安全的类StringBuffer,Vector等
            
    - 无锁

        - 锁是一种悲观的操作
        - 无锁是一种乐观的方式
        - 无锁的实现方式CAS(Compare And Swap),非阻塞的同步。
        
            ```
            CAS(V,E,N)
                V - 要更新的变量的当前值
                E - 期望当前的V的值
                N - V要更新的值
            
            当且仅当 V==E 的时候 ，把E的值赋给V
            
            ```
        - 在应用层面判断多线程的干扰，如果有干扰(V!=E)，则通知线程重试

        e.g. AtomicInteger
    
      
        
                
        
                
                    
        


    
    



