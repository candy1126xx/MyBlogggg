---
layout:     post                    # 使用的布局
title:      ActivityManagerService               # 标题 
subtitle:   ActivityManagerService #副标题
date:       2017-03-18              # 时间
author:     candy1126xx                      # 作者
header-img: img/home-bg-o.jpg    #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签
    - Android框架层
---

## 简介
AMS运行于SystemServer进程。负责管理全局的“窗口”、“应用服务”、“内容提供者”、“广播接收者”组件。同时它作为组件和应用进程的桥梁，使得开发者不必管理应用进程的创建与结束。

## 启动后的准备工作
1. 获取SystemServer的Context和ActivityThread作为AMS的运行环境；
2. 创建ServiceThread线程和对应的MainHandler；即工作线程；
3. 创建UiHandler；主要功能是显示各种系统对话框；
4. 创建一些用来管理服务和广播的集合；
5. 启动ProcessStateService；
6. 启动AppOpsService；
7. 设置一些参数；
8. 创建ResentTask组件；
9. 创建ActivityStackSupervisor组件；
10. 创建TaskPersister组件；

## 管理应用进程
#### 数据结构
在AMS中，应用进程信息被封装为ProcessRecord对象；并用3个集合管理所有应用进程的ProcessRecord对象。

* mProcessNames(ArrayMap<String,SparseArray<ProcessRecord>>)：其中ArrayMap的键是进程名，SparseArray的键是uid，值是ProcessRecord；它保存着系统中所有应用进程(包括已启动和将要启动)的ProcessRecord；
* mPidsLocked(SparseArray<ProcessRecord>)：其中SparseArray的键是pid，值是ProcessRecord；它保存着系统中所有已启动的进程的ProcessRecord；
* mLruProcesses(ArrayList)：保存了已启动进程的ProcessRecord，用于Lru算法。其中元素按成员变量lruWeight从小到大排列；lruWeight是调用updateLruProcessLocked()的时间 - 偏移量；偏移量从小到大：有Activity < 没有Activity有ContentProvider < 什么都没有。

#### 创建进程
在启动Activity的流程中，AMS先用PMS构建目标Activity的ActivityRecord，其中包含了目标进程的“processName”和“uid”；然后根据“processName”和“uid”在AMS的成员变量mProcessNames中查找对应的ProcessRecord，如果能找到对应的ProcessRecord，并且ProcessRecord的成员变量thread不为空，即AMS向目标进程发送信息的客户端ApplicationThreadProxy存在，则不需要启动新进程。

否则调用Process.start()启动新进程；当新进程启动后，新进程会向AMS发送信息“我启动了，并告诉你我的processName和uid”；AMS根据processName和uid找到刚才创建的ProcessRecord，读取应用进程信息封装为AppBindData，并把ProcessRecord加入mLruProcess；然后向应用进程发送信息“你的进程信息是AppBindData”；应用进程根据AppBindData创建Application，并调用Application.onCreate()。

#### 结束进程
在创建进程之后，和在销毁Activity的流程中，会调用updateOomAdjLocked()，按lruWeight从大到小(即倒序)遍历mLruProcesses，计算每个ProcessRecord的adj值，并把adj超过ProcessList.HIDDEN_APP_MIN_ADJ的记为“HIDDEN”；当“HIDDEN”的个数超过mProcessLimite(15)时，再来就直接杀掉。ADJ从小到大：有Activity在前台 < 正在处理广播 < 正在处理Service < 有Activity在后台 < 什么组件都没有。

## 管理Activity
#### 如何组织全局Activity？
DisplayContent就是屏幕；一个DisplayContent中有两个Stack，一个时桌面所在的HomeStack，另一个是应用进程所在的ActivityStack，它们的调度规则是：如果有Task被调度到前台，就把ActivityStack调度到前台；否则，把HomeStack调度到前台。

#### 任务Task
任务Task组织了一系列Activity，这些Activity可以属于一个应用，也可以不属于；同时，一个应用的Activity可以集中在1个Task，也可以分布在多个Task。这主要是为了弥补移动端大多情况只能显示1个窗口带来的麻烦。

#### taskAffinity属性
这个参数标识了一个Activity所需TaskRecord的名字，默认情况下，所有Activity所需的TaskRecord的名字为应用的包名；

一个TaskRecord的名字决定于这个TaskRecord的root activity的taskAffinity；

在概念上，具有相同的affinity的activity（即设置了相同taskAffinity属性的activity）属于同一个TaskRecord；
为一个activity的taskAffinity设置一个空字符串，表明这个activity不属于任何TaskRecord；

taskAffinity属性不对standard和singleTop模式有任何影响，即时你指定了该属性为其他不同的值，这两种启动模式下不会创建新的task。

#### 启动Activity
启动Activity有4种模式：

1. standard：默认模式，会在启动时创建一个新实例；
2. singleTop：当前栈中已有该Activity的实例并且该实例位于栈顶时，不会新建实例，而是复用栈顶的实例，并且会将Intent对象传入，回调onNewIntent方法；当前栈中已有该Activity的实例但是该实例不在栈顶时，其行为和standard启动模式一样，依然会创建一个新的实例；当前栈中不存在该Activity的实例时，其行为同standard启动模式；
3. singleTask：首先会根据taskAffinity去寻找当前是否存在一个对应名字的任务栈。如果不存在，则会创建一个新的Task，并创建新的Activity实例入栈到新创建的Task中去；如果存在，则得到该任务栈，查找该任务栈中是否存在该Activity实例，如果存在实例，则将它上面的Activity实例都出栈，然后回调启动的Activity实例的onNewIntent方法；如果不存在该实例，则新建Activity，并入栈；特别地，将两个不同App中的Activity设置为相同的taskAffinity，这样虽然在不同的应用中，但是Activity会被分配到同一个Task中去；
4. singleInstance：在整个系统中是单例的。如果在启动这样的Activiyt时，已经存在了一个实例，那么会把它所在的Task调度到前台，重用这个实例；否则，新建一个Task，并且其中只运行这一个Activity

#### 自动关闭Activity
当虚拟机(进程)的可用内存不足最大值的1/4时会触发关闭Activity：

1. 找到ApplicationThreadProxy对应的ProcessRecord；
2. 遍历其中的ActivityRecord：如果是处于“销毁”或“将要销毁”状态，什么也不做；否则找到Activity所属Task的TaskRecord，添加到ArrayList<TaskRecord>；
3. 遍历所有Stack，遍历其中所有TaskRecord，遍历其中所有Activity，遍历的顺序是从最先创建开始；如果Activity是属于进程ProcessRecord的，且可关闭，就关闭Activty；最多关闭ArrayList<TaskRecord>.size()/4个不同TaskRecord中的Activity；
4. “可关闭“是指：不可见且不处于Resume状态且不处于Pause状态；
5. 这里的“关闭”会保存Activity的状态，并且不清除ActivityRecord在ActivityStack、TaskRecord、ProcessRecord中的记录，当回退到它时可重建。

#### 手动关闭Activity
1. 应用进程向AMS发送信息“请求关闭Activity，并指明“ActivityRecord.Token、resultCode、resultData”；
2. 在AMS中，根据ActivityRecord.Token找到ActivityRecord，及其所属TaskRecord、ActivityStack；
3. 修改ActivityRecord的状态：finishing = true；
4. 找到ActivityStack中、除要关闭的Activity外、最上面的ActivityRecord，把它设置为焦点；
5. 如果是要向之前的ActivityRecord传递信息的，那么要关闭的ActivityRecord的resultTo必不为空，并且把resultCode、resultData封装成ActivityResult，添加到之前ActivityRecord的results；
6. 如果要关闭的ActivityRecord是当前显示的ActivityRecord，通知WMS准备它的关闭动画，并准备删除它的窗口；
7. 把要关闭的ActivityRecord的状态改为Pausing；向应用进程发送信息“请暂停ActivityRecord.Token标示的Activity“；
8. 在应用进程中，根据ActivityRecord.Token找到ActivityClientRecord及对应的Activity；
9. 改变Activity的状态：mFinished = true，mCalled = false；
10. 执行Activity生命周期onPause();
11. 改变ActivityClientRecord的状态：paused = true；
12. 调用并删除OnActivityPausedListener；
13. 应用进程向AMS发送信息“ActivityRecord.Token标示的Activity已经暂停了”；
14. 在AMS中，根据Token找到ActivityRecord，并把它的状态改为Paused；
15. 从相关集合中删除ActivityRecord(mStoppingActivities、mGoingToSleepActivities、mWaitingVisibleActivities)；
16. 把ActivityRecord的状态改为Finishing；
17. 从ProcessRecord的activities中删除ActivityRecord，如果activities为空了，清除HeavyNotification，并执行一次进程Lru；
18. AMS向应用进程发送信息“请你销毁ActivityRecord.Token标示的Activity”；
19. 在应用进程中，根据ActivityRecord.Token找到ActivityClientRecord及对应的Activity、Class<Activity>、WindowToken；
20. 执行Activity的生命周期onStop()， 并修改其状态：stopped = true；
21. 执行Activity的生命周期onDetroy()；
22. 从ActivityThread的mActivities中删除ActivityRecord.Token及对应的ActivityClientRecord；
23. 通知WMS删除Activity的DecorView，并关闭WindowToken所标示的所有窗口；
24. ContextImpl相关操作performFinalCleanup；
25. 应用进程向AMS发送信息“ActivityRecord.Token标示的Activity已经销毁了”；
26. 在AMS中，根据Token找到ActivityRecord，并把它的状态改为Detroying；
27. 清除ActivityRecord相关记录；
28. 把ActivityRecord的状态改为Detroyed；
29. 通知WMS删除WindowToken；
30. 从TaskRecord中删除ActivityRecord，如果TaskRecord空了，删除TaskRecord；
31. 找到ActivityStack中、最上面的ActivityRecord，Resume它。

## 管理Task
#### 数据结构
在AMS中，用TaskRecord来描述一个Task。

#### 创建TaskRecord
如果源Activity是null(比如从桌面启动)，或者源Activity的启动模式是singleInstance，或者目标Activity的启动模式是singleInstance或singleTask，或者启动目标Activity的Intent有FLAG_ACTIVITY_NEW_TASK标记且没有FLAG_ACTIVITY_MULTIPLE_TASK标记，就先找可重用的TaskRecord；否则用源Activity的TaskRecord；

如果给定了一个TaskRecord，检查该TaskRecord，如果该TaskRecord的baseIntent的CompnentName与目标Activity的CompnentName相同，而且rootActivity是null，那么目标Activity可以重用这个TaskRecord；否则遍历现存的TaskRecord，如果某TaskRecord的baseIntent和affinityIntent的ComponentName与目标Activity的相同，那么目标Activity可以重用这个TaskRecord；

如果没有可重用的TaskRecord，那么就需要新建TaskRecord。

## 与应用进程通信
应用向AMS传递信息：接口是IActivityManager；业务层服务端是ActivityManagerNative；业务层客户端是ActivityManagerProxy，实现层是ActivityManagerService；装饰层是ActivityManager。
AMS向应用传递信息：接口是IApplicationThread；业务层服务端是ApplicationThreadNative；业务层客户端是ApplicationThreadProxy；实现层是ApplicationThread。

当应用进程启动后，应用进程调用ActivityManagerNative.getDefaulte()，向SystemServer进程申请一个向AMS传递信息的客户端，即一个ActivityManagerProxy对象；然后，调用ActivityManagerProxy的attachApplication()，向AMS传递信息“你可以向我发消息了，我的服务端是ApplicationThread”，在这个过程中，ApplicationThread经过Binder机制转换为AMS使用的客户端ApplicationThreadProxy；在ActivityManagerService的attachApplication()实现中，先用应用进程的pid找到对应的ProcessRecord，然后把ApplicationThreadProxy的引用保存到ProcessRecord的成员变量thread。这样，AMS也可以向应用进程传递信息了；同时thread是否为null也作为AMS判断应用进程是否已经启动的标志。

## 应用启动流程
#### 点击桌面应用图标
1. 由PMS收集目标Activity的信息，构建ActivityInfo；
2. 由源进程的ApplicationThreadProxy在AMS找到对应的ProcessRecord；
3. 由源Activity的ActivityRecord.Token在AMS找到对应的ActivityRecord；
4. 由目标ActivityInfo构建目标ActivityRecord，ActivityRecord的成员变量processName、uid标示了所属进程；
5. 计算目标Stack，一般来说就是ActivityStack；
6. 计算目标Task，构建TaskRecord，并把TaskRecord插入到ActivityStack顶部；
7. 把目标ActivityRecord插入目标TaskRecord的顶部；
8. 目标ActivityRecord向WMS申请WindowToken；
9. 目标ActivityRecord被添加到mLRUActivities；
10. 使HomeStack顶部的Activity(也就是桌面)进入Pause状态，WMS准备好目标Activity的入场动画；
11. 由目标ActivityRecord中保存的目标进程的processName和uid，在AMS中的mProcessNames查找目标ProcessReocrd；
12. 此时一定找不到，也说明目标进程没有启动，所以新建ProcessRecord，并加入mProcessNames；
13. 调用Process.start()开启目标进程，开启成功后，把目标ProcessRecord加入mPidsSelfLocked；
14. 新进程启动后，先建立与AMS的双向通信，在这个过程中，AMS会根据pid找到对应的ProcessRecord；并向应用进程传递“应用进程名、ApplicationInfo、根Activity的ComponentName、启动参数、要启动的服务”等信息；
15. ActivityThread根据传递过来的信息构建AppBindData，传递给主线程；并添加虚拟机内存监听，当空间占用超过最大值的3/4时，应用进程会向AMS发送消息，让它释放一些本进程的Activity；
16. 主线程根据AppBindData构建“应用”运行所需的Android环境及从PMS查询相关信息，包括“Configuration、ResourcesManager、ApplicationInfo、LoadedApk、Application”等，并调用Application.onCreate()；
17. 同时在SystemServer进程，AMS找到ActivityStack栈顶的ActvityRecord，把ProcessRecord的引用保存到成员变量app；然后向应用进程传递信息：“你该启动ActivityRecord对应的Activity了”(scheduleLaunchActivity)；最后把ProcessRecord加入mLruProcesses，进行一次进程Lru；
18. ActivityThread根据传递过来的ActivityRecord构建ActivityClientRecord，构建对应的Activity；如果Application还没有创建好，此时创建；创建Activity运行所需Android环境，包括“Context”等；
19. 调用Activity的attach()，创建PhoneWindow、UiThread、MainThred；获取向WMS传递信息的客户端；
20. Activity执行生命周期onCreate() -> onStart() -> onPostCreate()；
21. ActivityThread把ActivityClientRecord添加到mActivities；
22. Activity执行生命周期onResume()；
23. 初始化DecorView；
24. ActivityThread向AMS发送信息“Activity已经Resume了”(activityResumed)；
25. AMS在activityResumed()的实现中，修改对应的ActivityRecord的状态。