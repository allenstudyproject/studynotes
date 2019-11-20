一、应用启动流程
=============

应用进程不存在的情况下，从点击桌面应用图标，到应用启动（冷启动），大概会经历以下流程：

>1. Launcher startActivity
>2. AMS startActivity
>3. Zygote fork 进程
>4. ActivityThread main()
> >4.1 ActivityThread attach
> >
> >4.2 handleBindApplication
> >
> >**4.3 attachBaseContext**
> >
> >4.4 installContentProviders
> >
> >**4.5 Application onCreate**
>5. ActivityThread 进入loop循环
>6. **Activity生命周期回调，onCreate、onStart、onResume**

整个启动流程我们能干预的主要是 4.3、4.5 和6，应用启动优化主要从这三个地方入手。理想状况下，这三个地方如果不做任何耗时操作，那么应用启动速度就是最快的，但是现实很骨感，很多开源库接入第一步一般都是在Application onCreate方法初始化，有的甚至直接内置ContentProvider，直接在ContentProvider中初始化框架，不给你优化的机会。

二、启动分类
==========

- 冷启动

当启动应用时，后台没有该应用的进程（常见如：进程被杀、首次启动等），这时系统会重新创建一个新的进程分配给该应用.

- 暖启动

当启动应用时，后台已有该应用的进程（常见如：按back键、home键，应用虽然会退出，但是该应用的进程是依然会保留在后台，可进入任务列表查看），所以在已有进程的情况下，这种启动会从已有的进程中来启动应用

- 热启动

相比暖启动，热启动时应用做的工作更少，启动时间更短。热启动产生的场景很多，常见如：用户使用返回键退出应用，然后马上又重新启动应用.

热启动和暖启动因为会从已有的进程中来启动，不会再创建和初始化Application

三、启动优化

>* 闪屏页优化
>* MultipDex优化
>* 第三方库懒加载
>* 线程优化

3.1 闪屏页优化

消除启动时的白屏/黑屏，市面上大部分App都采用了这种方法，非常简单，是一个障眼法，不会缩短实际冷启动时间

styles.xml 增加一个主题叫AppThemeWelcome

    <style name="AppThemeWelcome" parent="Theme.AppCompat.NoActionBar">
      ...
        <item name="android:windowBackground">@drawable/logo</item>  <!-- 默认背景-->
    </style>

闪屏页设置这个主题，或者全局给Application设置

    <activity android:name=".ui.activity.DemoSplashActivity"
            android:configChanges="orientation|screenSize|keyboardHidden"
            android:theme="@style/AppThemeWelcome"
            android:screenOrientation="portrait">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />

                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>


这样的话启动Activity之后背景会一直在，所以在Activity的onCreate方法中切换成正常主题

    protected void onCreate(@Nullable Bundle savedInstanceState) {
      setTheme(R.style.AppTheme); //切换正常主题
      super.onCreate(savedInstanceState);

3.2 MultiDex 优化

在Android 4.4的机器MultiDex.install(context)会耗时

知道了MultiDex原理之后，可以理解install过程为什么耗时，因为涉及到解压apk取出dex、压缩dex、将dex文件通过反射转换成DexFile对象、反射替换数组。

在主进程Application 的 attachBaseContext 方法中判断如果需要使用MultiDex，则创建一个临时文件，然后开一个进程（LoadDexActivity），显示Loading，异步执行MultiDex.install 逻辑，执行完就删除临时文件并finish自己。
主进程Application 的 attachBaseContext 进入while代码块，定时轮循临时文件是否被删除，如果被删除，说明MultiDex已经执行完，则跳出循环，继续正常的应用启动流程。
注意LoadDexActivity 必须要配置在main dex中。

3.3 第三方库懒加载

很多第三方开源库都说在Application中进行初始化，十几个开源库都放在Application中，肯定对冷启动会有影响，所以可以考虑按需初始化，例如Glide，可以放在自己封装的图片加载类中，调用到再初始化，其它库也是同理，让Application变得更轻。

这种方式一般是在主页空闲的时候，将其它页面的数据加载好，保存到内存或数据库，等到打开该页面的时候，判断已经预加载过，直接从内存或数据库读取数据并显示。

3.4 线程优化

线程是程序运行的基本单位，线程的频繁创建是耗性能的，所以大家应该都会用线程池。单个cpu情况下，即使是开多个线程，同时也只有一个线程可以工作，所以线程池的大小要根据cpu个数来确定。


