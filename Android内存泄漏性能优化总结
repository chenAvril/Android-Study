** 本文为转载的优质好文，[原文链接](https://github.com/GeniusVJR/LearningNotes/blob/master/Part1/Android/Android%E5%86%85%E5%AD%98%E6%B3%84%E6%BC%8F%E6%80%BB%E7%BB%93.md)



# Android 内存泄漏总结

内存管理的目的就是让我们在开发中怎么有效的避免我们的应用出现内存泄漏的问题。内存泄漏大家都不陌生了，简单粗俗的讲，就是该被释放的对象没有释放，一直被某个或某些实例所持有却不再被使用导致 GC 不能回收。最近自己阅读了大量相关的文档资料，打算做个 总结 沉淀下来跟大家一起分享和学习，也给自己一个警示，以后 coding 时怎么避免这些情况，提高应用的体验和质量。

我会从 java 内存泄漏的基础知识开始，并通过具体例子来说明 Android 引起内存泄漏的各种原因，以及如何利用工具来分析应用内存泄漏，最后再做总结。


## Java 内存分配策略

Java 程序运行时的内存分配策略有三种,分别是静态分配,栈式分配,和堆式分配，对应的，三种存储策略使用的内存空间主要分别是静态存储区（也称方法区）、栈区和堆区。

* 静态存储区（方法区）：主要存放静态数据、全局 static 数据和常量。这块内存在程序编译时就已经分配好，并且在程序整个运行期间都存在。

* 栈区 ：当方法被执行时，方法体内的局部变量（其中包括基础数据类型、对象的引用）都在栈上创建，并在方法执行结束时这些局部变量所持有的内存将会自动被释放。因为栈内存分配运算内置于处理器的指令集中，效率很高，但是分配的内存容量有限。

* 堆区 ： 又称动态内存分配，通常就是指在程序运行时直接 new 出来的内存，也就是对象的实例。这部分内存在不使用时将会由 Java 垃圾回收器来负责回收。

## 栈与堆的区别：

在方法体内定义的（局部变量）一些基本类型的变量和对象的引用变量都是在方法的栈内存中分配的。当在一段方法块中定义一个变量时，Java 就会在栈中为该变量分配内存空间，当超过该变量的作用域后，该变量也就无效了，分配给它的内存空间也将被释放掉，该内存空间可以被重新使用。

堆内存用来存放所有由 new 创建的对象（包括该对象其中的所有成员变量）和数组。在堆中分配的内存，将由 Java 垃圾回收器来自动管理。在堆中产生了一个数组或者对象后，还可以在栈中定义一个特殊的变量，这个变量的取值等于数组或者对象在堆内存中的首地址，这个特殊的变量就是我们上面说的引用变量。我们可以通过这个引用变量来访问堆中的对象或者数组。

举个例子:

```
public class Sample {
    int s1 = 0;
    Sample mSample1 = new Sample();

    public void method() {
        int s2 = 1;
        Sample mSample2 = new Sample();
    }
}

Sample mSample3 = new Sample();
```

Sample 类的局部变量 s2 和引用变量 mSample2 都是存在于栈中，但 mSample2 指向的对象是存在于堆上的。
mSample3 指向的对象实体存放在堆上，包括这个对象的所有成员变量 s1 和 mSample1，而它自己存在于栈中。

结论：

局部变量的基本数据类型和引用存储于栈中，引用的对象实体存储于堆中。—— 因为它们属于方法中的变量，生命周期随方法而结束。

成员变量全部存储与堆中（包括基本数据类型，引用和引用的对象实体）—— 因为它们属于类，类对象终究是要被new出来使用的。

了解了 Java 的内存分配之后，我们再来看看 Java 是怎么管理内存的。

## Java是如何管理内存

Java的内存管理就是对象的分配和释放问题。在 Java 中，程序员需要通过关键字 new 为每个对象申请内存空间 (基本类型除外)，所有的对象都在堆 (Heap)中分配空间。另外，对象的释放是由 GC 决定和执行的。在 Java 中，内存的分配是由程序完成的，而内存的释放是由 GC 完成的，这种收支两条线的方法确实简化了程序员的工作。但同时，它也加重了JVM的工作。这也是 Java 程序运行速度较慢的原因之一。因为，GC 为了能够正确释放对象，GC 必须监控每一个对象的运行状态，包括对象的申请、引用、被引用、赋值等，GC 都需要进行监控。

监视对象状态是为了更加准确地、及时地释放对象，而释放对象的根本原则就是该对象不再被引用。

为了更好理解 GC 的工作原理，我们可以将对象考虑为有向图的顶点，将引用关系考虑为图的有向边，有向边从引用者指向被引对象。另外，每个线程对象可以作为一个图的起始顶点，例如大多程序从 main 进程开始执行，那么该图就是以 main 进程顶点开始的一棵根树。在这个有向图中，根顶点可达的对象都是有效对象，GC将不回收这些对象。如果某个对象 (连通子图)与这个根顶点不可达(注意，该图为有向图)，那么我们认为这个(这些)对象不再被引用，可以被 GC 回收。
以下，我们举一个例子说明如何用有向图表示内存管理。对于程序的每一个时刻，我们都有一个有向图表示JVM的内存分配情况。以下右图，就是左边程序运行到第6行的示意图。

![](http://www.ibm.com/developerworks/cn/java/l-JavaMemoryLeak/1.gif)

Java使用有向图的方式进行内存管理，可以消除引用循环的问题，例如有三个对象，相互引用，只要它们和根进程不可达的，那么GC也是可以回收它们的。这种方式的优点是管理内存的精度很高，但是效率较低。另外一种常用的内存管理技术是使用计数器，例如COM模型采用计数器方式管理构件，它与有向图相比，精度行低(很难处理循环引用的问题)，但执行效率很高。

## 什么是Java中的内存泄露

在Java中，内存泄漏就是存在一些被分配的对象，这些对象有下面两个特点，首先，这些对象是可达的，即在有向图中，存在通路可以与其相连；其次，这些对象是无用的，即程序以后不会再使用这些对象。如果对象满足这两个条件，这些对象就可以判定为Java中的内存泄漏，这些对象不会被GC所回收，然而它却占用内存。

在C++中，内存泄漏的范围更大一些。有些对象被分配了内存空间，然后却不可达，由于C++中没有GC，这些内存将永远收不回来。在Java中，这些不可达的对象都由GC负责回收，因此程序员不需要考虑这部分的内存泄露。

通过分析，我们得知，对于C++，程序员需要自己管理边和顶点，而对于Java程序员只需要管理边就可以了(不需要管理顶点的释放)。通过这种方式，Java提高了编程的效率。

![](http://www.ibm.com/developerworks/cn/java/l-JavaMemoryLeak/2.gif)

因此，通过以上分析，我们知道在Java中也有内存泄漏，但范围比C++要小一些。因为Java从语言上保证，任何对象都是可达的，所有的不可达对象都由GC管理。

对于程序员来说，GC基本是透明的，不可见的。虽然，我们只有几个函数可以访问GC，例如运行GC的函数System.gc()，但是根据Java语言规范定义， 该函数不保证JVM的垃圾收集器一定会执行。因为，不同的JVM实现者可能使用不同的算法管理GC。通常，GC的线程的优先级别较低。JVM调用GC的策略也有很多种，有的是内存使用到达一定程度时，GC才开始工作，也有定时执行的，有的是平缓执行GC，有的是中断式执行GC。但通常来说，我们不需要关心这些。除非在一些特定的场合，GC的执行影响应用程序的性能，例如对于基于Web的实时系统，如网络游戏等，用户不希望GC突然中断应用程序执行而进行垃圾回收，那么我们需要调整GC的参数，让GC能够通过平缓的方式释放内存，例如将垃圾回收分解为一系列的小步骤执行，Sun提供的HotSpot JVM就支持这一特性。

同样给出一个 Java 内存泄漏的典型例子，

```
Vector v = new Vector(10);
for (int i = 1; i < 100; i++) {
    Object o = new Object();
    v.add(o);
    o = null;   
}
```

在这个例子中，我们循环申请Object对象，并将所申请的对象放入一个 Vector 中，如果我们仅仅释放引用本身，那么 Vector 仍然引用该对象，所以这个对象对 GC 来说是不可回收的。因此，如果对象加入到Vector 后，还必须从 Vector 中删除，最简单的方法就是将 Vector 对象设置为 null。


**详细Java中的内存泄漏**

1.Java内存回收机制

不论哪种语言的内存分配方式，都需要返回所分配内存的真实地址，也就是返回一个指针到内存块的首地址。Java中对象是采用new或者反射的方法创建的，这些对象的创建都是在堆（Heap）中分配的，所有对象的回收都是由Java虚拟机通过垃圾回收机制完成的。GC为了能够正确释放对象，会监控每个对象的运行状况，对他们的申请、引用、被引用、赋值等状况进行监控，Java会使用有向图的方法进行管理内存，实时监控对象是否可以达到，如果不可到达，则就将其回收，这样也可以消除引用循环的问题。在Java语言中，判断一个内存空间是否符合垃圾收集标准有两个：一个是给对象赋予了空值null，以下再没有调用过，另一个是给对象赋予了新值，这样重新分配了内存空间。 

2.Java内存泄漏引起的原因

内存泄漏是指无用对象（不再使用的对象）持续占有内存或无用对象的内存得不到及时释放，从而造成内存空间的浪费称为内存泄漏。内存泄露有时不严重且不易察觉，这样开发者就不知道存在内存泄露，但有时也会很严重，会提示你Out of memory。j

Java内存泄漏的根本原因是什么呢？长生命周期的对象持有短生命周期对象的引用就很可能发生内存泄漏，尽管短生命周期对象已经不再需要，但是因为长生命周期持有它的引用而导致不能被回收，这就是Java中内存泄漏的发生场景。具体主要有如下几大类：

1、静态集合类引起内存泄漏：

像HashMap、Vector等的使用最容易出现内存泄露，这些静态变量的生命周期和应用程序一致，他们所引用的所有的对象Object也不能被释放，因为他们也将一直被Vector等引用着。 

例如

```
Static Vector v = new Vector(10);
for (int i = 1; i<100; i++)
{
Object o = new Object();
v.add(o);
o = null;
}
```

在这个例子中，循环申请Object 对象，并将所申请的对象放入一个Vector 中，如果仅仅释放引用本身（o=null），那么Vector 仍然引用该对象，所以这个对象对GC 来说是不可回收的。因此，如果对象加入到Vector 后，还必须从Vector 中删除，最简单的方法就是将Vector对象设置为null。



3、监听器

在java 编程中，我们都需要和监听器打交道，通常一个应用当中会用到很多监听器，我们会调用一个控件的诸如addXXXListener()等方法来增加监听器，但往往在释放对象的时候却没有记住去删除这些监听器，从而增加了内存泄漏的机会。

4、各种连接 

比如数据库连接（dataSourse.getConnection()），网络连接(socket)和io连接，除非其显式的调用了其close（）方法将其连接关闭，否则是不会自动被GC 回收的。对于Resultset 和Statement 对象可以不进行显式回收，但Connection 一定要显式回收，因为Connection 在任何时候都无法自动回收，而Connection一旦回收，Resultset 和Statement 对象就会立即为NULL。但是如果使用连接池，情况就不一样了，除了要显式地关闭连接，还必须显式地关闭Resultset Statement 对象（关闭其中一个，另外一个也会关闭），否则就会造成大量的Statement 对象无法释放，从而引起内存泄漏。这种情况下一般都会在try里面去的连接，在finally里面释放连接。

5、内部类和外部模块的引用

内部类的引用是比较容易遗忘的一种，而且一旦没释放可能导致一系列的后继类对象没有释放。此外程序员还要小心外部模块不经意的引用，例如程序员A 负责A 模块，调用了B 模块的一个方法如：
public void registerMsg(Object b);
这种调用就要非常小心了，传入了一个对象，很可能模块B就保持了对该对象的引用，这时候就需要注意模块B 是否提供相应的操作去除引用。

6、单例模式 

不正确使用单例模式是引起内存泄漏的一个常见问题，单例对象在初始化后将在JVM的整个生命周期中存在（以静态变量的方式），如果单例对象持有外部的引用，那么这个对象将不能被JVM正常回收，导致内存泄漏，考虑下面的例子：

```
class A{
public A(){
B.getInstance().setA(this);
}
....
}
//B类采用单例模式
class B{
private A a;
private static B instance=new B();
public B(){}
public static B getInstance(){
return instance;
}
public void setA(A a){
this.a=a;
}
//getter...
} 
```


显然B采用singleton模式，它持有一个A对象的引用，而这个A类的对象将不能被回收。想象下如果A是个比较复杂的对象或者集合类型会发生什么情况

## Android中常见的内存泄漏汇总
---

### 集合类泄漏

集合类如果仅仅有添加元素的方法，而没有相应的删除机制，导致内存被占用。如果这个集合类是全局性的变量 (比如类中的静态属性，全局性的 map 等即有静态引用或 final 一直指向它)，那么没有相应的删除机制，很可能导致集合所占用的内存只增不减。比如上面的典型例子就是其中一种情况，当然实际上我们在项目中肯定不会写这么 2B 的代码，但稍不注意还是很容易出现这种情况，比如我们都喜欢通过 HashMap 做一些缓存之类的事，这种情况就要多留一些心眼。

### 单例造成的内存泄漏

由于单例的静态特性使得其生命周期跟应用的生命周期一样长，所以如果使用不恰当的话，很容易造成内存泄漏。比如下面一个典型的例子，

```
public class AppManager {
private static AppManager instance;
private Context context;
private AppManager(Context context) {
this.context = context;
}
public static AppManager getInstance(Context context) {
if (instance == null) {
instance = new AppManager(context);
}
return instance;
}
}
```

这是一个普通的单例模式，当创建这个单例的时候，由于需要传入一个Context，所以这个Context的生命周期的长短至关重要：

1、如果此时传入的是 Application 的 Context，因为 Application 的生命周期就是整个应用的生命周期，所以这将没有任何问题。

2、如果此时传入的是 Activity 的 Context，当这个 Context 所对应的 Activity 退出时，由于该 Context 的引用被单例对象所持有，其生命周期等于整个应用程序的生命周期，所以当前 Activity 退出时它的内存并不会被回收，这就造成泄漏了。

正确的方式应该改为下面这种方式：

```
public class AppManager {
private static AppManager instance;
private Context context;
private AppManager(Context context) {
this.context = context.getApplicationContext();// 使用Application 的context
}
public static AppManager getInstance(Context context) {
if (instance == null) {
instance = new AppManager(context);
}
return instance;
}
}
```

或者这样写，连 Context 都不用传进来了：

```
在你的 Application 中添加一个静态方法，getContext() 返回 Application 的 context，

...

context = getApplicationContext();

...
   /**
     * 获取全局的context
     * @return 返回全局context对象
     */
    public static Context getContext(){
        return context;
    }

public class AppManager {
private static AppManager instance;
private Context context;
private AppManager() {
this.context = MyApplication.getContext();// 使用Application 的context
}
public static AppManager getInstance() {
if (instance == null) {
instance = new AppManager();
}
return instance;
}
}
```

### 匿名内部类/非静态内部类和异步线程

非静态内部类创建静态实例造成的内存泄漏

有的时候我们可能会在启动频繁的Activity中，为了避免重复创建相同的数据资源，可能会出现这种写法：

```
        public class MainActivity extends AppCompatActivity {
        private static TestResource mResource = null;
        @Override
        protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        if(mManager == null){
        mManager = new TestResource();
        }
        //...
        }
        class TestResource {
        //...
        }
        }
```
这样就在Activity内部创建了一个非静态内部类的单例，每次启动Activity时都会使用该单例的数据，这样虽然避免了资源的重复创建，不过这种写法却会造成内存泄漏，因为非静态内部类默认会持有外部类的引用，而该非静态内部类又创建了一个静态的实例，该实例的生命周期和应用的一样长，这就导致了该静态实例一直会持有该Activity的引用，导致Activity的内存资源不能正常回收。正确的做法为：

将该内部类设为静态内部类或将该内部类抽取出来封装成一个单例，如果需要使用Context，请按照上面推荐的使用Application 的 Context。当然，Application 的 context 不是万能的，所以也不能随便乱用，对于有些地方则必须使用 Activity 的 Context，对于Application，Service，Activity三者的Context的应用场景如下：

![](http://img.blog.csdn.net/20151123144226349?spm=5176.100239.blogcont.9.CtU1c4)

其中： NO1表示 Application 和 Service 可以启动一个 Activity，不过需要创建一个新的 task 任务队列。而对于 Dialog 而言，只有在 Activity 中才能创建

### 匿名内部类

android开发经常会继承实现Activity/Fragment/View，此时如果你使用了匿名类，并被异步线程持有了，那要小心了，如果没有任何措施这样一定会导致泄露

```
    public class MainActivity extends Activity {
    ...
    Runnable ref1 = new MyRunable();
    Runnable ref2 = new Runnable() {
        @Override
        public void run() {

        }
    };
       ...
    }
```

ref1和ref2的区别是，ref2使用了匿名内部类。我们来看看运行时这两个引用的内存：

![](http://img2.tbcdn.cn/L1/461/1/fb05ff6d2e68f309b94dd84352c81acfe0ae839e?spm=5176.100239.blogcont.10.CtU1c4)

可以看到，ref1没什么特别的。

但ref2这个匿名类的实现对象里面多了一个引用：

this$0这个引用指向MainActivity.this，也就是说当前的MainActivity实例会被ref2持有，如果将这个引用再传入一个异步线程，此线程和此Acitivity生命周期不一致的时候，就造成了Activity的泄露。

### Handler 造成的内存泄漏

Handler 的使用造成的内存泄漏问题应该说是最为常见了，很多时候我们为了避免 ANR 而不在主线程进行耗时操作，在处理网络任务或者封装一些请求回调等api都借助Handler来处理，但 Handler 不是万能的，对于 Handler 的使用代码编写一不规范即有可能造成内存泄漏。另外，我们知道 Handler、Message 和 MessageQueue 都是相互关联在一起的，万一 Handler 发送的 Message 尚未被处理，则该 Message 及发送它的 Handler 对象将被线程 MessageQueue 一直持有。

由于 Handler 属于 TLS(Thread Local Storage) 变量, 生命周期和 Activity 是不一致的。因此这种实现方式一般很难保证跟 View 或者 Activity 的生命周期保持一致，故很容易导致无法正确释放。

举个例子：

```
    public class SampleActivity extends Activity {

    private final Handler mLeakyHandler = new Handler() {
    @Override
    public void handleMessage(Message msg) {
      // ...
    }
    }

    @Override
    protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);

    // Post a message and delay its execution for 10 minutes.
    mLeakyHandler.postDelayed(new Runnable() {
      @Override
      public void run() { /* ... */ }
    }, 1000 * 60 * 10);

    // Go back to the previous Activity.
    finish();
    }
    }
```

在该 SampleActivity 中声明了一个延迟10分钟执行的消息 Message，mLeakyHandler 将其 push 进了消息队列 MessageQueue 里。当该 Activity 被 finish() 掉时，延迟执行任务的 Message 还会继续存在于主线程中，它持有该 Activity 的 Handler 引用，所以此时 finish() 掉的 Activity 就不会被回收了从而造成内存泄漏（因 Handler 为非静态内部类，它会持有外部类的引用，在这里就是指 SampleActivity）。

修复方法：在 Activity 中避免使用非静态内部类，比如上面我们将 Handler 声明为静态的，则其存活期跟 Activity 的生命周期就无关了。同时通过弱引用的方式引入 Activity，避免直接将 Activity 作为 context 传进去，见下面代码：

```
public class SampleActivity extends Activity {

  /**
   * Instances of static inner classes do not hold an implicit
   * reference to their outer class.
   */
  private static class MyHandler extends Handler {
    private final WeakReference<SampleActivity> mActivity;

    public MyHandler(SampleActivity activity) {
      mActivity = new WeakReference<SampleActivity>(activity);
    }

    @Override
    public void handleMessage(Message msg) {
      SampleActivity activity = mActivity.get();
      if (activity != null) {
        // ...
      }
    }
  }

  private final MyHandler mHandler = new MyHandler(this);

  /**
   * Instances of anonymous classes do not hold an implicit
   * reference to their outer class when they are "static".
   */
  private static final Runnable sRunnable = new Runnable() {
      @Override
      public void run() { /* ... */ }
  };

  @Override
  protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);

    // Post a message and delay its execution for 10 minutes.
    mHandler.postDelayed(sRunnable, 1000 * 60 * 10);

    // Go back to the previous Activity.
    finish();
  }
}
```

综述，即推荐使用静态内部类 + WeakReference 这种方式。每次使用前注意判空。

前面提到了 WeakReference，所以这里就简单的说一下 Java 对象的几种引用类型。

Java对引用的分类有 Strong reference, SoftReference, WeakReference, PhatomReference 四种。

![](https://gw.alicdn.com/tps/TB1U6TNLVXXXXchXFXXXXXXXXXX-644-546.jpg)

在Android应用的开发中，为了防止内存溢出，在处理一些占用内存大而且声明周期较长的对象时候，可以尽量应用软引用和弱引用技术。

软/弱引用可以和一个引用队列（ReferenceQueue）联合使用，如果软引用所引用的对象被垃圾回收器回收，Java虚拟机就会把这个软引用加入到与之关联的引用队列中。利用这个队列可以得知被回收的软/弱引用的对象列表，从而为缓冲器清除已失效的软/弱引用。

假设我们的应用会用到大量的默认图片，比如应用中有默认的头像，默认游戏图标等等，这些图片很多地方会用到。如果每次都去读取图片，由于读取文件需要硬件操作，速度较慢，会导致性能较低。所以我们考虑将图片缓存起来，需要的时候直接从内存中读取。但是，由于图片占用内存空间比较大，缓存很多图片需要很多的内存，就可能比较容易发生OutOfMemory异常。这时，我们可以考虑使用软/弱引用技术来避免这个问题发生。以下就是高速缓冲器的雏形：

首先定义一个HashMap，保存软引用对象。

```
private Map <String, SoftReference<Bitmap>> imageCache = new HashMap <String, SoftReference<Bitmap>> ();
```

再来定义一个方法，保存Bitmap的软引用到HashMap。

![](https://gw.alicdn.com/tps/TB1oW_FLVXXXXXuaXXXXXXXXXXX-679-717.jpg)

使用软引用以后，在OutOfMemory异常发生之前，这些缓存的图片资源的内存空间可以被释放掉的，从而避免内存达到上限，避免Crash发生。

如果只是想避免OutOfMemory异常的发生，则可以使用软引用。如果对于应用的性能更在意，想尽快回收一些占用内存比较大的对象，则可以使用弱引用。

另外可以根据对象是否经常使用来判断选择软引用还是弱引用。如果该对象可能会经常使用的，就尽量用软引用。如果该对象不被使用的可能性更大些，就可以用弱引用。

ok，继续回到主题。前面所说的，创建一个静态Handler内部类，然后对 Handler 持有的对象使用弱引用，这样在回收时也可以回收 Handler 持有的对象，但是这样做虽然避免了 Activity 泄漏，不过 Looper 线程的消息队列中还是可能会有待处理的消息，所以我们在 Activity 的 Destroy 时或者 Stop 时应该移除消息队列 MessageQueue 中的消息。

下面几个方法都可以移除 Message：

```
public final void removeCallbacks(Runnable r);

public final void removeCallbacks(Runnable r, Object token);

public final void removeCallbacksAndMessages(Object token);

public final void removeMessages(int what);

public final void removeMessages(int what, Object object);
```

### 尽量避免使用 static 成员变量

如果成员变量被声明为 static，那我们都知道其生命周期将与整个app进程生命周期一样。

这会导致一系列问题，如果你的app进程设计上是长驻内存的，那即使app切到后台，这部分内存也不会被释放。按照现在手机app内存管理机制，占内存较大的后台进程将优先回收，yi'wei如果此app做过进程互保保活，那会造成app在后台频繁重启。当手机安装了你参与开发的app以后一夜时间手机被消耗空了电量、流量，你的app不得不被用户卸载或者静默。

这里修复的方法是：

不要在类初始时初始化静态成员。可以考虑lazy初始化。
架构设计上要思考是否真的有必要这样做，尽量避免。如果架构需要这么设计，那么此对象的生命周期你有责任管理起来。

### 避免 override finalize()

1、finalize 方法被执行的时间不确定，不能依赖与它来释放紧缺的资源。时间不确定的原因是：
        虚拟机调用GC的时间不确定
        Finalize daemon线程被调度到的时间不确定

2、finalize 方法只会被执行一次，即使对象被复活，如果已经执行过了 finalize 方法，再次被 GC 时也不会再执行了，原因是：

含有 finalize 方法的 object 是在 new 的时候由虚拟机生成了一个 finalize reference 在来引用到该Object的，而在 finalize 方法执行的时候，该 object 所对应的 finalize Reference 会被释放掉，即使在这个时候把该 object 复活(即用强引用引用住该 object )，再第二次被 GC 的时候由于没有了 finalize reference 与之对应，所以 finalize 方法不会再执行。

3、含有Finalize方法的object需要至少经过两轮GC才有可能被释放。


### 资源未关闭造成的内存泄漏

对于使用了BraodcastReceiver，ContentObserver，File，游标 Cursor，Stream，Bitmap等资源的使用，应该在Activity销毁时及时关闭或者注销，否则这些资源将不会被回收，造成内存泄漏。

### 一些不良代码造成的内存压力

有些代码并不造成内存泄露，但是它们，或是对没使用的内存没进行有效及时的释放，或是没有有效的利用已有的对象而是频繁的申请新内存。

比如：
        Bitmap 没调用 recycle()方法，对于 Bitmap 对象在不使用时,我们应该先调用 recycle() 释放内存，然后才它设置为 null. 因为加载 Bitmap 对象的内存空间，一部分是 java 的，一部分 C 的（因为 Bitmap 分配的底层是通过 JNI 调用的 )。 而这个 recyle() 就是针对 C 部分的内存释放。
        构造 Adapter 时，没有使用缓存的 convertView ,每次都在创建新的 converView。这里推荐使用 ViewHolder。

## 总结

对 Activity 等组件的引用应该控制在 Activity 的生命周期之内； 如果不能就考虑使用 getApplicationContext 或者 getApplication，以避免 Activity 被外部长生命周期的对象引用而泄露。

尽量不要在静态变量或者静态内部类中使用非静态外部成员变量（包括context )，即使要使用，也要考虑适时把外部成员变量置空；也可以在内部类中使用弱引用来引用外部类的变量。

对于生命周期比Activity长的内部类对象，并且内部类中使用了外部类的成员变量，可以这样做避免内存泄漏：

        将内部类改为静态内部类
        静态内部类中使用弱引用来引用外部类的成员变量

Handler 的持有的引用对象最好使用弱引用，资源释放时也可以清空 Handler 里面的消息。比如在 Activity onStop 或者 onDestroy 的时候，取消掉该 Handler 对象的 Message和 Runnable.

在 Java 的实现过程中，也要考虑其对象释放，最好的方法是在不使用某对象时，显式地将此对象赋值为 null，比如使用完Bitmap 后先调用 recycle()，再赋为null,清空对图片等资源有直接引用或者间接引用的数组（使用 array.clear() ; array = null）等，最好遵循谁创建谁释放的原则。

正确关闭资源，对于使用了BraodcastReceiver，ContentObserver，File，游标 Cursor，Stream，Bitmap等资源的使用，应该在Activity销毁时及时关闭或者注销。

保持对对象生命周期的敏感，特别注意单例、静态对象、全局性集合等的生命周期。




# Handler内存泄漏分析及解决
---

### 一、介绍

首先，请浏览下面这段handler代码：

```
public class SampleActivity extends Activity {
  private final Handler mLeakyHandler = new Handler() {
    @Override
    public void handleMessage(Message msg) {
      // ... 
    }
  }
}
```

在使用handler时，这是一段很常见的代码。但是，它却会造成严重的内存泄漏问题。在实际编写中，我们往往会得到如下警告：

```
 ⚠ In Android, Handler classes should be static or leaks might occur.

```

### 二、分析

1、 Android角度

当Android应用程序启动时，framework会为该应用程序的主线程创建一个Looper对象。这个Looper对象包含一个简单的消息队列Message Queue，并且能够循环的处理队列中的消息。这些消息包括大多数应用程序framework事件，例如Activity生命周期方法调用、button点击等，这些消息都会被添加到消息队列中并被逐个处理。

另外，主线程的Looper对象会伴随该应用程序的整个生命周期。

然后，当主线程里，实例化一个Handler对象后，它就会自动与主线程Looper的消息队列关联起来。所有发送到消息队列的消息Message都会拥有一个对Handler的引用，所以当Looper来处理消息时，会据此回调[Handler#handleMessage(Message)]方法来处理消息。

2、 Java角度

在java里，非静态内部类 和 匿名类 都会潜在的引用它们所属的外部类。但是，静态内部类却不会。

###三、泄漏来源

请浏览下面一段代码：

```
public class SampleActivity extends Activity {

  private final Handler mLeakyHandler = new Handler() {
    @Override
    public void handleMessage(Message msg) {
      // ...
    }
  }

  @Override
  protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);

    // Post a message and delay its execution for 10 minutes.
    mLeakyHandler.postDelayed(new Runnable() {
      @Override
      public void run() { /* ... */ }
    }, 1000 * 60 * 10);

    // Go back to the previous Activity.
    finish();
  }
}
```

当activity结束(finish)时，里面的延时消息在得到处理前，会一直保存在主线程的消息队列里持续10分钟。而且，由上文可知，这条消息持有对handler的引用，而handler又持有对其外部类（在这里，即SampleActivity）的潜在引用。这条引用关系会一直保持直到消息得到处理，从而，这阻止了SampleActivity被垃圾回收器回收，同时造成应用程序的泄漏。

<<<<<<< HEAD
注意，上面代码中的Runnable类--非静态匿名类--同样持有对其外部类的引用。从而也导致泄漏。

### 四、泄漏解决方案

首先，上面已经明确了内存泄漏来源：

只要有未处理的消息，那么消息会引用handler，非静态的handler又会引用外部类，即Activity，导致Activity无法被回收，造成泄漏；

Runnable类属于非静态匿名类，同样会引用外部类。

为了解决遇到的问题，我们要明确一点：静态内部类不会持有对外部类的引用。所以，我们可以把handler类放在单独的类文件中，或者使用静态内部类便可以避免泄漏。

另外，如果想要在handler内部去调用所在的外部类Activity，那么可以在handler内部使用弱引用的方式指向所在Activity，这样统一不会导致内存泄漏。

对于匿名类Runnable，同样可以将其设置为静态类。因为静态的匿名类不会持有对外部类的引用。

```
public class SampleActivity extends Activity {

  /**
   * Instances of static inner classes do not hold an implicit
   * reference to their outer class.
   */
  private static class MyHandler extends Handler {
    private final WeakReference<SampleActivity> mActivity;

    public MyHandler(SampleActivity activity) {
      mActivity = new WeakReference<SampleActivity>(activity);
    }

    @Override
    public void handleMessage(Message msg) {
      SampleActivity activity = mActivity.get();
      if (activity != null) {
        // ...
      }
    }
  }

  private final MyHandler mHandler = new MyHandler(this);

  /**
   * Instances of anonymous classes do not hold an implicit
   * reference to their outer class when they are "static".
   */
  private static final Runnable sRunnable = new Runnable() {
      @Override
      public void run() { /* ... */ }
  };

  @Override
  protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);

    // Post a message and delay its execution for 10 minutes.
    mHandler.postDelayed(sRunnable, 1000 * 60 * 10);

    // Go back to the previous Activity.
    finish();
  }
}
```

### 五、小结

虽然静态类与非静态类之间的区别并不大，但是对于Android开发者而言却是必须理解的。至少我们要清楚，如果一个内部类实例的生命周期比Activity更长，那么我们千万不要使用非静态的内部类。最好的做法是，使用静态内部类，然后在该类里使用弱引用来指向所在的Activity。

原文链接：

[http://www.jianshu.com/p/cb9b4b71a820](http://www.jianshu.com/p/cb9b4b71a820)
=======
虽然静态类与非静态类之间的区别并不大，但是对于Android开发者而言却是必须理解的。至少我们要清楚，如果一个内部类实例的生命周期比Activity更长，那么我们千万不要使用非静态的内部类。最好的做法是，使用静态内部类，然后在该类里使用弱引用来指向所在的Activity。

原文链接：

[http://www.jianshu.com/p/cb9b4b71a820](http://www.jianshu.com/p/cb9b4b71a820)


# Android性能优化

## 合理管理内存

### 节制的使用Service

如果应用程序需要使用Service来执行后台任务的话，只有当任务正在执行的时候才应该让Service运行起来。当启动一个Service时，系统会倾向于将这个Service所依赖的进程进行保留，系统可以在LRUcache当中缓存的进程数量也会减少，导致切换程序的时候耗费更多性能。我们可以使用IntentService，当后台任务执行结束后会自动停止，避免了Service的内存泄漏。

### 当界面不可见时释放内存

当用户打开了另外一个程序，我们的程序界面已经不可见的时候，我们应当将所有和界面相关的资源进行释放。重写Activity的onTrimMemory()方法，然后在这个方法中监听TRIM_MEMORY_UI_HIDDEN这个级别，一旦触发说明用户离开了程序，此时就可以进行资源释放操作了。

### 当内存紧张时释放内存

onTrimMemory()方法还有很多种其他类型的回调，可以在手机内存降低的时候及时通知我们，我们应该根据回调中传入的级别来去决定如何释放应用程序的资源。

### 避免在Bitmap上浪费内存
读取一个Bitmap图片的时候，千万不要去加载不需要的分辨率。可以压缩图片等操作。

### 是有优化过的数据集合
Android提供了一系列优化过后的数据集合工具类，如SparseArray、SparseBooleanArray、LongSparseArray，使用这些API可以让我们的程序更加高效。HashMap工具类会相对比较低效，因为它需要为每一个键值对都提供一个对象入口，而SparseArray就避免掉了基本数据类型转换成对象数据类型的时间。

### 知晓内存的开支情况

* 使用枚举通常会比使用静态常量消耗两倍以上的内存，尽可能不使用枚举
* 任何一个Java类，包括匿名类、内部类，都要占用大概500字节的内存空间
* 任何一个类的实例要消耗12-16字节的内存开支，因此频繁创建实例也是会在一定程序上影响内存的
* 使用HashMap时，即使你只设置了一个基本数据类型的键，比如说int，但是也会按照对象的大小来分配内存，大概是32字节，而不是4字节，因此最好使用优化后的数据集合

### 谨慎使用抽象编程

在Android使用抽象编程会带来额外的内存开支，因为抽象的编程方法需要编写额外的代码，虽然这些代码根本执行不到，但是也要映射到内存中，不仅占用了更多的内存，在执行效率上也会有所降低。所以需要合理的使用抽象编程。

### 尽量避免使用依赖注入框架

使用依赖注入框架貌似看上去把findViewById()这一类的繁琐操作去掉了，但是这些框架为了要搜寻代码中的注解，通常都需要经历较长的初始化过程，并且将一些你用不到的对象也一并加载到内存中。这些用不到的对象会一直站用着内存空间，可能很久之后才会得到释放，所以可能多敲几行代码是更好的选择。

### 使用多个进程

谨慎使用，多数应用程序不该在多个进程中运行的，一旦使用不当，它甚至会增加额外的内存而不是帮我们节省内存。这个技巧比较适用于哪些需要在后台去完成一项独立的任务，和前台是完全可以区分开的场景。比如音乐播放，关闭软件，已经完全由Service来控制音乐播放了，系统仍然会将许多UI方面的内存进行保留。在这种场景下就非常适合使用两个进程，一个用于UI展示，另一个用于在后台持续的播放音乐。关于实现多进程，只需要在Manifast文件的应用程序组件声明一个android:process属性就可以了。进程名可以自定义，但是之前要加个冒号，表示该进程是一个当前应用程序的私有进程。

## 分析内存的使用情况

系统不可能将所有的内存都分配给我们的应用程序，每个程序都会有可使用的内存上限，被称为堆大小。不同的手机堆大小不同，如下代码可以获得堆大小：

```
ActivityManager manager = (ActivityManager)getSystemService(Context.ACTIVITY_SERVICE);
int heapSize = manager.getMemoryClass();
```
结果以MB为单位进行返回，我们开发时应用程序的内存不能超过这个限制，否则会出现OOM。

### Android的GC操作

Android系统会在适当的时机触发GC操作，一旦进行GC操作，就会将一些不再使用的对象进行回收。GC操作会从一个叫做Roots的对象开始检查，所有它可以访问到的对象就说明还在使用当中，应该进行保留，而其他的对系那个就表示已经不再被使用了。

### Android中内存泄漏
Android中的垃圾回收机制并不能防止内存泄漏的出现导致内存泄漏最主要的原因就是某些长存对象持有了一些其它应该被回收的对象的引用，导致垃圾回收器无法去回收掉这些对象，也就是出现内存泄漏了。比如说像Activity这样的系统组件，它又会包含很多的控件甚至是图片，如果它无法被垃圾回收器回收掉的话，那就算是比较严重的内存泄漏情况了。
举个例子，在MainActivity中定义一个内部类，实例化内部类对象，在内部类新建一个线程执行死循环，会导致内部类资源无法释放，MainActivity的控件和资源无法释放，导致OOM,可借助一系列工具，比如LeakCanary。

## 高性能编码优化

都是一些微优化，在性能方面看不出有什么显著的提升的。使用合适的算法和数据结构是优化程序性能的最主要手段。

### 避免创建不必要的对象
不必要的对象我们应该避免创建：

* 如果有需要拼接的字符串，那么可以优先考虑使用StringBuffer或者StringBuilder来进行拼接，而不是加号连接符，因为使用加号连接符会创建多余的对象，拼接的字符串越长，加号连接符的性能越低。
* 在没有特殊原因的情况下，尽量使用基本数据类型来代替封装数据类型，int比Integer要更加有效，其它数据类型也是一样。
* 当一个方法的返回值是String的时候，通常需要去判断一下这个String的作用是什么，如果明确知道调用方会将返回的String再进行拼接操作的话，可以考虑返回一个StringBuffer对象来代替，因为这样可以将一个对象的引用进行返回，而返回String的话就是创建了一个短生命周期的临时对象。
* 基本数据类型的数组也要优于对象数据类型的数组。另外两个平行的数组要比一个封装好的对象数组更加高效，举个例子，Foo[]和Bar[]这样的数组，使用起来要比Custom(Foo,Bar)[]这样的一个数组高效的多。

尽可能地少创建临时对象，越少的对象意味着越少的GC操作。

### 静态优于抽象
如果你并不需要访问一个对系那个中的某些字段，只是想调用它的某些方法来去完成一项通用的功能，那么可以将这个方法设置成静态方法，调用速度提升15%-20%，同时也不用为了调用这个方法去专门创建对象了，也不用担心调用这个方法后是否会改变对象的状态(静态方法无法访问非静态字段)。

### 对常量使用static final修饰符
```
static int intVal = 42;  
static String strVal = "Hello, world!";  
```
编译器会为上面的代码生成一个初始方法，称为<clinit>方法，该方法会在定义类第一次被使用的时候调用。这个方法会将42的值赋值到intVal当中，从字符串常量表中提取一个引用赋值到strVal上。当赋值完成后，我们就可以通过字段搜寻的方式去访问具体的值了。

final进行优化:

```
static final int intVal = 42;  
static final String strVal = "Hello, world!";  
```

这样，定义类就不需要<clinit>方法了，因为所有的常量都会在dex文件的初始化器当中进行初始化。当我们调用intVal时可以直接指向42的值，而调用strVal会用一种相对轻量级的字符串常量方式，而不是字段搜寻的方式。

这种优化方式只对基本数据类型以及String类型的常量有效，对于其他数据类型的常量是无效的。

### 使用增强型for循环语法

```
static class Counter {  
    int mCount;  
}  
  
Counter[] mArray = ...  
  
public void zero() {  
    int sum = 0;  
    for (int i = 0; i < mArray.length; ++i) {  
        sum += mArray[i].mCount;  
    }  
}  
  
public void one() {  
    int sum = 0;  
    Counter[] localArray = mArray;  
    int len = localArray.length;  
    for (int i = 0; i < len; ++i) {  
        sum += localArray[i].mCount;  
    }  
}  
  
public void two() {  
    int sum = 0;  
    for (Counter a : mArray) {  
        sum += a.mCount;  
    }  
}  
```

zero()最慢，每次都要计算mArray的长度，one()相对快得多，two()fangfa在没有JIT(Just In Time Compiler)的设备上是运行最快的，而在有JIT的设备上运行效率和one()方法不相上下，需要注意这种写法需要JDK1.5之后才支持。

Tips:ArrayList手写的循环比增强型for循环更快，其他的集合没有这种情况。因此默认情况下使用增强型for循环，而遍历ArrayList使用传统的循环方式。

### 多使用系统封装好的API

系统提供不了的Api完成不了我们需要的功能才应该自己去写，因为使用系统的Api很多时候比我们自己写的代码要快得多，它们的很多功能都是通过底层的汇编模式执行的。
举个例子，实现数组拷贝的功能，使用循环的方式来对数组中的每一个元素一一进行赋值当然可行，但是直接使用系统中提供的System.arraycopy()方法会让执行效率快9倍以上。

### 避免在内部调用Getters/Setters方法

面向对象中封装的思想是不要把类内部的字段暴露给外部，而是提供特定的方法来允许外部操作相应类的内部字段。但在Android中，字段搜寻比方法调用效率高得多，我们直接访问某个字段可能要比通过getters方法来去访问这个字段快3到7倍。但是编写代码还是要按照面向对象思维的，我们应该在能优化的地方进行优化，比如避免在内部调用getters/setters方法。

## 布局优化技巧
---
### 重用布局文件
**<include>**

<include>标签可以允许在一个布局当中引入另一个布局，那么比如说我们程序的所有界面都有一个公共的部分，这个时候最好的做法就是将这个公共的部分提取到一个独立的布局中，然后每个界面的布局文件当中来引用这个公共的布局。

Tips:如果我们要在<include>标签中覆写layout属性，必须要将layout_width和layout_height这两个属性也进行覆写，否则覆写xiaoguo将不会生效。

**<merge>**

<merge>标签是作为<include>标签的一种辅助扩展来使用的，它的主要作用是为了防止在引用布局文件时引用文件时产生多余的布局嵌套。布局嵌套越多，解析起来就越耗时，性能就越差。因此编写布局文件时应该让嵌套的层数越少越好。

举例：比如在LinearLayout里边使用一个<include>布局。<include>里边又有一个LinearLayout，那么其实就存在了多余的布局嵌套，使用merge可以解决这个问题。

### 仅在需要时才加载布局

某个布局当中的元素不是一起显示出来的，普通情况下只显示部分常用的元素，而那些不常用的元素只有在用户进行特定操作时才会显示出来。

举例：填信息时不是需要全部填的，有一个添加更多字段的选项，当用户需要添加其他信息的时候，才将另外的元素显示到界面上。用VISIBLE性能表现一般，可以用ViewStub。ViewStub也是View的一种，但是没有大小，没有绘制功能，也不参与布局，资源消耗非常低，可以认为完全不影响性能。

```
<ViewStub   
        android:id="@+id/view_stub"  
        android:layout="@layout/profile_extra"  
        android:layout_width="match_parent"  
        android:layout_height="wrap_content"  
        />  
```

```
public void onMoreClick() {  
    ViewStub viewStub = (ViewStub) findViewById(R.id.view_stub);  
    if (viewStub != null) {  
        View inflatedView = viewStub.inflate();  
        editExtra1 = (EditText) inflatedView.findViewById(R.id.edit_extra1);  
        editExtra2 = (EditText) inflatedView.findViewById(R.id.edit_extra2);  
        editExtra3 = (EditText) inflatedView.findViewById(R.id.edit_extra3);  
    }  
}  
```

tips：ViewStub所加载的布局是不可以使用<merge>标签的，因此这有可能导致加载出来出来的布局存在着多余的嵌套结构。


# LeakCanary的工作过程以及原理

> 本文是转载的！ 原文地址：http://blog.csdn.net/zivensonice/article/details/51639763

先说一下，这篇文章，是博主看到的少有的好文，感觉写的非常通俗易通。

# 曾经检测内存泄露的方式

让我们来看看在没有LeakCanary之前，我们怎么来检测内存泄露 

1. Bug收集 
通过Bugly、友盟这样的统计平台，统计Bug，了解OutOfMemaryError的情况。 

2. 重现问题 
对Bug进行筛选，归类，排除干扰项。然后为了重现问题，有时候你必须找到出现问题的机型，因为有些问题只会在特定的设备上才会出现。为了找到特定的机型，可能会想尽一切办法，去买、去借、去求人（14年的时候，上家公司专门派了一个商务去广州找了一家租赁手机的公司，借了50台手机回来，600块钱一天）。然后，为了重现问题，一遍一遍的尝试，去还原当时OutOfMemaryError出现的原因，用最原始、最粗暴的方式。 

3. Dump导出hprof文件 
使用Eclipse ADT的DDMS，观察Heap，然后点击手动GC按钮(Cause GC)，观察内存增长情况，导出hprof文件。 
主要观测的两项数据： 

3-1. Heap Size的大小，当资源增加到堆空余空间不够的时候，系统会增加堆空间的大小，但是超过可分配的最大值（比如手机给App分配的最大堆空间为128M）就会发生OutOfMemaryError,这个时候进程就会被杀死。这个最大堆空间，不同手机会有不同的值，跟手机内存大小和厂商定制过后的系统存在关联。 

3-2. Allocated堆中已分配的大小，这是应用程序实际占用的大小，资源回收后，这项数据会变小。 
查看操作前后的堆数据，看是否存在内存泄露，比如反复打开、关闭一个页面，看看堆空间是否会一直增大。 

![](http://img.blog.csdn.net/20160611180718063)

4. 然后使用MAT内存分析工具打开，反复查看找到那些原本应该被回收掉的对象。 

5. 计算这个对象到GC roots的最短强引用路径。 

6. 确定那个路径中那个引用不该有，然后修复问题。

很麻烦，不是吗。现在有一个类库可以直接解决这个问题


## LeakCanary

### 使用方式

使用AndroidStudio，在Module.app的build.gradle中引入

```
dependencies {
   debugCompile 'com.squareup.leakcanary:leakcanary-android:1.4-beta2'
   releaseCompile 'com.squareup.leakcanary:leakcanary-android-no-op:1.4-beta2'
   testCompile 'com.squareup.leakcanary:leakcanary-android-no-op:1.4-beta2'
 }
```

然后在Application中重写onCreate()方法

```
public class ExampleApplication extends Application {
  @Override public void onCreate() {
    super.onCreate();
    LeakCanary.install(this);
  }
}
```

在Activity中写一些导致内存泄露的代码，当发生内存泄露了，会在通知栏弹出消息，点击跳转到泄露页面

![](http://img.blog.csdn.net/20160612000027141)

LeakCanary 可以做到非常简单方便、低侵入性地捕获内存泄漏代码，甚至很多时候你可以捕捉到 Android 系统组件的内存泄漏代码，最关键是不用再进行（捕获错误+Bug归档+场景重现+Dump+Mat分析） 这一系列复杂操作，6得不行。


## 原理分析

### 如果我们自己实现

首先，设想如果让我们自己来实现一个LeakCanary，我们怎么来实现。 
按照前面说的曾经检测内存的方式，我想，大概需要以下几个步骤： 
1. 检测一个对象，查看他是否被回收了。

2. 如果没有被回收，使用DDMS的dump导出.hprof文件，确定是否内存泄露，如果泄露了导出最短引用路径 

3. 把最短引用路径封装到一个对象中，用Intent发送给Notification，然后点击跳转到展示页，页面展示

## 检测对象，是否被回收

我们来看看，LeakCanary是不是按照这种方式实现的。除了刚才说的只需要在Application中的onCreate方法注册LeakCanary.install(this);这种方式。 查看源码，使用官方给的Demo示例代码中，我们发现有一个RefWatcher对象，也可以用来监测，看看它是如何使用的。 
MainActivity.class

```
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        RefWatcher refWatcher = LeakCanary.androidWatcher(getApplicationContext(),
                new ServiceHeapDumpListener(getApplicationContext(), DisplayLeakService.class),
                AndroidExcludedRefs.createAppDefaults().build());
        refWatcher.watch(this);
  }
```


就是把MainActivity作为一个对象监测起来，查看``refWatcher.watch(this)``的实现



```
public void watch(Object watchedReference) {
    watch(watchedReference, "");
  }

  /**
   * Watches the provided references and checks if it can be GCed. This method is non blocking,
   * the check is done on the {@link Executor} this {@link RefWatcher} has been constructed with.
   *
   * @param referenceName An logical identifier for the watched object.
   */
  public void watch(Object watchedReference, String referenceName) {
    Preconditions.checkNotNull(watchedReference, "watchedReference");
    Preconditions.checkNotNull(referenceName, "referenceName");
    if (debuggerControl.isDebuggerAttached()) {
      return;
    }
    final long watchStartNanoTime = System.nanoTime();
    String key = UUID.randomUUID().toString();
    retainedKeys.add(key);
    final KeyedWeakReference reference =
        new KeyedWeakReference(watchedReference, key, referenceName, queue);

    watchExecutor.execute(new Runnable() {
      @Override public void run() {
        ensureGone(reference, watchStartNanoTime);
      }
    });
  }
```


可以总结出他的实现步骤如下： 
1. 先检查监测对象是否为空，为空抛出异常 

2. 如果是在调试Debugger过程中允许内存泄露出现，不再监测。因为这个时候监测的对象是不准确的，而且会干扰我们调试代码。 

3. 给监测对象生成UUID唯一标识符，存入Set集合，方便查找。 

4. 然后定义了一个KeyedWeakReference，查看下KeyedWeakReference是个什么玩意




```
public final class KeyedWeakReference extends WeakReference<Object> {
  public final String key;
  public final String name;

  KeyedWeakReference(Object referent, String key, String name,
      ReferenceQueue<Object> referenceQueue) {
    super(Preconditions.checkNotNull(referent, "referent"), Preconditions.checkNotNull(referenceQueue, "referenceQueue"));
    this.key = Preconditions.checkNotNull(key, "key");
    this.name = Preconditions.checkNotNull(name, "name");
  }
}
```

原来KeyedWeakReference就是对WeakReference进行了一些加工，是一种装饰设计模式，其实就是弱引用的衍生类。配合前面的Set retainedKeys使用，retainedKeys代表的是没有被GC回收的对象，referenceQueue中的弱引用代表的是被GC了的对象，通过这两个结构就可以明确知道一个对象是不是被回收了。（ 一个对象在referenceQueue可以找到当时在retainedKeys中找不到，那么肯定被回收了，没有内存泄漏一说） 

5. 接着看上面的执行过程，然后通过线程池开启了一个异步任务方法ensureGone。watchExecutor看看这个实体的类实现—AndroidWatchExecutor，查看源码


```
public final class AndroidWatchExecutor implements Executor {

  static final String LEAK_CANARY_THREAD_NAME = "LeakCanary-Heap-Dump";
  private static final int DELAY_MILLIS = 5000;

  private final Handler mainHandler;
  private final Handler backgroundHandler;

  public AndroidWatchExecutor() {
    mainHandler = new Handler(Looper.getMainLooper());
    HandlerThread handlerThread = new HandlerThread(LEAK_CANARY_THREAD_NAME);
    handlerThread.start();
    backgroundHandler = new Handler(handlerThread.getLooper());
  }

  @Override public void execute(final Runnable command) {
    if (isOnMainThread()) {
      executeDelayedAfterIdleUnsafe(command);
    } else {
      mainHandler.post(new Runnable() {
        @Override public void run() {
          executeDelayedAfterIdleUnsafe(command);
        }
      });
    }
  }

  private boolean isOnMainThread() {
    return Looper.getMainLooper().getThread() == Thread.currentThread();
  }

  private void executeDelayedAfterIdleUnsafe(final Runnable runnable) {
    // This needs to be called from the main thread.
    Looper.myQueue().addIdleHandler(new MessageQueue.IdleHandler() {
      @Override public boolean queueIdle() {
        backgroundHandler.postDelayed(runnable, DELAY_MILLIS);
        return false;
      }
    });
  }
}
```


做得事情就是，通过主线程的mainHandler转发到后台backgroundHandler执行任务，后台线程延迟DELAY_MILLIS这么多时间执行 

6. 具体执行的任务在ensureGone()方法里面

```
void ensureGone(KeyedWeakReference reference, long watchStartNanoTime) {
    long gcStartNanoTime = System.nanoTime();
    //记录观测对象的时间
    long watchDurationMs = NANOSECONDS.toMillis(gcStartNanoTime - watchStartNanoTime);
    //清除在queue中的弱引用 保留retainedKeys中剩下的对象
    removeWeaklyReachableReferences();
    //如果剩下的对象中不包含引用对象，说明已被回收，返回||调试中,返回
    if (gone(reference) || debuggerControl.isDebuggerAttached()) {
      return;
    }
    //请求执行GC
    gcTrigger.runGc();
    //再次清理一次对象
    removeWeaklyReachableReferences();
    if (!gone(reference)) {
      long startDumpHeap = System.nanoTime();
      //记录下GC执行时间
      long gcDurationMs = NANOSECONDS.toMillis(startDumpHeap - gcStartNanoTime);
      //Dump导出hprof文件
      File heapDumpFile = heapDumper.dumpHeap();

      if (heapDumpFile == null) {
        // Could not dump the heap, abort.
        return;
      }
      //记录下Dump和文件导出用的时间
      long heapDumpDurationMs = NANOSECONDS.toMillis(System.nanoTime() - startDumpHeap);
      //分析hprof文件
      heapdumpListener.analyze(
          new HeapDump(heapDumpFile, reference.key, reference.name, excludedRefs, watchDurationMs,
              gcDurationMs, heapDumpDurationMs));
    }
  }

  private boolean gone(KeyedWeakReference reference) {
    return !retainedKeys.contains(reference.key);
  }
```


```
private void removeWeaklyReachableReferences() {
    // WeakReferences are enqueued as soon as the object to which they point to becomes weakly
    // reachable. This is before finalization or garbage collection has actually happened.
    KeyedWeakReference ref;
    while ((ref = (KeyedWeakReference) queue.poll()) != null) {
      retainedKeys.remove(ref.key);
    }
  }
```


这里我们思考两个问题： 
1. retainedKeys和queue怎么关联起来的？这里的removeWeaklyReachableReferences方法就实现了我们说的 retainedKeys代表的是没有被GC回收的对象，queue中的弱引用代表的是被GC了的对象，之间的关联， 一个对象在queue可以找到当时在retainedKeys中找不到，那么肯定被回收了。gone()返回true说明对象已被回收，不需要观测了。 

2. 为什么执行removeWeaklyReachableReferences()两次？为了保证效率，如果对象被回收，没必要再通知GC执行，Dump操作等等一系列繁琐步骤，况且GC是一个线程优先级极低的线程，就算你通知了，她也不一定会执行，基于这一点，我们分析的观测对象的时机就显得尤为重要了，在对象被回收的时候召唤观测。



### 何时执行观测对象

我们观测的是一个Activity，Activity这样的组件都存在生命周期，在他生命周期结束的时，观测他如果还存活的话 就肯定就存在内存泄露了，进一步推论，Activity的生命周期结束就关联到它的onDestory()方法，也就是只要重写这个方法就可以了。

```
@Override
    protected void onDestroy() {
        super.onDestroy();
        refWatcher.watch(this);
    }
```

在MainActivity中加上这行代码就好了，但是我们显然不想每个Activity都这样干，都是同样的代码为啥要重复着写，当然解决办法呼之欲出：


```
private final Application.ActivityLifecycleCallbacks lifecycleCallbacks =
      new Application.ActivityLifecycleCallbacks() {
        @Override public void onActivityCreated(Activity activity, Bundle savedInstanceState) {
        }

        @Override public void onActivityStarted(Activity activity) {
        }

        @Override public void onActivityResumed(Activity activity) {
        }

        @Override public void onActivityPaused(Activity activity) {
        }

        @Override public void onActivityStopped(Activity activity) {
        }

        @Override public void onActivitySaveInstanceState(Activity activity, Bundle outState) {
        }

        @Override public void onActivityDestroyed(Activity activity) {
          ActivityRefWatcher.this.onActivityDestroyed(activity);
        }
      };
    void onActivityDestroyed(Activity activity) {
        refWatcher.watch(activity);
    }
```



LeakCanary源码是这样做的，通过ActivityLifecycleCallbacks转发，然后在install()中使用这个接口，这就实现了我们只需要调用LeakCanary.install(this);这句代码在Application中就可以实现监测

```
public final class LeakCanary {
    public static RefWatcher install(Application application) {
        return install(application, DisplayLeakService.class, AndroidExcludedRefs.createAppDefaults().build());
    }

    public static RefWatcher install(Application application, Class<? extends AbstractAnalysisResultService> listenerServiceClass, ExcludedRefs excludedRefs) {
        if(isInAnalyzerProcess(application)) {
            return RefWatcher.DISABLED;
        } else {
            enableDisplayLeakActivity(application);
            ServiceHeapDumpListener heapDumpListener = new ServiceHeapDumpListener(application, listenerServiceClass);
            RefWatcher refWatcher = androidWatcher(application, heapDumpListener, excludedRefs);
            ActivityRefWatcher.installOnIcsPlus(application, refWatcher);
            return refWatcher;
        }
    }
```


不需要在每个Activity方法的结束再多写几行onDestroy()代码，但是这个方法有个缺点，看注释


// If you need to support Android < ICS, override onDestroy() in your base activity.




```
 		//ICS
        October 2011: Android 4.0.
        public static final int ICE_CREAM_SANDWICH = 14;

```

如果是SDK 14 android 4.0以下的系统，不具备这个接口，也就是还是的通过刚才那种方式重写onDestory()方法。而且只实现了ActivityRefWatcher.installOnIcsPlus(application, refWatcher);对Activity进行监测，如果是服务或者广播还需要我们自己实现

### 分析hprof文件

接着分析，查看文件解析类发现他是个转发工具类


```
public final class ServiceHeapDumpListener implements HeapDump.Listener {
  ...
  @Override public void analyze(HeapDump heapDump) {
      Preconditions.checkNotNull(heapDump, "heapDump");
     //转发给HeapAnalyzerService
    HeapAnalyzerService.runAnalysis(context, heapDump, listenerServiceClass);
  }
}
```

通过IntentService运行在另一个进程中执行分析任务



```
public final class HeapAnalyzerService extends IntentService {

  private static final String LISTENER_CLASS_EXTRA = "listener_class_extra";
  private static final String HEAPDUMP_EXTRA = "heapdump_extra";

  public static void runAnalysis(Context context, HeapDump heapDump,
      Class<? extends AbstractAnalysisResultService> listenerServiceClass) {
    Intent intent = new Intent(context, HeapAnalyzerService.class);
    intent.putExtra(LISTENER_CLASS_EXTRA, listenerServiceClass.getName());
    intent.putExtra(HEAPDUMP_EXTRA, heapDump);
    context.startService(intent);
  }

  @Override protected void onHandleIntent(Intent intent) {
    String listenerClassName = intent.getStringExtra(LISTENER_CLASS_EXTRA);
    HeapDump heapDump = (HeapDump) intent.getSerializableExtra(HEAPDUMP_EXTRA);

    ExcludedRefs androidExcludedDefault = createAndroidDefaults().build();
    HeapAnalyzer heapAnalyzer = new HeapAnalyzer(androidExcludedDefault, heapDump.excludedRefs);
    //获取分析结果
    AnalysisResult result = heapAnalyzer.checkForLeak(heapDump.heapDumpFile, heapDump.referenceKey);
    AbstractAnalysisResultService.sendResultToListener(this, listenerClassName, heapDump, result);
  }
}
```

查看heapAnalyzer.checkForLeak代码

```
public AnalysisResult checkForLeak(File heapDumpFile, String referenceKey) {
    long analysisStartNanoTime = System.nanoTime();

    if (!heapDumpFile.exists()) {
      Exception exception = new IllegalArgumentException("File does not exist: " + heapDumpFile);
      return AnalysisResult.failure(exception, since(analysisStartNanoTime));
    }

    ISnapshot snapshot = null;
    try {
      // 加载hprof文件
      snapshot = openSnapshot(heapDumpFile);
      //找到泄露对象
      IObject leakingRef = findLeakingReference(referenceKey, snapshot);

      // False alarm, weak reference was cleared in between key check and heap dump.
      if (leakingRef == null) {
        return AnalysisResult.noLeak(since(analysisStartNanoTime));
      }

      String className = leakingRef.getClazz().getName();
      // 最短引用路径
      AnalysisResult result =
          findLeakTrace(analysisStartNanoTime, snapshot, leakingRef, className, true);
      //如果没找到  尝试排除系统进程干扰的情况下找出最短引用路径
      if (!result.leakFound) {
        result = findLeakTrace(analysisStartNanoTime, snapshot, leakingRef, className, false);
      }

      return result;
    } catch (SnapshotException e) {
      return AnalysisResult.failure(e, since(analysisStartNanoTime));
    } finally {
      cleanup(heapDumpFile, snapshot);
    }
  }
```

到这里，我们就找到了泄露对象的最短引用路径，剩下的工作就是发送消息给通知，然后点击通知栏跳转到我们另一个App打开绘制出路径即可。

### 补充—排除干扰项

但是我们在找出最短引用路径的时候，有这样一段代码，他是干什么的呢

```
// 最短引用路径
      AnalysisResult result =
          findLeakTrace(analysisStartNanoTime, snapshot, leakingRef, className, true);
      //如果没找到  尝试排除系统进程干扰的情况下找出最短引用路径
      if (!result.leakFound) {
        result = findLeakTrace(analysisStartNanoTime, snapshot, leakingRef, className, false);
```


查看findLeakTrace()

```
private AnalysisResult findLeakTrace(long analysisStartNanoTime, ISnapshot snapshot,
      IObject leakingRef, String className, boolean excludingKnownLeaks) throws SnapshotException {

    ExcludedRefs excludedRefs = excludingKnownLeaks ? this.excludedRefs : baseExcludedRefs;

    PathsFromGCRootsTree gcRootsTree = shortestPathToGcRoots(snapshot, leakingRef, excludedRefs);

    // False alarm, no strong reference path to GC Roots.
    if (gcRootsTree == null) {
      return AnalysisResult.noLeak(since(analysisStartNanoTime));
    }

    LeakTrace leakTrace = buildLeakTrace(snapshot, gcRootsTree, excludedRefs);

    return AnalysisResult.leakDetected(!excludingKnownLeaks, className, leakTrace, since(analysisStartNanoTime));
  }
```


唯一的不同是excludingKnownLeaks 从字面意思也很好理解，是否排除已知内存泄露

其实是这样的，在我们系统中本身就存在一些内存泄露的情况，这是上层App工程师无能为力的。但是如果是厂商或者做Android Framework层的工程师可能需要关心这个，于是做成一个参数配置的方式，让我们灵活选择岂不妙哉。当然，默认是会排除系统自带泄露情况的，不然打开App，弹出一堆莫名其妙的内存泄露，我们还无能为力，着实让人惶恐，而且我们还可以自己配置。 
通过ExcludedRefs这个类


```
public final class ExcludedRefs implements Serializable {

  public final Map<String, Set<String>> excludeFieldMap;
  public final Map<String, Set<String>> excludeStaticFieldMap;
  public final Set<String> excludedThreads;

  private ExcludedRefs(Map<String, Set<String>> excludeFieldMap,
      Map<String, Set<String>> excludeStaticFieldMap, Set<String> excludedThreads) {
    // Copy + unmodifiable.
    this.excludeFieldMap = unmodifiableMap(new LinkedHashMap<String, Set<String>>(excludeFieldMap));
    this.excludeStaticFieldMap = unmodifiableMap(new LinkedHashMap<String, Set<String>>(excludeStaticFieldMap));
    this.excludedThreads = unmodifiableSet(new LinkedHashSet<String>(excludedThreads));
  }

  public static final class Builder {
    private final Map<String, Set<String>> excludeFieldMap = new LinkedHashMap<String, Set<String>>();
    private final Map<String, Set<String>> excludeStaticFieldMap = new LinkedHashMap<String, Set<String>>();
    private final Set<String> excludedThreads = new LinkedHashSet<String>();

    public Builder instanceField(String className, String fieldName) {
        Preconditions.checkNotNull(className, "className");
      Preconditions.checkNotNull(fieldName, "fieldName");
      Set<String> excludedFields = excludeFieldMap.get(className);
      if (excludedFields == null) {
        excludedFields = new LinkedHashSet<String>();
        excludeFieldMap.put(className, excludedFields);
      }
      excludedFields.add(fieldName);
      return this;
    }

    public Builder staticField(String className, String fieldName) {
        Preconditions.checkNotNull(className, "className");
        Preconditions.checkNotNull(fieldName, "fieldName");
      Set<String> excludedFields = excludeStaticFieldMap.get(className);
      if (excludedFields == null) {
        excludedFields = new LinkedHashSet<String>();
        excludeStaticFieldMap.put(className, excludedFields);
      }
      excludedFields.add(fieldName);
      return this;
    }

    public Builder thread(String threadName) {
        Preconditions.checkNotNull(threadName, "threadName");
      excludedThreads.add(threadName);
      return this;
    }

    public ExcludedRefs build() {
      return new ExcludedRefs(excludeFieldMap, excludeStaticFieldMap, excludedThreads);
    }
  }
}
```


参考源码的使用方法，如下 
排除staticField干扰 

![](http://img.blog.csdn.net/20160612212219104)

![](http://img.blog.csdn.net/20160612212336653)

![](http://img.blog.csdn.net/20160612212357090)


