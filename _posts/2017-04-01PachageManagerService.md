---
layout:     post                    # 使用的布局
title:      PackageManagerService               # 标题 
subtitle:   PackageManagerService #副标题
date:       2017-04-01              # 时间
author:     candy1126xx                      # 作者
header-img: img/home-bg-o.jpg    #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签
    - Android框架层
---

## 简介
PMS运行于SystemServer进程。负责应用安装，管理所有已安装应用的组件信息。

## 启动后的准备工作
在PackageManagerService构造函数中：

1. 构建DisplayMetrics；
2. 构建Settings；
3. 向Setting添加“system”、“phone”、“log”、“nfc”进程的SharedUserSetting；
4. 由WMS向DisplayMetrics赋值；
5. 创建工作线程及PackageHandler；
6. 创建三个路径：mAppDataDir指向/data/data目录；mUserAppDataDir指向/data/user目录；mDrmAppPrivateInstallDir指向/data/app-private目录；
7. 扫描解析/system/etc/permissions目录下的xml文件，这些文件记录了系统进程运行需要的权限、硬件特性、第三方库；
8. 扫描解析/data/system/packages.xml，这个文件相当于注册表，记录了所有已安装应用的基本信息；
9. 对第三方库进行dex优化；
10. 扫描解析/system/frameworks、/system/app、/vendor/app三个目录下的apk文件，这些文件是系统应用的文件
11. 每扫描一个apk文件就生成一个PackageParser.Package对象；即PackageParser.Package对象封装了一个应用的Manifest文件的所有信息；
12. 扫描解析/data/app、/data/app-private两个目录下的外部应用的apk文件；
13. 把所有PackageParser.Package对象加入PMS管理；
14. 把PMS注册到SystemServiceManager；
15. 判断是否是开机后第一次启动(即不是手机重启的情况)；
16. dex优化；
17. 通知系统“PMS准备好了”。

## DisplayMetrics
DisplayMetrics类封装了关于显示屏的信息，比如尺寸、分辨率、字体缩放等。数据来源是WMS：

```
DisplayMetrics metrics = new DisplayMetrics();
getWindowManager().getDefaultDisplay().getMetrics(metrics);
```

## Settings
Settings类封装了系统中所有应用的设置信息(对应到“设置 - 应用管理”)。其中一条设置信息封装为SharedUserSetting，SharedUserSetting中封装的信息有：pkgFlags(例如ApplicationInfo.FLAG_SYSTEM标示这是系统核心应用)、name(同uid)、gids/userId(定义在Process类中的各个系统进程的gid/uid常量)、packages(类型为HashSet<PackageSetting>，代表一组设置信息)。即一组PackageSetting对应一个SharedUserSetting；一个SharedUserSetting可通过uid共享。

管理SharedUserSetting的集合类有：

mSharedUsers类型为HashMap<String, SharedUserSetting>，其键为uid(例如“android.uid.system”)；管理所有应用；

mUserIds类型为ArrayList<Object>，其键为uid与FIRST\_APPLICATION_UID的差，值为应用进程的SharedUserSetting；它的作用是可以由索引找到SharedUserSetting，比mSharedUsers快。

mOtherUserIds类型为SparseArray<Object>，其键为uid与FIRST\_APPLICATION_UID的差，值为系统进程的SharedUserSetting；它的作用是可以由索引找到SharedUserSetting，比mSharedUsers快。

## APK文件、Package对象、应用、进程的关系
* 应用可分为“系统核心应用”、“系统内置应用”、“外部应用”三类。
* 每个应用，在编码阶段每个Module有一个Manifest文件；编译之后所有Manifest文件被合成一个Manifest文件；PMS可以解析一个Manifest文件生成PackageParser.Package对象。
* 应用既可以是单进程的，也可以是多进程的。也可以多个应用运行于一个进程。
* 一个应用可以先由一个APK启动，之后再安装运行其他APK。(参考360的插件化框架)
* Manifest文件中的manifest标签的package属性是一个应用的唯一标志。新安装的APK如果package属性与已安装的APK相同，则会检查应用签名，如果签名不对则不能安装，如果签名正确则会当作升级。
* 在插件化框架中，若要实现多个APK共享数据(磁盘上的安装目录下的数据)，则这些APK必须拥有相同的uid，即这些APK中的Manifest文件中的manifest标签的sharedUserId属性必须是相同的。

## Manifest文件
#### 标签
* manifest：最外层的标签，必须有。
* application：manifest内标签，必须有。
* activity/service/receiver/provider：application内标签。声明一个Activity/Service/BroadcastReceiver/ContentProvider。
* intent-filter：activity/service/receiver内标签，声明该组件的IntentFilter。
* permission：application内标签。声明一个自定义权限。当其他应用要调用该应用的数据时，必须拥有这些权限。
* meta-data：application内标签。声明一个键值对。常用于保存持久化的应用设置信息。
* activity-alias：application内标签。声明一个快捷方式。
* uses-permission：manifest内标签。声明应用运行所需的系统权限。
* uses-library：manifest内标签。声明应用运行所需的Android类库。如果用户设备没有所需类库，PMS将阻止安装应用。
* supports-screens：manifest内标签。声明对各种屏幕的支持情况。
* uses-configuration：manifest内标签。声明对各种输入设备的支持情况。
* uses-feature：manifest内标签。声明应用运行所需的硬件特性。如果用户设备没有所需硬件，PMS将阻止安装应用。
* uses-sdk：manifest内标签。声明应用运行所需的Android版本。如果用户设备的Android版本小于设置的最小版本，PMS将阻止安装应用。
* instrumentation：manifest内标签。声明监控程序相关设置。

#### manifest标签属性
* package：指定本应用内java主程序包的包名，它也是这个应用进程的进程名。
* sharedUserId：拥有相同uid的多个APK可共享数据。
* sharedUserLabel：共享数据时每个APK的标示。
* versionCode/versionName：应用版本。
* installLocation：可选值：
 - preferExternal：系统会优先考虑将APK安装到SD卡上。用户可以选择为内部ROM存储上；如果SD存储已满，也会安装到内部存储上。
 - auto：系统将会根据存储空间决定。
 - internalOnly：必须安装到内部。

#### application标签属性
* allowTaskReparenting：
* backupAgent：允许将应用数据备份到云端目录，目录路径由指定类名确定。
* label：应用名。
* icon：应用图标。
* name：指明应用实现的Application子类。
* enabled：Android系统能否实例化应用组件。这是一个全局设置，若某个组件与全局设置不同，可以在该组件标签下自行设置。
* hasCode：如果应用不包含Java代码，则可以设置为false。
* killAfterRestore：
* permission：
* presistent：应用是否在任何时候都保持运行状态。
* process：应用运行的进程名，默认值为manifest标签下package属性。
* taskAffinity：设置所有Activity所属的TaskRecord名，默认值为manifest标签下package属性。是一个全局设置。
* theme：设置所有Activity所用的主题。是一个全局设置。

#### activity/service/receiver标签属性
* alwaysRetainTaskState：
* clearTaskOnLaunch：热启动时是否清除Task。
* configChanges：
* excludeFromRecents：是否可被显示在最近打开的activity列表里。
* finishOnTaskLaunch：
* launchMode：四种启动模式。
* multiprocess：是否允许多进程。
* noHistory：
* stateNotNeeded：被销毁或者成功重启时是否保存状态。
* windowSoftInputMode：设置软键盘模式。
* screenOrientation：屏幕方向。
 - unspecified：由系统自动判断显示方向。
 - landscape：横屏模式。
 - portrait：竖屏模式。
 - user：用户当前首选的方向。
 - behind：和该Activity下面的那个Activity的方向一致。
 - sensor：有物理的感应器来决定。如果用户旋转设备这屏幕会横竖屏切换。
 - nosensor：忽略物理感应器，这样就不会随着用户旋转设备而更改了

## 管理应用信息
#### 读取Manifest文件
PMS把Manifest文件转换为各种数据结构的目的是为了“更快速地查询”。

PMS.mActivities(ActivityIntentResolver)保存所有已安装应用的Activity信息。ActivityIntentResolver中
成员变量mActivities的类型为ArrayMap，键为Activity的类名(ComponentName)，值为Activity的描述类(PackageParser.Activity)。PackageParser.Activity的成员变量info(ActivityInfo)记录了Activity的信息，可用于启动Activity；intents(ArrayList<ActivityIntentInfo>)记录了所有可以启动这个Activity的IntentFilter。

成员变量mFilters的类型为ArraySet，元素类型为ActivityIntentInfo，记录了所有可以启动这个Activity的IntentFilter。

成员变量mTypeToFilter、mBaseTypeToFilter、mWildTypeToFilter、mSchemeToFilter、mActionToFilter、mTypedActionToFilter是mFilters的子集，保存某一类的ActivityIntentInfo。它们的类型是ArrayMap<String, ActivityIntentInfo>，键为ActivityIntentInfo的标示，比如mActionToFilter的键就是Action字符串(对应Manifest中action属性)、mTypeToFilter的键就是Type字符(比如“image/jpg”)。

当PMS扫描解析Manifest文件后，Activity的信息保存在PackageParser.Package.activies中；然后，需要用PackageParser.Package.activies填充PMS.mActivities；ActivityIntentResolver.addActivity()用于向PMS.mActivities.mActivities添加元素，在此过程中，intents会被添加到mFilters及某个子集中；之后的匹配查询操作会根据不同的情况，从不同的集合中查找符合条件的ActivityInfo，然后根据ActivityInfo启动Activity。是以空间换时间。

mServices、mReceivers、mProviders与mActivities类似。

#### Intent与IntentFilter
在匹配过程中用到的Intent属性：

* Action：String；标示一个特定的“动作”；
* Data：“动作”要操作的数据；是一个Uri，所以也代表了Scheme；
* Category：目标组件的类别；
* Type：MIME类型；通常是需要操作某一种文件(如mp3文件)时使用；
* Component：在希望指定目标组件时使用；
* Extras：其他信息；

在匹配过程中用到的IntentFilter属性：

* 与Intent对应，有Action、Catrgory、Data；
* Priority：优先级。决定了匹配返回的List<ResolveInfo>的排序。

#### 匹配流程
ApplicationPackageManager是工具类，提供查询指定Package的各种相关信息的方法，内部是通过IPackageManager.Stub向PMS发送信息实现的；同时注意到PMS是无法向应用进程发送信息的。

在ApplicationPackageManager.queryIntentActivities() -> PMS.queryIntentActivities()中：

1. 检查Intent是否指明了组件名，如果指明了，通过组件名可以直接查询PMS.mActivities.mActivities得到需要的ActivityInfo，并构建ResolveInfo返回；
2. 否则，检查Intent是否指明了包名，如果指明了，先获取PackageParse.Package对象，然后找出在PackageParse.Package.activities中的、符合条件的ActivityIntentInfo，构建成ResolveInfo列表返回；
3. 否则，通过PMS.mActivities内的IntentFilter的集合，查找符合条件的ActivityIntentInfo，构建成ResolveInfo列表返回。

Intent的Type形如“image/jpg”，具体的匹配逻辑是：

1. 如果“image”是“\*”，即没有设置Type，但设置了Action，则从mActionToFilter获取键为Action的ActivityIntentInfo生成1个列表；
2. 如果“image“不是“\*”，但是“jpg”是“*”，则从mBaseTypeToFilter获取键为“image“的ActivityIntentInfo生成1个列表，再从mWildTypeToFilter获取键为“image“的ActivityIntentInfo生成1个列表；
3. 如果是“image/jpg”，则从mTypeToFilter获取键为“image/jpg”的ActivityIntentInfo生成1个列表，再从mWildTypeToFilter获取键为“image“的ActivityIntentInfo生成1个列表；
4. 如果设置了Data(Scheme)，则从mSchemeToFilter获取键为Scheme的ActivityIntentInfo生成1个列表；
5. 遍历所有列表，检测每个元素的Action、Data、Category是否匹配，用匹配的构建ResolveInfo加入最终列表；
6. 最终列表根据mPriority排序，mPriority越大，在呈现给用户的应用选择对话框中的位置越靠前。

## 应用安装
在PackageManagerService.installPackage()中：
#### 接受安装请求
1. 检查客户端是否拥有安装APK的权限；
2. 向工作线程发送消息“请复制APK到安装目录，并为安装过程做必要初始化，相关信息为InstallParams”；
3. 在工作线程中，绑定DefaultContainerService服务(组件)；
4. 把InstallParams加入待安装队列mPendingInstalls(ArrayList)；
5. 向工作线程发送消息“启动安装”；
6. 在工作线程中，取mPendingInstalls首项；
7. 首先根据flags判断“期望安装位置”：PackageManager.INSTALL_EXTERNAL/PackageManager.INSTALL_INTERNAL；
8. 然后获取与DeviceStorageMonitorService服务(进程)通信的客户端；
9. 然后调用DeviceStorageMonitorService.getMinimalPackageInfo()，由APK文件地址得到一个PackageInfoLite对象；
10. 根据“期望安装位置”和PackageLite对象得到“最终安装位置”；(以下为INSTALL_INTERNAL的情况)
11. 校验APK(目前还没有具体实现)；
12. 把APK文件复制到/data/app目录下；
13. 向工作线程抛一个Runnable，在Runnable.run()中进行安装过程；

#### 2个写入
在PackageManagerService.installPackageLI()中：

1. 输入的参数为InstallArgs、PackageInstalledInfo；
2. 构建一个PackagePaser对象，解析APK文件，生成PackagePaser.Package对象；这一步不解析application及其内标签；把其他信息写入/data/system/packages.xml；
3. 判断是否为“升级”。如果已安装的包信息中有与Package对象的包名相同的，则为升级；
4. 校验签名和新增权限；
5. 再构建一个PackagePaser对象，解析APK文件，生成PackagePaser.Package对象，并加入PMS管理；这一步只解析application及其内标签；
6. 清除安装过程中生成的数据及缓存。

#### 广播结果并检查请求队列
1. 把安装结果封装成一个PostInstallData对象，并向工作线程发送消息“安装完毕，安装结果为PostInstallData”；
2. 在工作线程中，如果安装成功，则发送PACKAGE_ADDED广播；如果升级成功，则发送PACKAGE_REPLACE广播；同时携带packageName；
3. 删除mPendingInstalls首项；
4. 检查mPendingInstalls是否为空，如果为空，解绑DefaultContainerService服务；否则再向工作线程发送消息“启动安装”。

