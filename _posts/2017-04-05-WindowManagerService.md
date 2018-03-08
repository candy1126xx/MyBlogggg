---
layout:     post                    # 使用的布局
title:      WindowManagerService               # 标题 
subtitle:   WindowManagerService #副标题
date:       2017-04-05              # 时间
author:     candy1126xx                      # 作者
header-img: img/home-bg-o.jpg    #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签
    - Android框架层
---

## 简介
WMS运行于SystemServer进程的DisplayThread线程。DisplayThread线程由DMS创建，其中运行了DMS、WMS、IMS。WMS负责所有窗口的排序和布局、创建和销毁Native层Surface、把触摸事件分发给焦点窗口、锁屏。

这篇仅记录管理窗口部分。

## 重要的类
* WindowState：在WMS中代表一个窗口；封装了窗口通信服务端、客户端；封装了窗口参数、系统UI可见性；封装了窗口坐标、窗口区域；封装了窗口动画；等等。
* WindowToken：
 - 针对系统窗口：封装了Binder对象作为窗口所属组件的标示(token)；其客户端BinderProxy对象作为向窗口所属组件发送信息的客户端；封装了窗口类型(windowType)；封装了同属一个组件的所有父窗口和子窗口的WindowState(windows)。
 - 针对非Activity窗口：封装了ViewRootImpl.W对象作为窗口所属组件的标示(token)；其客户端ViewRootImpl.W.Proxy对象作为向窗口所属进程发送信息的客户端；其余同上。
* AppWindowToken：针对Activity窗口；封装了ActivityRecord.Token对象作为窗口所属组件的标示(appToken)；同时它也用于向窗口在AMS中对应的ActivityRecord发送信息(因为AMS和WMS在同一进程，所以这里用的是服务端)；封装了它与其它的所有子窗口的WindowState(allAppWindows)；封装了Activity所属Task的引用(Task)、是否全屏(appFullscreen)、期望方向(requestedOrientation)等信息；
* ViewRootImpl.W：实现了IWindow接口；其中封装了Session的引用和某个ViewRootImpl的弱引用；可作为其他组件向应用进程中的某个指定ViewRootImpl发送信息的服务端；对应用窗口/子窗口具有唯一性，作为应用窗口/子窗口在WMS.mWindowMap中的唯一标示。
* ViewRootImpl.W.Proxy：其他组件向应用进程中的某个指定ViewRootImpl发送信息的客户端。
* ActivityRecord.Token：实现了IApplicationToken接口；其中封装了ActivityManagerService的引用和某个ActivityRecord的弱引用；可作为其他组件向AMS中的某个指定ActivityRecord发送信息的服务端；对ActivityRecord具有唯一性，作为对应Activity组件(应用父窗口)在WMS.mTokenMap中的唯一标示。
* ActivityRecord.Token.Proxy：其他组件向AMS中的某个指定ActivityRecord发送信息的客户端。
* WindowManagerService.mTokenMap：以窗口所属组件的标示(Binder/ActivityRecord.Token/ViewRootImpl.W)为键，以WindowToken/AppWindowToken为值，主要作用是与窗口所属组件通信。
* WindowManagerService.mWindowMap：以窗口唯一标示ViewRootImpl.W为键，以WindowState为值，主要作用是管理应用窗口/子窗口。
* Task：WMS中的Task概念。
 - mAppTokens：按AMS中TaskRecord中ActivityRecord的顺序排列AppWindowToken。
* TaskStack：WMS中的Stack概念。
 - mTasks：按AMS中ActivityStack中TaskRecord的顺序排列Task。

## 启动后的准备工作
在SystemServer.startOtherServices() -> WindowManagerService.main() -> WMS构造函数：

1. 获取向DMS发送信息的客户端DisplayManagerInternal；
2. 构建并初始化DisplaySetting；
3. 构建PhoneWindowManager，并注册到LocalServices；
4. 构建PointerEventDispatcher；
5. 构建SurfaceSession；
6. 构建DisplayContent；
7. 构建KeyguardDisableHandler；
8. 获取向PowerManagerService发送信息的客户端PowerManager和PowerManagerInternal；
9. 构建WakeLock；
10. 构建AppTrasition；
11. 构建WindowAnimator；
12. 注册锁屏/解锁广播接收器；
13. 把向WMS发送信息的客户端WindowManagerInter注册到LocalServices，供其它系统服务组件使用；
14. 初始化PhoneWindowManager；
15. 建立与SurfaceControl的通信，构建FocusedStackFrame，再关闭与SurfaceControl的通信。

## PhoneWindowManager
PhoneWindowManager实现了WindowManagerPolicy接口，提供了“设置Display、添加/删除/布局窗口”的具体实现。

* setInitialDisplaySize()：根据给定的Display的宽高设置方向、状态栏高度、导航栏高度；
* setDisplayOverscan()：设置Display的Overscan区域；
* checkAddPermission()：检测进程是否有添加窗口的权限；原则是应用进程只能添加APPLICATION\_WINDOW、SUB\_WINDOW、TOAST；
* adjustWindwoParamsLw()：整理应用进程传来的窗口参数；只针对TYPE\_SYSTEM\_OVERLAY和TYPE\_STATUS_BAR；如果有FLAG\_DRAW\_SYSTEM\_BAR\_BACKGROUND，将会添加SYSTEM\_UI\_FLAG\_LAYOUT\_FULLSCREEN和SYSTEM\_UI\_FLAG\_LAYOUT\_HIDE\_NAVIGATION；
* adjustConfigurationLw()：修改WMS计算的Configuration；
* windowTypeToLayerLw()：根据type返回int用于窗口排序，值越大越在上面；重要的几种：WALLPAPER == APPLICATION_WINDOW < SYSTEM_DIALOG < TOAST < SYSTEM_ALERT < INPUT_METHOD < STATUS_BAR < SYSTEM_OVERLAY < NAVIGATION_BAR；
* subWindowTypeToLayerLw()：根据type返回int用于子窗口排序，值越大越在上面，正值表示在所依附的窗口的上面，负值表示在所依附的窗口的下面；APPLICATION_MEDIA < 父窗口 < APPLICATION_PANEL == APPLICATION_ATTACHED_DIALOG;
* addStartingWindow()：添加一个窗口，用于在应用窗口开启之前显示，告知用户应用正在启动；构建一个PhoneWindow，类型为APPLICATION_STARTING，无法获取焦点，无法接收触摸事件，并将生成的控件树添加到WMS；
* removeStartingWindow()：删除StartingWindow，应该在应用第一个窗口显示之后调用；
* prepareAddWindowLw()：会在添加窗口前调用，默认实现是检查添加的窗口不能是StartusBar、NavigationBar、Keyguard，即这三类是单例；
* removeWindowLw()：会在删除窗口前调用，默认实现只针对StartusBar、NavigationBar、Keyguard，把相应引用置为null；
* beginLayoutLw()：窗口布局第一阶段；根据给定的Display的尺寸和方向，计算“Display11区域”；
* layoutWindowLw()：窗口布局第二阶段；根据“11区域”、窗口参数计算窗口区域；
* finishLayoutLw()：窗口布局第三阶段；默认没有实现；
* beginPostLayoutPolicyLw()：窗口策略分发第一阶段；
* applyPostLayoutPolicyLw()：窗口策略分发第二阶段；
* finishPostLayoutPolicyLw()：窗口策略分发第三阶段；

## 与窗口通信
#### 业务层服务端
WMS要向窗口所属组件发送信息，就需要持有一个相关的“服务端”的引用，在需要时调用asInterface()转换为“客户端”。不同的窗口，“服务端”是不同的。这些“服务端”在WMS被封装为不同的对象，保存在WMS.mTokenMap。

* 系统窗口的通信服务端是一个实现了IBinder接口的对象，具体是什么要看是什么系统服务组件；
* Activity窗口的通信服务端是ActivityRecord.Token，封装它的数据结构是AppWindowToken；
* 非Activity窗口的通信服务端是ViewRootImpl.W，封装它的数据结构是WindowToken。

即WMS向Activity窗口发送信息时，是使用ActiviRecord.Token对象向AMS发送信息(不跨进程)；向非Activity窗口发送信息时，是使用ViewRootImpl.W.Proxy对象向应用进程发送信息(跨进程)。

#### 与AMS通信建立过程
在WMS启动后，调用ActivityManagerService.setWindowManager()，把WindowManagerService的引用保存在成员变量mWindowManager；此后AMS可通过mWindowManager向WMS发送信息。

在启动Activity时，创建的ActivityRecord.Token对象封装了ActivityManagerService的引用和某个ActivityRecord的弱引用，并调用WindowManagerService.addAppToken()把ActivityRecord.Token对象的引用传到WMS；在WMS中，创建AppWindowToken作为值，ActivityRecord.Token对象作为键，保存在成员变量mTokenMap；此后WMS可通过mTokenMap向AMS中的指定ActivityRecord发送信息。

#### 与应用进程通信建立过程
在WMS启动后，会把WMS注册到SystemSeviceManager；调用IWindowManager.Stub.asInterface(ServiceManager.getService("window"))可以获得向WMS发送信息的客户端IWindowManager.Stub.Proxy对象；调用Proxy.openSession()可获得客户端Session.Proxy对象；此后应用进程可以通过Session.Proxy对象向WMS发送信息；此外，应用进程的View框架提供了PhoneWindow类，PhoneWindow提供了窗口管理控件的策略，还用WindowManagerImpl类和WindowManagerGlobal类封装了向WMS发送信息的过程，其中WindowManagerGlobal使用了单例模式，因此，应用进程不必显示创建/删除窗口，而可以通过Activity组件隐式创建/删除窗口。

在应用进程向WMS添加非Activity窗口时，调用Session.Proxy.addToDisplay()把窗口的ViewRootImpl.W对象传给WMS；创建WindowToken作为值，ViewRootImpl.W对象作为键，保存在WindowManagerService.mTokenMap；此后WMS可以通过ViewRootImpl.W.Proxy对象向应用进程发中指定的非Activity窗口发送消息。

## 窗口的唯一标示
WMS以“唯一标示”区分各个窗口。系统窗口都是单例，所以不需要“唯一标示”。而对于应用窗口，不论是父窗口还是子窗口，都是使用ViewRootImpl.W对象作为“唯一标示”。这些“唯一标示”在WMS被封装为WidnowState，保存在WMS.mWindowMap。

在应用进程中，父窗口对应的有PhoneWindow对象，子窗口没有；因此子窗口默认不获取焦点、不响应输入事件、不独占触摸事件、显示时不添加DimLayer；

对于子窗口，WindowState.mAttachedWindow是所依附的父窗口，WindowState.mChildWindows必为null，WindowState.mAttachedWindow.mChildWindows是按type排序的所有兄弟窗口；相应地，对于父窗口，WindowState.mAttachedWindow必为null，WindowState.mChildWindows是按type排序的所有子窗口。

## 创建窗口
#### 应用父窗口
包括**建立通信**、**创建WindowState**、**窗口排序**、**窗口布局**四步。
###### 建立通信
1. 在ActivityRecord的构造方法中，创建了ActivityRecord.Token对象；
2. 在启动栈顶ActivityRecord阶段(ActivityStack.startActivityLocked())，调用WindowManagerService.addAppToken()创建了AppWindowToken，并把ActivityRecord.Token、AppWindowToken存入mTokenMap；
3. _TODO：创建Task、TaskStack。_
4. 按ActivityRecord在TaskRecord.mActivities中的index，把AppWindowToken存入Task.mAppTokens；
5. 调用WindowManagerService.moveTaskToTop()，把DisplayContent.mAppTokens中AppWindowToken的顺序按DisplayContent中TaskStack的顺序、TaskStack中Task的顺序、Task中AppWindowToken的顺序整理一遍。

###### 创建WindowState
1. 在Activity.attach()中，创建了PhoneWindow。
2. 在Activity.onResume()后，调用WindowManagerImpl.addView() -> WindowManagerGlobal.addView()；创建ViewRootImpl，并把DecorView存入mViews，把ViewRootImpl存入mRoots，把WindowManager.LayoutParams存入mParams。
3. 调用ViewRootImpl.setView() -> Session.addToDisplay() -> WindowManagerService.addWindow()；创建WindowState，并把ViewRootImpl.W、WindowState存入mWindowMap，最后创建SurfaceSession；

###### 窗口排序
1. 调用addWindowToListInOrderLocked()，把WindowState加入“全局窗口管理”DisplayContent.mWindows。DisplayContent.mWindows中的元素按type、插入先后排序，具体操作是：
 - 系统窗口：addFreeWindowToList()，插到所有窗口的最上面；
 - 应用父窗口：addAppWindowToList()，在所属Task中，若是BASE_WINDOW在Task的最下面，若是STARTING_WINDOW或新创建的WindowState在Task的最上面；
 - 应用子窗口：addAttachedWindow()，先找到所依附的应用窗口，然后比较其兄弟窗口的mSubLayer，越大越靠上。
2. 调用assignLayersLocked()，遍历DisplayContent.mWindows，使mBaseLayers相同的相邻元素的mLayers递增。

###### 窗口布局
1. 根据给定的Display的尺寸和方向，计算“Display11区域”；
2. 根据“Display11区域”、各个窗口的参数，分别计算各个窗口区域。