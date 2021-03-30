# Activity是什么？

我们都知道android中有四大组件（Activity 活动，Service 服务，Content Provider 内容提供者，BroadcastReceiver 广播接收器），Activity是我们用的最多也是最基本的组件，因为应用的所有操作都与用户相关，Activity 提供窗口来和用户进行交互。

官方文档这么说：
　　

> An activity is a single, focused thing that the user can do. Almost all activities interact with the user, so the Activity class takes care of creating a window for you in which you can place your UI with setContentView(View). 
> 
> 
> 大概的意思：

>activity是独立平等的，用来处理用户操作。几乎所有的activity都是用来和用户交互的，所以activity类会创建了一个窗口，开发者可以通过setContentView(View)的接口把UI放到给窗口上。

　　Android中的activity全都归属于task管理 。task 是多个 activity 的集合，这些 activity 按照启动顺序排队存入一个栈（即“back stack”）。android默认会为每个App维持一个task来存放该app的所有activity，task的默认name为该app的packagename。

　　当然我们也可以在AndroidMainfest.xml中申明activity的taskAffinity属性来自定义task，但不建议使用，如果其他app也申明相同的task，它就有可能启动到你的activity，带来各种安全问题（比如拿到你的Intent）。

#Activity的内部调用过程

　　上面已经说了，系统通过堆栈来管理activity，当一个新的activity开始时，它被放置在堆栈的顶部和成为运行活动，以前的activity始终保持低于它在堆栈，而不会再次到达前台，直到新的活动退出。

　　还是上这张官网的activity_lifecycle图：
　　![这里写图片描述](http://img.blog.csdn.net/20160425171711054)　

 

 - 首先打开一个新的activity实例的时候，系统会依次调用

> onCreate（）  -> onStart() -> onResume() 然后开始running

 　　running的时候被覆盖了（从它打开了新的activity或是被锁屏，但是它**依然在前台**运行， lost focus but is still visible），系统调用onPause();

 
>　该方法执行activity暂停，通常用于提交未保存的更改到持久化数据，停止动画和其他的东西。但这个activity还是完全活着（它保持所有的状态和成员信息，并保持连接到**窗口管理器**）

接下来它有三条出路
①用户返回到该activity就调用onResume()方法重新running 

②用户回到桌面或是打开其他activity，就会调用onStop()进入停止状态（保留所有的状态和成员信息，**对用户不可见**）

③系统内存不足，拥有更高限权的应用需要内存，那么该activity的进程就可能会被系统回收。（回收onRause()和onStop()状态的activity进程）要想重新打开就必须重新创建一遍。

如果用户返回到onStop()状态的activity（又显示在前台了），系统会调用

> onRestart() ->  onStart() -> onResume() 然后重新running

在activity结束（调用finish ()）或是被系统杀死之前会调用onDestroy()方法释放所有占用的资源。


> activity生命周期中三个嵌套的循环

　

 - activity的完整生存期会在 onCreate() 调用和 onDestroy() 调用之间发生。　
 
 - activity的可见生存期会在 onStart() 调用和 onStop() 调用之间发生。系统会在activity的整个生存期内多次调用 onStart() 和onStop()， 因为activity可能会在显示和隐藏之间不断地来回切换。　
 
 - activity的前后台切换会在 onResume() 调用和 onPause() 之间发生。
 因为这个状态可能会经常发生转换，为了避免切换迟缓引起的用户等待，这两个方法中的代码应该相当地轻量化。

## activity被回收的状态和信息保存和恢复过程

```
public class MainActivity extends Activity {

	@Override
	protected void onCreate(Bundle savedInstanceState) {
		if(savedInstanceState!=null){ //判断是否有以前的保存状态信息
			 savedInstanceState.get("Key"); 
			 }
		super.onCreate(savedInstanceState);
		setContentView(R.layout.activity_main);
	}
   @Override
protected void onSaveInstanceState(Bundle outState) {
	// TODO Auto-generated method stub
	 //可能被回收内存前保存状态和信息，
	   Bundle data = new Bundle(); 
	   data.putString("key", "last words before be kill");
	   outState.putAll(data);
	super.onSaveInstanceState(outState);
}
   @Override
protected void onRestoreInstanceState(Bundle savedInstanceState) {
	// TODO Auto-generated method stub
	   if(savedInstanceState!=null){ //判断是否有以前的保存状态信息
			 savedInstanceState.get("Key"); 
			 }
	super.onRestoreInstanceState(savedInstanceState);
}
}
```

> onSaveInstanceState方法

　　在activity　可能被回收之前　调用,用来保存自己的状态和信息，以便回收后重建时恢复数据（在onCreate()或onRestoreInstanceState()中恢复）。旋转屏幕重建activity会调用该方法，但其他情况在onRause()和onStop()状态的activity不一定会调用 ，下面是该方法的文档说明。


> One example of when onPause and onStop is called and not this method is when a user navigates back from activity B to activity A: there is no need to call onSaveInstanceState on B because that particular instance will never be restored, so the system avoids calling it. An example when onPause is called and not onSaveInstanceState is when activity B is launched in front of activity A: the system may avoid calling onSaveInstanceState on activity A if it isn't killed during the lifetime of B since the state of the user interface of A will stay intact. 


也就是说，系统灵活的来决定调不调用该方法，**但是如果要调用就一定发生在onStop方法之前，但并不保证发生在onPause的前面还是后面。**

> onRestoreInstanceState方法

　　这个方法在onStart 和 onPostCreate之间调用，在onCreate中也可以状态恢复，但有时候需要所有布局初始化完成后再恢复状态。

　　onPostCreate：一般不实现这个方法，当程序的代码开始运行时，它调用系统做最后的初始化工作。

# 启动模式

## 启动模式什么？
　　
　　简单的说就是定义activity 实例与task 的关联方式。
　　
## 为什么要定义启动模式？ 

　　 为了实现一些默认启动（standard）模式之外的需求：
　　 

 - 让某个 activity 启动一个新的 task （而不是被放入当前 task ）
 
 - 让 activity 启动时只是调出已有的某个实例（而不是在 back stack 顶创建一个新的实例）　
 
 - 或者，你想在用户离开 task 时只保留根 activity，而 back stack 中的其它 activity 都要清空

##怎样定义启动模式？

　　定义启动模式的方法有两种：

### 使用 manifest 文件

　　在 manifest 文件中activity声明时，利用 activity 元素的 launchMode 属性来设定 activity 与 task 的关系。

　　

```
 <activity
            ．．．．．．
            android:launchMode="standard"
             >
           ．．．．．．．
        </activity>
```

> 注意： 你用 launchMode 属性为 activity 设置的模式可以被启动 activity 的 intent 标志所覆盖。

####有哪些启动模式？


 - "standard" （默认模式）　
 
　　当通过这种模式来启动Activity时,　Android总会为目标 Activity创建一个新的实例,并将该Activity添加到当前Task栈中。这种方式不会启动新的Task,只是将新的 Activity添加到原有的Task中。　
　　
 - "singleTop"　
 
 　　该模式和standard模式基本一致,但有一点不同:当将要被启动的Activity已经位于Task栈顶时,系统不会重新创建目标Activity实例,而是直接复用Task栈顶的Activity。
 
 - "singleTask"

　　Activity在同一个Task内只有一个实例。
　　
　　如果将要启动的Activity不存在,那么系统将会创建该实例,并将其加入Task栈顶；　

　　如果将要启动的Activity已存在,且存在栈顶,直接复用Task栈顶的Activity。　

　　如果Activity存在但是没有位于栈顶,那么此时系统会把位于该Activity上面的所有其他Activity全部移出Task,从而使得该目标Activity位于栈顶。

 - "singleInstance"　

　　无论从哪个Task中启动目标Activity,只会创建一个目标Activity实例且会用一个全新的Task栈来装载该Activity实例（全局单例）.

　　如果将要启动的Activity不存在,那么系统将会先创建一个全新的Task,再创建目标Activity实例并将该Activity实例放入此全新的Task中。

　　如果将要启动的Activity已存在,那么无论它位于哪个应用程序,哪个Task中;系统都会把该Activity所在的Task转到前台,从而使该Activity显示出来。

### 使用 Intent 标志

　　在要启动 activity 时，你可以在传给 startActivity() 的 intent 中包含相应标志，以修改 activity 与 task 的默认关系。

　　

```
　　　　　Intent i = new Intent(this,ＮewActivity.class);
		i.setFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
		startActivity(i);
```

#### 可以通过标志修改的默认模式有哪些？

　　　

 - FLAG_ACTIVITY_NEW_TASK

　　与"singleTask"模式相同，在新的 task 中启动 activity。如果要启动的 activity 已经运行于某 task 中，则那个 task 将调入前台。

 - FLAG_ACTIVITY_SINGLE_TOP

　　与 "singleTop"模式相同，如果要启动的 activity位于back stack 顶，系统不会重新创建目标Activity实例,而是直接复用Task栈顶的Activity。

 - FLAG_ACTIVITY_CLEAR_TOP

　　**此种模式在launchMode中没有对应的属性值。**如果要启动的 activity 已经在当前 task 中运行，则不再启动一个新的实例，且所有在其上面的 activity 将被销毁。

####　关于启动模式的一些建议

　　 一般不要改变 activity 和 task 默认的工作方式。 如果你确定有必要修改默认方式，请保持谨慎，并确保 activity 在启动和从其它 activity 返回时的可用性，多做测试和安全方面的工作。


# Intent Filter

　　android的3个核心组件——Activity、services、广播接收器——是通过intent传递消息的。intent消息用于在运行时绑定不同的组件。
　　在 Android 的 AndroidManifest.xml 配置文件中可以通过 intent-filter 节点为一个 Activity 指定其 Intent Filter，以便告诉系统该 Activity 可以响应什么类型的 Intent。

## intent-filter 的三大属性

### Action 

　　一个 Intent Filter 可以包含多个 Action，Action 列表用于标示 Activity 所能接受的“动作”，它是一个用户自定义的字符串。

　　

```
<intent-filter > 
 <action android:name="android.intent.action.MAIN" /> 
 <action android:name="com.scu.amazing7Action" /> 
……
 </intent-filter>
```

在代码中使用以下语句便可以启动该Intent 对象：

```
Intent i=new Intent(); 
i.setAction("com.scu.amazing7Action");
```
Action 列表中包含了“com.scu.amazing7Action”的 Activity 都将会匹配成功

### URL

　　在 intent-filter 节点中，通过 data节点匹配外部数据，也就是通过 URI 携带外部数据给目标组件。

　　

```
<data android:mimeType="mimeType" 
	android:scheme="scheme" 
	 android:host="host"
	 android:port="port" 
	 android:path="path"/>
```
注意：只有data的所有的属性都匹配成功时 URI 数据匹配才会成功

### Category 

　　为组件定义一个 类别列表，当 Intent 中包含这个类别列表的所有项目时才会匹配成功。

```
<intent-filter . . . >
   <action android:name="code android.intent.action.MAIN" />
   <category android:name="code　android.intent.category.LAUNCHER" />
</intent-filter>
```

## Activity 种 Intent Filter 的匹配过程

　　①加载所有的Intent Filter列表
　　②去掉action匹配失败的Intent Filter
　　③去掉url匹配失败的Intent Filter
　　④去掉Category匹配失败的Intent Filter
　　⑤判断剩下的Intent Filter数目是否为0。如果为0查找失败返回异常；如果大于0，就按优先级排序，返回最高优先级的Intent Filter



# 开发中Activity的一些问题

 - 

 一般设置Activity为非公开的

　　

```
<activity  
．．．．．． 
android:exported="false" /> 
```

注意：非公开的Activity不能设置intent-filter，以免被其他activity唤醒（如果拥有相同的intent-filter）。

 - 不要指定activity的taskAffinity属性

 - 不要设置activity的LaunchMode（保持默认）

　　注意Activity的intent最好也不要设定为FLAG_ACTIVITY_NEW_TASK

 - 在匿名内部类中使用this时加上activity类名（类名.this,不一定是当前activity）

 - 设置activity全屏
 
  　　在其 onCreate()方法中加入：

```
// 设置全屏模式
 getWindow().setFlags(WindowManager.LayoutParams.FLAG_FULLSCREEN, WindowManager.LayoutParams.FLAG_FULLSCREEN); 
 // 去除标题栏
 requestWindowFeature(Window.FEATURE_NO_TITLE);
```

　　
# 什么是服务？　　

　　Service是一个应用程序组件，它能够在后台执行一些耗时较长的操作，并且不提供用户界面。服务能被其它应用程序的组件启动，即使用户切换到另外的应用时还能保持后台运行。此外，应用程序组件还能与服务绑定，并与服务进行交互，甚至能进行进程间通信（IPC）。 比如，服务可以处理网络传输、音乐播放、执行文件I/O、或者与content provider进行交互，所有这些都是后台进行的。


# Service 与 Thread 的区别


　　服务仅仅是一个组件，即使用户不再与你的应用程序发生交互，它仍然能在后台运行。因此，应该只在需要时才创建一个服务。

　　如果你需要在主线程之外执行一些工作，但仅当用户与你的应用程序交互时才会用到，那你应该创建一个新的线程而不是创建服务。 比如，如果你需要播放一些音乐，但只是当你的activity在运行时才需要播放，你可以在onCreate()中创建一个线程，在onStart()中开始运行，然后在onStop()中终止运行。还可以考虑使用AsyncTask或HandlerThread来取代传统的Thread类。

　　**由于无法在不同的 Activity 中对同一 Thread 进行控制**，这个时候就要考虑用服务实现。如果你使用了服务，它默认就运行于应用程序的主线程中。因此，如果服务执行密集计算或者阻塞操作，你仍然应该在服务中创建一个新的线程来完成（避免ANR）。

# 服务的分类

## 按运行分类

　　

 - 前台服务

　　前台服务是指那些经常会被用户关注的服务，因此内存过低时它不会成为被杀的对象。 前台服务必须提供一个状态栏通知，并会置于“正在进行的”（“Ongoing”）组之下。这意味着只有在服务被终止或从前台移除之后，此通知才能被解除。
　　例如，用服务来播放音乐的播放器就应该运行在前台，因为用户会清楚地知晓它的运行情况。 状态栏通知可能会标明当前播放的歌曲，并允许用户启动一个activity来与播放器进行交互。

　　要把你的服务请求为前台运行，可以调用startForeground()方法。此方法有两个参数：唯一标识通知的整数值、状态栏通知Notification对象。例如：

 　

```
Notification notification = new Notification(R.drawable.icon, getText(R.string.ticker_text),System.currentTimeMillis());

Intent notificationIntent = new Intent(this,ExampleActivity.class);

PendingIntent pendingIntent = PendingIntent.getActivity(this, 0, notificationIntent, 0);

notification.setLatestEventInfo(this, getText(R.string.notification_title),
        getText(R.string.notification_message), pendingIntent);
        
startForeground(ONGOING_NOTIFICATION, notification);
```

　　要从前台移除服务，请调用stopForeground()方法，这个方法接受个布尔参数，表示是否同时移除状态栏通知。此方法不会终止服务。不过，如果服务在前台运行时被你终止了，那么通知也会同时被移除。

 - 后台服务

## 按使用分类　　

 - 本地服务

　　用于应用程序内部，实现一些耗时任务，并不占用应用程序比如Activity所属线程，而是单开线程后台执行。
　　调用Context.startService()启动，调用Context.stopService()结束。在内部可以调用Service.stopSelf() 或 Service.stopSelfResult()来自己停止。

 - 远程服务

　　用于Android系统内部的应用程序之间，可被其他应用程序复用，比如天气预报服务，其他应用程序不需要再写这样的服务，调用已有的即可。可以定义接口并把接口暴露出来，以便其他应用进行操作。客户端建立到服务对象的连接，并通过那个连接来调用服务。调用Context.bindService()方法建立连接，并启动，以调用 Context.unbindService()关闭连接。多个客户端可以绑定至同一个服务。如果服务此时还没有加载，bindService()会先加载它。

# Service生命周期

　　![这里写图片描述](http://img.blog.csdn.net/20160503165018575)

Service生命周期方法：
```
public class ExampleService extends Service {
    int mStartMode;       // 标识服务被杀死后的处理方式
    IBinder mBinder;      // 用于客户端绑定的接口
    boolean mAllowRebind; // 标识是否使用onRebind

    @Override
    public void onCreate() {
        // 服务正被创建
    }
    @Override
    public int onStartCommand(Intent intent, int flags, int startId) {
        // 服务正在启动，由startService()调用引发
        return mStartMode;
    }
    @Override
    public IBinder onBind(Intent intent) {
        // 客户端用bindService()绑定服务
        return mBinder;
    }
    @Override
    public boolean onUnbind(Intent intent) {
        // 所有的客户端都用unbindService()解除了绑定
        return mAllowRebind;
    }
    @Override
    public void onRebind(Intent intent) {
        // 某客户端正用bindService()绑定到服务,
        // 而onUnbind()已经被调用过了
    }
    @Override
    public void onDestroy() {
        // 服务用不上了，将被销毁
    }
}
```

> 请注意onStartCommand()方法必须返回一个整数。这个整数是描述系统在杀死服务之后应该如何继续运行。onStartCommand()的返回值必须是以下常量之一：

> START_NOT_STICKY 　
如果系统在onStartCommand()返回后杀死了服务，则不会重建服务了，除非还存在未发送的intent。 当服务不再是必需的，并且应用程序能够简单地重启那些未完成的工作时，这是避免服务运行的最安全的选项。　
　
> START_STICKY 
如果系统在onStartCommand()返回后杀死了服务，则将重建服务并调用onStartCommand()，但不会再次送入上一个intent， 而是用null intent来调用onStartCommand() 。除非还有启动服务的intent未发送完，那么这些剩下的intent会继续发送。 这适用于媒体播放器（或类似服务），它们不执行命令，但需要一直运行并随时待命。　

> START_REDELIVER_INTENT 
如果系统在onStartCommand()返回后杀死了服务，则将重建服务并用上一个已送过的intent调用onStartCommand()。任何未发送完的intent也都会依次送入。这适用于那些需要立即恢复工作的活跃服务，比如下载文件。

　　服务的生命周期与activity的非常类似。不过，更重要的是你需密切关注服务的创建和销毁环节，因为后台运行的服务是不会引起用户注意的。

　　服务的生命周期——从创建到销毁——可以有两种路径：

 - 一个started服务

　　这类服务由其它组件调用startService()来创建。然后保持运行，且必须通过调用stopSelf()自行终止。其它组件也可通过调用stopService() 终止这类服务。服务终止后，系统会把它销毁。

　　如果一个Service被startService 方法多次启动，那么onCreate方法只会调用一次，onStart将会被调用多次（对应调用startService的次数），并且系统只会创建Service的一个实例（因此你应该知道只需要一次stopService调用）。该Service将会一直在后台运行，而不管对应程序的Activity是否在运行，直到被调用stopService，或自身的stopSelf方法。当然如果系统资源不足，android系统也可能结束服务。

　　

 - 　一个bound服务
 
　　服务由其它组件（客户端）调用bindService()来创建。然后客户端通过一个IBinder接口与服务进行通信。客户端可以通过调用unbindService()来关闭联接。多个客户端可以绑定到同一个服务上，当所有的客户端都解除绑定后，系统会销毁服务。（服务不需要自行终止。）

　　如果一个Service被某个Activity 调用 Context.bindService 方法绑定启动，不管调用 bindService 调用几次，onCreate方法都只会调用一次，同时onStart方法始终不会被调用。当连接建立之后，Service将会一直运行，除非调用Context.unbindService 断开连接或者之前调用bindService 的 Context 不存在了（如Activity被finish的时候），系统将会自动停止Service，对应onDestroy将被调用。

![这里写图片描述](http://img.blog.csdn.net/20160503165622881)　

　　这两条路径并不是完全隔离的。也就是说，你可以绑定到一个已经用startService()启动的服务上。例如，一个后台音乐服务可以通过调用startService()来启动，传入一个指明所需播放音乐的 Intent。 之后，用户也许需要用播放器进行一些控制，或者需要查看当前歌曲的信息，这时一个activity可以通过调用bindService()与此服务绑定。在类似这种情况下，stopService()或stopSelf()不会真的终止服务，除非所有的客户端都解除了绑定。

> 　　当在旋转手机屏幕的时候，当手机屏幕在“横”“竖”变换时，此时如果你的 Activity 如果会自动旋转的话，旋转其实是 Activity 的重新创建，因此旋转之前的使用 bindService 建立的连接便会断开（Context 不存在了）。

# 在manifest中声明服务

　　无论是什么类型的服务都必须在manifest中申明，格式如下：

```
<manifest ... >
  ...
  <application ... >
      <service android:name=".ExampleService" />
      ...
  </application>
</manifest>
```

Service 元素的属性有：

> android:name　　-------------　　服务类名

>android:label　　--------------　　服务的名字，如果此项不设置，那么默认显示的服务名则为类名

>android:icon　　--------------　　服务的图标

>android:permission　　-------　　申明此服务的权限，这意味着只有提供了该权限的应用才能控制或连接此服务

>android:process　　----------　　表示该服务是否运行在另外一个进程，如果设置了此项，那么将会在包名后面加上这段字符串表示另一进程的名字

>android:enabled　　----------　　如果此项设置为 true，那么 Service 将会默认被系统启动，不设置默认此项为 false

>android:exported　　---------　　表示该服务是否能够被其他应用程序所控制或连接，不设置默认此项为 false　

　　android:name是唯一必需的属性——它定义了服务的类名。与activity一样，服务可以定义intent过滤器，使得其它组件能用隐式intent来调用服务。如果你想让服务只能内部使用（其它应用程序无法调用），那么就不必（也不应该）提供任何intent过滤器。
　　此外，如果包含了android:exported属性并且设置为"false"， 就可以确保该服务是你应用程序的私有服务。即使服务提供了intent过滤器，本属性依然生效。　

# startService 启动服务

　　从activity或其它应用程序组件中可以启动一个服务，调用startService()并传入一个Intent（指定所需启动的服务）即可。

```
	Intent intent = new Intent(this, MyService.class);
	startService(intent);
```

服务类：

```
public class MyService extends Service {

	  /**
     * onBind 是 Service 的虚方法，因此我们不得不实现它。
     * 返回 null，表示客服端不能建立到此服务的连接。
     */
	@Override
	public IBinder onBind(Intent intent) {
		// TODO Auto-generated method stub
		return null;
	}
    
	@Override
    public void onCreate() {
        super.onCreate();
    }
     
    @Override
 public int onStartCommand(Intent intent, int flags, int startId)  　　 {  	
    //接受传递过来的intent的数据 
     return START_STICKY; 
    };
     
    @Override
    public void onDestroy() {
        super.onDestroy();
    }
	
}
```

　　一个started服务必须自行管理生命周期。也就是说，系统不会终止或销毁这类服务，除非必须恢复系统内存并且服务返回后一直维持运行。 因此，服务必须通过调用stopSelf()自行终止，或者其它组件可通过调用stopService()来终止它。

# bindService 启动服务　　

　　当应用程序中的activity或其它组件需要与服务进行交互，或者应用程序的某些功能需要暴露给其它应用程序时，你应该创建一个bound服务，并通过进程间通信（IPC）来完成。

方法如下：

```
 Intent intent=new Intent(this,BindService.class); 
 bindService(intent, ServiceConnection conn, int flags)  

```

> 注意bindService是Context中的方法，当没有Context时传入即可。

在进行服务绑定的时，其flags有：

 - Context.BIND_AUTO_CREATE    
 
　　表示收到绑定请求的时候，如果服务尚未创建，则即刻创建，在系统内存不足需要先摧毁优先级组件来释放内存，且只有驻留该服务的进程成为被摧毁对象时，服务才被摧毁　

 - Context.BIND_DEBUG_UNBIND    　
 
　　通常用于调试场景中判断绑定的服务是否正确，但容易引起内存泄漏，因此非调试目的的时候不建议使用

 - Context.BIND_NOT_FOREGROUND    　
 
　　表示系统将阻止驻留该服务的进程具有前台优先级，仅在后台运行。

服务类：

```
public class BindService extends Service {

	 // 实例化MyBinder得到mybinder对象；
	private final MyBinder binder = new MyBinder();
	
	  /**
     * 返回Binder对象。
     */
	@Override
	public IBinder onBind(Intent intent) {
		// TODO Auto-generated method stub
		return binder;
	}
    
     /**
      * 新建内部类MyBinder，继承自Binder(Binder实现IBinder接口),
      * MyBinder提供方法返回BindService实例。
      */
　　public class MyBinder extends Binder{
        
        public BindService getService(){
            return BindService.this;
        }
    }
     

	@Override
	public boolean onUnbind(Intent intent) {
		// TODO Auto-generated method stub
		return super.onUnbind(intent);
	}
}
```

启动服务的activity代码：

```
public class MainActivity extends Activity {

	/** 是否绑定 */  
	boolean mIsBound = false; 
	BindService mBoundService;
	@Override
	protected void onCreate(Bundle savedInstanceState) {
		super.onCreate(savedInstanceState);
		setContentView(R.layout.activity_main);
		doBindService();
	}
	/**
	 * 实例化ServiceConnection接口的实现类,用于监听服务的状态
	 */
	private ServiceConnection conn = new ServiceConnection() {  
		  
	    @Override  
	    public void onServiceConnected(ComponentName name, IBinder service) {  
	        BindService mBoundService = ((BindService.MyBinder) service).getService();  
	        
	    }  
	  
	    @Override  
	    public void onServiceDisconnected(ComponentName name) {  
	        mBoundService = null;  
	     
	    }  
	}; 
	
	/** 绑定服务 */  
	public void doBindService() {  
	    bindService(new Intent(MainActivity.this, BindService.class), conn,Context.BIND_AUTO_CREATE);  
	    mIsBound = true;  
	}  
	
	/** 解除绑定服务 */  
	public void doUnbindService() {  
	    if (mIsBound) {  
	        // Detach our existing connection.  
	        unbindService(conn);  
	        mIsBound = false;  
	    }  
	} 
	
	@Override
	protected void onDestroy() {
		// TODO Auto-generated method stub
		super.onDestroy();
		
		doUnbindService();
	}
}
```

> 注意在AndroidMainfest.xml中对Service进行显式声明


判断Service是否正在运行：

```
private boolean isServiceRunning() {
    ActivityManager manager = (ActivityManager) getSystemService(ACTIVITY_SERVICE);
    
 　{
        if ("com.example.demo.BindService".equals(service.service.getClassName())) 　　{
            return true;
        }
    }
    return false;
}
```


# IntentService定义

　　IntentService继承与Service，用来处理异步请求。客户端可以通过startService(Intent)方法传递请求给IntentService。IntentService在onCreate()函数中通过HandlerThread单独开启一个线程来依次处理所有Intent请求对象所对应的任务。　
　　
　　这样以免事务处理阻塞主线程（ＡＮＲ）。执行完所一个Intent请求对象所对应的工作之后，如果没有新的Intent请求达到，则**自动停止**Service；否则执行下一个Intent请求所对应的任务。　
　　
　　IntentService在处理事务时，还是采用的Handler方式，创建一个名叫ServiceHandler的内部Handler，并把它直接绑定到HandlerThread所对应的子线程。 ServiceHandler把处理一个intent所对应的事务都封装到叫做**onHandleIntent**的虚函数；因此我们直接实现虚函数onHandleIntent，再在里面根据Intent的不同进行不同的事务处理就可以了。
另外，IntentService默认实现了Onbind（）方法，返回值为null。

使用IntentService需要实现的两个方法：
　　

 - 构造函数　

　　IntentService的构造函数一定是**参数为空**的构造函数，然后再在其中调用super("name")这种形式的构造函数。因为Service的实例化是系统来完成的，而且系统是用参数为空的构造函数来实例化Service的
 
 - 实现虚函数onHandleIntent

　　在里面根据Intent的不同进行不同的事务处理。　
　　
好处：处理异步请求的时候可以减少写代码的工作量，比较轻松地实现项目的需求。

# IntentService与Service的区别

　　Service不是独立的进程，也不是独立的线程，它是依赖于应用程序的主线程的，不建议在Service中编写耗时的逻辑和操作，否则会引起ANR。

　　IntentService 它创建了一个独立的工作线程来处理所有的通过onStartCommand()传递给服务的intents（把intent插入到工作队列中）。通过工作队列把intent逐个发送给onHandleIntent()。　
　　
　　不需要主动调用stopSelft()来结束服务。因为，在所有的intent被处理完后，系统会自动关闭服务。

　　 默认实现的onBind()返回null。

# IntentService实例介绍

　　首先是myIntentService.java

```
public class myIntentService extends IntentService {

	//------------------必须实现-----------------------------
	
	public myIntentService() {
		super("myIntentService");
		// 注意构造函数参数为空，这个字符串就是worker thread的名字
	}

	@Override
	protected void onHandleIntent(Intent intent) {
		//根据Intent的不同进行不同的事务处理 
        String taskName = intent.getExtras().getString("taskName");  
        switch (taskName) {
		case "task1":
			Log.i("myIntentService", "do task1");
			break;
		case "task2":
			Log.i("myIntentService", "do task2");
			break;
		default:
			break;
		}		
	}
  //--------------------用于打印生命周期--------------------	
   @Override
  public void onCreate() {
		Log.i("myIntentService", "onCreate");
	super.onCreate();
}
	
	@Override
	public int onStartCommand(Intent intent, int flags, int startId) {
		Log.i("myIntentService", "onStartCommand");
		return super.onStartCommand(intent, flags, startId);
	}
	
	@Override
	public void onDestroy() {
		Log.i("myIntentService", "onDestroy");
		super.onDestroy();
	}
}

```
然后记得在Manifest.xml中注册服务

```
 <service android:name=".myIntentService">
            <intent-filter >  
                <action android:name="cn.scu.finch"/>  
            </intent-filter>     
        </service>
```

最后在Activity中开启服务

```
public class MainActivity extends Activity {
	@Override
	protected void onCreate(Bundle savedInstanceState) {
		// TODO Auto-generated method stub
		super.onCreate(savedInstanceState);
		
		//同一服务只会开启一个worker thread，在onHandleIntent函数里依次处理intent请求。
		
        Intent i = new Intent("cn.scu.finch");  
        Bundle bundle = new Bundle();  
        bundle.putString("taskName", "task1");  
        i.putExtras(bundle);  
        startService(i);  
           
        Intent i2 = new Intent("cn.scu.finch");  
        Bundle bundle2 = new Bundle();  
        bundle2.putString("taskName", "task2");  
        i2.putExtras(bundle2);  
        startService(i2); 
        
        startService(i);  //多次启动
	}
}
```

运行结果：

![这里写图片描述](http://img.blog.csdn.net/20160513135411037)　

　　IntentService在onCreate()函数中通过HandlerThread单独开启一个线程来依次处理所有Intent请求对象所对应的任务。　
　　
　　通过onStartCommand()传递给服务intent被**依次**插入到工作队列中。工作队列又把intent逐个发送给onHandleIntent()。

　　

> 注意：
> 它只有一个工作线程，名字就是构造函数的那个字符串，也就是“myIntentService”，我们知道多次开启service，只会调用一次onCreate方法（创建一个工作线程），多次onStartCommand方法（用于传入intent通过工作队列再发给onHandleIntent函数做处理）。

# 1.ContentProvider是什么？

　　ContentProvider（内容提供者）是Android的四大组件之一，管理android以结构化方式存放的数据，以相对安全的方式封装数据（表）并且提供简易的处理机制和统一的访问接口供**其他程序**调用。　
　　
　　Android的数据存储方式总共有五种，分别是：Shared Preferences、网络存储、文件存储、外储存储、SQLite。但一般这些存储都只是在单独的一个应用程序之中达到一个数据的共享，有时候我们需要操作其他应用程序的一些数据，就会用到ContentProvider。而且Android为常见的一些数据提供了默认的ContentProvider（包括音频、视频、图片和通讯录等）。

　　但注意ContentProvider它也只是一个中间人，真正操作的数据源可能是数据库，也可以是文件、xml或网络等其他存储方式。

# ２.URL

 　　URL（统一资源标识符）代表要操作的数据，可以用来标识每个ContentProvider，这样你就可以通过指定的URI找到想要的ContentProvider,从中获取或修改数据。
 　　在Android中URI的格式如下图所示：

![这里写图片描述](http://img.blog.csdn.net/20160505154322045)　


　　

 - Ａ
　
　　　schema，已经由Android所规定为：content://．　
　　　
 - Ｂ

　　　主机名（Authority），是URI的授权部分，是唯一标识符，用来定位ContentProvider。

> Ｃ部分和D部分：是每个ContentProvider内部的路径部分

 - Ｃ

　　　指向一个对象集合，一般用表的名字，如果没有指定D部分，则返回全部记录。

 - Ｄ

　　　指向特定的记录，这里表示操作user表id为7的记录。如果要操作user表中id为7的记录的name字段， D部分变为       **/7/name**即可。


> URI模式匹配通配符
> 
> *：匹配的任意长度的任何有效字符的字符串。 
> 
> ＃：匹配的任意长度的数字字符的字符串。 
> 
> 如：
>  
>  content://com.example.app.provider/*
>  匹配provider的任何内容url 
>  
>  content://com.example.app.provider/table3/# 
>  匹配table3的所有行

　　
## 2.１MIME

 　　MIME是指定某个扩展名的文件用一种应用程序来打开，就像你用浏览器查看PDF格式的文件，浏览器会选择合适的应用来打开一样。Android中的工作方式跟HTTP类似，ContentProvider会根据URI来返回MIME类型，ContentProvider会返回一个包含两部分的字符串。MIME类型一般包含两部分，如：

　　

> text/html
text/css
text/xml
application/pdf

　　分为类型和子类型，Android遵循类似的约定来定义MIME类型，每个内容类型的Android MIME类型有两种形式：多条记录（集合）和单条记录。

　　集合记录：

```
vnd.android.cursor.dir/自定义
```

　　单条记录：

```
vnd.android.cursor.item/自定义
```

　　vnd表示这些类型和子类型具有非标准的、供应商特定的形式。Android中类型已经固定好了，不能更改，只能区别是集合还是单条具体记录，子类型可以按照格式自己填写。
　　在使用Intent时，会用到MIME，根据Mimetype打开符合条件的活动。

　　下面分别介绍Android系统提供了两个用于操作Uri的工具类：ContentUris和UriMatcher。
## 2.２ ContentUris

 　　ContetnUris包含一个便利的函数withAppendedId()来向URI追加一个id。

　

```
Uri uri = Uri.parse("content://cn.scu.myprovider/user")
Uri resultUri = ContentUris.withAppendedId(uri, 7); 

//生成后的Uri为：content://cn.scu.myprovider/user/7
```

　　同时提供parseId(uri)方法用于从URL中获取ID:

```
Uri uri = Uri.parse("content://cn.scu.myprovider/user/7")
long personid = ContentUris.parseId(uri);
//获取的结果为:7
```

## 2.３UriMatcher

　　UriMatcher本质上是一个文本过滤器，用在contentProvider中帮助我们过滤，分辨出查询者想要查询哪个数据表。

　　举例说明：

 - 第一步，初始化：

```
UriMatcher matcher = new UriMatcher(UriMatcher.NO_MATCH);
//常量UriMatcher.NO_MATCH表示不匹配任何路径的返回码
```

 - 第二步，注册需要的Uri：

```
//USER 和 USER_ID是两个int型数据
matcher.addURI("cn.scu.myprovider", "user", USER);
matcher.addURI("cn.scu.myprovider", "user/#",USER_ID);
//如果match()方法匹配content://cn.scu.myprovider/user路径，返回匹配码为USER
```

 - 第三部，与已经注册的Uri进行匹配:

```
/* 
     * 如果操作集合，则必须以vnd.android.cursor.dir开头 
     * 如果操作非集合，则必须以vnd.android.cursor.item开头 
     * */  
    @Override  
    public String getType(Uri uri) {  
    Uri uri = Uri.parse("content://" + "cn.scu.myprovider" + "/user");  
        switch(matcher.match(uri)){  
        case USER:  
            return "vnd.android.cursor.dir/user";  
        case USER_ID:  
            return "vnd.android.cursor.item/user";  
        }  
    } 
``` 

# 3.ContentProvider的主要方法

　　　

> public boolean onCreate()

　　ContentProvider创建后　或　打开系统后其它应用第一次访问该ContentProvider时调用。

> public Uri insert(Uri uri, ContentValues values)

　　外部应用向ContentProvider中添加数据。

> public int delete(Uri uri, String selection, String[] selectionArgs)

　　外部应用从ContentProvider删除数据。

> public int update(Uri uri, ContentValues values, String selection, String[] selectionArgs)：

　　外部应用更新ContentProvider中的数据。

> public Cursor query(Uri uri, String[] projection, String selection, String[] selectionArgs, String sortOrder)　

　　供外部应用从ContentProvider中获取数据。
　

> public String getType(Uri uri)

　　该方法用于返回当前Url所代表数据的MIME类型。

# ４.ContentResolver

　　ContentResolver通过URI来查询ContentProvider中提供的数据。除了URI以 外，还必须知道需要获取的数据段的名称，以及此数据段的数据类型。如果你需要获取一个特定的记录，你就必须知道当前记录的ID，也就是URI中D部分。

　　ContentResolver 类提供了与ContentProvider类相同签名的四个方法：

　　

> public Uri insert(Uri uri, ContentValues values)　//添加

> public int delete(Uri uri, String selection, String[] selectionArgs)　//删除
> 
> public int update(Uri uri, ContentValues values, String selection, String[] selectionArgs)　//更新
> 
> public Cursor query(Uri uri, String[] projection, String selection, String[] selectionArgs, String sortOrder)//获取

实例代码：

```
ContentResolver resolver =  getContentResolver();
Uri uri = Uri.parse("content://cn.scu.myprovider/user");

//添加一条记录
ContentValues values = new ContentValues();
values.put("name", "fanrunqi");
values.put("age", 24);
resolver.insert(uri, values);  

//获取user表中所有记录
Cursor cursor = resolver.query(uri, null, null, null, "userid desc");
while(cursor.moveToNext()){
   //操作
}

//把id为1的记录的name字段值更改新为finch
ContentValues updateValues = new ContentValues();
updateValues.put("name", "finch");
Uri updateIdUri = ContentUris.withAppendedId(uri, 1);
resolver.update(updateIdUri, updateValues, null, null);

//删除id为2的记录
Uri deleteIdUri = ContentUris.withAppendedId(uri, 2);
resolver.delete(deleteIdUri, null, null);
```

# 5.ContentObserver

　　　 ContentObserver(内容观察者)，目的是观察特定Uri引起的数据库的变化，继而做一些相应的处理，它类似于数据库技术中的触发器(Trigger)，当ContentObserver所观察的Uri发生变化时，便会触发它.

    下面是使用内容观察者监听短信的例子：

```
public class MainActivity extends Activity {
 
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
         
//注册观察者Observser    
this.getContentResolver().registerContentObserver(Uri.parse("content://sms"),true,new SMSObserver(new Handler()));
 
    }
 
    private final class SMSObserver extends ContentObserver {
 
        public SMSObserver(Handler handler) {
            super(handler);
 
        }
 
     
        @Override
        public void onChange(boolean selfChange) {
 
 Cursor cursor = MainActivity.this.getContentResolver().query(
Uri.parse("content://sms/inbox"), null, null, null, null);
 
            while (cursor.moveToNext()) {
                StringBuilder sb = new StringBuilder();
 
                sb.append("address=").append(
                        cursor.getString(cursor.getColumnIndex("address")));
 
                sb.append(";subject=").append(
                        cursor.getString(cursor.getColumnIndex("subject")));
 
                sb.append(";body=").append(
                        cursor.getString(cursor.getColumnIndex("body")));
 
                sb.append(";time=").append(
                        cursor.getLong(cursor.getColumnIndex("date")));
 
                System.out.println("--------has Receivered SMS::" + sb.toString());
 
                 
            }
 
        }
 
    }
}
```
  同时可以在ContentProvider发生数据变化时调用
getContentResolver().notifyChange(uri, null)来通知注册在此URI上的访问者。

 　　

```
public class UserContentProvider extends ContentProvider {
   public Uri insert(Uri uri, ContentValues values) {
      db.insert("user", "userid", values);
      getContext().getContentResolver().notifyChange(uri, null);
   }
}
```
# 6.实例说明

 　　数据源是SQLite, 用ContentResolver操作ContentProvider。

![这里写图片描述](http://img.blog.csdn.net/20160505195143099)

Constant.java（储存一些常量）

```
public class Constant {  
      
    public static final String TABLE_NAME = "user";  
      
    public static final String COLUMN_ID = "_id";  
    public static final String COLUMN_NAME = "name";  
       
       
    public static final String AUTOHORITY = "cn.scu.myprovider";  
    public static final int ITEM = 1;  
    public static final int ITEM_ID = 2;  
       
    public static final String CONTENT_TYPE = "vnd.android.cursor.dir/user";  
    public static final String CONTENT_ITEM_TYPE = "vnd.android.cursor.item/user";  
       
    public static final Uri CONTENT_URI = Uri.parse("content://" + AUTOHORITY + "/user");  
}  
```

  DBHelper.java(操作数据库)

```
public class DBHelper extends SQLiteOpenHelper {  
  
    private static final String DATABASE_NAME = "finch.db";    
    private static final int DATABASE_VERSION = 1;    
  
    public DBHelper(Context context) {  
        super(context, DATABASE_NAME, null, DATABASE_VERSION);  
    }  
  
    @Override  
    public void onCreate(SQLiteDatabase db)  throws SQLException {  
        //创建表格  
        db.execSQL("CREATE TABLE IF NOT EXISTS "+ Constant.TABLE_NAME + "("+ Constant.COLUMN_ID +" INTEGER PRIMARY KEY AUTOINCREMENT," + Constant.COLUMN_NAME +" VARCHAR NOT NULL);");  
    }  
  
    @Override  
    public void onUpgrade(SQLiteDatabase db, int oldVersion, int newVersion)  throws SQLException {  
        //删除并创建表格  
        db.execSQL("DROP TABLE IF EXISTS "+ Constant.TABLE_NAME+";");  
        onCreate(db);  
    }  
}  

```

　MyProvider.java(自定义的ContentProvider)　

```
public class MyProvider extends ContentProvider {    
    
    DBHelper mDbHelper = null;    
    SQLiteDatabase db = null;    
    
    private static final UriMatcher mMatcher;    
    static{    
        mMatcher = new UriMatcher(UriMatcher.NO_MATCH);    
        mMatcher.addURI(Constant.AUTOHORITY,Constant.TABLE_NAME, Constant.ITEM);    
        mMatcher.addURI(Constant.AUTOHORITY, Constant.TABLE_NAME+"/#", Constant.ITEM_ID);    
    }    
    
  
    @Override    
    public String getType(Uri uri) {    
        switch (mMatcher.match(uri)) {    
        case Constant.ITEM:    
            return Constant.CONTENT_TYPE;    
        case Constant.ITEM_ID:    
            return Constant.CONTENT_ITEM_TYPE;    
        default:    
            throw new IllegalArgumentException("Unknown URI"+uri);    
        }    
    }    
    
    @Override    
    public Uri insert(Uri uri, ContentValues values) {    
        // TODO Auto-generated method stub    
        long rowId;    
        if(mMatcher.match(uri)!=Constant.ITEM){    
            throw new IllegalArgumentException("Unknown URI"+uri);    
        }    
        rowId = db.insert(Constant.TABLE_NAME,null,values);    
        if(rowId>0){    
            Uri noteUri=ContentUris.withAppendedId(Constant.CONTENT_URI, rowId);    
            getContext().getContentResolver().notifyChange(noteUri, null);    
            return noteUri;    
        }    
    
        throw new SQLException("Failed to insert row into " + uri);    
    }    
    
    @Override    
    public boolean onCreate() {    
        // TODO Auto-generated method stub    
        mDbHelper = new DBHelper(getContext());    
    
        db = mDbHelper.getReadableDatabase();    
    
        return true;    
    }    
    
    @Override    
    public Cursor query(Uri uri, String[] projection, String selection,    
            String[] selectionArgs, String sortOrder) {    
        // TODO Auto-generated method stub    
        Cursor c = null;    
        switch (mMatcher.match(uri)) {    
        case Constant.ITEM:    
            c =  db.query(Constant.TABLE_NAME, projection, selection, selectionArgs, null, null, sortOrder);    
            break;    
        case Constant.ITEM_ID:    
            c = db.query(Constant.TABLE_NAME, projection,Constant.COLUMN_ID + "="+uri.getLastPathSegment(), selectionArgs, null, null, sortOrder);    
            break;    
        default:    
            throw new IllegalArgumentException("Unknown URI"+uri);    
        }    
    
        c.setNotificationUri(getContext().getContentResolver(), uri);    
        return c;    
    }    
    
    @Override    
    public int update(Uri uri, ContentValues values, String selection,    
            String[] selectionArgs) {    
        // TODO Auto-generated method stub    
        return 0;    
    }

	@Override
	public int delete(Uri uri, String selection, String[] selectionArgs) {
		// TODO Auto-generated method stub
		return 0;
	}    
    
}    
```
MainActivity.java(ContentResolver操作)

```
public class MainActivity extends Activity {
    private ContentResolver mContentResolver = null; 
    private Cursor cursor = null;  
         @Override
        protected void onCreate(Bundle savedInstanceState) {
        	// TODO Auto-generated method stub
        	super.onCreate(savedInstanceState);
        	setContentView(R.layout.activity_main);
        	
        	   TextView tv = (TextView) findViewById(R.id.tv);
				
        		mContentResolver = getContentResolver();  
        		tv.setText("添加初始数据 ");
                for (int i = 0; i < 10; i++) {  
                    ContentValues values = new ContentValues();  
                    values.put(Constant.COLUMN_NAME, "fanrunqi"+i);  
                    mContentResolver.insert(Constant.CONTENT_URI, values);  
                } 
                
            	tv.setText("查询数据 ");
                cursor = mContentResolver.query(Constant.CONTENT_URI, new String[]{Constant.COLUMN_ID,Constant.COLUMN_NAME}, null, null, null);  
                if (cursor.moveToFirst()) {
                	String s = cursor.getString(cursor.getColumnIndex(Constant.COLUMN_NAME));
                	tv.setText("第一个数据： "+s);
                }
        }
         
}  

```

最后在manifest申明

```
<provider android:name="MyProvider" android:authorities="cn.scu.myprovider" />
```

[本文中代码下载](http://download.csdn.net/detail/amazing7/9511234)

　

# BroadcastReceiver的定义

　　广播是一种广泛运用的在应用程序之间传输信息的机制，主要用来监听系统或者应用发出的广播信息，然后根据广播信息作为相应的逻辑处理，也可以用来传输少量、频率低的数据。

　　在实现开机启动服务和网络状态改变、电量变化、短信和来电时通过接收系统的广播让应用程序作出相应的处理。

　　BroadcastReceiver 自身并不实现图形用户界面，但是当它收到某个通知后， BroadcastReceiver 可以通过启动 Service 、启动 Activity 或是 NotificationMananger 提醒用户。

# BroadcastReceiver使用注意

　　当系统或应用发出广播时，将会扫描系统中的所有广播接收者，通过action匹配将广播发送给相应的接收者，接收者收到广播后将会产生一个广播接收者的实例，执行其中的onReceiver()这个方法；特别需要注意的是这个实例的生命周期只有10秒，如果10秒内没执行结束onReceiver()，系统将会报错。　
　　
　　在onReceiver()执行完毕之后，该实例将会被销毁，所以不要在onReceiver()中执行耗时操作，也不要在里面创建子线程处理业务（因为可能子线程没处理完，接收者就被回收了，那么子线程也会跟着被回收掉）；正确的处理方法就是通过in调用activity或者service处理业务。

# BroadcastReceiver的注册

　　BroadcastReceiver的注册方式有且只有两种，一种是静态注册（推荐使用），另外一种是动态注册，广播接收者在注册后就开始监听系统或者应用之间发送的广播消息。

**接收短信广播示例**：

定义自己的BroadcastReceiver 类

```
public class MyBroadcastReceiver extends BroadcastReceiver {
 
// action 名称
String SMS_RECEIVED = "android.provider.Telephony.SMS_RECEIVED" ;
 
    public void onReceive(Context context, Intent intent) {
 
       if (intent.getAction().equals( SMS_RECEIVED )) {
           // 一个receiver可以接收多个action的，即可以有多个intent-filter，需要在onReceive里面对intent.getAction(action name)进行判断。
       }
    }
}
```

## 静态方式

　　在AndroidManifest.xml的application里面定义receiver并设置要接收的action。

```
< receiver android:name = ".MyBroadcastReceiver" > 

 < intent-filter android:priority = "777" >             
<action android:name = "android.provider.Telephony.SMS_RECEIVED" />
</ intent-filter > 

</ receiver >
```

　　这里的priority取值是　-1000到1000，值越大优先级越高，同时注意加上系统接收短信的限权。

``` 
< uses-permission android:name ="android.permission.RECEIVE_SMS" />
```

　　静态注册的广播接收者是一个常驻在系统中的全局监听器，当你在应用中配置了一个静态的BroadcastReceiver，安装了应用后而无论应用是否处于运行状态，广播接收者都是已经常驻在系统中了。同时应用里的所有receiver都在清单文件里面，方便查看。　
　　
　　要销毁掉静态注册的广播接收者，可以通过调用PackageManager将Receiver禁用。

## 动态方式

　　在Activity中声明BroadcastReceiver的扩展对象，在onResume中注册，onPause中卸载.
　　　


```

public class MainActivity extends Activity {
	MyBroadcastReceiver receiver;
	@Override
	 protected void onResume() {
		// 动态注册广播 (代码执行到这才会开始监听广播消息，并对广播消息作为相应的处理)
		receiver = new MyBroadcastReceiver();
		IntentFilter intentFilter = new IntentFilter( "android.provider.Telephony.SMS_RECEIVED" );
		registerReceiver( receiver , intentFilter);	
		super.onResume();
	}
	@Override
	protected void onPause() {  
		// 撤销注册 (撤销注册后广播接收者将不会再监听系统的广播消息)
		unregisterReceiver(receiver);
		super.onPause();
	}
}


```




## 静态注册和动态注册的区别

　　1、静态注册的广播接收者一经安装就常驻在系统之中，不需要重新启动唤醒接收者；
　　动态注册的广播接收者随着应用的生命周期，由registerReceiver开始监听，由unregisterReceiver撤销监听，如果应用退出后，没有撤销已经注册的接收者应用应用将会报错。

　　2、当广播接收者通过intent启动一个activity或者service时，如果intent中无法匹配到相应的组件。　
　　动态注册的广播接收者将会导致应用报错
　　而静态注册的广播接收者将不会有任何报错，因为自从应用安装完成后，广播接收者跟应用已经脱离了关系。　

# 发送BroadcastReceiver

## 发送广播主要有两种类型：

### 普通广播
 
 应用在需要通知各个广播接收者的情况下使用，如 开机启动

使用方法：sendBroadcast() 

```
Intent intent = new Intent("android.provider.Telephony.SMS_RECEIVED"); 
//通过intent传递少量数据
intent.putExtra("data", "finch"); 
// 发送普通广播
sendBroadcast(Intent); 
```

　　普通广播是完全异步的，可以在同一时刻（逻辑上）被所有接收者接收到，所有满足条件的 BroadcastReceiver 都会随机地执行其 onReceive() 方法。
　　同级别接收是先后是随机的；级别低的收到广播；
　　消息传递的效率比较高，并且无法中断广播的传播。

　　
　　
### 有序广播

应用在需要有特定拦截的场景下使用，如黑名单短信、电话拦截。　

使用方法：

　``sendOrderedBroadcast(intent, receiverPermission);``

　``receiverPermission`` ：一个接收器必须持以接收您的广播。如果为 null ，不经许可的要求（一般都为null）。

```
//发送有序广播
 sendOrderedBroadcast(intent, null);
```

　　在有序广播中，我们可以在前一个广播接收者将处理好的数据传送给后面的广播接收者，也可以调用abortBroadcast()来终结广播的传播。

```
public void onReceive(Context arg0, Intent intent) {
　　//获取上一个广播的bundle数据
　　Bundle bundle = getResultExtras(true);//true：前一个广播没有结果时创建新的Bundle；false：不创建Bundle
　　bundle.putString("key", "777");
　　//将bundle数据放入广播中传给下一个广播接收者
　　setResultExtras(bundle);　
　　
　　//终止广播传给下一个广播接收者
　　abortBroadcast();
}
```
　　高级别的广播收到该广播后，可以决定把该广播是否截断掉。
　　同级别接收是先后是随机的，如果先接收到的把广播截断了，同级别的例外的接收者是无法收到该广播。
　　
## 异步广播

使用方法：``sendStickyBroadcast()`` ：

　　发出的Intent当接收Activity（动态注册）重新处于onResume状态之后就能重新接受到其Intent.（the Intent will be held to be re-broadcast to future receivers）。就是说sendStickyBroadcast发出的最后一个Intent会被保留，下次当Activity处于活跃的时候又会接受到它。

发这个广播需要权限：

```
<uses-permission android:name="android.permission.BROADCAST_STICKY" />
```

卸载该广播：

```
removeStickyBroadcast(intent);
```
　　在卸载之前该intent会保留，接收者在可接收状态都能获得。

## 异步有序广播

　　使用方法：``sendStickyOrderedBroadcast(intent, resultReceiver, scheduler,
       initialCode, initialData, initialExtras)``：

　　这个方法具有有序广播的特性也有异步广播的特性；
　　同时需要限权：

```
 <uses-permission android:name="android.permission.BROADCAST_STICKY" /> 
```

# 总结

　　

 - 静态广播接收的处理器是由PackageManagerService负责，当手机启动或者新安装了应用的时候，PackageManagerService会扫描手机中所有已安装的APP应用，将AndroidManifest.xml中有关注册广播的信息解析出来，存储至一个全局静态变量当中。

 - 动态广播接收的处理器是由ActivityManagerService负责，当APP的服务或者进程起来之后，执行了注册广播接收的代码逻辑，即进行加载，最后会存储在一个另外的全局静态变量中。需要注意的是：

　　 1、 这个并非是一成不变的，当程序被杀死之后，已注册的动态广播接收器也会被移出全局变量，直到下次程序启动，再进行动态广播的注册，当然这里面的顺序也已经变更了一次。

　　 2、这里也并没完整的进行广播的排序，只记录的注册的先后顺序，并未有结合优先级的处理。

 - 广播发出的时候，广播接收者接收的顺序如下：

　　１．当广播为**普通广播**时，有如下的接收顺序：

　　无视优先级
　　动态优先于静态
　　同优先级的动态广播接收器，**先注册的大于后注册的**
　　同优先级的静态广播接收器，**先扫描的大于后扫描的**　

　　２．如果广播为**有序广播**，那么会将动态广播处理器和静态广播处理器合并在一起处理广播的消息，最终确定广播接收的顺序：　
　　
　　优先级高的先接收　
　　同优先级的动静态广播接收器，**动态优先于静态**
　　同优先级的动态广播接收器，**先注册的大于后注册的**
　　同优先级的静态广播接收器，**先扫描的大于后扫描的**　


# 一些常用的系统广播的action 和permission 

 - 开机启动


```
<action android:name="android.intent.action.BOOT_COMPLETED"/> 
```

```
<uses-permission android:name="android.permission.RECEIVE_BOOT_COMPLETED" />  
```

 - 网络状态

```
<action android:name="android.net.conn.CONNECTIVITY_CHANGE"/>  
```

```
<uses-permission android:name="android.permission.ACCESS_NETWORK_STATE"/> 
```

　　　网络是否可用的方法：

```
  public static boolean isNetworkAvailable(Context context) {  
        ConnectivityManager mgr = (ConnectivityManager) context.getSystemService(Context.CONNECTIVITY_SERVICE);  
        NetworkInfo[] info = mgr.getAllNetworkInfo();  
        if (info != null) {  
            for (int i = 0; i < info.length; i++) {  
      if (info[i].getState() == NetworkInfo.State.CONNECTED) {  
                    return true;  
                }  
            }  
        }  
        return false;  
    } 
```

 - 电量变化

```
<action android:name="android.intent.action.BATTERY_CHANGED"/>  
```
BroadcastReceiver 的onReceive方法：

```
public void onReceive(Context context, Intent intent) {  
        int currLevel = intent.getIntExtra(BatteryManager.EXTRA_LEVEL, 0);  //当前电量  　
        
        int total = intent.getIntExtra(BatteryManager.EXTRA_SCALE, 1);    //总电量  
        int percent = currLevel * 100 / total;  
        Log.i(TAG, "battery: " + percent + "%");  
    }  
```


> 在Android中实现异步任务机制有两种方式，**Handler**和**AsyncTask**。 
> 
> 
> 本篇就说说AsyncTask的异步实现。


## 1、什么时候使用 AsnyncTask

　　在上一篇文章已经说了，主线程主要负责控制UI页面的显示、更新、交互等。  为了有更好的用户体验，UI线程中的操作要求越短越好。

　　我们把耗时的操作（例如网络请求、数据库操作、复杂计算）放到单独的子线程中操作，以避免主线程的阻塞。但是在子线程中不能更新ＵＩ界面，这时候需要使用handler。

　　但如果耗时的操作太多，那么我们需要开启太多的子线程，这就会给系统带来巨大的负担，随之也会带来性能方面的问题。在这种情况下我们就可以考虑使用类AsyncTask来异步执行任务，不需要子线程和handler，就可以完成异步操作和刷新UI。


　　不要随意使用AsyncTask,除非你必须要与UI线程交互.默认情况下使用Thread即可,要注意需要将线程优先级调低.AsyncTask适合处理短时间的操作,长时间的操作,比如下载一个很大的视频,这就需要你使用自己的线程来下载,不管是断点下载还是其它的.

## ２、AsnyncTask原理


　　AsyncTask主要有二个部分：一个是与主线程的交互，另一个就是线程的管理调度。虽然可能多个AsyncTask的子类的实例，但是AsyncTask的内部Handler和ThreadPoolExecutor都是进程范围内共享的，其都是static的，也即属于类的，类的属性的作用范围是CLASSPATH，因为一个进程一个VM，所以是AsyncTask控制着进程范围内所有的子类实例。　

　　AsyncTask内部会创建一个进程作用域的线程池来管理要运行的任务，也就就是说当你调用了AsyncTask的execute()方法后，AsyncTask会把任务交给线程池，由线程池来管理创建Thread和运行Therad。


## ３、AsyncTask介绍
　　Android的AsyncTask比Handler更轻量级一些（只是代码上轻量一些，而实际上要比handler更耗资源），适用于简单的异步处理。
　　
　　Android之所以有Handler和AsyncTask，都是为了不阻塞主线程（UI线程），因为UI的更新只能在主线程中完成，因此异步处理是不可避免的。

　　AsyncTask：对线程间的通讯做了包装，是后台线程和UI线程可以简易通讯：后台线程执行异步任务，将result告知UI线程。

使用AsyncTask分为两步：　

①　继承AsyncTask类实现自己的类

```
public abstract class AsyncTask<Params, Progress, Result> {
```

> Params: 输入参数，对应excute()方法中传递的参数。如果不需要传递参数，则直接设为void即可。
> 
> Progress：后台任务执行的百分比
> 
> Result：返回值类型，和doInBackground（）方法的返回值类型保持一致。

②复写方法

 最少要重写以下这两个方法：

 - doInBackground(Params…) 

　　在**子线程**（其他方法都在主线程执行）中执行比较耗时的操作，不能更新ＵＩ，可以在该方法中调用publishProgress(Progress…)来更新任务的进度。Progress方法是AsycTask中一个final方法只能调用不能重写。

 - onPostExecute(Result)

　　使用在doInBackground 得到的结果处理操作UI， 在主线程执行，任务执行的结果作为此方法的参数返回。
　　
有时根据需求还要实现以下三个方法：

 - onProgressUpdate(Progress…) 

　　可以使用进度条增加用户体验度。 此方法在主线程执行，用于显示任务执行的进度。

 - onPreExecute()

　　这里是最终用户调用Excute时的接口，当任务执行之前开始调用此方法，可以在这里显示进度对话框。

 - onCancelled()  

　　用户调用取消时，要做的操作



## ４、AsyncTask示例

按照上面的步骤定义自己的异步类：

```
public class MyTask extends AsyncTask<String, Integer, String> {  
    //执行的第一个方法用于在执行后台任务前做一些UI操作  
    @Override  
    protected void onPreExecute() {  
       
    }  
   
    //第二个执行方法,在onPreExecute()后执行，用于后台任务,不可在此方法内修改UI
    @Override  
    protected String doInBackground(String... params) {  
         //处理耗时操作
        return "后台任务执行完毕";  
    }  
      
   /*这个函数在doInBackground调用publishProgress(int i)时触发，虽然调用时只有一个参数  
    但是这里取到的是一个数组,所以要用progesss[0]来取值  
    第n个参数就用progress[n]来取值   */
    @Override  
    protected void onProgressUpdate(Integer... progresses) {  
    	//"loading..." + progresses[0] + "%"
        super.onProgressUpdate(progress);  
    }  
      
    /*doInBackground返回时触发，换句话说，就是doInBackground执行完后触发  
    这里的result就是上面doInBackground执行后的返回值，所以这里是"后台任务执行完毕"  */
    @Override  
    protected void onPostExecute(String result) { 
    	
    }  
      
    //onCancelled方法用于在取消执行中的任务时更改UI  
    @Override  
    protected void onCancelled() {  
    	
    }  
}
```

在主线程申明该类的对象，调用对象的execute（）函数开始执行。

```
MyTask ｔ= new MyTask();
t.execute();//这里没有参数
```


## 5、使用AsyncTask需要注意的地方

 - AsnycTask内部的Handler需要和主线程交互，所以AsyncTask的实例必须在UI线程中创建

 - AsyncTaskResult的doInBackground(mParams)方法执行异步任务运行在子线程中，其他方法运行在主线程中，可以操作UI组件。

 - 一个AsyncTask任务只能被执行一次。

 - 运行中可以随时调用AsnycTask对象的cancel(boolean)方法取消任务，如果成功，调用isCancelled()会返回true，并且不会执行 onPostExecute() 方法了，而是执行 onCancelled() 方法。

 - 对于想要立即开始执行的异步任务，要么直接使用Thread，要么单独创建线程池提供给AsyncTask。默认的AsyncTask不一定会立即执行你的任务，除非你提供给他一个单独的线程池。如果不与主线程交互，直接创建一个Thread就可以了。

 　　


# 1、Handler的由来

  
 　　当程序第一次启动的时候，Android会同时启动一条主线程（ Main Thread）来负责处理与UI相关的事件，我们叫做UI线程。

　　Android的UI操作并不是线程安全的（出于性能优化考虑），意味着如果多个线程并发操作UI线程，可能导致线程安全问题。 

　　为了解决Android应用多线程问题—Android平台只允许UI线程修改Activity里的UI组建，就会导致新启动的线程无法改变界面组建的属性值。

　　**简单的说：**当主线程队列处理一个消息超过5秒,android 就会抛出一个 ANP(无响应)的异常,所以,我们需要把一些要处理比较长的消息,放在一个单独线程里面处理,把处理以后的结果,返回给主线程运行,就需要用的Handler来进行线程建的通信。


# ２、Handler的作用

  

２.１　让线程延时执行

主要用到的两个方法：

 - ``final boolean	postAtTime(Runnable r, long uptimeMillis)``

 - ``final boolean	postDelayed(Runnable r, long delayMillis)``

２.２　让任务在其他线程中执行并返回结果

分为两个步骤：

 - 在新启动的线程中发送消息

　　使用Handler对象的sendMessage()方法或者SendEmptyMessage()方法发送消息。

 - 在主线程中获取处理消息

　　重写Handler类中处理消息的方法（void handleMessage(Message msg)），当新启动的线程发送消息时，消息发送到与之关联的MessageQueue。而Hanlder不断地从MessageQueue中获取并处理消息。


# ３、Handler更新UI线程一般使用


　　

 - 首先要进行Handler 申明，复写handleMessage方法( 放在主线程中)

```
private Handler handler = new Handler() {

		@Override
		public void handleMessage(Message msg) {
			// TODO 接收消息并且去更新UI线程上的控件内容
			if (msg.what == UPDATE) {
				// 更新界面上的textview
				tv.setText(String.valueOf(msg.obj));
			}
			super.handleMessage(msg);
		}
	};
```

 - 子线程发送Message给ui线程表示自己任务已经执行完成，主线程可以做相应的操作了。

```
new Thread() {
			@Override
			public void run() {
				// TODO 子线程中通过handler发送消息给handler接收，由handler去更新TextView的值
				try {
					   //do something
						
						Message msg = new Message();
						msg.what = UPDATE;					
						msg.obj = "更新后的值" ;
						handler.sendMessage(msg);
					}
				} catch (InterruptedException e) {
					e.printStackTrace();
				}
			}
		}.start();
```


# 4、Handler原理分析


4.1 　Handler的构造函数


①　public　Handler()

②　public　Handler(Callbackcallback)

③　public　Handler(Looperlooper)

④　public　Handler(Looperlooper, Callbackcallback) 　


 - 第①个和第②个构造函数都没有传递Looper，这两个构造函数都将通过调用Looper.myLooper()获取当前线程绑定的Looper对象，然后将该Looper对象保存到名为mLooper的成员字段中。 　
　　下面来看①②个函数源码：
　　

```
113    public Handler() {
114        this(null, false);
115    }

127    public Handler(Callback callback) {
128        this(callback, false);
129    }

//他们会调用Handler的内部构造方法

188    public Handler(Callback callback, boolean async) {
189        if (FIND_POTENTIAL_LEAKS) {
190      final Class<? extends Handler> klass = getClass();
191      if ((klass.isAnonymousClass() ||klass.isMemberClass()
         || klass.isLocalClass()) &&
192                    (klass.getModifiers() & Modifier.STATIC) == 0) {
193                Log.w(TAG, "The following Handler class should be static or leaks might occur: " +
194                    klass.getCanonicalName());
195            }
196        }
197//************************************
198        mLooper = Looper.myLooper();
199        if (mLooper == null) {
200            throw new RuntimeException(
201                "Can't create handler inside thread that has not called Looper.prepare()");
202        }
203        mQueue = mLooper.mQueue;
204        mCallback = callback;
205        mAsynchronous = async;
206    }

```

  　　我们看到暗红色的重点部分：

　　通过Looper.myLooper()获取了当前线程保存的Looper实例，又通过这个Looper实例获取了其中保存的MessageQueue（消息队列）。**每个Handler 对应一个Looper对象，产生一个MessageQueue**
　　

 - 第③个和第④个构造函数传递了Looper对象，这两个构造函数会将该Looper保存到名为mLooper的成员字段中。 
　　下面来看③④个函数源码：

```
136    public Handler(Looper looper) {
137        this(looper, null, false);
138    }　

147    public Handler(Looper looper, Callback callback) {
148        this(looper, callback, false);
149    }
//他们会调用Handler的内部构造方法

227    public Handler(Looper looper, Callback callback, boolean async) {
228        mLooper = looper;
229        mQueue = looper.mQueue;
230        mCallback = callback;
231        mAsynchronous = async;
232    }

```

 - 第②个和第④个构造函数还传递了Callback对象，Callback是Handler中的内部接口，需要实现其内部的handleMessage方法，Callback代码如下:

```
80     public interface Callback {
81         public boolean More ...handleMessage(Message msg);
82     }
```

　　Handler.Callback是用来处理Message的一种手段，如果没有传递该参数，那么就应该重写Handler的handleMessage方法，也就是说为了使得Handler能够处理Message，我们有两种办法： 
　　
　1. 向Hanlder的构造函数传入一个Handler.Callback对象，并实现Handler.Callback的handleMessage方法 　
　　
　2. 无需向Hanlder的构造函数传入Handler.Callback对象，但是需要重写Handler本身的handleMessage方法 　
　　　
　　　也就是说无论哪种方式，我们都得通过某种方式实现handleMessage方法，这点与Java中对Thread的设计有异曲同工之处。 

4.2　Handle发送消息的几个方法源码

```
   public final boolean sendMessage(Message msg)
    {
        return sendMessageDelayed(msg, 0);
    }
```

```
   public final boolean sendEmptyMessageDelayed(int what, long delayMillis) {
        Message msg = Message.obtain();
        msg.what = what;
        return sendMessageDelayed(msg, delayMillis);
    }
```

```
 public final boolean sendMessageDelayed(Message msg, long delayMillis)
    {
        if (delayMillis < 0) {
            delayMillis = 0;
        }
        return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);
    }
```

```
 public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
        MessageQueue queue = mQueue;
        if (queue == null) {
            RuntimeException e = new RuntimeException(
                    this + " sendMessageAtTime() called with no mQueue");
            Log.w("Looper", e.getMessage(), e);
            return false;
        }
        return enqueueMessage(queue, msg, uptimeMillis);
    }
```

我们可以看出他们最后都调用了sendMessageAtTime（），然后返回了enqueueMessage方法，下面看一下此方法源码：

```
626    private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
　　　　　　//把当前的handler作为msg的target属性
627        msg.target = this;
628        if (mAsynchronous) {
629            msg.setAsynchronous(true);
630        }
631        return queue.enqueueMessage(msg, uptimeMillis);
632    }
```

在该方法中有两件事需要注意：  

1. msg.target = this 

 　　该代码将Message的target绑定为当前的Handler
   
   <br>
   
2. queue.enqueueMessage
　　
　　变量queue表示的是Handler所绑定的消息队列MessageQueue，通过调用queue.enqueueMessage(msg, uptimeMillis)我们将Message放入到消息队列中。

过下图可以看到完整的方法调用顺序： 

![流程图](http://img.blog.csdn.net/20160516125245477)　


# ５、Looper原理分析
 

　　我们一般在主线程申明Handler，有时我们需要继承Thread类实现自己的线程功能，当我们在里面申明Handler的时候会报错。其原因是主线程中已经实现了两个重要的Looper方法，下面看一看ActivityThread.java中main方法的源码：

```
public static void main(String[] args) {
            //......省略
5205        Looper.prepareMainLooper();//>
5206
5207        ActivityThread thread = new ActivityThread();
5208        thread.attach(false);
5209
5210        if (sMainThreadHandler == null) {
5211            sMainThreadHandler = thread.getHandler();
5212        }
5213
5214        AsyncTask.init();
5215
5216        if (false) {
5217            Looper.myLooper().setMessageLogging(new
5218   LogPrinter(Log.DEBUG, "ActivityThread"));
5219        }
5220
5221        Looper.loop();//>
5222
5223        throw new RuntimeException("Main thread loop unexpectedly exited");
5224    }
5225}
```

5.1　首先看prepare()方法

　　

```
70     public static void prepare() {
71         prepare(true);
72     }
73 
74     private static void prepare(boolean quitAllowed) {
　　　　　//证了一个线程中只有一个Looper实例
75         if (sThreadLocal.get() != null) {
76             throw new RuntimeException("Only one Looper may be created per thread");
77         }
78         sThreadLocal.set(new Looper(quitAllowed));
79     }
```
该方法会调用Looper构造函数同时实例化出MessageQueue和当前thread.

```
186    private Looper(boolean quitAllowed) {
187        mQueue = new MessageQueue(quitAllowed);
188        mThread = Thread.currentThread();
189    } 

182    public static MessageQueue myQueue() {
183        return myLooper().mQueue;
184    }

```


　　prepare()方法中通过ThreadLocal对象实现Looper实例与线程的绑定。（不清楚的可以查看　[ThreadLocal的使用规则和源码分析](http://blog.csdn.net/amazing7/article/details/51313851)）　


５.2 　loop()方法　　

```
109    public static void loop() {
110        final Looper me = myLooper();
111        if (me == null) {
112            throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
113        }
114        final MessageQueue queue = me.mQueue;
115
118        Binder.clearCallingIdentity();
119        final long ident = Binder.clearCallingIdentity();
120
121        for (;;) {
122            Message msg = queue.next(); // might block
123            if (msg == null) {
124               
125                return;
126            }
127
129            Printer logging = me.mLogging;
130            if (logging != null) {
131                logging.println(">>>>> Dispatching to " + msg.target + " " +
132                        msg.callback + ": " + msg.what);
133            }
//重点****
135            msg.target.dispatchMessage(msg);
136
137            if (logging != null) {
138                logging.println("<<<<< Finished to " + msg.target + " " + msg.callback);
139            }
140
142            // identity of the thread wasn't corrupted.
143            final long newIdent = Binder.clearCallingIdentity();
144            if (ident != newIdent) {
145                Log.wtf(TAG, "Thread identity changed from 0x"
146                        + Long.toHexString(ident) + " to 0x"
147                        + Long.toHexString(newIdent) + " while dispatching to "
148                        + msg.target.getClass().getName() + " "
149                        + msg.callback + " what=" + msg.what);
150            }
151
152            msg.recycleUnchecked();
153        }
154    }
```
 　　首先looper对象不能为空，就是说loop()方法调用必须在prepare()方法的后面。

　Looper一直在不断的从消息队列中通过MessageQueue的next方法获取Message，然后通过代码msg.target.dispatchMessage(msg)让该msg所绑定的Handler（Message.target）执行dispatchMessage方法以实现对Message的处理。 

Handler的dispatchMessage的源码如下：

```
93     public void dispatchMessage(Message msg) {
94         if (msg.callback != null) {
95             handleCallback(msg);
96         } else {
97             if (mCallback != null) {
98                 if (mCallback.handleMessage(msg)) {
99                     return;
100                }
101            }
102            handleMessage(msg);
103        }
104    }
```
 　　我们可以看到Handler提供了三种途径处理Message，而且处理有前后优先级之分：首先尝试让postXXX中传递的Runnable执行，其次尝试让Handler构造函数中传入的Callback的handleMessage方法处理，最后才是让Handler自身的handleMessage方法处理Message。



# ６、如何在子线程中使用Handler


  　　Handler本质是从当前的线程中获取到Looper来监听和操作MessageQueue，当其他线程执行完成后回调当前线程。

　　子线程需要先prepare（）才能获取到Looper的，是因为在子线程只是一个普通的线程，其ThreadLoacl中没有设置过Looper，所以会抛出异常，而在Looper的prepare（）方法中sThreadLocal.set(new Looper())是设置了Looper的。

６.1　实例代码

　定义一个类实现Runnable接口或继承Thread类（一般不继承）。

```
    class Rub implements Runnable {  
  
        public Handler myHandler;  
        // 实现Runnable接口的线程体 
        @Override  
        public void run() {  
        	
         /*①、调用Looper的prepare()方法为当前线程创建Looper对象并，
          创建Looper对象时，它的构造器会自动的创建相对应的MessageQueue*/
            Looper.prepare();  
            
            /*.②、创建Handler子类的实例，重写HandleMessage()方法，该方法处理除当前线程以外线程的消息*/
             myHandler = new Handler() {  
                @Override  
                public void handleMessage(Message msg) {  
                    String ms = "";  
                    if (msg.what == 0x777) {  
                     
                    }  
                }  
  
            };  
            //③、调用Looper的loop()方法来启动Looper让消息队列转动起来
            Looper.loop();  
        }
    }
```

注意分成三步：　

１．调用Looper的prepare()方法为当前线程创建Looper对象，创建Looper对象时，它的构造器会创建与之配套的MessageQueue。 　

２．有了Looper之后，创建Handler子类实例，重写HanderMessage()方法，该方法负责处理来自于其他线程的消息。 　

３．调用Looper的looper()方法启动Looper。

　　然后使用这个handler实例在任何其他线程中发送消息，最终处理消息的代码都会在你创建Handler实例的线程中运行。



# ７、总结  　　

**Handler**:
 
- 发送消息，它能把消息发送给Looper管理的MessageQueue。 

- 处理消息，并负责处理Looper分给它的消息。
 
- Handler的构造方法，会首先得到当前线程中保存的Looper实例，进而与Looper实例中的MessageQueue想关联。　
 
- Handler的sendMessage方法，会给msg的target赋值为handler自身，然后加入MessageQueue中。　　
　　　
   
   
**Looper**:

- 每个线程只有一个Looper，它负责管理对应的MessageQueue，会不断地从MessageQueue取出消息，并将消息分给对应的Hanlder处理。 　


- 主线程中，系统已经初始化了一个Looper对象，因此可以直接创建Handler即可，就可以通过Handler来发送消息、处理消息。 程序自己启动的子线程，程序必须自己创建一个Looper对象，并启动它，调用``Looper.prepare()``方法。 

- prapare()方法：保证每个线程最多只有一个Looper对象。 　

- looper()方法：启动Looper，使用一个死循环不断取出MessageQueue中的消息，并将取出的消息分给对应的Handler进行处理。 　

**MessageQueue**:

- 由Looper负责管理，它采用先进先出的方式来管理Message。　


**Message**:

- Handler接收和处理的消息对象。 


　　

# 概述

　　Android 也提供了几种方法用来保存数据，使得这些数据即使在程序结束以后依然不会丢失。这些方法有：　　　　　

 - **文本文件**：可以保存在应用程序自己的目录下，安装的每个app都会在/data/data/目录下创建个文件夹，名字和应用程序中AndroidManifest.xml文件中的package一样。　　
    　
 - **SDcard保存**：
 
 - **Preferences保存**：这也是一种经常使用的数据存储方法，因为它们对于用户而言是透明的，并且从应用安装的时候就存在了。
 - **Assets保存**：用来存储一些只读数据，Assets是指那些在assets目录下的文件，这些文件在你将你的应用编译打包之前就要存在，并且可以在应用程序运行的时候被访问到。但有时候我们需要对保存的数据进行一些复杂的操作，或者数据量很大，超出了文本文件和Preference的性能能的范围，所以需要一些更加高效的方法来管理，从Android1.5开始，Android就自带SQLite数据库了。
　　SQLite它是一个独立的，无需服务进程，支持事务处理，可以使用SQL语言的数据库。

# SQLite的特性

 1、 ACID事务 　
　　

>ACID：
> 　　指数据库事务正确执行的四个基本要素的缩写。包含：原子性（Atomicity）、一致性（Consistency）、隔离性（Isolation）、持久性（Durability）。一个支持事务（Transaction）的数据库，必需要具有这四种特性，否则在事务过程（Transaction processing）当中无法保证数据的正确性，交易过程极可能达不到交易方的要求。

2、 零配置 – 无需安装和管理配置 　

3、储存在单一磁盘文件中的一个完整的数据库 　

4、数据库文件可以在不同字节顺序的机器间自由的共享 

5、支持数据库大小至2TB 　

6、 足够小, 大致3万行C代码, 250K 　

7、比一些流行的数据库在大部分普通数据库操作要快 　

8、简单, 轻松的API 　

9、 包含TCL绑定, 同时通过Wrapper支持其他语言的绑定 　

> http://www.sqlite.org/tclsqlite.html

10、良好注释的源代码, 并且有着90%以上的测试覆盖率  

11、 独立: 没有额外依赖  

12、 Source完全的Open, 你可以用于任何用途, 包括出售它  

13、支持多种开发语言，C，PHP，Perl，Java，ASP.NET，Python 


# Android 中使用 SQLite 

  　　Activites 可以通过 Content Provider 或者 Service 访问一个数据库。

## 创建数据库

　　Android 不自动提供数据库。在 Android 应用程序中使用 SQLite，必须自己创建数据库，然后创建表、索引，填充数据。Android 提供了 SQLiteOpenHelper 帮助你创建一个数据库，你只要继承 SQLiteOpenHelper 类根据开发应用程序的需要，封装创建和更新数据库使用的逻辑就行了。　
　　
　　SQLiteOpenHelper 的子类，至少需要实现三个方法：　

```
public class DatabaseHelper extends SQLiteOpenHelper {

	/**
	 * @param context  上下文环境（例如，一个 Activity）
	 * @param name   数据库名字
	 * @param factory  一个可选的游标工厂（通常是 Null）
	 * @param version  数据库模型版本的整数
	 * 
	 * 会调用父类 SQLiteOpenHelper的构造函数
	 */ 
	public DatabaseHelper(Context context, String name, CursorFactory factory, int version) {
		super(context, name, factory, version);
		
	}

	/**
	 *  在数据库第一次创建的时候会调用这个方法
	 *  
	 *根据需要对传入的SQLiteDatabase 对象填充表和初始化数据。
	 */
	@Override
	public void onCreate(SQLiteDatabase db) {

	}

	/**
	 * 当数据库需要修改的时候（两个数据库版本不同），Android系统会主动的调用这个方法。
	 * 一般我们在这个方法里边删除数据库表，并建立新的数据库表.
	 */
	@Override
	public void onUpgrade(SQLiteDatabase db, int oldVersion, int newVersion) {
		//三个参数，一个 SQLiteDatabase 对象，一个旧的版本号和一个新的版本号

	}

	@Override
	public void onOpen(SQLiteDatabase db) {
		// 每次成功打开数据库后首先被执行
		super.onOpen(db);
	}
}
```

继承SQLiteOpenHelper之后就拥有了以下两个方法：

 - getReadableDatabase() 　创建或者打开一个查询数据库

 - getWritableDatabase()　创建或者打开一个可写数据库

```
DatabaseHelper database = new DatabaseHelper(context);//传入一个上下文参数
SQLiteDatabase db = null;
db = database.getWritableDatabase();
```
　　上面这段代码会返回一个 SQLiteDatabase 类的实例，使用这个对象，你就可以查询或者修改数据库。　

SQLiteDatabase类为我们提供了很多种方法，而较常用的方法如下：

> (int) delete(String table,String whereClause,String[] whereArgs)

　　删除数据行

> (long) insert(String table,String nullColumnHack,ContentValues values)

　　	添加数据行

> (int) update(String table, ContentValues values, String whereClause, String[] whereArgs)

　　更新数据行

> (void) execSQL(String sql)

　　	执行一个SQL语句，可以是一个select或其他的sql语句

> (void) close()

　　	关闭数据库

> (Cursor) query(String table, String[] columns, String selection, String[] selectionArgs, String groupBy, String having, String orderBy, String limit)

　　查询指定的数据表返回一个带游标的数据集。

 各参数说明：
table：表名称
colums：列名称数组
selection：条件子句，相当于where
selectionArgs：条件语句的参数数组
groupBy：分组
having：分组条件
orderBy：排序类
limit：分页查询的限制
Cursor：返回值，相当于结果集ResultSet


> (Cursor) rawQuery(String sql, String[] selectionArgs)

　　运行一个预置的SQL语句，返回带游标的数据集（与上面的语句最大的区别就是防止SQL注入）


　　当你完成了对数据库的操作（例如你的 Activity 已经关闭），需要调用 SQLiteDatabase 的 Close() 方法来释放掉数据库连接。

## 创建表和索引

　　为了创建表和索引，需要调用 SQLiteDatabase 的 execSQL() 方法来执行 DDL 语句。如果没有异常，这个方法没有返回值。
　　例如，你可以执行如下代码：
　　

```
 db.execSQL("CREATE TABLE user(_id INTEGER PRIMARY KEY   
        AUTOINCREMENT, username TEXT, password TEXT);");
```

　　这条语句会创建一个名为 user的表，表有一个列名为 _id，并且是主键，这列的值是会自动增长的整数。另外还有两列：username( 字符 ) 和 password( 字符  )。 SQLite 会自动为主键列创建索引。
　　通常情况下，第一次创建数据库时创建了表和索引。要 删除表和索引，需要使用 execSQL() 方法调用 DROP INDEX 和 DROP TABLE 语句。

## 添加数据　

　　有两种方法可以给表添加数据。

①可以使用 execSQL() 方法执行 INSERT, UPDATE, DELETE 等语句来更新表的数据。execSQL() 方法适用于所有不返回结果的 SQL 语句。例如：

```
String sql = "insert into user(username,password) values ('finch','123456');//插入操作的SQL语句
db.execSQL(sql);//执行SQL语句
``` 

②使用 SQLiteDatabase 对象的 insert()。

 　　

```
ContentValues cv = new ContentValues();
cv.put("username","finch");//添加用户名
cv.put("password","123456"); //添加密码
db.insert("user",null,cv);//执行插入操作
```

## 更新数据（修改）

①使用SQLiteDatabase 对象的  update()方法。

```
ContentValues cv = new ContentValues();
cv.put("password","654321");//添加要更改的字段及内容
String whereClause = "username=?";//修改条件
String[] whereArgs = {"finch"};//修改条件的参数
db.update("user",cv,whereClause,whereArgs);//执行修改
```
该方法有四个参数：　
 　　表名；
 　　列名和值的 ContentValues 对象；　
 　　可选的 WHERE 条件；　
 　　可选的填充 WHERE 语句的字符串，这些字符串会替换 WHERE 条件中的“？”标记，update() 根据条件，更新指定列的值.　

②使用execSQL方式的实现

```
String sql = "update [user] set password = '654321' where username="finch";//修改的SQL语句
db.execSQL(sql);//执行修改
``` 


## 删除数据

①使用SQLiteDatabase 对象的delete()方法。

```
String whereClause = "username=?";//删除的条件
String[] whereArgs = {"finch"};//删除的条件参数
db.delete("user",whereClause,whereArgs);//执行删除
```

②使用execSQL方式的实现

```
String sql = "delete from user where username="finch";//删除操作的SQL语句
db.execSQL(sql);//执行删除操作
```

## 查询数据

①使用 rawQuery() 直接调用 SELECT 语句

```
Cursor c = db.rawQuery("select * from user where username=?",new Stirng[]{"finch"});

if(cursor.moveToFirst()) {
    String password = c.getString(c.getColumnIndex("password"));
}
``` 

　　返回值是一个 cursor 对象，这个对象的方法可以迭代查询结果。
如果查询是动态的，使用这个方法就会非常复杂。例如，当你需要查询的列在程序编译的时候不能确定，这时候使用 query() 方法会方便很多。

②通过query实现查询

　　query() 方法用 SELECT 语句段构建查询。
　　SELECT 语句内容作为 query() 方法的参数，比如：要查询的表名，要获取的字段名，WHERE 条件，包含可选的位置参数，去替代 WHERE 条件中位置参数的值，GROUP BY 条件，HAVING 条件。
　　除了表名，其他参数可以是 null。所以代码可写成：

```
Cursor c = db.query("user",null,null,null,null,null,null);//查询并获得游标
if(c.moveToFirst()){//判断游标是否为空
    for(int i=0;i<c.getCount();i++){　
c.move(i);//移动到指定记录
String username = c.getString(c.getColumnIndex("username");
String password = c.getString(c.getColumnIndex("password"));
    }
}
```
### 使用游标

　　不管你如何执行查询，都会返回一个 Cursor，这是 Android 的 SQLite 数据库游标，使用游标，你可以：　　

 - 通过使用 getCount() 方法得到结果集中有多少记录；　
 
 - 通过 moveToFirst(), moveToNext(), 和 isAfterLast() 方法遍历所有记录；

 - 通过 getColumnNames() 得到字段名；

 - 通过 getColumnIndex() 转换成字段号；

 - 通过 getString()，getInt() 等方法得到给定字段当前记录的值；

 - 通过 requery() 方法重新执行查询得到游标；

 - 通过 close() 方法释放游标资源；
 
例如，下面代码遍历 user表:

```
 Cursor result=db.rawQuery("SELECT _id, username, password FROM user"); 
    result.moveToFirst(); 
    while (!result.isAfterLast()) { 
        int id=result.getInt(0); 
        String name=result.getString(1); 
        String password =result.getString(2); 
        // do something useful with these 
        result.moveToNext(); 
      } 
      result.close();
```

>　　移动开发本质上就是手机和服务器之间进行通信，需要从服务端获取数据。反复通过网络获取数据是比较耗时的，特别是访问比较多的时候，会极大影响了性能，Android中可通过缓存机制来减少频繁的网络操作，减少流量、提升性能。

# 实现原理

　　把不需要实时更新的数据缓存下来，通过时间或者其他因素　来判别是读缓存还是网络请求，这样可以缓解服务器压力，一定程度上提高应用响应速度，并且支持离线阅读。
　　

# Bitmap的缓存

　　在许多的情况下(像 ListView, GridView 或 ViewPager 之类的组件 )我们需要一次性加载大量的图片，在屏幕上显示的图片和所有待显示的图片有可能需要马上就在屏幕上无限制的进行滚动、切换。

  　　像ListView, GridView 这类组件，它们的子项当不可见时，所占用的内存会被回收以供正在前台显示子项使用。垃圾回收器也会释放你已经加载了的图片占用的内存。如果你想让你的UI运行流畅的话，就不应该每次显示时都去重新加载图片。保持一些内存和文件缓存就变得很有必要了。

## 使用内存缓存

　　通过预先消耗应用的一点内存来存储数据，便可快速的为应用中的组件提供数据，是一种典型的以**空间换时间**的策略。
　　LruCache  类（Android v4 Support Library 类库中开始提供）非常适合来做图片缓存任务 ，它可以使用一个LinkedHashMap  的强引用来保存最近使用的对象，并且当它保存的对象占用的内存总和超出了为它设计的最大内存时会把**不经常使用**的对象成员踢出以供垃圾回收器回收。

　　给LruCache 设置一个合适的内存大小，需考虑如下因素：

 - 还剩余多少内存给你的activity或应用使用
 - 屏幕上需要一次性显示多少张图片和多少图片在等待显示
 - 手机的大小和密度是多少（密度越高的设备需要越大的 缓存）
 - 图片的尺寸（决定了所占用的内存大小）
 - 图片的访问频率（频率高的在内存中一直保存）
 - 保存图片的质量（不同像素的在不同情况下显示）

	
具体的要根据应用图片使用的具体情况来找到一个合适的解决办法，一个设置 LruCache 例子：

```
private LruCache<String, Bitmap> mMemoryCache;

@Override
protected void onCreate(Bundle savedInstanceState) {
    ...
    // 获得虚拟机能提供的最大内存，超过这个大小会抛出OutOfMemory的异常
    final int maxMemory = (int) (Runtime.getRuntime().maxMemory() / 1024);

    // 用１／８的内存大小作为内存缓存
    final int cacheSize = maxMemory / 8;

    mMemoryCache = new LruCache<String, Bitmap>(cacheSize) {
        @Override
        protected int sizeOf(String key, Bitmap bitmap) {
            // 这里返回的不是item的个数，是cache的size（单位1024个字节）
            return bitmap.getByteCount() / 1024;
        }
    };
    ...
}

public void addBitmapToMemoryCache(String key, Bitmap bitmap) {
    if (getBitmapFromMemCache(key) == null) {
        mMemoryCache.put(key, bitmap);
    }
}

public Bitmap getBitmapFromMemCache(String key) {
    return mMemoryCache.get(key);
}
```
　　当为ImageView加载一张图片时，会先在LruCache 中看看有没有缓存这张图片，如果有的话直接更新到ImageView中，如果没有的话，一个后台线程会被触发来加载这张图片。

```
public void loadBitmap(int resId, ImageView imageView) {
    final String imageKey = String.valueOf(resId);

    // 查看下内存缓存中是否缓存了这张图片
    final Bitmap bitmap = getBitmapFromMemCache(imageKey);
    if (bitmap != null) {
        mImageView.setImageBitmap(bitmap);
    } else {
        mImageView.setImageResource(R.drawable.image_placeholder);
BitmapWorkerTask task = new BitmapWorkerTask(mImageView);
        task.execute(resId);
    }
}
```
   在图片加载的Task中，需要把加载好的图片加入到内存缓存中。

```
class BitmapWorkerTask extends AsyncTask<Integer, Void, Bitmap> {
    ...
    // 在后台完成
    @Override
    protected Bitmap doInBackground(Integer... params) {
        final Bitmap bitmap = decodeSampledBitmapFromResource(
                getResources(), params[0], 100, 100));
    addBitmapToMemoryCache(String.valueOf(params[0]), bitmap);
        return bitmap;
    }
    ...
}
```

## 使用磁盘缓存

　　内存缓存能够快速的获取到最近显示的图片，但不一定就能够获取到。当数据集过大时很容易把内存缓存填满（如GridView ）。你的应用也有可能被其它的任务（比如来电）中断进入到后台，后台应用有可能会被杀死，那么相应的内存缓存对象也会被销毁。 当你的应用重新回到前台显示时，你的应用又需要一张一张的去加载图片了。

   　磁盘文件缓存能够用来处理这些情况，保存处理好的图片，当内存缓存不可用的时候，直接读取在硬盘中保存好的图片，这样可以有效的减少图片加载的次数。读取磁盘文件要比直接从内存缓存中读取要慢一些，而且需要在一个UI主线程外的线程中进行，因为磁盘的读取速度是不能够保证的，磁盘文件缓存显然也是一种以**空间换时间**的策略。

　　如果图片使用非常频繁的话，一个 ContentProvider 可能更适合代替去存储缓存图片，比如图片gallery 应用。

　　下面是一个DiskLruCache的部分代码：

```
private DiskLruCache mDiskLruCache;
private final Object mDiskCacheLock = new Object();
private boolean mDiskCacheStarting = true;
private static final int DISK_CACHE_SIZE = 1024 * 1024 * 10; // 10MB
private static final String DISK_CACHE_SUBDIR = "thumbnails";

@Override
protected void onCreate(Bundle savedInstanceState) {
    ...
    // 初始化内存缓存
    ...
    // 在后台线程中初始化磁盘缓存
    File cacheDir = getDiskCacheDir(this, DISK_CACHE_SUBDIR);
    new InitDiskCacheTask().execute(cacheDir);
    ...
}

class InitDiskCacheTask extends AsyncTask<File, Void, Void> {
    @Override
    protected Void doInBackground(File... params) {
        synchronized (mDiskCacheLock) {
            File cacheDir = params[0];
  mDiskLruCache = DiskLruCache.open(cacheDir, DISK_CACHE_SIZE);
　 mDiskCacheStarting = false; // 结束初始化
　 mDiskCacheLock.notifyAll(); // 唤醒等待线程
        }
        return null;
    }
}

class BitmapWorkerTask extends AsyncTask<Integer, Void, Bitmap> {
    ...
    // 在后台解析图片
    @Override
    protected Bitmap doInBackground(Integer... params) {
        final String imageKey = String.valueOf(params[0]);

        // 在后台线程中检测磁盘缓存
        Bitmap bitmap = getBitmapFromDiskCache(imageKey);

        if (bitmap == null) { // 没有在磁盘缓存中找到图片
 final Bitmap bitmap = decodeSampledBitmapFromResource(
                    getResources(), params[0], 100, 100));
        }

        // 把这个final类型的bitmap加到缓存中
        addBitmapToCache(imageKey, bitmap);

        return bitmap;
    }
    ...
}

public void addBitmapToCache(String key, Bitmap bitmap) {
    // 先加到内存缓存
    if (getBitmapFromMemCache(key) == null) {
        mMemoryCache.put(key, bitmap);
    }

    //再加到磁盘缓存
    synchronized (mDiskCacheLock) {
        if (mDiskLruCache != null && mDiskLruCache.get(key) == null) {
            mDiskLruCache.put(key, bitmap);
        }
    }
}

public Bitmap getBitmapFromDiskCache(String key) {
    synchronized (mDiskCacheLock) {
        // 等待磁盘缓存从后台线程打开
        while (mDiskCacheStarting) {
            try {
                mDiskCacheLock.wait();
            } catch (InterruptedException e) {}
        }
        if (mDiskLruCache != null) {
            return mDiskLruCache.get(key);
        }
    }
    return null;
}

public static File getDiskCacheDir(Context context, String uniqueName) {
    // 优先使用外缓存路径，如果没有挂载外存储，就使用内缓存路径
final String cachePath =
            Environment.MEDIA_MOUNTED.equals(Environment.getExternalStorageState()) ||
!isExternalStorageRemovable() ?getExternalCacheDir(context).getPath():context.getCacheDir().getPath();

    return new File(cachePath + File.separator + uniqueName);
}
```

　　不能在UI主线程中进行这项操作，因为初始化磁盘缓存也需要对磁盘进行操作。上面的程序片段中，一个锁对象确保了磁盘缓存没有初始化完成之前不能够对磁盘缓存进行访问。

　　 内存缓存在UI线程中进行检测，磁盘缓存在UI主线程外的线程中进行检测，当图片处理完成之后，分别存储到内存缓存和磁盘缓存中。

## 设备配置参数改变时加载问题

　　由于应用在运行的时候设备配置参数可能会发生改变，比如设备朝向改变，会导致Android销毁你的Activity然后按照新的配置重启，这种情况下，我们要避免重新去加载处理所有的图片，让用户能有一个流畅的体验。   

   　使用Fragment 能够把内存缓存对象传递到新的activity实例中，调用setRetainInstance(true)) 方法来保留Fragment实例。当activity重新创建好后， 被保留的Fragment依附于activity而存在，通过Fragment就可以获取到已经存在的内存缓存对象了，这样就可以快速的获取到图片，并设置到ImageView上，给用户一个流畅的体验。

下面是一个示例程序片段：

```
private LruCache<String, Bitmap> mMemoryCache;

@Override
protected void onCreate(Bundle savedInstanceState) {
    ...
RetainFragment mRetainFragment =            RetainFragment.findOrCreateRetainFragment(getFragmentManager());
    mMemoryCache = RetainFragment.mRetainedCache;
    if (mMemoryCache == null) {
        mMemoryCache = new LruCache<String, Bitmap>(cacheSize) {
            ... //像上面例子中那样初始化缓存
        }
        mRetainFragment.mRetainedCache = mMemoryCache;
    }
    ...
}

class RetainFragment extends Fragment {
    private static final String TAG = "RetainFragment";
    public LruCache<String, Bitmap> mRetainedCache;

    public RetainFragment() {}

    public static RetainFragment findOrCreateRetainFragment(FragmentManager fm) {
        RetainFragment fragment = (RetainFragment) fm.findFragmentByTag(TAG);
        if (fragment == null) {
            fragment = new RetainFragment();
        }
        return fragment;
    }

    @Override
    public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        // 使得Fragment在Activity销毁后还能够保留下来
        setRetainInstance(true);
    }
}
```

　　可以在不适用Fragment（没有界面的服务类Fragment）的情况下旋转设备屏幕。在保留缓存的情况下，你应该能发现填充图片到Activity中几乎是瞬间从内存中取出而没有任何延迟的感觉。任何图片优先从内存缓存获取，没有的话再到硬盘缓存中找，如果都没有，那就以普通方式加载图片。
　　
参考：

[Caching Bitmaps](http://developer.android.com/training/displaying-bitmaps/cache-bitmap.html)

[LruCache](http://developer.android.com/reference/android/util/LruCache.html)

# 使用SQLite进行缓存

　　网络请求数据完成后，把文件的相关信息（如url（一般作为唯一标示），下载时间，过期时间）等存放到数据库。下次加载的时候根据url先从数据库中查询，如果查询到并且时间未过期，就根据路径读取本地文件，从而实现缓存的效果。

　　注意：缓存的数据库是存放在/data/data/<package>/databases/目录下，是占用内存空间的，如果缓存累计，容易浪费内存，需要及时清理缓存。

# 文件缓存

　　思路和一般缓存一样，把需要的数据存储在文件中，下次加载时判断文件是否存在和过期（使用File.lastModified()方法得到文件的最后修改时间，与当前时间判断），存在并未过期就加载文件中的数据，否则请求服务器重新下载。

　　注意，无网络环境下就默认读取文件缓存中的。


## 1、概述
  

 　　Android提供了5种方式来让用户保存持久化应用程序数据。根据自己的需求来做选择，比如数据是否是应用程序私有的，是否能被其他程序访问，需要多少数据存储空间等，分别是：　
 　　
</br>

①　使用SharedPreferences存储数据　

②　文件存储数据

③　 SQLite数据库存储数据

④　使用ContentProvider存储数据

⑤　网络存储数据　

Android提供了一种方式来暴露你的数据（甚至是私有数据）给其他应用程序 - ContentProvider。它是一个可选组件，可公开读写你应用程序数据。


## 2、SharedPreferences存储

  

　　SharedPreference类提供了一个总体框架，使您可以保存和检索的任何基本数据类型（ boolean, float, int, long, string）的持久键-值对（基于XML文件存储的“key-value”键值对数据）。

　　通常用来存储程序的一些配置信息。其存储在“data/data/程序包名/shared_prefs目录下。

　　xml 处理时Dalvik会通过自带底层的本地XML Parser解析，比如XMLpull方式，这样对于内存资源占用比较好。　

**2.1** 　我们可以通过以下两种方法获取SharedPreferences对象（通过Context）：

> ①　getSharedPreferences (String name, int mode)

　　当我们有多个SharedPreferences的时候，根据第一个参数name获得相应的SharedPreferences对象。


>②　getPreferences (int mode)

　　如果你的Activity中只需要一个SharedPreferences的时候使用。 

这里的mode有四个选项：

```
Context.MODE_PRIVATE
```

　　该SharedPreferences数据只能被本应用程序读、写。

```
Context.MODE_WORLD_READABLE
```

　　该SharedPreferences数据能被其他应用程序读，但不能写。

```
Context.MODE_WORLD_WRITEABLE
```

　　该SharedPreferences数据能被其他应用程序读和写。

```
Context.MODE_MULTI_PROCESS
```

　　sdk2.3后添加的选项，当多个进程同时读写同一个SharedPreferences时它会检查文件是否修改。  

**2.2** 　向Shared Preferences中**写入值**
　
首先要通过 SharedPreferences.Editor获取到Editor对象；

然后通过Editor的putBoolean() 或 putString()等方法存入值；

最后调用Editor的commit()方法提交；

```
//Use 0 or MODE_PRIVATE for the default operation 
SharedPreferences settings = getSharedPreferences("fanrunqi", 0);
SharedPreferences.Editor editor = settings.edit();
editor.putBoolean("isAmazing", true); 

// 提交本次编辑
editor.commit();
```
同时Edit还有两个常用的方法：

> editor.remove(String key) ：下一次commit的时候会移除key对应的键值对 
> 	
>editor.clear()：移除所有键值对

**2.3** 　从Shared Preferences中**读取值** 

　　读取值使用 SharedPreference对象的getBoolean()或getString()等方法就行了（没Editor 啥子事）。

```
SharedPreferences settings = getSharedPreferences("fanrunqi", 0);
boolean isAmazing= settings.getBoolean("isAmazing",true);
```

**2.４** 　Shared Preferences的优缺点

　　可以看出来Preferences是很轻量级的应用，使用起来也很方便，简洁。但存储数据类型比较单一（只有基本数据类型），无法进行条件查询，只能在不复杂的存储需求下使用，比如保存配置信息等。


## 3、文件数据存储

  

### 3.1 使用内部存储

　　当文件被保存在内部存储中时，默认情况下，文件是应用程序私有的，其他应用不能访问。当用户卸载应用程序时这些文件也跟着被删除。

　　文件默认存储位置：/data/data/包名/files/文件名。

### 3.1.1 创建和写入一个内部存储的私有文件：

①　调用Context的openFileOutput()函数，填入文件名和操作模式，它会返回一个FileOutputStream对象。

②　通过FileOutputStream对象的write()函数写入数据。

③　 FileOutputStream对象的close ()函数关闭流。

例如：

```
		String FILENAME = "a.txt";
		String string = "fanrunqi";

		try {
			FileOutputStream fos = openFileOutput(FILENAME, Context.MODE_PRIVATE);
			fos.write(string.getBytes());
			fos.close();
		} catch (Exception e) {
			e.printStackTrace();
		}
```

在 ``openFileOutput(String name, int mode)``方法中
 

 - name参数:　用于指定文件名称，不能包含路径分隔符“/” ，如果文件不存在，Android 会自动创建它。
 
 
 - mode参数：用于指定操作模式，分为四种：

> Context.MODE_PRIVATE = 0

　　为默认操作模式，代表该文件是私有数据，只能被应用本身访问，在该模式下，写入的内容会覆盖原文件的内容。

> Context.MODE_APPEND = 32768

　　该模式会检查文件是否存在，存在就往文件追加内容，否则就创建新文件。　

> Context.MODE_WORLD_READABLE = 1

　　表示当前文件可以被其他应用读取。

> MODE_WORLD_WRITEABLE

　　表示当前文件可以被其他应用写入。

### 3.1.2 读取一个内部存储的私有文件：

① 调用openFileInput( )，参数中填入文件名，会返回一个FileInputStream对象。

② 使用流对象的 read()方法读取字节

③ 调用流的close()方法关闭流

例如：

```
	String FILENAME = "a.txt";
		try {
            FileInputStream inStream = openFileInput(FILENAME);
            int len = 0;
            byte[] buf = new byte[1024];
            StringBuilder sb = new StringBuilder();
            while ((len = inStream.read(buf)) != -1) {
                sb.append(new String(buf, 0, len));
            }
            inStream.close();
        } catch (Exception e) {
            e.printStackTrace();
        } 
```

其他一些经常用到的方法：

 - getFilesDir()：　得到内存储文件的绝对路径

 - getDir()：　在内存储空间中**创建**或**打开一个已经存在**的目录

 - deleteFile()：　删除保存在内部存储的文件。　　
 - fileList()：　返回当前由应用程序保存的文件的数组（内存储目录下的全部文件）。　



　　
### 3.1.３　保存编译时的静态文件

　　如果你想在应用编译时保存静态文件，应该把文件保存在项目的　**res/raw/**　目录下，你可以通过 openRawResource()方法去打开它（传入参数R.raw.filename），这个方法返回一个 InputStream流对象你可以读取文件但是不能修改原始文件。

```
InputStream is = this.getResources().openRawResource(R.raw.filename);
```

### 3.1.４　保存内存缓存文件

　　有时候我们只想缓存一些数据而不是持久化保存，可以使用getCacheDir（）去打开一个文件，文件的存储目录（ /data/data/包名/cache ）是一个应用专门来保存临时缓存文件的内存目录。

　　当设备的内部存储空间比较低的时候，Android可能会删除这些缓存文件来恢复空间，但是你不应该依赖系统来回收，要自己维护这些缓存文件把它们的大小限制在一个合理的范围内，比如1ＭＢ．当你卸载应用的时候这些缓存文件也会被移除。


## 3.２ 使用外部存储（sdcard）

　　因为内部存储容量限制，有时候需要存储数据比较大的时候需要用到外部存储，使用外部存储分为以下几个步骤：

### 3.2.1　添加外部存储访问限权
　
　　首先，要在AndroidManifest.xml中加入访问SDCard的权限，如下:

```
　<!-- 在SDCard中创建与删除文件权限 --> 
    <uses-permission android:name="android.permission.MOUNT_UNMOUNT_FILESYSTEMS"/> 

   <!-- 往SDCard写入数据权限 --> 
    <uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE"/>
```

### 3.2.２　检测外部存储的可用性

　　在使用外部存储时我们需要检测其状态，它可能被连接到计算机、丢失或者只读等。下面代码将说明如何检查状态：

```
//获取外存储的状态
String state = Environment.getExternalStorageState();
if (Environment.MEDIA_MOUNTED.equals(state)) {
    // 可读可写
    mExternalStorageAvailable = mExternalStorageWriteable = true;
} else if (Environment.MEDIA_MOUNTED_READ_ONLY.equals(state)) {
    // 可读
} else {
    // 可能有很多其他的状态，但是我们只需要知道，不能读也不能写  
}
```

### 3.2.3　访问外部存储器中的文件
　

**１、如果 API 版本大于或等于８**，使用

> getExternalFilesDir (String type)

　　该方法打开一个外存储目录，此方法需要一个类型，指定你想要的子目录，如类型参数DIRECTORY_MUSIC和 DIRECTORY_RINGTONES（传null就是你应用程序的文件目录的根目录）。通过指定目录的类型，确保Android的媒体扫描仪将扫描分类系统中的文件（例如，铃声被确定为铃声）。如果用户卸载应用程序，这个目录及其所有内容将被删除。

例如：
```
File file = new File(getExternalFilesDir(null), "fanrunqi.jpg");
```

**２、如果API 版本小于 8** （7或者更低）

 

>  getExternalStorageDirectory ()

通过该方法打开外存储的根目录，你应该在以下目录下写入你的应用数据，这样当卸载应用程序时该目录及其所有内容也将被删除。

```
/Android/data/<package_name>/files/
```

读写数据：

```
if(Environment.getExternalStorageState().equals(Environment.MEDIA_MOUNTED)){  
		    File sdCardDir = Environment.getExternalStorageDirectory();//获取SDCard目录  "/sdcard"        

		       File saveFile = new File(sdCardDir,"a.txt"); 
		        
		       //写数据
		        try {
		        	FileOutputStream fos= new FileOutputStream(saveFile); 
		        	fos.write("fanrunqi".getBytes()); 
					fos.close();
				} catch (Exception e) {
					e.printStackTrace();
				} 
				
				//读数据
				 try {
		        	FileInputStream fis= new FileInputStream(saveFile); 
		        	int len =0;
		        	byte[] buf = new byte[1024];
		        	StringBuffer sb = new StringBuffer();
		        	while((len=fis.read(buf))!=-1){
		        		sb.append(new String(buf, 0, len));
		        	}
		        	fis.close();
				} catch (Exception e) {
					e.printStackTrace();
				}  
		}
``` 

　　我们也可以在　/Android/data/package_name/cache/目录下做外部缓存。

部分翻译于：[android-data-storage](http://www.android-doc.com/guide/topics/data/data-storage.html)

# ４、 网络存储数据
 
## HttpUrlConnection

 　　HttpUrlConnection是Java.net包中提供的API，我们知道Android SDK是基于Java的，所以当然优先考虑HttpUrlConnection这种最原始最基本的API，其实大多数开源的联网框架基本上也是基于JDK的HttpUrlConnection进行的封装罢了，掌握HttpUrlConnection需要以下几个步骤：
　　
1、将访问的路径转换成URL。

> URL url = new URL(path);

2、通过URL获取连接。

 

> HttpURLConnection conn = (HttpURLConnection) url.openConnection();
 

3、设置请求方式。

 

> conn.setRequestMethod(GET);
 

4、设置连接超时时间。

 

>conn.setConnectTimeout(5000);

5、设置请求头的信息。

> conn.setRequestProperty(User-Agent, Mozilla/5.0 (compatible; MSIE 9.0; Windows NT 6.1; Trident/5.0));

7、针对不同的响应码，做不同的操作（请求码200，表明请求成功，获取返回内容的输入流）

工具类：

```
public class StreamTools {
	/**
	 * 将输入流转换成字符串
	 * 
	 * @param is
	 *            从网络获取的输入流
	 * @return
	 */
	public static String streamToString(InputStream is) {
		try {
			ByteArrayOutputStream baos = new ByteArrayOutputStream();
			byte[] buffer = new byte[1024];
			int len = 0;
			while ((len = is.read(buffer)) != -1) {
				baos.write(buffer, 0, len);
			}
			baos.close();
			is.close();
			byte[] byteArray = baos.toByteArray();
			return new String(byteArray);
		} catch (Exception e) {
			Log.e(tag, e.toString());
			return null;
		}
	}
}
```

### HttpUrlConnection发送GET请求

```
public static String loginByGet(String username, String password) {
		String path = http://192.168.0.107:8080/WebTest/LoginServerlet?username= + username + &password= + password;
		try {
			URL url = new URL(path);
			HttpURLConnection conn = (HttpURLConnection) url.openConnection();
			conn.setConnectTimeout(5000);
			conn.setRequestMethod(GET);
			int code = conn.getResponseCode();
			if (code == 200) {
				InputStream is = conn.getInputStream(); // 字节流转换成字符串
				return StreamTools.streamToString(is);
			} else {
				return 网络访问失败;
			}
		} catch (Exception e) {
			e.printStackTrace();
			return 网络访问失败;
		}
	}
```

### HttpUrlConnection发送POST请求

```
public static String loginByPost(String username, String password) {
		String path = http://192.168.0.107:8080/WebTest/LoginServerlet;
		try {
			URL url = new URL(path);
			HttpURLConnection conn = (HttpURLConnection) url.openConnection();
			conn.setConnectTimeout(5000);
			conn.setRequestMethod(POST);
			conn.setRequestProperty(Content-Type, application/x-www-form-urlencoded);
			String data = username= + username + &password= + password;
			conn.setRequestProperty(Content-Length, data.length() + );
			// POST方式，其实就是浏览器把数据写给服务器
			conn.setDoOutput(true); // 设置可输出流
			OutputStream os = conn.getOutputStream(); // 获取输出流
			os.write(data.getBytes()); // 将数据写给服务器
			int code = conn.getResponseCode();
			if (code == 200) {
				InputStream is = conn.getInputStream();
				return StreamTools.streamToString(is);
			} else {
				return 网络访问失败;
			}
		} catch (Exception e) {
			e.printStackTrace();
			return 网络访问失败;
		}
	}
```

## HttpClient

　　HttpClient是开源组织Apache提供的Java请求网络框架，其最早是为了方便Java服务器开发而诞生的，是对JDK中的HttpUrlConnection各API进行了封装和简化，提高了性能并且降低了调用API的繁琐，Android因此也引进了这个联网框架，我们再不需要导入任何jar或者类库就可以直接使用，值得注意的是Android官方已经宣布不建议使用HttpClient了。

### HttpClient发送GET请求

1、 创建HttpClient对象

2、创建HttpGet对象，指定请求地址（带参数）

3、使用HttpClient的execute(),方法执行HttpGet请求，得到HttpResponse对象

4、调用HttpResponse的getStatusLine().getStatusCode()方法得到响应码

5、调用的HttpResponse的getEntity().getContent()得到输入流，获取服务端写回的数据

```
public static String loginByHttpClientGet(String username, String password) {
		String path = http://192.168.0.107:8080/WebTest/LoginServerlet?username=
				+ username + &password= + password;
		HttpClient client = new DefaultHttpClient(); // 开启网络访问客户端
		HttpGet httpGet = new HttpGet(path); // 包装一个GET请求
		try {
			HttpResponse response = client.execute(httpGet); // 客户端执行请求
			int code = response.getStatusLine().getStatusCode(); // 获取响应码
			if (code == 200) {
				InputStream is = response.getEntity().getContent(); // 获取实体内容
				String result = StreamTools.streamToString(is); // 字节流转字符串
				return result;
			} else {
				return 网络访问失败;
			}
		} catch (Exception e) {
			e.printStackTrace();
			return 网络访问失败;
		}
	}
```

### HttpClient发送POST请求

1，创建HttpClient对象

2，创建HttpPost对象，指定请求地址

3，创建List，用来装载参数

4，调用HttpPost对象的setEntity()方法，装入一个UrlEncodedFormEntity对象，携带之前封装好的参数

5，使用HttpClient的execute()方法执行HttpPost请求，得到HttpResponse对象

6， 调用HttpResponse的getStatusLine().getStatusCode()方法得到响应码

7， 调用的HttpResponse的getEntity().getContent()得到输入流，获取服务端写回的数据

```
public static String loginByHttpClientPOST(String username, String password) {
		String path = http://192.168.0.107:8080/WebTest/LoginServerlet;
		try {
			HttpClient client = new DefaultHttpClient(); // 建立一个客户端
			HttpPost httpPost = new HttpPost(path); // 包装POST请求
			// 设置发送的实体参数
			List parameters = new ArrayList();
			parameters.add(new BasicNameValuePair(username, username));
			parameters.add(new BasicNameValuePair(password, password));
			httpPost.setEntity(new UrlEncodedFormEntity(parameters, UTF-8));
			HttpResponse response = client.execute(httpPost); // 执行POST请求
			int code = response.getStatusLine().getStatusCode();
			if (code == 200) {
				InputStream is = response.getEntity().getContent();
				String result = StreamTools.streamToString(is);
				return result;
			} else {
				return 网络访问失败;
			}
		} catch (Exception e) {
			e.printStackTrace();
			return 访问网络失败;
		}
	}
``` 

参考：

  　　[Android开发请求网络方式详解](http://www.2cto.com/kf/201501/368943.html)
## Android提供的其他网络访问框架

　　HttpClient和HttpUrlConnection的两种网络访问方式编写网络代码，需要自己考虑很多，获取数据或许可以，但是如果要将手机本地数据上传至网络，根据不同的web端接口，需要组织不同的数据内容上传，给手机端造成了很大的工作量。
　　
　　目前有几种快捷的网络开发开源框架，给我们提供了非常大的便利。下面是这些项目Github地址，有文档和Api说明。
　　
[android-async-http](https://github.com/loopj/android-async-http)　

[http-request](https://github.com/kevinsawicki/http-request)

[okhttp](https://github.com/square/okhttp)


# ５、 SQLite数据库存储数据

　　
　　前面的文章[ SQLite的使用入门](http://blog.csdn.net/amazing7/article/details/51375012)已经做了详细说明，这里就不在多说了。


# ６、 使用ContentProvider存储数据
　　同样可以查看　[ContentProvider实例详解](http://blog.csdn.net/amazing7/article/details/51324022)

 

RecyclerView

一、简介

这个是谷歌官方出的控件，使我们可以非常简单的做出列表装的一个控件，当然recyclerview的功能不止这些，它还可以做出瀑布流的效果，这是一个非常强大的控件，内部自带viewholder可以使我们非常简单的完成许多操作，正在一步一步取代listview这个控件，当然它也有一些小的缺点，那就是谷歌官方并没有直接给我写出它的点击事件的接口，但是这并难不倒我们，我们可以自己写一个回调的接口，实现点击事件，在这里我不仅要为大家介绍recyclerview的item的点击事件，还要为大家介绍，一个item中局部的点击事件，还有添加header、footer，还有添加不同类别的item的布局。可以说彻底的读懂了这篇文章，我们对recyclerview就有了一个新的认识了。

二、涉及到的知识点

  
 - item的点击事件
 - item里面内容的点击事件
 - 为recycler view添加header和footer
 - 为item添加不同的布局
 
三、实现代码

```
/**
 * Created by linSir on 16/7/24.管理地址界面的适配器
 */
public class ManageAddressAdapter extends RecyclerView.Adapter<RecyclerView.ViewHolder> {//在这里我们要继承自官方为我们写好的适配器

    public OnTitleClickListener mListener;
    private List<AllAddress> mList;  //用户列表

    public ManageAddressAdapter() {
        mList = new ArrayList<>();
    }

    public void setList(List<AllAddress> list) {//从外界传入一个list
        mList.clear();
        mList.addAll(list);
        notifyDataSetChanged();
    }


    @Override
    public RecyclerView.ViewHolder onCreateViewHolder(ViewGroup parent, int viewType) {//重写方法，用上我们下面写好的viewHoler
        return new ManageAddressViewHolder(LayoutInflater.from(parent.getContext()).inflate(R.layout.item_manage_address, parent, false));
    }

    @Override public void onBindViewHolder(RecyclerView.ViewHolder holder, int position) {//在这里我们将数据和控件绑定起来
        ManageAddressViewHolder mHolder = (ManageAddressViewHolder) holder;
        mHolder.userName.setText(mList.get(position).getShipName());
        mHolder.userTel.setText(mList.get(position).getPhone());

        String address = mList.get(position).getProvince() + " " + mList.get(position).getCity()
                + " " + mList.get(position).getArea() + " " + mList.get(position).getDetail();
        mHolder.address.setText(address);

        mHolder._default.setOnClickListener(new ClickListener(String.valueOf(position)));//在这里我们设置了点击事件

    }

    @Override public int getItemCount() {
        return mList.size();
    }


    public static class ManageAddressViewHolder extends RecyclerView.ViewHolder {

        private TextView userName;
        private TextView userTel;
        private TextView address;
        private TextView _default;

        public ManageAddressViewHolder(View itemView) {
            super(itemView);
            userName = (TextView) itemView.findViewById(R.id.manage_userName);
            userTel = (TextView) itemView.findViewById(R.id.manage_userTel);
            address = (TextView) itemView.findViewById(R.id.manage_userAddress);
            _default = (TextView) itemView.findViewById(R.id.manage_default);


        }
    }


    public class ClickListener implements View.OnClickListener {//在这里我们重写了点击事件
        private String id;

        public ClickListener(String id) {
            this.id = id;
        }

        @Override public void onClick(View view) {
            if (mListener != null) {
                mListener.onTitleClick(id);
            }
        }
    }


    public void setOnTitleClickListener(OnTitleClickListener listener) {//自己写了一个方法，用上我们的接口
        mListener = listener;
    }


    public interface OnTitleClickListener {//自己写了一个点击事件的接口
        void onTitleClick(String id);
    }


}
```

通过以上的代码，我们已经为recyclerView里面的item里面的textview添加成功了点击事件，我们只需要在调用它的界面实现这个接口，然后重写点击事件的方法，就可以实现这个textview的点击事件了，下面我们一起看一下代码：

```
/**
 * Created by linSir on 16/7/24.管理地址界面
 */
public class ManageAddressActivity extends AppCompatActivity implements ManageAddressAdapter.OnTitleClickListener {//实现上文我们写好的点击事件的接口

    private ManageAddressAdapter mAdapter;
    private List<AllAddress> list;
    @BindView(R.id.manage_address_recyclerView) RecyclerView mRecyclerView;

    @Override protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_manage);
        ButterKnife.bind(this);
        list = new ArrayList<AllAddress>();
        mAdapter = new ManageAddressAdapter();
        LinearLayoutManager linearLayoutManager = new LinearLayoutManager(this);
        mAdapter.setOnTitleClickListener(this);//声明一下
        mRecyclerView.setAdapter(mAdapter);
        mRecyclerView.setLayoutManager(linearLayoutManager);//这里千万不要了为recyclerview设置布局


    }

    @OnClick(R.id.back_manage_address)
    public void back() {
        finish();
    }

    @OnClick(R.id.manage_add_address)
    public void add() {
        Intent intent = new Intent(ManageAddressActivity.this, EditAddress.class);
        startActivity(intent);
    }

    @Override
    protected void onResume() {
        super.onResume();
        final HttpResultListener<List<AllAddress>> listener;

        listener = new HttpResultListener<List<AllAddress>>() {
            @Override
            public void onSuccess(List<AllAddress> allAddresses) {
                Toast.makeText(ManageAddressActivity.this, "获取收货成功", Toast.LENGTH_SHORT).show();
                mAdapter.setList(allAddresses);
                list = allAddresses;

            }

            @Override
            public void onError(Throwable e) {
                Log.i("lin", "----lin----> 获取收货地址失败 " + e.toString());
            }
        };
        ApiService5.getInstance().allAddress(listener, 1);
    }

    @Override public void onTitleClick(String id) {//这里便是我们重写了的点击事件
        Log.i("lin", "----lin----> onTitleClick 的 id " + id);
    }
}

```

以上的代码，我们实现了，为recyclerView的一个item中的textview添加的点击事件，下面我要为大家介绍一下，如何为recyclerView中item添加不同的布局文件，并且如何为item整个添加点击事件：

```
/**
 * Created by lin_sir on 2016/7/7.部分商品的展示,的适配器
 */
public class BuyerRecyclerAdapter extends RecyclerView.Adapter<RecyclerView.ViewHolder> {

    public static final int FOOTER_TYPE = 0;//最后一个的类型
    public static final int HAS_IMG_TYPE = 1;//有图片的类型

    private List<FamousPageModel> dataList;

    private ProgressBar mProgress;
    private TextView mNoMore;

    public BuyerRecyclerAdapter() {
        dataList = new ArrayList<>();
    }

    public void addData(List<FamousPageModel> list) {
        dataList.addAll(list);
        notifyDataSetChanged();
    }

    @Override
    public RecyclerView.ViewHolder onCreateViewHolder(ViewGroup parent, int viewType) {
        if (viewType == FOOTER_TYPE) {
            return new FooterItemViewHolder(LayoutInflater.from(parent.getContext()).inflate(R.layout.item_footer, parent, false));
        } else {
            return new BuyerItemViewHolder(LayoutInflater.from(parent.getContext()).inflate(R.layout.item_buyer, parent, false));
        }

    }

    @Override
    public void onBindViewHolder(RecyclerView.ViewHolder holder, int position) {
        int type = getItemViewType(position);
        if (type == FOOTER_TYPE) {
            bindFooterView((FooterItemViewHolder) holder);
        } else {
            bindView((BuyerItemViewHolder) holder, dataList.get(position));
        }
    }

    @Override
    public int getItemViewType(int position) {
        if (position + 1 == getItemCount()) {
            return FOOTER_TYPE;
        } else {
            FamousPageModel news = dataList.get(position);
            return HAS_IMG_TYPE;
        }
    }

    private void bindView(BuyerItemViewHolder holder, FamousPageModel data) {

        String productName = data.getProductName();
        String[] products = productName.split(" ");
        if (products.length != 1) {
            productName = products[1] + " 等商品";
        }

        String count = data.getNum();
        int sum = 0;
        String[] counts = count.split(" ");
        if (counts.length == 2) {
            sum = Integer.parseInt(counts[0]) + Integer.parseInt(counts[1]);
            count = String.valueOf(sum);
        }

        if (counts.length == 3) {
            sum = Integer.parseInt(counts[0]) + Integer.parseInt(counts[1]) + Integer.parseInt(counts[2]);
            count = String.valueOf(sum);
        }

        holder.count.setText(count);
        holder.goods_name.setText(productName);
        holder.price.setText(data.getTotal());
        if (data.getUser() != null) {
            holder.user_name.setText(data.getUser().getName());

        }

        String imgUrl = data.getPicture();

        if (imgUrl != null) {
            ImageUtil.requestImg(BaseApplication.get().getAppContext(), imgUrl, holder.img);
        }

//        Picasso.with(BaseApplication.get().getAppContext()).load(data.getPicture()).into(holder.img);
    }


    @Override
    public int getItemCount() {
        return dataList == null ? 0 : dataList.size() + 1;
    }

    public static class BuyerItemViewHolder extends RecyclerView.ViewHolder {

        private ImageView img;
        private TextView price;
        private TextView goods_name;
        private TextView count;
        private TextView user_name;


        public BuyerItemViewHolder(View itemView) {

            super(itemView);

            img = (ImageView) itemView.findViewById(R.id.iv_user_item_buyer2);
            price = (TextView) itemView.findViewById(R.id.tv_product_price);
            goods_name = (TextView) itemView.findViewById(R.id.tv_product_name);
            count = (TextView) itemView.findViewById(R.id.tv_product_number);
            user_name = (TextView) itemView.findViewById(R.id.tv_user_name);

        }
    }


    /**
     * 刷新列表
     */
    public void refreshList(List<FamousPageModel> list) {
        dataList.clear();
        dataList.addAll(list);
        notifyDataSetChanged();
    }

    /**
     * 加载更多
     */
    public void loadMoreList(List<FamousPageModel> list) {
        dataList.addAll(list);
        notifyDataSetChanged();
    }

    /**
     * 显示没有更多
     */
    public void showNoMore() {
        if (getItemCount() > 0) {
            if (mProgress != null && mNoMore != null) {
                mNoMore.setVisibility(View.VISIBLE);
                mProgress.setVisibility(View.GONE);
            }
        }
    }


    /**
     * 显示加载更多
     */
    public void showLoadMore() {
        if (mProgress != null && mNoMore != null) {
            mProgress.setVisibility(View.VISIBLE);
            mNoMore.setVisibility(View.GONE);
        }
    }

    private void bindFooterView(FooterItemViewHolder viewHolder) {
        mProgress = viewHolder.mProgress;
        mNoMore = viewHolder.tvNoMore;
    }


    public static class FooterItemViewHolder extends RecyclerView.ViewHolder {

        private ProgressBar mProgress;
        private TextView tvNoMore;

        public FooterItemViewHolder(View itemView) {
            super(itemView);
            mProgress = (ProgressBar) itemView.findViewById(R.id.pb_footer_load_more);
            tvNoMore = (TextView) itemView.findViewById(R.id.tv_footer_no_more);
        }
    }


    /**
     * 获取点击的 item,如果是最后一个,则返回 null
     */
    public FamousPageModel getClickItem(int position) {
        if (position < dataList.size()) {
            return dataList.get(position);
        } else {
            return null;
        }
    }

}

```

在这里我们首先重写了getItemViewType，这里面我目前只写了两种类型，那就是带有图片的类型，和不带有图片的类型，并且我让不带图片的类型出现在了最后面，当然大家可以随意的设置，我们只需要根据它们的特征为他们分好类，然后根据不同的类型加载不同的布局文件即可。


```
/**
 * Created by lin_sir on 2016/7/7. buyer界面
 */
public class FragmentBuyer extends Fragment implements SwipeRefreshLayout.OnRefreshListener {

    private static int CURRENT_PAGE = 1;                            //获取需要请求的页号
    private RecyclerView recyclerView;                              //recyclerView
    private BuyerRecyclerAdapter madapter;                          //适配器
    private List<FamousPageModel> mlist;                            //一个装载数据的集合
    private LinearLayoutManager linearLayoutManager;                //linearLayoutManger
    private HttpResultListener<List<FamousPageModel>> listener;     //数据请求的回调接口
    private SwipeRefreshLayout refreshLayout;                       //下拉刷新控件
    private boolean isLoadMore;                                     //是否加载更多

    @Nullable
    @Override
    public View onCreateView(LayoutInflater inflater, @Nullable ViewGroup container, @Nullable Bundle savedInstanceState) {
        View view = inflater.inflate(R.layout.fragment_buyer, container, false);
        initviews(view);
        initListener();
        refreshData();
        ButterKnife.bind(this, view);
        return view;
    }

    private void initListener() {
        listener = new HttpResultListener<List<FamousPageModel>>() {
            @Override
            public void onSuccess(List<FamousPageModel> list) {
                Log.i("lin", "----lin---->  refresh  success");
                refreshLayout.setRefreshing(false);
                if (isLoadMore) {
                    mlist = list;
                    madapter.loadMoreList(mlist);
                    madapter.notifyDataSetChanged();
                    madapter.showNoMore();
                } else {
                    mlist = list;
                    madapter.refreshList(list);
                    madapter.notifyDataSetChanged();
                }
            }

            @Override
            public void onError(Throwable e) {
                Log.i("lin", "----lin---->   refresh  error  " + e.toString());
                refreshLayout.setRefreshing(false);
                madapter.showNoMore();
            }
        };
    }

    private void initviews(View view) {

        mlist = new ArrayList<>();

        refreshLayout = (SwipeRefreshLayout) view.findViewById(R.id.refresh_news);
        recyclerView = (RecyclerView) view.findViewById(R.id.rv_buyer);

        refreshLayout.setColorSchemeResources(R.color.blue_500, R.color.purple_500, R.color.green_500);
        refreshLayout.setOnRefreshListener(this);
        linearLayoutManager = new LinearLayoutManager(getActivity());
        linearLayoutManager.setOrientation(OrientationHelper.VERTICAL);

        madapter = new BuyerRecyclerAdapter();
        madapter.addData(mlist);

        recyclerView.setLayoutManager(linearLayoutManager);
        recyclerView.setAdapter(madapter);
        recyclerView.addOnScrollListener(new OnRecyclerScrollListener());
        recyclerView.addOnItemTouchListener(new RecyclerItemClickListener(getActivity(), onItemClickListener));
    }

    /**
     * 刷新时,默认请求第一页的数据
     */
    private void refreshData() {
        refreshLayout.setRefreshing(true);
        isLoadMore = false;
        CURRENT_PAGE = 1;
        requestData(1);
    }

    /**
     * 加载更多
     */
    private void loadMoreData() {
        refreshLayout.setRefreshing(false);// 加载更多与刷新不能同时存在
        isLoadMore = true;
        requestData(++CURRENT_PAGE);
    }

    public class OnRecyclerScrollListener extends RecyclerView.OnScrollListener {

        int lastVisibleItem = 0;

        @Override
        public void onScrollStateChanged(RecyclerView recyclerView, int newState) {
            super.onScrollStateChanged(recyclerView, newState);

            if (madapter != null && newState == RecyclerView.SCROLL_STATE_IDLE && lastVisibleItem + 1 == madapter.getItemCount()) {
                loadMoreData();
            }
        }

        @Override
        public void onScrolled(RecyclerView recyclerView, int dx, int dy) {
            super.onScrolled(recyclerView, dx, dy);
            lastVisibleItem = linearLayoutManager.findLastVisibleItemPosition();
        }
    }

    /**
     * 请求数据
     */
    private void requestData(int page) {
        madapter.showLoadMore();
        ApiService2.getInstance().famous(listener, page);
    }

    @Override
    public void onRefresh() {
        refreshData();
    }

    private RecyclerItemClickListener.OnItemClickListener onItemClickListener = new RecyclerItemClickListener.OnItemClickListener() {//这里我们便实现了点击事件
        @Override
        public void onItemClick(View view, int position) {
            Toast.makeText(getActivity(), "您点击的是：   " + position, Toast.LENGTH_SHORT).show();
            
        }
    };


    @OnClick(R.id.release_distance_buyer)
    public void release() {
        Intent intent = new Intent(getActivity(), ReleaseOrderActivity.class);
        startActivity(intent);


    }

    @Override public void onDestroy() {
        super.onDestroy();
    }

}

```

以上便是根据我最近做的项目粗略的整理了一下，recyclerview的简单用法，在这里我也用到了，下拉刷新，上拉加载更多，等等等等的方法，写这些也是为了让自己在以后遇到同样的需求的时候，可以很快的写出，如果大家也有这些小问题的可以和我一起交流。


recyclerView和ListView的异同

---

* ViewHolder是用来保存视图引用的类，无论是ListView亦或是RecyclerView。只不过在ListView中，ViewHolder需要自己来定义，且这只是一种推荐的使用方式，不使用当然也可以，这不是必须的。只不过不使用ViewHolder的话，ListView每次getView的时候都会调用findViewById(int)，这将导致ListView性能展示迟缓。而在RecyclerView中使用RecyclerView.ViewHolder则变成了必须，尽管实现起来稍显复杂，但它却解决了ListView面临的上述不使用自定义ViewHolder时所面临的问题。
* 我们知道ListView只能在垂直方向上滚动，Android API没有提供ListView在水平方向上面滚动的支持。或许有多种方式实现水平滑动，但是请相信我，ListView并不是设计来做这件事情的。但是RecyclerView相较于ListView，在滚动上面的功能扩展了许多。它可以支持多种类型列表的展示要求，主要如下：

	1. LinearLayoutManager，可以支持水平和竖直方向上滚动的列表。
	2. StaggeredGridLayoutManager，可以支持交叉网格风格的列表，类似于瀑布流或者Pinterest。
	3. GridLayoutManager，支持网格展示，可以水平或者竖直滚动，如展示图片的画廊。

* 列表动画是一个全新的、拥有无限可能的维度。起初的Android API中，删除或添加item时，item是无法产生动画效果的。后面随着Android的进化，Google的Chat Hasse推荐使用ViewPropertyAnimator属性动画来实现上述需求。
相比较于ListView，RecyclerView.ItemAnimator则被提供用于在RecyclerView添加、删除或移动item时处理动画效果。同时，如果你比较懒，不想自定义ItemAnimator，你还可以使用DefaultItemAnimator。

* ListView的Adapter中，getView是最重要的方法，它将视图跟position绑定起来，是所有神奇的事情发生的地方。同时我们也能够通过registerDataObserver在Adapter中注册一个观察者。RecyclerView也有这个特性，RecyclerView.AdapterDataObserver就是这个观察者。ListView有三个Adapter的默认实现，分别是ArrayAdapter、CursorAdapter和SimpleCursorAdapter。然而，RecyclerView的Adapter则拥有除了内置的内DB游标和ArrayList的支持之外的所有功能。RecyclerView.Adapter的实现的，我们必须采取措施将数据提供给Adapter，正如BaseAdapter对ListView所做的那样。
* 在ListView中如果我们想要在item之间添加间隔符，我们只需要在布局文件中对ListView添加如下属性即可：

	```
	 android:divider="@android:color/transparent"
	 android:dividerHeight="5dp"
	```
* ListView通过AdapterView.OnItemClickListener接口来探测点击事件。而RecyclerView则通过RecyclerView.OnItemTouchListener接口来探测触摸事件。它虽然增加了实现的难度，但是却给予开发人员拦截触摸事件更多的控制权限。
* ListView可以设置选择模式，并添加MultiChoiceModeListener，如下所示：


```
listView.setChoiceMode(ListView.CHOICE_MODE_MULTIPLE_MODAL);
listView.setMultiChoiceModeListener(new MultiChoiceModeListener() {
public boolean onCreateActionMode(ActionMode mode, Menu menu) { ... }
public void onItemCheckedStateChanged(ActionMode mode, int position,
long id, boolean checked) { ... }
    public boolean onActionItemClicked(ActionMode mode, MenuItem item) {
        switch (item.getItemId()) {
            case R.id.menu_item_delete_crime:
            CrimeAdapter adapter = (CrimeAdapter)getListAdapter();
            CrimeLab crimeLab = CrimeLab.get(getActivity());
            for (int i = adapter.getCount() - 1; i >= 0; i--) {
                if (getListView().isItemChecked(i)) {
                    crimeLab.deleteCrime(adapter.getItem(i));
                }
          }
        mode.finish();
        adapter.notifyDataSetChanged();
        return true;
        default:
            return false;
}
    public boolean onPrepareActionMode(ActionMode mode, Menu menu) { ... }
    public void onDestroyActionMode(ActionMode mode) { ... }
});

```
	而RecyclerView则没有此功能。
	
　　

　　
　　
  　　

