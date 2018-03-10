---
layout:     post                    # 使用的布局
title:      ContentProvider组件               # 标题 
subtitle:   ContentProvider组件       #副标题
date:       2017-03-31              # 时间
author:     candy1126xx                      # 作者
header-img: img/home-bg-o.jpg    #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签
    - Android应用层
---

ContentProvider的作用是暴露某应用的数据，提供增删查改这些数据的接口，并在过程中有权限校验等。所有ContentProvider都由AMS管理，但是增删查改的实际处理逻辑是运行在ContentProvider所属进程中的。

ContentResolver的作用是根据Uri查找匹配的ContentProvider，获取一个Cursor对象，用它操作这个ContentProvider管理的数据。也提供了打开文件流等方法。

## 向ContentProvider通信的Binder结构
**业务层**

* 定义接口：IContentProvider
* 服务端：ContentProviderNative
* 客户端：ContentProviderProxy

**实现层**ContentProvider.Transport/ContentProvider子类

## ContentResolver
#### query()：获取Cursor对象
在ContentResolver.query()中，

1. 由Uri获取unstableProvider；此过程是抽象方法acquireUnstableProvider()完成的；
2. 调用unstableProvider.query()获取Cursor；如果获取失败，由Uri获取stableProvider，此过程是抽象方法acquireProvider()完成的；然后再调用stableProvider.query()获取Cursor；
3. 把Cursor包装成CursorWrapperInner并返回；
4. 释放unstableProvider和stableProvider；此过程是抽象方法releaseProvider()和releaseUnstableProvider()完成的。

## AMS管理ContentProvider
#### ApplicationContentResolver
调用Activity.getContentResolver()获取到的是ApplicationContentResolver对象。

ApplicationContentResolver继承自ContentResolver，所有方法实现都是转调ActivityThread对象的相应方法，ActivityThread再向AMS发送信息。

另外，ApplicationContentResolver的创建是在应用进程启动过程中进行的。

#### 获取向ContentProvider通信的客户端

```
// 一条通信连接
class ContentProviderConnection {

}

class ContentProviderRecord {
	IContentProvider provider; // 通信服务端
	ArrayList<ContentProviderConnection> connections； // 存在的连接
}

class ActivityManagerService {
    // 其中用多个集合保存了Uri域名与ContentProviderRecord的对应关系
	final ProviderMap mProviderMap;
}
```

ActivityThread.acquireProvider() -> AMS.getContentProvider()。参数ApplicationThread对象,Uri中的域名部分。

1. 由ApplicationThread对象获取ProcessRecord对象；
2. 从mProviderMap中由Uri域名获取ContentProviderRecord对象；
3. 假设获取不到，即第一次有进程申请这个ContentProviderRecord对象；
4. 由PMS获取ContentProviderInfo对象；
5. 创建ContentProviderRecord对象；
6. 检测：ContentProvider的所属进程是否启动，如果没启动要启动；
7. 创建ContentProviderConnection对象；
8. 可能会执行一次进程Lru；
9. 把ContentProviderRecord对象放入mProviderMap；
10. 返回ContentProviderRecord.newHolder(ContentProviderConnection)创建的ContentProviderHolder对象。

#### 应用主动加载自己的ContentProvider

## 增删查改其它应用的数据库
```
// 增
getContentResolver().insert()；
// 删
getContentResolver().delete()；
// 查
Cursor c = getContentResolver().quary()；
c.getColumnNames()；
c.getColumnIndex()；
c.getColumnName()；
c.getBlob()/getString()/getShort()等；
// 改
getContentResolver().update()；
```

## 读出写入其它应用的文件
```
getContentResolver().openTypedAssetFileDescriptor()；
getContentResolver().openFileDescriptor()；
getContentResolver().openOutputStream()；
getContentResolver().openInputStream()；
```