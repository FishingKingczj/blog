---
title: 'Android 面试基础知识点整理'
layout: post
tags:
    - interview 
    - android
---
Android 面试中比较重要的知识点，主要包括  
四大组件，View，通信，消息，进程，性能，开源库等 
内容很长，主要是提炼问题的精华解答，对于一些比较重要的问题，之后会单独整理相关的源码分析  
另外复习过程中还有一些比较生僻或者比较小的点，在最后慢慢补充吧

<!--more-->

**目录**

* TOC
{:toc}

# Activity
## Activity 生命周期
![Activity生命周期](http://t1.dt8.co/o_1d6amlbu5qn2m9m1aorps41jlba.png-w.jpg)
onCreate、onDestroy  
onStart、onStop 代表着活动是否可见  
onResume、onPause 代表活动是否处于可交互状态  

**生命周期流程**  
- 启动A: `onCreate` -> `onStart` -> `onResume`
- A启动B：`A.onPause` -> `A.onStop` （如果B是透明的或者是对话框，则`A.onStop`不会调用）
- 启动B后，返回A：`onRestart` -> `onStart` -> `onResume`
- back键：`onPause` -> `onStop` -> `onDestroy`

**saveInstanceState**  
保存并恢复Activity状态：  
`onSaveInstanceState` （`onStop`之前调用，和`onPause`没有必然的先后顺序）  
`onRestoreInstanceState` （`onStart`之后调用，和`onResume`没有必然的先后顺序）  
调用条件：
- 活动因为内存不足被回收
- 配置改变（比如手机方向，添加android:configChanged属性后不会触发，会调用`onConfiguration`函数）

```java
@Override
public void onSaveInstanceState(Bundle savedInstanceState){
    super.onSaveInstanceState(savedInstanceState);
    savedInstanceState.putString("message", this.message.getText().toString());
}

@Override
public void onRestoreInstanceState(Bundle savedInstanceState){
    super.onRestoreInstanceState(savedInstanceState);
    this.message = savedInstanceState.getString("message");
}
```

## Activity 启动
**启动模式**  
- standard 标准模式
  - 每次启动会创建一个新的Activity
  - A启动B，B会位于A的栈中
  - 默认的启动模式

- singleTop 栈顶复用模式
  - 要启动的Activity在栈顶则直接使用，不创建新的Activity
  - 第二次启动在栈顶，会调用`onNewIntent`、`onResume`方法，`onCreate`、`onStart`不会调用

- singleTask 栈内复用模式
  - 要启动的Activity在栈内则直接使用，不创建新的Activity
  - 第二次启动在栈顶，会调用`onNewIntent`、`onResume`方法，`onCreate`、`onStart`不会调用

- singleInstance 单一实例模式
  - 要启动的Activity会新建一个栈，并且此Activity会**独占**这个栈

- 应用
  - standard：Activity默认模式
  - singleTop：当外界多次跳转到一个页面是可以使用这个模式，比如从一些下拉栏通知界面点击进入一个页面的情景，避免了因为多次启动导致的需要返回多次的情况
  - singleTask：可用于应用的主界面，外界多次启动时不会受子页面干扰，clearTop效果也会清除主页面之上的页面
  - singleInstance：可用于和程序分离的页面，比如通话页面、闹铃提醒页面

**onNewIntent()方法**  
用singletop，singleinstance，singletask三种启动模式的activity，被intent复用开启时会走`onNewIntent`方法  
按Home键退出，再长按Home键进入，此时`onNewIntent`不被访问，因为再次进入的时候没有被发起Intent  
按Home键后再点击手机屏幕图标打开这个应用，走`onNewIntent`方法，因为点击图标就是用intent开启应用

**Intent Flag**  
Android Intent 常用的Flag有以下几种，一般是组合使用
- FLAG_ACTIVITY_NEW_TASK  
  首先会查找是否存在和被启动的Activity具有相同的亲和性的Task，如果有，则直接把这个Task整体移动到前台，并保持Task中的**状态不变（无论Activity是否在Task顶）**，如果没有，则新建一个Task来存放被启动的activity
- FLAG_ACTIVITY_CLEAR_TOP  
  相当于singleInstance的用法
- FLAG_ACTIVITY_SINGLE_TOP  
  相当于singleTop的用法
- FLAG_ACTIVITY_CLEAR_TASK  
  跳转Activity所在的Task中所有Activity全部清除，然后创建要跳转的Activity

**affinity**  
affinity是指Activity的归属，也就是该Activity属于哪个Task，一般情况下在同一个应用中，启动的Activity都在同一个Task中。如果一个Activity没有显式的指明`taskAffinity`，那么它的这个属性就等于Application指明的`taskAffinity`，如果Application也没有指明，那么该`taskAffinity`的值就等于应用的包名。  
- 根据affinity重新为Activity选择宿主Task（与`allowTaskReparenting`属性配合使用）  
  Activity的`allowTaskReparenting`为true时，Activity就拥有了一个转移所在Task的能力。具体点来说，就是一个Activity现在是处于某个Task当中的，但是它与另外一个Task具有相同的affinity值，那么当另外这个任务切换到前台的时候，该Activity就可以转移到现在的这个任务当中。`allowTaskReparenting`默认是继承至application中的false。    
  >例子： 一个天气预报程序，它有一个用于显示天气信息的Activity，allowTaskReparenting属性设置成true，这个Activity和天气预报程序的所有其它Activity具体相同的affinity值。这个时候，你自己的应用程序通过Intent去启动了这个用于显示天气信息的Activity，那么此时这个Activity应该是和你的应用程序是在同一个任务当中的。但是当把天气预报程序切换到前台的时候，这个Activity会被转移到天气预报程序的任务当中，并显示出来。如果将你自己的应用切换到前台，发现你自己应用Task里的那个Activity消失了。

- 启动一个Activity过程中Intent使用了`FLAG_ACTIVITY_NEW_TASK`标记，根据affinity查找或创建一个新的具有对应affinity的task  
  当调用`startActivity()`方法来启动一个Activity时，默认是将它放入到当前的任务当中。但是，如果在Intent中加入了`FLAG_ACTIVITY_NEW_TASK`的话：
  1. 系统会去检查这个Activity的affinity是否与当前Task的affinity相同。如果相同的话就会把它放入到当前Task当中。
  2. 如果不同则会先去检查是否已经有一个名字与该Activity的affinity相同的Task，如果有，这个Task将被调到前台，同时这个Activity将显示在这个Task的顶端
  3. 如果没有的话，系统将会尝试为这个Activity创建一个新的Task。
  4. 如果一个Activity在manifest文件中声明的启动模式是singleTask，那么他被启动的时候，行为模式会和前面提到的指定`FLAG_ACTIVITY_NEW_TASK`一样。 

- 对启动模式的影响
  - standard/singleTop  
    MainActivity和TestActivity的taskAffinity不相同，但是它们仍然被放入同一个任务栈中。
  - singleTask
    - 当A和B的taskAffinity相同时：第一次创建B的实例时，并不会启动新的task，而是直接将B添加到A所在的task， 当B的实例已经存在时，将B所在task中位于B之上的全部Activity都删除，B就成为栈顶元素，实现跳转到B的功能。
    - 当A和B的taskAffinity不同时：第一次创建B的实例时，会启动新的task，然后将B添加到新建的task中， 当B的实例引进存在，将B所在task中位于B之上的全部Activity都删除，B就成为栈顶元素（也是root Activity），实现跳转到B的功能。

**IntentFilter匹配**  
Intent隐式启动的三个属性：action、category、data  
- action：代码中有一个及以上与xml过滤规则中的相同即可
- category：代码中所有的必须与xml过滤规则中的相同
- data：同action

>代码中隐式启动时，会默认添加android.intent.category.DEFAULT，所以xml必须含有此属性才能隐式启动  
在同一个应用内，能使用显示启动，就尽量使用显示启动，增加程序的效率

# Service

>**Service 运行在主线程中，它并不是一个新的线程，所以并不能执行耗时操作。如果要在 Service 中执行耗时操作，需要使用 Thread**

## 启动方式
- started  
  其它组件调用 `startService()` 启动一个 Service。一旦启动，Service 将一直运行在后台，即使启动这个 Service 的组件已经被销毁。通常一个被 start 的 Service 会在后台执行单独的操作，也并不需要给启动它的组件返回结果。只有当 Service 自己调用 `stopSelf()` 或者其它组件调用 `stopService()` 才会终止。
  1. 定义一个类继承于Service
  2. 在Manifest.xml文件中配置该Service
  3. 使用Context的startService(Intent)方法启动该Service
  4. 不再使用时使用stopService(Intent)方法停止该服务

- bind  
  其它组件可以调用 `bindService()` 来绑定一个 Service。这种方式会让 Service 和启动它的组件绑定在一起，当启动它的组件销毁的时候，Service 也会自动进行 unBind 操作。同一个 Service 可以被多个组件绑定，只有所有绑定它的组件都进行了 unBind 操作，这个 Service 才会被销毁。
  1. 定义一个类继承Service，创建一个继承与Binder的实例对象，并提供公共方法供客户端调用。
  2. 实现onBind()方法，返回Binder实例
  3. 在Manifest.xml文件中配置该Service
  4. 在客户端中，实现ServiceConnection实例，从onServiceConnected()回调方法接收Binder，并使用bindService绑定服务。注：onServiceDiscounnection方法是在服务崩溃或者服务杀死导致的连接中断时调用

## 生命周期
![Service生命周期](http://t1.dt8.co/o_1d6amljajvph1l0q1ur71gjb1ak1a.png-w.jpg)
- 完整生命周期（entire lifetime）  
  从 `onCreate()` 被调用，到 `onDestroy()` 返回。和 Activity 类似，一般在 `onCreate()` 方法中做一些初始化的工作，在 `onDestroy()` 中做一些资源释放的工作。
  >如，若 Service 在后台播放一个音乐，就需要在 `onCreate()` 方法中开启一个线程启动音乐，并在 `onDestroy()` 中结束线程。
- 活动生命周期（activity lifetime）   
  从 `onStartCommand()` 或 `onBind()` 回调开始，由相应的 `startService()` 或 `bindService()` 调用。start 方式的活动生命周期结束就意味着完整证明周期的结束，而 bind 方式，当 `onUnbind()` 返回后，Service 的活动生命周期结束。

## IntentService
1. 继承IntentService
2. 重写`onHandleIntent(Intent intent)`，根据intent参数不同执行不同的任务
3. 注册Service
4. `startService(startServiceIntent)`启动，可以启动多次，每启动一次，就会新建一个work thread，但IntentService的实例始终只有一个
5. 当所有请求处理完成后，自动停止 Service

```java
@Override
  protected void onHandleIntent(Intent intent) {
      //Intent是从Activity发过来的, 携带识别参数, 根据参数不同执行不同的任务
      String action = intent.getExtras().getString("param");
      if (action.equals("oper1")) {
          System.out.println("Operation1");
          // 耗时操作
      } else if (action.equals("oper2")) {
          System.out.println("Operation2");
          // 耗时操作
      }
  }
```

```java
public class TestActivity extends Activity {
    @Override
    public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.main);
        
        //可以启动多次, 每启动一次, 就会新建一个work thread, 但IntentService的实例始终只有一个
        //Operation 1
        Intent startServiceIntent = new Intent("com.test.intentservice");
        Bundle bundle = new Bundle();
        bundle.putString("param", "oper1");
        startServiceIntent.putExtras(bundle);
        startService(startServiceIntent);
        
        //Operation 2
        Intent startServiceIntent2 = new Intent("com.test.intentservice");
        Bundle bundle2 = new Bundle();
        bundle2.putString("param", "oper2");
        startServiceIntent2.putExtras(bundle2);
        startService(startServiceIntent2);
    }
}
```

## Service保活 
- 在`onStartCommand`方法中将flag设置为START_STICKY
- 注册时，在xml中设置android:priority
- 在`onStartCommand`方法中将进程设置为前台进程
- 在`onDestroy`方法中重启service
- 用`AlarmManager.setRepeating()`方法循环发送闹钟广播，接收的时候调用service的`onstart`方法

## Service与Activity通信
- Binder
- Broadcast

# Boardcast

## 使用方法
1. 创建BroadcastReceiver子类
2. 注册广播接收器
   - 静态注册，在AndroidManifest.xml 中，通过标签注册
   - 动态注册，通过调用Context的`registerReceiver()`方法即可动态注册BroadcastReceiver，用`unregisterReceiver()`销毁广播接收者（尽量在 `onPause()` 进行注销）
3. `sendBroadcast(intent)`发送广播接收者

```java
//实例化BroadcastReceiver子类 & IntentFilter
BroadcastReceiver broadcastReceiver = new BroadcastReceiver();
IntentFilter intentFilter = new IntentFilter();

//设置接收广播的类型
intentFilter.addAction(android.net.conn.CONNECTIVITY_CHANGE);

//调用Context的registerReceiver()方法进行动态注册
registerReceiver(BroadcastReceiver, intentFilter);
```

## 广播类型
- 普通广播（Normal Broadcast）

```java
  Intent intent = new Intent();
  //对应BroadcastReceiver中intentFilter的action
  intent.setAction(BROADCAST_ACTION);
  //发送广播
  sendBroadcast(intent);
```

- 系统广播（System Broadcast）  
  当使用系统广播时，只需要在注册广播接收者时定义相关的action即可，并不需要手动发送广播，当系统有相关操作时会自动进行系统广播。
- 有序广播（Ordered Broadcast）  
  有序广播指的是发送出去的广播被广播接收者按照先后顺序接收的广播。  
  广播接受者按照Priority属性值从大到小排序，Priority属性相同者，动态注册的广播优先。  
  先接收的广播接收者可以对广播进行截断，即后接收的广播接收者不再接收到此广播，先接收的广播接收者可以对广播进行修改，那么后接收的广播接收者将接收到被修改后的广播  
  有序广播的使用过程与普通广播非常类似，差异仅在于广播的发送方式为 `sendOrderedBroadcast(intent)`
- App应用内广播（Local Broadcast）  
  可理解为一种局部广播，广播的发送者和接收者都同属于一个App。它相对全局广播的优势在于：安全性高，效率高 。
  - 将全局广播设置成局部广播  
    1. 注册广播时将exported属性设置为false，使得非本App内部发出的此广播不被接收
    2. 在广播发送和接收时，增设相应权限permission，用于权限验证
    3. 发送广播时指定该广播接收器所在的包名，此广播将只会发送到此包中的App内与之相匹配的有效广播接收器中(使用Intent.setPackage方法)。
  - 使用封装好的LocalBroadcastManager类  
    使用方式上与全局广播几乎相同，只是注册/取消注册广播接收器和发送广播时将参数的context变成了`LocalBroadcastManager`的单一实例instance

# 数据存储
- SharedPreferences
- 文件存储    
- SQLite
- ContentProvider
- 网络存储

## SharedPreferences
SharedPreferences使用xml格式，以键值对的方式来存储数据。  
它存贮在文件系统的/data/data/your_app_package_name/shared_prefs/目录下，可以被处在同一个应用中的所有Activity 访问。

**使用**  
1. 通过Context的`getSharedPreferences(String name,int mode)`方法来获取SharedPreferences的实例  
   name 用于指定SharedPreferences文件的名称（格式为xml文件）  
   mode 用于指定操作模式  
   - MODE_PRIVATE：默认操作模式，表示只有当前应用程序才可以对这个文件进行读写
   - MODE_WORLD_READABLE：指定此文件对其他程序只读。
   - MODE_WORLD_WRITEABLE：指定此文件能被其他程序读写。
2. 调用SharedPreferences对象的`edit()`方法来获取一个`SharedPreferences.Editor`对象  
3. 使用SharedPreferences.Editor进行数据操作`putxxx()`, `getxxx()`, `remove`, `clear`
4. 调用`commit()`或`apply()`将添加的数据提交

**commit 和apply**  
- apply没有返回值而commit返回boolean表明修改是否提交成功 
- apply异步提交到硬件磁盘, 而commit是同步提交到硬件磁盘，因此，在多个并发的提交commit的时候，他们会等待正在处理的commit保存到磁盘后在操作，从而降低了效率。而apply只是原子的提交到内存，后面有调用apply的函数的将会直接覆盖前面的内存数据，这样从一定程度上提高了很多效率
- apply方法不会提示任何失败的提示
- 在一个进程中，sharedPreference是单实例，一般不会出现并发冲突，如果对提交的结果不关心的话，建议使用apply，当然需要确保提交成功且有后续操作的话，还是需要用commit的

## 文件存储   
Context 类中提供  
`openFileOutput()`方法，用于将数据存储到指定的文件中，返回一个 FileOutputStream 对象  
`openFileInput()` 方法，用于从文件中读取数据，返回一个 FileInputStream 对象。  
操作和Java文件读写一致

## SQLite

## ContentProvider
用于进程间进行数据交互和共享，即跨进程通信

**统一资源标识符（URI）**  
用于唯一标识ContentProvider和其中的数据，外界进程通过uri找到对应的ContentProvider，再进行数据操作  
 
`<srandard_prefix>://<authority>/<data_path>/<id>`  
`<srandard_prefix>`：ContentProvider的srandard_prefix始终是content://  
`<authority>`：ContentProvider的名称  
`<data_path>`：请求的数据类型  
`<id>`：指定请求的特定数据  

Android系统ContentProvider包括
- Browser：存储如浏览器的信息
- CallLog：存储通话记录等信息
- Contacts：存储联系人等信息
- MediaStore：存储媒体文件的信息
- Settings：存储设备的设置和首选项信息

**ContentProvider**  
ContentProvider需要在AndroidManifest.xml文件中进行配置。而且某个应用程序通过ContentProvider暴露了自己的数据操作接口，那么不管该应用程序是否启动，其他应用程序都可以通过这个接口来操作它的内部数据，也可以在配置中设置访问权限防止其他应用程序访问数据。

**ContentResolver**  
ContentResolver，内容访问者。可以通过ContentResolver来操作ContentProvider所暴露处理的接口。一般使用Content.getContentResolver()方法获取ContentResolver对象。  ContentResolver类对所有的ContentProvider进行统一管理。外部进程通过 ContentResolver类从而与ContentProvider类进行交互。  
ContentResolver的很多方法与ContentProvider一一对应。

# View

## 事件分发机制
**事件类型**  
1. ACTION_DOWN：手指按下屏幕的瞬间产生该事件
2. ACTION_MOVE：手指在屏幕上移动时候产生该事件
3. ACTION_UP：手指从屏幕上松开的瞬间产生该事件

**事件传递方法**  
- `dispatchTouchEvent(MotionEvent event)`  
   如果MotionEvent传递给了View，那么该方法一定会被调用  
   返回是否消费了当前事件。可能是View本身的`onTouchEvent`方法消费，也可能是子View的`dispatchTouchEvent`方法中消费。返回true表示事件被消费，本次的事件终止。返回false表示View以及子View均没有消费事件，将调用父View的`onTouchEvent`方法
- `onInterceptTouchEvent(MotionEvent ev)`
   事件拦截，当一个**ViewGroup**在接到MotionEvent事件序列时候，调用此方法判断是否需要拦截。特别注意，这是ViewGroup特有的方法，View并没有拦截方法  
   返回是否拦截事件传递，返回true表示拦截了事件，那么事件将不再向下分发而是调用View本身的onTouchEvent方法。返回false表示不做拦截，事件将向下分发到子View的dispatchTouchEvent方法。
- `onTouchEvent(MotionEvent ev)`
   真正对MotionEvent进行处理方法，在`dispatchTouchEvent`进行调用。
   返回true表示事件被消费，本次的事件终止。返回false表示事件没有被消费，将调用父View的`onTouchEvent`方法

**处理流程**  
1. 事件从`Activity.dispatchTouchEvent()`开始传递，只要没有被停止或者拦截，从最上层的view(viewGroup)开始一直往下(子view)传递。子view可以通过`onTouchEvent()`对事件进行处理。 
2. 事件由父view(viewGroup)传递给子view，viewGroup可以通过`onInterceptTouchEvent()`对事件进行拦截，停止其往下传递。 
3. 如果事件从上往下传递的过程中没有被拦截，最底层的view也没有消耗事件，事件会反向往上传递，这时父view(ViewGroup)可以进行消费，如果还是没有被消费的话，最后会到Activity的`onTouchEvent()`方法中处理。 
4. 如果 View 没有对 ACTION_DOWN 进行消费，之后的其他事件不会传递过来。 
5. OnTouchListener 优先于 `onTouchEvent()` 对事件进行消费。 

## View绘制流程

**流程**  
- Measure 用来计算 View 的实际大小  
  测量流程从 `performMeasure` 方法开始，分发给 ViewGroup ，由 ViewGroup 在它的 `measureChild` 方法中传递给子 View。ViewGroup 通过遍历自身所有的子 View，并逐个调用子 View 的 `measure` 方法实现测量操作。  
  `measure` 方法最终的测量是通过回调 `onMeasure` 方法实现的，可以通过重写这个方法实现自定义View的测量。
- Layout 用来确定 View 在父容器的布局位置  
  父容器获取子 View 的位置参数后，调用子 View 的 layout 方法并将位置参数传入  
  由`performLayout`调用`child.layout`，子类如果是 ViewGroup 类型，则重写 `onLayout` 方法，实现 ViewGroup 中所有 View 控件的布局
- Draw 用来将控件绘制出来  
  绘制的流程从 performDraw 方法开始，调用每个 View 的 draw 方法绘制每个具体的 View

### 自定义View
1. 在`onMeasure()`方法中，测量自定义控件的大小.
2. 在`onLayout()`方法中来确定控件显示位置。
3. 在`onDraw()`方法中，利用Canvas与Paint来绘制要显示的内容

**构造函数**  

```java
/**
  * 在new的时候调用
  */
public MyView(Context context) {
    this(context, null);
}

/**
  * 在布局中使用
  */
public MyView(Context context, @Nullable AttributeSet attrs) {
    this(context, attrs, 0);
}

/**
  * 布局layout中调用,但是会有style
  */
public MyView(Context context, @Nullable AttributeSet attrs, int defStyleAttr) {
    super(context, attrs, defStyleAttr);
}
```

**`onMeasure()`**  

自定义view的测量方法，返回View的实际大小  
```java
@Override
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    super.onMeasure(widthMeasureSpec, heightMeasureSpec);
    //布局的宽高都由这个方法指定
    //指定控件的宽高,需要测量
    //获取宽高的模式
    int widthMode = MeasureSpec.getMode(widthMeasureSpec);
    int heightMode = MeasureSpec.getMode(heightMeasureSpec);
    /**
      * MeasureSpec.AT_MOST : 在布局中指定 wrap_content
      * MeasureSpec.EXACTLY : 在布局中指定确切的值 100dp   match_parent fill_parent
      * MeasureSpec.UNSPECIFIED : 尽可能的大,很少能用到，ListView, ScrollView 在测量子布局的时候会用UNSPECIFIED
      */
    //获取宽高的大小
    int widthSize = MeasureSpec.getSize(widthMeasureSpec);
    int heightSize = MeasureSpec.getSize(heightMeasureSpec);
    // ... 进行操作重新计算width和height
    setMeasuredDimension(newwidth, newheight);
}
```

**`onLayout()` (ViewGroup)**

```java
// layout的作用是ViewGroup用来确认子元素的位置
public void layout(int l, int t, int r, int b) {
  // 初始化mLeft mRight mBottom mTop四个值，这一步完成后，view在父容器中的位置就确定下来了
   boolean changed = isLayoutModeOptical(mParent) ?
                setOpticalFrame(l, t, r, b) : setFrame(l, t, r, b);
  // 接着会调用onLayout方法，在onLayout中去循环遍历子元素，确定子元素的位置
  onLayout(changed, l, t, r, b);
}

/**
当ViewGroup的位置被确定后，它在onLyaout中会遍历所有的子元素并调用其layout方法，在layout方法中又会调用onLayout方法并调用每一个子view的layout()方法，把之前在onMeasure()里保存下来的它们的位置作为参数传进去，然它们进行自我布局
*/
/**
  * 实现一个子View上下居中，分隔20dp排布的ViewGroup
  * @param changed 当前View的大小和位置改变了 
  * @param l,t,r,b 
  * 放置父控件的矩形可用空间（除去margin和padding的空间）
  * 的左上角的left、top以及右下角right、bottom值
  */
@Override  
protected void onLayout(boolean changed, int l, int t, int r, int b) {
  int padding = 20;
  // 保存到父View左侧的距离
  int mLeft = padding;
  
  // 循环所有子View
  for (int i = 0; i < getChildCount(); i++) {  
    View child = getChildAt(i);  
    // 取出当前子View长宽  
    int childViewHeight = childView.getMeasuredHeight();
    int childViewWidth = childView.getMeasuredWidth();
    // 让子View在竖直方向上显示在父View中间位置
    int mTop = (t + b - childViewHeight) / 2;
    // 调用layout给每一个子View设定位置l,t,r,b
    childView.layout(mLeft, mTop, mLeft + childViewWidth, mTop
        + childViewHeight);
    // 改变下一个子View到父View左侧的距离
    mLeft += childViewWidth;
  }  
}  
```

> `getLeft(), getTop(), getRight(), getBottom`  
`getWidth() = getRight() - getLeft()`  
`getHeight() = getBottom() - getTop()`  

>`getWidth/Height()`和`getMeasuredWidth/Height()`的区别： 
`getWidth()`, `getLeft()`等这些函数是View相对于其父View的位置。而`getMeasuredWidth()`, `getMeasuredHeight()`是测量后该View的实际值  
实际上在当屏幕可以包裹内容的时候，他们的值是相等的，只有当View超出屏幕后，才能看出他们的区别：`getMeasuredHeight()`是实际View的大小，与屏幕无关，而`getHeight`的大小此时则是屏幕的大小。当超出屏幕后，`getMeasuredHeight()`等于`getHeight()`加上屏幕之外没有显示的大小

**`onDraw()` 使用canvas/paint等工具绘制View**

```java
@Override
protected void onDraw(Canvas canvas) {
    super.onDraw(canvas);
    // 使用canvas/paint等工具绘制View
}
```

**自定义属性**

```xml
<resources>
  <!--name 自定义View的名字 TextView-->
  <declare-styleable name="TextView">
    <!-- name 属性名称
      format 格式: string 文字 color 颜色 dimension 宽高,字体大小
      integer 数字 reference 资源(drawable)-->
    <attr name="text" format="string" />
    <attr name="textColor" format="color" />
    <attr name="textSize" format="dimension" />
    <attr name="maxLength" format="integer" />
    <attr name="background" format="reference|color" />
    <!-- 枚举 -->
    <attr name="inputType">
      <enum name="number" value="1" />
      <enum name="text" value="2" />
      <enum name="password" value="3" />
    </attr>
  </declare-styleable>
</resources>
```

# Fragment
**特点**  
- 模块化（Modularity）：我们不必把所有代码全部写在Activity中，而是把代码写在各自的Fragment中。
- 可重用（Reusability）：多个Activity可以重用一个Fragment。
- 可适配（Adaptability）：根据硬件的屏幕尺寸、屏幕方向，能够方便地实现不同的布局，这样用户体验更好。

**使用**  
1. 创建继承Fragment的类，重写`onCreateView()`
2. 把Fragment添加到Activity中  
    - 静态添加：在xml中通过`<fragment>`的方式添加，缺点是一旦添加就不能在运行时删除。
    - 动态添加：运行时添加，这种方式比较灵活，因此建议使用这种方式。

**生命周期**  
![Fragment生命周期](http://t1.dt8.co/o_1d6amfooh9a21ar51j6o1tqs1fmfc.png-w.jpg)

对应Activity的onCreate()  
- onAttach()：Fragment和Activity相关联时调用。可以通过该方法获取 Activity引用，还可以通过getArguments()获取参数。
- onCreate()：Fragment被创建时调用。
- onCreateView()：创建Fragment的布局。
- onActivityCreated()：当Activity完成onCreate()时调用。

对应Activity的onStart()
- onStart()：当Fragment可见时调用。

对应Activity的onResume()
- onResume()：当Fragment可见且可交互时调用。

对应Activity的onPause()
- onPause()：当Fragment不可交互但可见时调用。

对应Activity的onPause()
- onStop()：当Fragment不可见时调用。

对应Activity的onPause()
- onDestroyView()：当Fragment的UI从视图结构中移除时调用。
- onDestroy()：销毁Fragment时调用。
- onDetach()：当Fragment和Activity解除关联时调用。

**通信**  
- Fragment to Activity  
  使用Fragment接口的方式进行通信
- Activity to Fragment  
  获取Fragment对象，并调用Fragment的方法即可
- Fragment之间  
  以Activity为中介进行通信

# ListView/RecyclerView
主要是从View/GroupView的绘制/测量、内存角度考虑

## ListView优化
- getView方法中尽量少使用逻辑，少创建对象
- 降低item的布局的深度，减少测量与绘制
- convertView 缓存已经加载好的View
- ViewHolder缓存控件的实例

## RecyclerView
- 设置 item 等高，减少测量次数
- 使用RecycledViewPool指定缓存大小
- 避免创建过多对象
- 局部刷新

# 动画
- View 动画
- 帧动画
- 属性动画

# Context
在加载资源、启动一个新的Activity、获取系统服务、获取内部文件路径、创建View操作时等都需要Context的参与  
Context字面意思上下文，或者叫做场景，是对一系列系统服务接口的封装，包括：内部资源、包、类加载、I/O操作、权限、主线程、IPC和组件启动等操作的管理

## Context 的使用
![Context](http://t1.dt8.co/o_1d6amfenc1h5njga1k11139o1d1af.png-w.jpg)
>1. 启动Activity在这些类中是可以的，但是需要创建一个新的task。一般情况不推荐。
>2. 在这些类中去layout inflate是合法的，但是会使用系统默认的主题样式，如果你自定义了某些样式可能不会被使用。 
>* ContentProvider、BroadcastReceiver之所以在上述表格中，是因为在其内部方法中都有一个context用于使用。

## Context造成内存泄漏
- 涉及各种系统、APP资源，使用的地方太多，在流程和引用关系上造成混乱，比如：异步的网络请求、View动画等，在流程上处理不当，导致无法释放  
- 杂乱无章的传值，让很多对象都Hold一份Context，开辟了过多无法回收的资源  
- Drawable对象、Bitmap对象回收不及时，甚至与View绑定  
- 单例的滥用，导致Context长期被引用无法释放

解决方案
- 没必要传值的时候，尽量使用Application的Context，这样保证Context即可以全局使用，又不会创建多份
- 在与Activity等组件耦合，必须要使用Activity的Context的时候，考虑使用弱引用，避免循环持有Context

# IPC 跨进程通信

## Serializable / Parcelable / Binder
- `Serializable`是Java提供的一个序列化接口，它是一个空接口，为对象标准的序列化和反序列化操作。使用`Serializable`来实现序列化相当简单。  
接口
- `Parcelable`是Android中的序列化方式，内部包装了可序列化的数据，可以在Binder中自由传输，在序列化过程中需要实现的功能有序列化、反序列化和内容描述序列化功能有`writeToParcel`方法来完成，最终是通过`Parcelable`中的一系列`write`方法来完成的。用法如下
- Binder是Android中的一个类，实现了IBinder接口。
  - 从IPC角度来说，Binder是Android中的一种跨进程通信方式，Binder还可以理解为一种虚拟的物理设备，它的设备驱动是/dev/binder，该通信方式在Linux中没有。
  - 从Android Framework角度来说，Binder是ServiceManager连接各种Manager(ActivityManager、WindowManager等等)和相应ManagerService的桥梁。
  - 从Android应用层来说，Binder是客户端和服务端进行通信的媒介，当bindService的时候，服务端会返回一个包含了服务端业务调用的Binder对象，通过Binder对象，客户端就可以获取服务端提供的服务或者数据，这里的服务包括普通服务和基于AIDL的服务。

## 通信方法
- Bundler  
  在Intent中传递Bundle数据，由于Bundle实现了Parcelable接口，所以它可以方便地在不同的进程间传输。
- 文件共享  
  两个进程间通过读/写同一个文件来交换数据，比如A进程把数据写入文件，B进程通过读取这个文件来获取数据。
- Messenger  
  Messenger可以在不同进程中传递Message对象，在Message中放入我们需要传递的数据，就可以轻松地实现数据的进程间传递。底层实现是AIDL。
- AIDL
- ContentProvider  
  ContentProvider的底层实现同样也是Binder。
- Socket

# 消息机制
### Handler
- 创建Handler  
  handler创建时会关联一个looper，默认的构造方法将关联当前线程的looper
- 重写`handleMessage(Message msg)`
  用于处理收到message时Handler的执行任务，将UI处理过程写在这里
- 发送消息  
  使用 `post(Runnable)`, `postAtTime(Runnable, long)`, `postDelayed(Runnable, long)`, `sendEmptyMessage(int)`, `sendMessage(Message)`, `sendMessageAtTime(Message, long)`, `sendMessageDelayed(Message, long)`这些方法向MessageQueue上发送消息，可以post一个Runnable对UI进行处理，Runnable还是在UI主线程中运行的，而不会另外开启线程运行

### Looper/MessageQueue
- 每个线程有且最多只能有一个Looper对象
- Looper内部有一个 MessageQueue，loop()方法调用后线程开始不断从队列中取出消息执行
- Looper使一个线程变成Looper线程
- Activity的生命周期都是依靠主线程的Looper.loop，当收到不同Message时采用相应措施，一旦退出消息循环，程序也就退出了

# 多线程

## Handler + Thread / HandlerThread
即使用Thread进行异步任务，在[Handler](#Handler)中更新UI   
HandlerThread对这个过程进行了封装，本质原理是相同的

## AsyncTask

```java
// 三种泛型类型分别代表 启动任务执行的输入参数后, 台任务执行的进度, 后台计算结果的类型
public abstract class AsyncTask<Params, Progress, Result>
```

**使用**  
1. `execute()`，执行一个异步任务，调用方法。
2. `onPreExecute()`，在`execute()`被调用后立即执行，一般用来在执行后台任务前对UI做一些标记。
3. `doInBackground()`，在`onPreExecute()`完成后立即执行，用于执行较为费时的操作，此方法将接收输入参数和返回计算结果。在执行过程中可以调用`publishProgress()`来更新进度信息。
4. `onProgressUpdate()`，在调用`publishProgress()`时，此方法被执行，直接将进度信息更新到UI组件上。
5. `onPostExecute()`，当后台操作结束时，此方法将会被调用，计算结果将做为参数传递到此方法中，直接将结果显示到UI组件上。

**注意点**  
- 异步任务的实例必须在UI线程中创建。
- `execute()`方法必须在UI线程中调用。
- 不要手动调用`onPreExecute()`，`doInBackground()`，`onProgressUpdate()`，`onPostExecute()`
- 不能在`doInBackground()`中更新UI
- 一个任务实例只能执行一次，如果执行第二次将会抛出异常
- 注意内存泄漏

**并行执行**    
AsyncTask默认串行执行，通过一个阻塞队列BlockingQuery<Runnable>存储待执行的任务  
可以通过asyncTask.executeOnExecutor()，利用静态线程池THREAD_POOL_EXECUTOR提供一定数量的线程  进行并发执行

## ThreadPoolExecutor
线程池，适合一组异步任务

**意义**  
1. 重用线程池中的线程， 避免因为线程的创建和销毁所带来的性能开销
2. 有效控制线程池中的最大并发数，避免大量线程之间因为相互抢占系统资源而导致的阻塞现象
3. 能够对线程进行简单的管理，可提供定时执行和按照指定时间间隔循环执行等功能

**使用**  
ThreadPoolExecutor 间接实现了Executor接口。提供了一系列参数来配置线程池，通过不同的参数配置实现不同功能特性的线程池  
android中的线程池都是直接或是间接通过配置 ThreadPoolExecutor 来实现的  
Executors类提供了4个工厂方法用于创建4种不同特性的线程池：  
- `FixedThreadPool`
  只有核心线程数，并且没有超时机制，因此核心线程即使闲置时，也不会被回收，因此能更快的响应外界的请求
- `CachedThreadPool`
  没有核心线程，非核心线程数量没有限制，超时为60秒。适用于执行大量耗时较少的任务，当线程闲置超过60秒时就会被系统回收掉，当所有线程都被系统回收后，它几乎不占用任何系统资源
- `ScheduledThreadPool`
  核心线程数是固定的，非核心线程数量没有限制， 没有超时机制，主要用于执行定时任务和具有固定周期的重复任务
- `SingleThreadExecutor`
  只有一个核心线程，并没有超时机制，意义在于统一所有的外界任务到一个线程中， 这使得在这些任务之间不需要处理线程同步的问题

# 图片缓存
三级缓存：  
- 内存缓存，LruCache
- 本地缓存，DiskLruCache
- 网络缓存
- 图片加载，实现一个Loader

使用Glide可以简单地实现图片的三级缓存

# OOM 内存泄漏
## 单例造成的内存泄露
单例的生命周期等同于应用程序的生命周期，不正确地使用单例模式会造成内存泄露。  
**解决方法**
1. 使单例模式引用的对象的生命周期等同于应用生命周期，包括引用context.getApplicationContext()
2. 传递局部生命周期对象时，尽量使用弱引用（WeakReference）

## 非静态内部类 / 匿名类
非静态内部类自动获得外部类的强引用，而且它的生命周期甚至比外部类更长，因此在其中持有局部生命周期的对象就会导致内存泄漏。比如：
- 匿名AsyncTask
- 一个匿名的 Thread，进行耗时的操作，如果 Activity 被销毁而 Thread 中的耗时操作没有结束的话，便会产生内存泄露
- 一个匿名的 Handler，采用 sendMessageDelayed() 方法来发送消息，这时如果 MainActivity 被销毁，而 Handler 里面的消息还没发送完毕的话，Activity 的内存也不会被回收

**解决方法**  
将非静态内部类和匿名类变成静态内部类

## 集合类  
集合类添加元素后，仍引用着集合元素对象，导致该集合中的元素对象无法被回收，从而导致内存泄露  
**解决方法**  
在集合元素使用之后从集合中删除，等所有元素都使用完之后，将集合置空。

## 其他情况
1. 需要手动关闭的对象没有关闭  
  - 网络、文件等流忘记关闭  
  - 手动注册广播时，退出时忘记 unregisterReceiver()  
  - Service 执行完后忘记 stopSelf()  
  - EventBus 等观察者模式的框架忘记手动解除注册  
2. static 关键字修饰的成员变量
3. ListView 的 Item 泄露

## 检查工具
Leakcanary

# ANR 程序无响应
- InputDispatching Timeout：5秒内无法响应屏幕触摸事件或键盘输入事件
- BroadcastQueue Timeout ：在执行前台广播（BroadcastReceiver）的onReceive()函数时10秒没有处理完成，后台为60秒。
- Service Timeout ：前台服务20秒内，后台服务在200秒内没有执行完毕。
- ContentProvider Timeout ：ContentProvider的publish在10s内没进行完。

**解决方案**  
- 尽量避免在主线程（UI线程）中作耗时操作
- traces.txt文件分析

# 开源库
## Retrofit
一个网络加载框架。底层使用OKHttp封装  
- 包含特别多注解，方便简化代码量
- 解耦
- 支持同步、异步和RxJava
- 可以配置不同的反序列化工具来解析数据，如json/xml
- 请求速度快，使用非常方便灵活
- 支持很多的开源库

## RxAndroid
RxAndroid是RxJava的Android扩展，是一个基于可观测序列组成的异步的、基于事件的库。这个异步库可以让我们用非常简洁的代码来处理复杂数据流或者事件。  
它是通过一种扩展的观察者模式，使用Observer和Observable来实现的

## EventBus
用于Android的事件发布-订阅总线，简化了应用程序内各个组件之间进行通信和跨进程通信的复杂度，尤其是碎片之间进行通信的问题，可以避免由于使用广播通信而带来的诸多不便

三个角色：  
- Event：事件
- Subscriber：事件订阅者
- Publisher：事件的发布者，可以在任意线程里发布事件

四种线程模型：
- POSTING：事件处理函数的线程跟发布事件的线程在同一个线程。
- MAIN：表示事件处理函数的线程在主线程(UI线程)。
- BACKGROUND：表示事件处理函数的线程在后台线程，因此不能进行UI操作。如果发布事件的线程是主线程(UI线程)，那么事件处理函数将会开启一个后台线程，如果果发布事件的线程是在后台线程，那么事件处理函数就使用该线程。
- ASYNC：表示无论事件发布的线程是哪一个，事件处理函数始终会新建一个子线程运行，同样不能进行UI操作。

## Picasso/Glide/Fresco
图片加载框架，提供图片缓存，异步加载，变换等功能

# MVP/MVVC
## MVP
MVP框架由三部分组成：
- View负责显示
- Presenter负责逻辑处理
- Model提供数据。  
最主要的变化就是将Activity中负责业务逻辑的代码移到Presenter中，Activity只充当MVP中的View，负责界面初始化以及建立界面控件与Presenter的关联。  
但也有一些缺点，比如：
- Model到Presenter的数据传递过程需要通过回调
- View(Activity)需要持有Presenter的引用，同时，Presenter也需要持有View(Activity)的引用，增加了控制的复杂度；
- MVC中Activity的代码很臃肿，转移到MVP的Presenter中，同样造成了Presenter在业务逻辑复杂时的代码臃肿。

## MVVM
View和Model之间通过Android Data Binding技术，实现视图和数据的**双向绑定**  
ViewModel持有Model的引用，通过Model的方法请求数据，获取数据后，通过Callback的方式回到ViewModel中，由于ViewModel与View的双向绑定，使得界面得以实时更新。同时，界面输入的数据变化时，由于双向绑定技术，ViewModel中的数据得以实时更新。  
在Android中实现MVVM架构的核心支撑技术是[Data binding](https://github.com/LyndonChin/MasteringAndroidDataBinding)技术。  
但是，Data Binding依然有很多支持的不好的组件（listview，recyclerView等），不可能通过给所有组件绑定ViewModel从而实现Activity没有业务逻辑操作。另外，ViewModel获取数据之后，也不可能把所有数据直接绑定到界面，有些也需要通过回调传到Activity中。

# 其他
## Kotlin
## WebView
## 屏幕适配/旋转