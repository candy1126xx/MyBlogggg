---
layout:     post                    # 使用的布局
title:      SystemServer进程              # 标题 
subtitle:   SystemServer进程 #副标题
date:       2017-03-15              # 时间
author:     candy1126xx                      # 作者
header-img: img/home-bg-o.jpg    #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签
    - Android框架层
---

## 简介
SystemServer进程是Java层的第二个进程。其中运行着所有Java层的系统服务，并管理着这些服务的“服务端”。

## 启动后的准备工作
对比应用进程，可以看出是很相似的。

在ZygoteInit.handleSystemServerProcess()中。

1. 关闭从Zygote那里继承下来的Socket；
2. 设置一些进程参数；
3. 做些常规初始化；
4. 在Native层启动一个线程池，用于Binder通信；
5. 设置进程名为“system_server”；
6. 调用com.android.server.SystemServer类的main函数。

## 启动系统服务
在SystemServer.main() -> SystemServer.run()中：

1. 在Java层加载libandroid_servers.so；
2. 在Native层启动SurfaceFlinger进程；
3. 在Java层创建SystemContext；
4. 创建LocalServices；
5. 创建SystemServiceManager，并注册到LocalServices；
6. 启动各个系统级服务；
7. 主线程进入Loop循环，用于接收系统级服务的工作线程发送来的信息；
8. 系统级服务启动后都运行于SystemServer进程，其中AMS有两个线程“主线程”和“工作线程”，其它服务都只有“工作线程”。

## Android运行环境
Android希望淡化进程的概念，转而使用组件的概念。Android中的组件有：“应用”、“窗口”、“系统服务”、“应用服务”、“内容提供者”、“广播接收者”。

怎么在“进程”和“组件”之间转换呢？每个进程对应一个ActivityThread。ActivityThread的功能主要有两个：控制运行在该进程的组件的生命周期；为“应用”、“窗口”、“服务”组件创建Context。

Context封装了“组件运行的环境信息”，并提供了“与环境交互的接口”和“与其他组件交互的接口”。针对不同的组件，Android提供了不同的Context子类。

#### 系统服务：ContextImpl

* 获取AssetManager；
* 获取Resource；
* 获取向PMS发送信息的客户端PackageManager；
* 获取向ContentProvider发送信息的工具类ContentResolver；
* 获取主线程Loopper；
* 获取所属应用的ApplicationContext；
* 设置/获取所属应用的主题；
* 获取所属应用的类加载器；
* 获取所属应用的包名；
* 获取所属应用的应用信息ApplicationInfo；
* 获取所属应用的资源文件夹(/data/app/包名/res/)路径；
* 获取所属应用的代码文件夹(/data/app/包名/)路径；
* 获取所属应用的某个SP文件路径；
* 获取操作所属应用的某个SP文件的工具类SharedPreferences；
* 打开某个文件(必须在所属应用的文件集中)的输入流/输出流；
* 删除某个文件(必须在所属应用的文件集中)；
* 获取所属应用的文件集的路径(/data/data/包名/files/)；
* 获取SD卡上某类资源的文件夹路径；
* 获取所属应用在SD卡上的独占文件夹的路径；
* 获取所属应用的缓存文件夹的路径(/data/data/包名/cache/)；
* 打开或创建所属应用的某个数据库；
* 删除所属应用的某个数据库；
* 获取所属应用的数据库文件夹(/data/data/包名/databases/)；
* 列出所有数据库文件；
* 开启某个“窗口”组件(必须设置FLAG_NEW_TASK)；
* 开启多个“窗口”组件(必须设置FLAG_NEW_TASK)；
* 向其他应用发送一个Intent；
* 发送/删除一条广播；
* 注册/注销一个“广播接收者”组件；
* 启动/停止/绑定/解绑一个“应用服务”组件；
* 检查所属应用是否拥有某项权限；

#### 应用：Application

* 作为ContextImpl的装饰类，有ContextImpl的所有接口；
* 注册/注销ComponentCallback；
* 注册/注销ActivityLifecycleCallback

#### 窗口：Activity

* 作为ContextImpl的装饰类，有ContextImpl的所有接口；
* 获取/重置开启本组件的Intent；
* 获取所属应用的Application；
* 获取向WMS发送信息的客户端WindowManager；
* 获取操作窗口的工具类PhoneWindow；
* 获取当前焦点的控件；
* 获取FragmentManager；
* 获取/设置控件树；
* 获取/设置Transition动画；
* 为所属应用申请某些权限；
* 开启某个“窗口”组件(不必设置FLAG_NEW_TASK)；
* 开启多个“窗口”组件(不必设置FLAG_NEW_TASK)；
* 设置对之前窗口的返回结果；
* 获取之前窗口的类名/包名；
* 设置控件树的可见性；
* 获取本组件的关闭状态(是正在关闭还是已经关闭了)；
* 销毁本组件；
* 销毁本组件及所属Task中的其他窗口组件；
* 销毁本组件及所属Task；
* 关闭但不销毁本组件；即关闭本窗口，但不删除AMS中记录，当用户返回本窗口时会重建本窗口；
* 获取/设置窗口方向；
* 获取所属Task的ID；
* 判断本组件是否是所属Task的root；
* 把所属Task移动到后台；

#### Application 和 Activity 的 区别
* 从组件生命周期的角度考虑，Application的生命周期和“应用”一样长，Activity的生命周期和“窗口”一样长。当引用对象的生命周期比窗口的生命周期长，就不能用Activity，否则会造成内存泄漏。
* 从WMS的角度考虑，Activity可以对应到WindowToken，而Application无法对应到。所以当要添加子窗口时，必须用Activity。
* 从AMS的角度考虑，Activity可以对应到TaskRecord，而Application无法对应到。所以当要启动“窗口”组件时，要么用Activity，要么用Application + FLAG\_NEW_TASK。

#### getApplication() 和 getApplicationContext() 的区别
同一个Application对象在3个阶段被赋值到3个对象的引用。

* 当应用包信息被PMS管理后，被应用对应的LoadedApk对象引用。即LoadedApk.mApplication。
* 当应用启动后，被应用对应的ActivityThread对象引用。即ActivityThread.mApplication。
* 当某个“窗口”组件启动后，被该组件引用。即Activity.mApplication。

getApplication()返回的是Activity.mApplication；getApplicationContext()返回的是LoadedApk.mApplication或ActivityThread.mApplication。

## LocalServices、SystemServiceManager
在Binder机制建立前，系统服务注册在LocalServices。LocalServices不要求注册的服务继承自Binder和SystemService类。

在Binder机制建立后，LocalServices就没有用了。注册在LocalServices的服务都会再注册到SystemServiceManager，之后注册的服务也都直接注册到SystemServiceManager。SystemServiceManager要求注册的服务必须继承自Binder和SystemService类。SystemServiceManager的作用是管理系统服务的生命周期，与Binder通信无关。

## 管理系统服务的“服务端”
应用进程要获取系统服务的客户端，可以通过ServiceManager/ServiceManagerNative向ServiceManager进程发送消息。ServiceManagerNative是Binder机制业务层的服务端；ServiceManager是装饰层，增加了缓存和单例功能。