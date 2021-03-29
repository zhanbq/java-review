# java基础

## 1. 谈谈你对 Java 平台的理解？“Java 是解释执行”，这句话正确吗？

java是一种面向对象的语言,特点:

1."一次编写,到处运行"的跨平台.

> java通过字节码和JVM这种跨平台的抽象,屏蔽的操作系统和硬件细节,这就是跨平台的基础.

2.内存的自动回收和分配

jre是java的运行环境,包含了JVM和java类库,而jdk可以看做是jre的一个超集,提供更多的工具,编译器和各种诊断工具.



"java是解释执行"这种说法不太准确.java源码首先经过javac编译器编译为字节码,然后,运行时通过JVM内嵌的解释器转换为机器码.Oracle Hotspot JVM,提供了JIT即时编译器,动态编译,将热点代码直接编译成机器码,这部分热点代码属于编译执行,而不是解释执行了.

## 2. 请对比 Exception 和 Error，另外，运行时异常与一般异常有什么区别？

Exception和Error都是继承了Throwable类,java中只有Throwable类型的实例才可以被抛出或者捕获.

Exception和Error是对不同异常情况的分类.

> Exception是程序正常运行中,可以预料的的异常,需要主动捕获,进行相应的处理.
>
> Error是在正常情况下,不太可能出现的情况,大部分Error都会导致程序出于非正常的不可恢复的状态.这种异常不需要也不方便捕获,常见的OutOfMemoryError,都是Error的子类.

Exception又分为已检查和未检查异常,可检查必须显示捕获,这是编译器检查的一部分.未检查异常就是运行期异常,类似NullPointerException,ArrayIndexOutOfBoundsException,通常这类异常可以编码避免,具体根据需要来判断是否需要捕获,并不会在编译期强制要求.

### 2.1 NoClassDefFoundError 和 ClassNotFoundException 有什么区别

ClassNotfoundException时在编译时JVM加载不到类或者找不到类导致的

而NoClassDefError是在运行时JVM加载不到类或者找不到类

### 2.2 异常处理的性能开销

1. try-catch代码段会产生额外的性能开销,会影响JVM对代码进行优化,利用try-catch控制代码流程远比ifelse低效

2. java每实例化一个Exception,都会对当时的栈进行快照,这是一个相对比较重的操作.如果发生的非常频繁,就必须重视这种开销.

> 当我们的服务出现反应变慢、吞吐量下降的时候，检查发生最频繁的 Exception 也是一种思路。关于诊断后台变慢的问题，我会在后面的 Java 性能基础模块中系统探讨。

## 3.谈谈 final、finally、 finalize 有什么不同？

**final可以用来修饰类,方法,变量.**

- final修饰的class代码不可以继承扩展,
- final修饰的变量不可以修改.
- final修饰的方法不可以被重写.

final不是不可变

```java
final List<String> strList = new ArrayList<>();
 strList.add("Hello");
 strList.add("world");  
 List<String> unmodifiableStrList = List.of("hello", "world");
 unmodifiableStrList.add("again");
```

final 只能约束 strList 这个引用不可以被赋值，但是 strList 对象行为不被 final 影响，添加元素等操作是完全正常的。

[List.of 方法](http://openjdk.java.net/jeps/269)创建的本身就是不可变 List，最后那句 add 是会在运行时抛出异常的。

java语言目前原生不可变的支持.如果要实现不可变的类,需要

- class自身是final修饰的
- 所有成员变量定义为private和final,并且不实现setter方法
- 构造对象式,成员变量使用深度拷贝来初始化,而不是直接赋值,这是一种防御措施,因为无法确定对象不会被其他人修改.
- 如果确实需要时间getter方法,或者其他可能会返回内部状态的方法,使用copy-on-waiter原则,创建私有copy.



**finally是java保证重点代码一定要被执行的一种机制.**

> 例如,可以使用try-finally或try-catch-finally来关闭JDBC连接,保证unlock锁释放等操作.



**finalize是基础类java.lang.Object的一个方法.**

当垃圾回收器将要回收对象所占内存之前被调用,让此对象处理它生前的最后事情.

finalize()的调用具有不确定行，只保证方法会调用，但不保证方法里的任务会被执行完（比如一个对象手脚不够利索，磨磨叽叽，还在自救的过程中，被杀死回收了）。使用不当会影响性能，导致程序死锁、挂起等。

finalize 的执行是和垃圾收集关联在一起的，一旦实现了非空的 finalize 方法，就会导致相应对象回收呈现数量级上的变慢，有人专门做过 benchmark，大概是 40~50 倍的下降。

> finalize 机制现在已经不推荐使用,并且在 JDK 9 开始被标记为 deprecated.
>
> 综上：finalize()方法并没有什么鸟用。
>
> 至于为什么会存在这样一个鸡肋的方法：书中说“它不是C/C++中的析构函数，而是Java刚诞生时为了使C/C++程序员更容易接受它所做出的一个妥协”。



## 4. 强引用,软引用,弱引用,幻象引用有什么区别?具体使用场景是什么？

不同的引用类型主要体现的是**对象不同的可达性状态和垃圾收集的影响.**

**强引用:** 就是我们常见的普通对象引用,

```java
Object obj = new Object()
```

obj就是强引用,通过关键字new创建的对象所关联的引用就是强引用.

当JVM内存空间不足,JVM宁愿抛出OutOfMemory运行时错误,使程序异常终止,也不会随意回收具有强引用的"存活"对象来解决内存不足的问题.

对于一个普通对象,如果没有其他的引用关系,只要超过了引用的作用域或者显示的将相应的强引用赋值为null,就是可以被垃圾收集的了,具体回收时机还要看垃圾收集策略.



**软引用:**软引用通过SoftReference类实现.

软引用的声明周期比强引用短一些.只有当JVM认为内存不足时,才回去试图回收软引用指向的对象:即JVM会确保在抛出OutOfMemoryError之前,清理软引用指向的对象.

```java
public class Test {  

    public static void main(String[] args){  
        System.out.println("开始");            
        A a = new A();            
        SoftReference<A> sr = new SoftReference<A>(a);  
        a = null;  
        if(sr!=null){  
            a = sr.get();  
        }  
        else{  
            a = new A();  
            sr = new SoftReference<A>(a);  
        }            
        System.out.println("结束");     
    }       

}

class A{  
    int[] a ;  
    public A(){  
        a = new int[100000000];  
    }  
} 
```

软引用可以和一个引用队列(ReferenceQueue)联合使用,如果软引用所引用的对象被垃圾回收,JVM就会把这个软引用加入到与之关联的引用队列中.后续我们可以调用ReferenceQueue的poll()方法来检查是否有它所关心的对象被回收. 如果队列为空,将返回一个null,否则该方法返回队列前面的一个Reference对象.

```java
	public static void main(String[] args) {
        //创建queue
        ReferenceQueue queue = new ReferenceQueue();
        //创建弱引用，并且传入queue，此时queue为空
        SoftReference reference = new SoftReference<Object>(new Object(), queue);
        System.out.println(reference.get());   //返回对象
        System.gc();  //GC
        Reference reference1 = null;
        //GC之后，弱引用被回收，reference 被添加进queue，这是remove就可以获得reference1 对象
        try {
            reference1 = queue.remove();//===>这里会阻塞
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println(reference1);

    }
```

> ===>之所以会阻塞 就是因为JVM的内存还足够.

**弱引用:  ** 弱引用通过WeakReference类实现。 弱引用的生命周期比软引用短。

弱引用通过WeakReference类实现.

在垃圾回收器线程扫描它所管辖的内存区域的过程中,一旦发现了具有弱引用的对象,不管当前内存空间足够与否,都会回收它的内存.由于垃圾回收器是一个优先级很低的线程,因此不一定会很快回收弱引用的对象

弱引用可以和一个引用队列（ReferenceQueue）联合使用,如果弱引用所引用的对象被垃圾回收,Java虚拟机就会把这个弱引用加入到与之关联的引用队列中.



**虚引用:** 虚引用仅仅是提供了一种确保对象被finalize以后,做某些事情的机制.如果一个对象仅持有虚引用,那么它就和没有任何引用一样,在任何时候都可能被垃圾回收机回收.

通过PhantomReference类来实现.无法通过虚引用访问对象的任何属性或函数.

虚引用必须和引用队列 （ReferenceQueue）联合使用.当垃圾回收器准备回收一个对象时,如果发现它还有虚引用,就会在回收对象的内存之前,把这个虚引用加入到与之关联的引用队列中.

```java
ReferenceQueue queue = new ReferenceQueue ();
PhantomReference pr = new PhantomReference (object, queue);
```



## 5. String、StringBuffer、StringBuilder有什么区别？

**String** 

是java语言非常重要基础的类,提供了构造和管理字符串的各种基本逻辑.

典型的Immutable(不可变)类,被声明成为final class,所有的属性也都是final的.所以String本身也是不可变长的,所以,类似拼接,裁剪字符串等操作,都会产生新的String对象.字符串的操作一般是很常用的操作,所以效率对应用的性能影响比较明显.

**StringBuffer**

为了解决字符串拼接产生太多中间对象而提供的一个类,我们可以用append或者add方法把字符串添加到已有序列的某位或者指定位置.

> 本质上,StringBuffer或StringBuilder在append是用char数组存储数据,而数组是可以扩容的,这样就不用不停的创建对象了.

> StringBuffer本质是一个线程安全的可修改的字符序列,它保证了线程安全,所以也带来了额外的性能开销,所以除非有线程安全的需要,不然还是使用StringBuilder.

**StringBuilder**

StringBuilder是java1.5中新增,在能力上和StringBuffer没有本质区别,因为他们继承了AbstractStringBuilder,区别仅在于方法是否加了synchronized关键字.

### 5.1 字符串的设计和实现考量

1. **String为什么是final的**

   准确的说Sting是immutablele(不可变)的.

   - String是immutablele类的典型实现,这样就原生的保证了线程安全.
   - 在拷贝构造函数中,由于不可变Immutable对象在拷贝时不需要额外复制数据.

2. **为了修改字符序列的目的,StringBuffer和StringBuilder底层都是利用可修改的char数组,JDK9以后变成了byte数组,**

3. **StingBuffer或StringBuilder内部的数组初始长度都是16.**

   太大浪费空间,太小可能需要重新创建足够大的数组.如果确定拼接会发生非常多次,而且是可以预计的,那么就可以指定合适的大小,避免多次扩容的开销.

   而扩容会产生多重开销,因为要抛弃原油数组,创建新的数组.

   新的数组容量计算方式如下:

   ```java
   	/**
   	返回至少与给定最小容量一样大的容量。 如果足够，则返回增加相同数量+ 2的当前容量。 除非给定的最小容量	大于该MAX_ARRAY_SIZE否则不会返回大于MAX_ARRAY_SIZE的容量
   	*/
   	private int newCapacity(int minCapacity) {
           // overflow-conscious code
           int newCapacity = (value.length << 1) + 2;
           if (newCapacity - minCapacity < 0) {
               newCapacity = minCapacity;
           }
           return (newCapacity <= 0 || MAX_ARRAY_SIZE - newCapacity < 0)
               ? hugeCapacity(minCapacity)
               : newCapacity;
       }
   
   	/**
   	如果minCapacity大于 小于 Integer.MAX_VALUE 直接抛出OutOfMemoryError
   	如果minCapacity小于MAX_ARRAY_SIZE(Integer.MAX_VALUE-8) 且 小于Integer.MAX_VALUE 则返回		minCapacity
   	如果minCapacity大于MAX_ARRAY_SIZE(Integer.MAX_VALUE-8) 返回MAX_ARRAY_SIZE
   	*/
   	private int hugeCapacity(int minCapacity) {
           if (Integer.MAX_VALUE - minCapacity < 0) { // overflow
               throw new OutOfMemoryError();
           }
           return (minCapacity > MAX_ARRAY_SIZE)
               ? minCapacity : MAX_ARRAY_SIZE;
       }
   
   ```

   

### 5.2 字符串缓存

如果把堆进行转储(Dump Heap),然后分析对象组成,会发现平均25%的对象时字符串,而且其中约半数是重复的.

String在Java6以后提供了intern()方法.目的是提示JVM把相应的字符串缓存起来,用来重复使用.

```java
/**
返回字符串对象的规范表示。
最初为空的字符串池由String类私有维护。
调用intern方法时，如果池已经包含equals(Object)方法确定的此String对象的字符串，则返回池中的字符串。 否则，将此String对象添加到池中，并返回对此String对象的引用。
由此可见，对于任何两个字符串s和t ， s.intern() == t.intern()是true当且仅当s.equals(t)是true 。
所有文字字符串和字符串值常量表达式都将被插入。 字符串文字是在Java™语言规范的第3.10.5节中定义的。

返回值：
与该字符串具有相同内容的字符串，但保证来自唯一字符串池
*/
public native String intern();

```

在我们创建字符串对象并调用intern()方法的时候,如果已经有缓存的字符串,就会返回缓存里的实例,否则就缓存起来.

被缓存的字符串是存在的PermGen里的,也就是"永久代",这个空间是有限的,也基本不会被FullGC之外的垃圾收集照顾到.所以如果使用不当就会导致OOM.

后面的版本中,这个缓存被放到了堆中,这样极大的避免了永久代沾满的问题.而且,默认缓存大小也由最初的1009到java7被修改为60013.

> intern是一种显示的排重机制,副作用就是需要写代码时明确的调用,一是不方便,二是日常开发中很难清楚的预计字符串重复的情况.

java8之后推出了一个新特性,G1GC下的字符串排重.它是通过将相同数据的字符串指向同一份数据来做到的,是JVM底层的改变,并不需要Java类库做了什么修改.

字符串的一些基础操作会直接利用 JVM 内部的 Intrinsic 机制，往往运行的就是特殊优化的本地代码，而根本就不是 Java 代码生成的字节码。Intrinsic 可以简单理解为，是一种利用 native 方式 hard-coded 的逻辑，算是一种特别的内联，很多优化还是需要直接使用特定的 CPU 指令

在 Java 9 中，我们引入了 Compact Strings 的设计，对字符串进行了大刀阔斧的改进。将数据存储方式从 char 数组，改变为一个 byte 数组加上一个标识编码的所谓 coder，并且将相关字符串操作类都进行了修改

极端情况下,字符串出现了能力的退化,比如字符串的大小.原来 char 数组的实现，字符串的最大长度就是数组本身的长度限制，但是替换成 byte 数组，同样数组长度下，存储能力是退化了一倍的！

通常的代码开发中,紧凑的字符串带来的其实是优势,更小的内存占用,更快的操作速度.