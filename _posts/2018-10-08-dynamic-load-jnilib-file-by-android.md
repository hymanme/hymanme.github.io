---
layout: post
title: Android 中动态加载 so 文件
summary: 本篇文章介绍 Android 开发中如何动态加载 jniLib，也就是动态加载 native 的 so 库文件，以及介绍了关于 CPU、微架构、指令集、android Abi 选择、动态加载 jniLib 相对于静态加载的优缺点等。最终以一个实例实现动态 jniLib 的加载。
date: 2018-10-08 17:57:02
categories: Android
tags: [Android, JniLib, so lib]
featured-img: stones
---

本篇文章介绍 Android 开发中如何动态加载 jniLib，也就是动态加载 native 的 so 库文件，以及介绍了关于 CPU、微架构、指令集、android Abi 选择、动态加载 jniLib 相对于静态加载的优缺点等。最终以一个实例实现动态 jniLib 的加载。

## CPU、核心、微架构、指令集、ABI

>  在进入主题之前，先让我们了解几个专有名词肯定是大有裨益的，当然你也可以直接跳过本节直接进入主题。

CPU 想必很好理解，即中央处理器，主要用来处理一些计算密集型的任务。而且单个 CPU 可以同时进行多个任务，多个任务实际上并不是真的同时执行，要靠 CPU 高速调度几个任务，顺序执行，只是速度太快，我们感觉是在同时执行，但是多任务之间调度会影响到性能。随着技术飞速发展，出现了多核心 CPU，其实 CPU 主要就是靠得核心在工作，多个核心即可实现真正意义上的同时处理任务。

CPU 执行计算任务时都需要遵从一定的规范，程序在被执行前都需要先翻译为 CPU 可以理解的语言。这种规范或语言就是指令集（ISA，Instruction Set Architecture）。程序被按照某种指令集的规范翻译为 CPU 可识别的底层代码的过程叫做编译（compile）。常见的指令集有 x86、ARM、MIPS 等，其中指令集可以扩展，比如 x86 指令集可以增加 64 位支持，就变成 x86_64，同理 ARM 可以扩展成 arm64-v8a，MIPS 可以扩展为 MIPS64。

核心的实现方式被称为微架构（microarchitecture）。指令集是一套规范，是公开的，指令集的实现（微架构）是一个极具技术含量的工作，而且即便是你有这个技术，你想设计某套指令集的微架构，还需要得到该指令集的授权，否则会吃官司。而微架构的设计直接影响了核心可以达到的最高频率、核心在一定频率下能执行的运算量、一定工艺水平下核心的能耗水平等等，也就是微架构设计技术的好坏，决定了设计出核心性能的高低。

常见的代号如 Haswell、Cortex-A15 等都是微架构的称号。他们都是使用了某种指令集语言设计出来的一种微架构代号，比如 HasWell 就是兼容 x86 指令集的微架构、Cortex-A15 就是兼容 ARM 指令集的微架构。值得注意的是，一款 CPU 使用了 ARM 指令集不等于它就使用了 ARM 研发的微架构，因为 Intel、高通、苹果、Nvidia等厂商都自行开发了兼容 ARM 指令集的微架构，这也就是前面讲到的指令集授权。

[ABI](https://link.jianshu.com/?t=https%3A%2F%2Fdeveloper.android.google.cn%2Fndk%2Fguides%2Fabis.html) 是应用程序二进制接口（Application Binary Interface），说明它也是一种规范，有点类似指令集的概率。不同 Android 手机使用不同的 CPU，因此支持不同的指令集。CPU 与指令集的每种组合都有其自己的应用二进制界面（或 *ABI*）， ABI 可以非常精确地定义应用的机器代码在运行时如何与系统交互。目前 Android 平台主流的有：`armeabi`、`armeabi-v7a`、`x86`、`mips`、`arm64-v8a`、`mips64`、`x86_64`。

## Android 中常见的 ABI

常见的 ABI 有：`armeabi`、`armeabi-v7a`、`x86`、`mips`、`arm64-v8a`、`mips64`、`x86_64`。

谷歌官网提到了我们要为每一种 CPU 架构（指令集）指定 ABI，但是指的庆幸的是各种 ABI 存在向下兼容的特性，也就是说我们实际上并没有必要为每一种 CPU 架构都设计一套对应的 ABI 出来，但是通过向下兼容会在一定程度上影响性能和实际的兼容性，所以为了得到最优的性能，我们建议针对特定的 CPU 架构实现其对应的 ABI。

> 您必须为应用要使用的每个 CPU 架构指定 ABI。— Android 开发者文档

不到万不得已，或者说性能以及兼容性要求不高的情况下，可以借鉴一下以下 ABI 兼容表，使用某个兼容的 ABI 来在其他 CPU 架构上运行。

**表 1.** ABI 和支持的指令集。

| ABI                                                          | 支持的指令集                                                 | 说明                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| [`armeabi`](https://developer.android.com/ndk/guides/abis?hl=zh-cn#armeabi) | ARMV5TE 和更高版本<br />Thumb-1                              | 无硬浮点。只兼容 armeabi                                     |
| [`armeabi-v7a`](https://developer.android.com/ndk/guides/abis?hl=zh-cn#v7a) | armeabi<br />Thumb-2<br />VFPv3-D16<br />其他（可选）        | 与 ARMv5、v6 设备不兼容。兼容 armeabi-v7a、armeabi           |
| [`arm64-v8a`](https://developer.android.com/ndk/guides/abis?hl=zh-cn#arm64-v8a) | AArch-64                                                     | 兼容arm64-v8a、armeabi-v7a、armeabi                          |
| [`x86`](https://developer.android.com/ndk/guides/abis?hl=zh-cn#x86) | x86 (IA-32)<br />MMXSSE/2/3<br />SSSE3                       | 不支持 MOVBE 或 SSE4。常见于 pc 端，兼容 x86、armeabi（待考证） |
| [`x86_64`](https://developer.android.com/ndk/guides/abis?hl=zh-cn#86-64) | x86-64<br />MMXSSE/2/3<br />SSSE3<br />SSE4.1、4.2<br />POPCNT | 兼容 x86_64、x86                                             |
| [`mips`](https://developer.android.com/ndk/guides/abis?hl=zh-cn#mips) | MIPS32r1 及更高版本                                          | 使用硬浮点，并且假设 CPU:FPU 时钟比率为 2:1 以获取最大兼容性。 不提供 micromips 或 MIPS16。只兼容 mips |
| [`mips64`](https://developer.android.com/ndk/guides/abis?hl=zh-cn#mips64) | MIPS64r6                                                     | 兼容 mips64、mips                                            |

## 静态加载 so 库

对于某些计算密集型的运算，我们大多不会在 java 层来实现，一般使用 native 层处理，比如一些视频的解码等，最终会通过 C/C++ 来实现，然后编译成 so（共享）库供 Android 端使用。

#### 使用步骤

1. 将不同 ABI 的 so 包拷贝到 Android 项目的`app\src\main\jniLibs\` 目录下![jniLib](http://wx1.sinaimg.cn/mw690/005X6W83gy1fw17larinvj30ao09mweu.jpg)

2. 修改项目目录下的 gradle 文件，配置当前 App 支持的 ABI 类型

   ```groovy
    defaultConfig {
           ...
           ndk {
               //添加当前app所支持的abi。以下未显示添加的 jniLibs 目录下的 ABI 不会被打包进 apk
               //意思就是 目前只会打包以下4中abi，其他的均不打包
               abiFilters 'armeabi', 'armeabi-v7a', 'arm64-v8a', 'x86'
               // 还可以添加 'x86', 'x86_64', 'mips', 'mips64'
           }
       }
   ```

3. 如果还有 jar 包，将对应 jar 包拷贝到 Android 项目的`app\libs\`目录下

4. 刷新一下 Gradle，这里主要是为了加载一下 jar 包

5. 加载 so，一般使用`System.load(String filePath)`、`System.loadLibrary(String libName)`来加载指定 so 库文件，其中前者可以加载指定路径下的 so 文件，后者传入库的名称，不包括`lib`前缀以及后缀名，只能加载`Java.library.path`路径下的库文件，可以通过`System.getProperty('java.library.path')`获取这个路径。实际实验中系统会去`getApplicationInfo().nativeLibraryDir`查找对应的 so 库。

6. 调用 native 方法验证

**PS**：abiFilters 显示支持的 ABI 下的 so 库必须完整，比如我加了`armeabi`，`armeabi-v7a`两种 ABI，那么这两个 abi 目录下必须都包含相同的 so 文件，比如`armeabi-v7a`下有 a.so 那么`armeabi`目录下也必须有 a.so 文件，否则则会报错。如果你没有指定 ABI 对应的 so 文件，你可以对照 ABI 兼容表，将其兼容的 abi 拷贝进欠缺该 abi 的目录下。

## 产生问题

使用静态加载 jniLibs 十分简单，只要按到步骤操作即可完成 native 代码的加载，但是这种方法会有一些局限性。

* 增加 apk 体积大小
* 多个 sdk so文件同时加载可能会出现冲突
* 针对不同的 sdk 版本，单个 so 的兼容性不是很好
* 不能动态的对 so 库进行更新，除非发布新版本

针对以上几个问题，我们发现，静态加载 so 不够灵活，十分的呆板，所有的 so 文件都是安装 软件时候被焊死在 apk 包中非常的不易扩展，那有没有什么方法可以解决或者说优化这些问题呢？有，那就是动态加载共享链接库，接下来进入主题。

## 动态加载 so 库

#### 何为动态加载?

动态加载即我们不将所需 so 库直接拷贝到 apk 包里面，也就是用户安装的 apk 文件里面不包含任何 so 文件，将所需的 so 文件放在云端，待用户安装完 apk 使用我们的 app 时，先判断用户的手机 CPU 架构，再通过某种策略选择合适的时间去云端下载该用户手机最优 ABI 架构的 so 库文件到本地，然后当用户实际需要使用 native 功能时，再手动`System.load(lib)`，最后使用 native 方法或者数据结构，至此动态加载完成。

#### 动态加载的优点

* 大大减小了 apk 体积大小，我们实际只需要当前 ABI 的库文件
* 可以做到动态更新 so 库，比如我们给 so 一个版本号来控制 so 的更新工作
* 可以更大粒度的实现兼容，不仅仅只针对 ABI 类型，还可以针对 SDK 版本，手机上下文等加载定制版 so 库
* 看上去很牛批

#### 实现步骤

1. 生成所需的各种 ABI 版本的 so 库文件，保存在云端，记录访问地址
2. 设计某种策略，在特定时间触发初始化任务
3. 初始化任务中，首先通过`Build.CPU_ABI`获取当前设备最优 ABI 架构，除了`Build.CPU_ABI`还有第二 ABI，通过`Build.CPU_ABI2`获取，sdk21 之后推荐使用``Build.SUPPORTED_ABIS`获取，此为一个数组，数组元素越靠前，越是当前设备最优 ABI
4. 去云端下载对应 ABI 类型的 so 文件到本地 app 的用户空间，如：`getDir("libs", Context.MODE_PRIVATE).getAbsolutePath()`，对应的路径为`data\data\package\app_libs\`
5. 使用 native 功能时，调用 `System.load(lib)`方法去加载刚下载下来的 so 库文件，`lib`为刚下载的库文件结对路径
6. 调用 native 方法验证

#### 实战

1. 新建静态加载 so 案例

   本案例使用的是声网 Agora 信令 SDK，参考 [Agora Signal](https://docs.agora.io/cn/2.1/addons/Signaling/Quickstart%20Guides/signal_android?platform=Android) ，配置好 sdk 并验证 sdk 是否可用，这一次主要是避免接下来动态加载中由于 sdk 问题而导致的异常

2. 删除 jniLibs 目录下所有文件只保留一个空的 jniLibs 目录，并将删除的 jniLib 中各种 ABI 文件分类上传至云端

   ![so cloud](http://wx2.sinaimg.cn/mw690/005X6W83gy1fw1agnoblxj30p2032t9a.jpg)

3. 修改 gradle 文件

   ```groovy
   defaultConfig {
           applicationId "com.hymane.dynamicloadso"
           minSdkVersion 16
           targetSdkVersion 28
           versionCode 1
           versionName "1.0"
           testInstrumentationRunner "androidx.test.runner.AndroidJUnitRunner"
           multiDexEnabled true
           resConfigs "zh"
           ndk {
               //选择要添加的对应cpu类型的.so库。
               abiFilters 'armeabi', 'armeabi-v7a', 'arm64-v8a', 'x86'
               // 还可以添加 'x86', 'x86_64', 'mips', 'mips64'
           }
       }
   ```

4. 获取当前设备的 ABI 版本

   ```java
   //获取当前设备的ABI
   public static AbiType getCurrentAbi() {
           return AbiType.getAbi(Build.CPU_ABI);
   }
   
   //定义一个 ABI 类型枚举
   public enum AbiType {
       ARMEABI("armeabi"), ARMEABI_V7A("armeabi-v7a"), ARM64_V8A("arm64-v8a"),
       X86("x86"), X86_64("x86_64"),
       MIPS("mips"), MIPS64("mips64");
       private String abiName;
       AbiType(String abiName) {
           this.abiName = abiName;
       }
       public String getAbiName() {
           return abiName;
       }
       public void setAbiName(String abiName) {
           this.abiName = abiName;
       }
       public static AbiType getAbi(String displayString) {
           if (displayString == null) {
               return null;
           }
           for (AbiType type : AbiType.values()) {
               if (type.getAbiName().equals(displayString.toLowerCase())) {
                   return type;
               }
           }
           return null; //not found
       }
   }
   ```

5. 通过当前 abi 去下载对应 so 文件

   ```java
       /***
        * so初始化工作，未初始化过则下载对应so至本地
        * 然后再进行load lib，如果so均已准备好，则直接load
        */
       private void init() {
           Log.d(TAG, "init:");
   
           if (abi == null) {
               Log.d(TAG, "init: error: no abi found");
               return;
           }
           Log.d(TAG, "init: abi=" + abi.getAbiName());
           Log.d(TAG, "init: abi path:"+System.getProperty("java.library.path"));
   
           Log.d(TAG, "init: "+getApplicationInfo().nativeLibraryDir);
   
           downloadSo(url + abi.getAbiName() + "/" + soName, getDir("libs", Context.MODE_PRIVATE).getAbsolutePath());
       }
   	//下载指定so文件，下载完毕进行加载load
       private void downloadSo(final String filePath, final String savePath) {
           Request request = new Request.Builder()
                   .url(filePath)
                   .build();
           OkHttpClient client = new OkHttpClient();
           File path = new File(savePath);
           if (!path.exists()) {
               path.mkdirs();
           }
           File file = new File(savePath, soName);
           if (file.exists()) {
               //已经下载
               Log.d(TAG, "downloadSo: so已存在，直接加载");
               load(file);
               return;
           }
           client.newCall(request)
                   .enqueue(new Callback() {
                       @Override
                       public void onFailure(Call call, IOException e) {
                           Log.d(TAG, "download onFailure: error=" + e.getMessage());
                       }
   
                       @Override
                       public void onResponse(Call call, Response response) throws IOException {
                           InputStream is = null;
                           byte[] buf = new byte[2048];
                           int len = 0;
                           FileOutputStream fos = null;
                           // 储存下载文件的目录
                           try {
                               is = response.body().byteStream();
                               long total = response.body().contentLength();
                               File file = new File(savePath, soName);
                               fos = new FileOutputStream(file);
                               long sum = 0;
                               while ((len = is.read(buf)) != -1) {
                                   fos.write(buf, 0, len);
                                   sum += len;
                                   int progress = (int) (sum * 1.0f / total * 100);
                                   // 下载中
                               }
                               fos.flush();
                               // 下载完成
   //                            listener.onDownloadSuccess();
                               Log.d(TAG, "download file(" + soName + ") success: save path:" + filePath);
                               load(file);
                           } catch (Exception e) {
                               Log.d(TAG, "download onFailure: error=" + e.getMessage());
                           } finally {
                               try {
                                   if (is != null)
                                       is.close();
                               } catch (IOException e) {
                               }
                               try {
                                   if (fos != null)
                                       fos.close();
                               } catch (IOException e) {
                               }
                           }
                       }
                   });
       }
   
       private void load(File soFile) {
           System.load(soFile.getAbsolutePath());
       }
   ```

6. 调用 native 方法验证

   ```java
   	/***
        * 初始化sdk
        */
       private void start() {
           Log.d(TAG, "start:");
           m_agoraAPI = AgoraAPIOnlySignal.getInstance(this, "your appkey");
           m_agoraAPI.login2("your appkey", "hymane", "_no_need_token", 0, null, 30, 3);
           m_agoraAPI.channelJoin("myChannel");
           m_agoraAPI.callbackSet(new AgoraAPI.CallBack() {
               @Override
               public void onLoginSuccess(int uid, int fd) {
                   super.onLoginSuccess(uid, fd);
                   Log.d(TAG, "onLoginSuccess: uid=" + uid);
               }
   
               @Override
               public void onLoginFailed(int ecode) {
                   super.onLoginFailed(ecode);
                   Log.d(TAG, "onLoginFailed: ecode=" + ecode);
               }
   
               @Override
               public void onMessageSendSuccess(String messageID) {
                   super.onMessageSendSuccess(messageID);
                   Log.d(TAG, "onMessageSendSuccess: messageId=" + messageID);
               }
   
               @Override
               public void onMessageChannelReceive(String channelID, String account, int uid, String msg) {
                   super.onMessageChannelReceive(channelID, account, uid, msg);
                   Log.d(TAG, "onMessageChannelReceive: channelId=" + channelID + " account=" + account + " message:" + msg);
               }
   
               @Override
               public void onChannelJoined(String channelID) {
                   super.onChannelJoined(channelID);
                   Log.d(TAG, "onChannelJoined: channelId=" + channelID);
               }
           });
       }
   
       /***
        * 使用sdk api
        * 验证是否正常
        */
       private void send() {
           Log.d(TAG, "send:");
           m_agoraAPI.messageChannelSend("myChannel", String.valueOf(count++), "");
       }
   ```

   当验证工作未出现异常，且打印出`onMessageChannelReceive`日志即表示动态加载成功。

**PS**：注意一点，通过以上你可能会发现会崩溃，错误为：

```java
2018-10-09 00:48:25.842 9384-9384/com.hymane.dynamicloadso E/AndroidRuntime: FATAL EXCEPTION: main
    Process: com.hymane.dynamicloadso, PID: 9384
    java.lang.UnsatisfiedLinkError: dalvik.system.PathClassLoader[DexPathList[[zip file "/data/app/com.hymane.dynamicloadso-3vJm7daseRaWhfeeF5pHpQ==/base.apk"],nativeLibraryDirectories=[/data/app/com.hymane.dynamicloadso-3vJm7daseRaWhfeeF5pHpQ==/lib/x86, /system/lib]]] couldn't find "libagora-sig-sdk-jni.so"
        at java.lang.Runtime.loadLibrary0(Runtime.java:1012)
        at java.lang.System.loadLibrary(System.java:1669)
        at io.agora.NativeAgoraAPI.<clinit>(NativeAgoraAPI.java:181)
        at io.agora.AgoraAPIOnlySignal.getInstance(AgoraAPIOnlySignal.java:57)
        at com.hymane.dynamicloadso.MainActivity.start(MainActivity.java:179)
        at com.hymane.dynamicloadso.MainActivity.access$100(MainActivity.java:26)
        at com.hymane.dynamicloadso.MainActivity$2.onClick(MainActivity.java:58)
        at android.view.View.performClick(View.java:6597)
        at android.view.View.performClickInternal(View.java:6574)
        at android.view.View.access$3100(View.java:778)
        at android.view.View$PerformClick.run(View.java:25885)
        at android.os.Handler.handleCallback(Handler.java:873)
        at android.os.Handler.dispatchMessage(Handler.java:99)
        at android.os.Looper.loop(Looper.java:193)
        at android.app.ActivityThread.main(ActivityThread.java:6669)
        at java.lang.reflect.Method.invoke(Native Method)
        at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:493)
        at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:858)
```

以上错误可以看出，是在指定目录（默认 app 私有空间）下未找到我们的 so 文件，我们明明手动加载了自己下载的 so 文件了，为什么没有生效？

这个问题主要是 agora sdk 初始化的时候已经加载了一次 so 文件，查看`agora-sig-sdk.jar`文件中源码可以发现，`NativeAgoraAPI`如下：

```java
public class NativeAgoraAPI implements IAgoraAPI {
    public NativeAgoraAPI() {
    }
	...
    //这里调用loadLibrary加载system lib下的so文件
    //但是我们不是静态加载so，系统没有将我们的so文件拷贝到系统lib下
    //导致找不到对应文件
    static {
        System.loadLibrary("agora-sig-sdk-jni");
    }
    ...
}
```

知道问题了，我们就可以解决了，只需要修改此处源码，将

```java
static {
  	System.loadLibrary("agora-sig-sdk-jni");
}
```

代码删掉就可以，然后我们自己 load 的 so 文件就生效了，拷贝初始化 sdk 所需的类以及需要改动的类（NativeAgoraAPI）的源码，保存在项目目录下的对应的包下面，此处需要为修改的类保存在 sdk 原包名`io.agora`下。

![modify code](http://wx4.sinaimg.cn/mw690/005X6W83gy1fw1b27han6j30do04gt8x.jpg)

接下来再初始化 sdk 调用 native 方法即可。

## 总结

随着项目的逐渐扩大，so 文件的量以及占 apk 体积大小也会逐渐增大，也就伴随着各种问题，比如前面提到的 so 库加载冲突，apk 体积过大等问题。而动态加载不仅可以解决以上问题还使得加载 so 库变得更加灵活，定制性更强，所以动态加载 so 是一个较好的方案。当然，这并不是一本万利，也有其值得深思熟虑的问题，比如何时去下载 so 文件，这就需要制定一定的策略了，由于下载需要时间，还需要考虑网络异常等问题，其间需要考虑的情景也就变得更加复杂。更有甚者，策略未处理完善，用户使用功能时，发现 so 文件未下载，还需要花时间去下载，这样用户就需要花时间去等待，极大降低了用户体验，尤其是那些用户一打开 app 就立马会使用到的 native 功能，所以实际开发中，最好是静态加载和动态加载一起配合使用，动态加载只用在那些用户接触需要较深的操作时候。

## 参考

* [https://developer.android.com/ndk/guides/abis](https://developer.android.com/ndk/guides/abis)
* [https://zhuanlan.zhihu.com/p/19893066](https://zhuanlan.zhihu.com/p/19893066)
* [http://www.ywnds.com/?p=3561](http://www.ywnds.com/?p=3561)
* [http://honghui.github.io/blog/20160311/android-xi-tong-ru-he-xuan-ze-native-so.html](http://honghui.github.io/blog/20160311/android-xi-tong-ru-he-xuan-ze-native-so.html)
* [https://blog.csdn.net/aqi00/article/details/72763742](https://blog.csdn.net/aqi00/article/details/72763742)