# Android 面试

#### 四大组件

- Activity

  1. Activity生命周期

     onCreate() - onStart() - onResume() - onPause() - onStop() - onDestroy()

  2. Activity启动模式

     Standard  SingleTop  SingleTask  SingleInstance

  3. Fragment的作用（开放式）

     模块化，解耦，复用等

- Service

  1. Service生命周期

     onCreate() - onStartCommand() - onDestroy()

     onCreate() - onBind() - onUnbind() - onDestroy()

  2. Activity与Service间的通讯

     1. Binder
     2. Broadcast

  3. Service与Thread的区别

     两者不是一类东西

     1. Service其实是运行在主线程中的，如果需要执行复杂耗时的操作，必须在Service中再创建一个Thread来执行任务
     2. Service的优先级高于后台挂起的Activity，系统可能在内存不足的时候优先杀死后台的Activity或者Thread，而不会轻易杀死Service组件
     3. Thread只是一个用来执行后台任务的工具类，它可以在Activity中被创建，也可以在Service中被创建

- ContentProvider

- BroadcastReceiver

  1. BroadcastReceiver的两种注册方式及区别

     静态注册 动态注册

#### 布局方式

- LinerLayout

- TableLayout

- FrameLayout

- RelativeLayout

- GridLayout

- AbsoluteLayout

- PercentFrameLayout

- PercentRelativeLayout

  PercentFrameLayout，PercentRelativeLayout是Google新添加的布局方式，可以在一定程度上解决Android页面适配问题，不常用。可以考察面试者是否会主动学习这些技术。

#### UI适配

1. 使用dp而非px
2. 尽量使用相对布局，使用相对位置确定View之间的关系
3. 可以使用百分百布局
4. 等...

#### 自定义View

1. View的绘制流程
2. Touch事件的传递机制

#### 数据存储

- SharedPreference

- 文件

- SQLite

  - SQLite数据库框架使用

    OrmLite、GreenDao、Room等

- ContentProvider

- 网络

  - 网络请求框架使用

    Volley、OkHttp、Retrofit、Retrofit2.0+RxJava2.0

#### 内存泄露

- 根本原因

  长生命周期对象持有短生命周期对象，导致短生命周期对象无法被释放

- 常见场景

  1. 单例导致内存泄露

     单例中持有短生命周期对象(比如：Activity)

  2. 静态变量导致内存泄露

     静态变量属性中包含短生命周期对象(比如：Activity)

  3. 非静态内部类导致内存泄露

     非静态内部类会持有外部类的引用，当非静态内部类对象的生命周期比外部类对象的生命周期长时（比如非静态内部类接受某种回调而被另外的类持有），就会导致内存泄露。

  4. 未取消注册或回调导致内存泄露

  5. 集合中的对象未清理造成内存泄露

  6. 资源未关闭或释放导致内存泄露

  7. 等

#### 进程间通讯(IPC)

- Bundle
- Messenger
- AIDL
- ContentProvider
- Broadcast
- Socket

#### 并发和多线程

同步和锁

#### 使用过的框架

1. Retrofit
2. RxJava
3. 等

#### 设计架构思想

1. Java常用设计模式
   - 单例模式
   - 建造者模式
   - 观察者模式
   - 等
2. 设计模式的原则
   - 开闭原则：实现热插拔，提高扩展性
   - 里氏代换：实现抽象的规范，实现子父类互相替换
   - 依赖倒转：针对接口编程，实现开闭原则的基础
   - 接口隔离：降低耦合度，接口单独设计，互相隔离
   - 最迪米特法则：又称不知道原则，功能模块尽量独立
   - 合成复用原则：尽量使用聚合，组合，而不是继承
3. Android的MVC，MVP，MVVM架构