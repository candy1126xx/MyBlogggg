---
layout:     post                    # 使用的布局
title:      ServiceManager进程               # 标题 
subtitle:   ServiceManager进程 #副标题
date:       2017-03-12              # 时间
author:     candy1126xx                      # 作者
header-img: img/home-bg-o.jpg    #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签
    - Android框架层
---

## 简介
ServiceManager进程是由init进程创建的。源码文件是service_native.c。它是Native层的、系统全局“服务端”的管理者。Java层系统全局“服务端”的管理者是运行在SystemServer进程的ServiceManager对象。

## 启动后做了些什么？
1. 创建一个binder_state结构体；
2. 系统调用open，以可读可写(O\_RDWR)模式打开文件"/dev/binder"，返回“文件描述符”到binder_state -> fd；
3. 创建一个binder_proc结构体保存Binder机制的全局信息；
4. 系统调用ioctl(BINDER_VERSION)，设置Binder Version；
5. binder_state -> mapsize = 128*1024；
6. 系统调用mmap，把binder\_state -> fd映射到内存，返回“映射区地址“到binder_state -> mapped；
7. 系统调用ioctl(BINDER\_SET\_CONTEXT\_MGR)，把Zygote进程设置为守护进程；
8. 系统调用ioctl(BINDER\_WRITE\_READ)，写入BC\_ENTER\_LOOP，告诉Binder驱动开始循环；
9. 开始循环，在一次循环中有两步操作：第一步从Binder驱动读出一个binder\_write\_read；第二步解析这个binder\_write\_read，执行相应操作。

## 服务端注册到ServiceManager
以MediaService为例

#### 1.获取ServiceManager对象在MediaService进程的客户端
在IServiceManager.cpp -> defaulteServiceManager()：

1. gDefaulteServiceManager是定义在Static.cpp中的单例；如果gDefaulteServiceManager不为null，直接返回；否则调用interface_cast<IServiceManager>(ProcessState::self() -> getContextObject(NULL))创建。
2. ProcessState::self() -> getContextObject(NULL)会返回一个句柄为0的BpBinder，即ServiceManager在通信层的客户端。
3. 调用BpBinder.queryLocalInterface()获取ServiceManager业务层客户端在MediaService进程的实例，不为null则返回实例的指针；否则以BpBinder为参数创建一个BpServiceManager实例，并返回它的指针。

至此，MediaService进程有了一个指向BpServiceManager实例的指针。客户端向ServiceManager发送信息，相当于调用BpServiceManager的方法 ，转调BpBinder -> transtact()，转调IPCThreadState::self() -> transact()。

#### 2.MediaService进程向Binder驱动写入数据
调用BpServiceManager -> addService()，转调BpBinder -> transtact(ADD\_SERIVCE\_TRANSACTION)，转调IPCThreadState::self() -> transact()：

1. 向mOut写入指令吗BC_TRANSACTION；
2. 创建一个binder\_transaction\_data，其中target.handle = 0，代表ServiceManager进程的句柄；code = ADD\_SERIVCE\_TRANSACTION，请求码；data\_xxx 写入了三样： IServiceManager -> getInterfaceDescriptor()、name即“android.os.service_manager”、指向位于MediaService进程中的MediaService服务端的指针；
3. 把binder\_transaction_data写入mOut；
4. 创建一个binder\_write_read，其中“写”的部分为mOut，“读”的部分为mIn；
5. 系统调用ioctl(BINDER\_WRITE\_READ)，写入binder\_write\_read；

至此，MediaService进程向Binder驱动写入了一个binder\_write\_read。

#### 3.ServiceManager进程从Binder驱动读出数据
数据是一个binder\_write\_read，其中指令码为BR\_TRANSACTION，请求码为SVC\_MGR\_ADD_SERVICE。

1. 转调service\_manager.c -> do\_add_service()；
2. 检测要注册的服务是否可以被添加到ServiceManager；只有当操作者是root或system进程，或者被添加的服务在allowed中定义过，才能被添加；
3. 搜索svclist，看服务是否已经存在；如果不存在，创建svcinfo，并添加到svclist。

至此，service\_manager.c -> svclist多了一个svcinfo，其中，handle为？？？，name为“android.os.service_manager”。