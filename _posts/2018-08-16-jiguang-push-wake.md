---
layout: post
title: 安卓手机提高推送到达率-极光同设备唤醒
summary: 安卓系统是一个高度开放的系统，自由度高，但是统一性就特别查，尤其是在国内，安卓系统接入不了GCM，只能通过长连接的形式来完成推送的服务...
date: 2018-08-16 14:34:22
categories: Android
tags: [Android, Push]
featured-img: stones
---

## 背景
安卓系统是一个高度开放的系统，自由度高，但是统一性就特别查，尤其是在国内，安卓系统接入不了GCM，只能通过长连接的形式来完成推送的服务。这种长连接是软件层次上的实现，所以避免不了如果软件被杀死之后，长连接就会断开，推送服务也就随即失效。iOS系统推送和安卓GCM类似，是统一的系统级别推送，到达率极高，几乎只要你手机开机、网络良好就能收到推送。但是国内安卓推送生态就很难看了。

## 实现
目前安卓系统推送的实现方式可以选择的也很多，大体如下：

1. 手动撸；自己实现长连接，建立推送服务，工作量大，极不推荐
2. 接入手机大厂推送；如小米、华为等大厂的推送，用户群体大，效果明显，尤其是小米推送，在 miui 系统上推送的到达率还是非常高的
3. 接入三方推送；如极光、友盟、腾讯信鸽、百度云推等，接入三方推送工作量较低，能快速完成推送接入。推荐

## 优化
### 问题
目前项目中用的是极光推送，较为基础的集成，但是推送到达率极低，因为 app 没有做保活功能，当软件退出之后，推送就不能及时到达，之所以说是不能及时到达，是因为当用户再次打开软件，未收到的消息会再次推送过来，极光推送免费版会缓存5条推送10天时间。
这样对于软件是极不友好的，有没有什么方法可以提升推送到达率呢？
### 方案
解决方案有很多，各有各的优缺点

#### 1. 与手机厂商合作，加入手机系统白名单
    效果明显，但是成本较高，要不你的软件很知名，要不你有钱，😑
#### 2. 软件保活
    只要软件不死，总能收到消息推送的，效果明显，但是保活不是一个简单的工作，尤其是在面对国内众多高定制的手机 rom。
#### 3. 接入手机厂商自家推送
    效果明显，类似和厂商合作，但是接入成本较高，而且还要接入各家推送。
#### 4. 三方推送优化
    效果一般，三方推送也有定制手机厂商的sdk，如友盟，极光，但是极光需要开通 vip 才可使用该功能，月费2400+。🙄
        
### 试试看
可以按方案从下往上来处理，毕竟先把容易实现的处理掉啊。就极光推送而言
#### 1. 推送配置
```
    <!-- Required SDK 核心功能-->
        <!-- 可配置android:process参数将PushService放在其他进程中 -->
        <service
            android:name="cn.jpush.android.service.PushService"
            android:enabled="true"
            android:exported="false">
            <intent-filter>
                <action android:name="cn.jpush.android.intent.REGISTER" />
                <action android:name="cn.jpush.android.intent.REPORT" />
                <action android:name="cn.jpush.android.intent.PushService" />
                <action android:name="cn.jpush.android.intent.PUSH_TIME" />
            </intent-filter>
        </service>

        <!-- since 3.0.9 Required SDK 核心功能-->
        <provider
            android:name="cn.jpush.android.service.DataProvider"
            android:authorities="${applicationId}.DataProvider"
            android:exported="false" />

        <!-- since 1.8.0 option 可选项。用于同一设备中不同应用的JPush服务相互拉起的功能。 -->
        <!-- 若不启用该功能可删除该组件，将不拉起其他应用也不能被其他应用拉起 -->
        <service
            android:name="cn.jpush.android.service.DaemonService"
            android:enabled="true"
            android:exported="true">
            <intent-filter>
                <action android:name="cn.jpush.android.intent.DaemonService" />
                <!--<category android:name="${applicationId}" />-->
            </intent-filter>
        </service>

        <!-- Required SDK核心功能-->
        <receiver
            android:name="cn.jpush.android.service.PushReceiver"
            android:enabled="true">
            <intent-filter android:priority="1000">
                <action android:name="cn.jpush.android.intent.NOTIFICATION_RECEIVED_PROXY" />
                <category android:name="${applicationId}" />
            </intent-filter>
            <intent-filter>
                <action android:name="android.intent.action.USER_PRESENT" />
                <action android:name="android.net.conn.CONNECTIVITY_CHANGE" />
            </intent-filter>
            <!-- Optional -->
            <intent-filter>
                <action android:name="android.intent.action.PACKAGE_ADDED" />
                <action android:name="android.intent.action.PACKAGE_REMOVED" />

                <data android:scheme="package" />
            </intent-filter>
        </receiver>

        <!-- Required SDK核心功能-->
        <activity
            android:name="cn.jpush.android.ui.PushActivity"
            android:configChanges="orientation|keyboardHidden"
            android:exported="false"
            android:theme="@android:style/Theme.NoTitleBar">
            <intent-filter>
                <action android:name="cn.jpush.android.ui.PushActivity" />

                <category android:name="android.intent.category.DEFAULT" />
                <category android:name="${applicationId}" />
            </intent-filter>
        </activity>

        <!-- SDK核心功能-->
        <activity
            android:name="cn.jpush.android.ui.PopWinActivity"
            android:configChanges="orientation|keyboardHidden"
            android:exported="false"
            android:theme="@style/MyDialogStyle">
            <intent-filter>
                <category android:name="android.intent.category.DEFAULT" />
                <category android:name="${applicationId}" />
            </intent-filter>
        </activity>

        <!-- Required SDK核心功能-->
        <service
            android:name="cn.jpush.android.service.DownloadService"
            android:enabled="true"
            android:exported="false">

        </service>

        <!-- Required SDK核心功能-->
        <receiver android:name="cn.jpush.android.service.AlarmReceiver" />


        <!-- User defined.  For test only  用户自定义的广播接收器-->
        <receiver
            android:name="${applicationId}.receiver.MyReceiver"
            android:enabled="true"
            android:exported="false">
            <intent-filter>
                <action android:name="cn.jpush.android.intent.REGISTRATION" />
                <!--Required  用户注册SDK的intent-->
                <action android:name="cn.jpush.android.intent.UNREGISTRATION" />
                <action android:name="cn.jpush.android.intent.MESSAGE_RECEIVED" />
                <!--Required  用户接收SDK消息的intent-->
                <action android:name="cn.jpush.android.intent.NOTIFICATION_RECEIVED" />
                <!--Required  用户接收SDK通知栏信息的intent-->
                <action android:name="cn.jpush.android.intent.NOTIFICATION_OPENED" />
                <!--Required  用户打开自定义通知栏的intent-->
                <action android:name="cn.jpush.android.intent.ACTION_RICHPUSH_CALLBACK" />
                <!--Optional 用户接受Rich Push Javascript 回调函数的intent-->
                <action android:name="cn.jpush.android.intent.CONNECTION" />
                <!-- 接收网络变化 连接/断开 since 1.6.3 -->
                <category android:name="${applicationId}" />
            </intent-filter>
        </receiver>
```
#### 2. 同设备唤醒
其中 `cn.jpush.android.service.DaemonService` 是极光推送一个可选 Service ，主要用户同一设备不同 app 互相唤醒。意思就是手机上装有多个接入极光推送的 app 之后，只要有一个 app 打开了，就可以收到推送。但是官方文档上只是注释上一笔带过，并没有说明这个具体使用方式。
如果添加了`category`那手机上只有具有相同`category`的软件才可以互相唤醒，当然你可以不填，会有默认的`category`，这样匹配率是最高的。这样设置完毕才算是真的同设备唤醒。




