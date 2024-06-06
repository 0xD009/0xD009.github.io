---
title: Android逆向工程基础
published: 2024-04-11
description: Android逆向相比C/C++逆向有更高的门槛，需要更多先验知识，本文将简单谈谈这些基础的常识抛砖引玉
tags:
  - Android
category: CTF or Security
draft: false
---

## 0x01 前言
Android 逆向一直是逆向工程的一个重要分支，随着移动端APP的势头越来越盛，其安全问题也称为重要的议题。

在 CTF 比赛中，Android 有时划在逆向工程方向，有时也设立独立的方向，凭个人感觉，简单题的主要逻辑都在 Java 层，Java层无论怎么混淆，相比 C++ 层（JNI 层）还是比较好逆的，而困难题除了 C++ 外，还可能涉及到 Android 系统或者生态独有的知识，对于无 Android 开发基础的选手来说比较不友好。

笔者学过一段时间的安卓开发，但是半途而废，后面转到安全方向之后发现上手 Android 逆向还挺快的，目前已经是大三了，打 CTF 也有一年半了，投了一些安全相关的实习岗位，由于个人发展重心一直在 CTF 比赛上，而且实践居多，遂打算通过写一些东西来整理以下思路，应对面试或者说工业界的考核。

笔者希望首先从 APK 文件和 Android 系统的常识开始写，后面总结一些 APK 文件的静态和动态分析工具和方法，最后做一些扩展或者说进阶，考虑学术界或者工业界关注的一些方向，比如某些框架和移动端的一些安全机制（如指纹）。

本篇的内容重心在提供一些基本的常识上，让初学者不至于一点思路都没有，但是各部分都只是泛泛地谈谈，抛砖引玉。对笔者本人来说也只起到整理思路的作用。

## 0x2 认识APK文件结构
从[52pojie](https://www.52pojie.cn/thread-1695796-1-1.html)的一个教学贴上直接拿到下面这张表：

| 文件                    | 注释                                                                                                 |
| --------------------- | -------------------------------------------------------------------------------------------------- |
| assets目录              | 存放APK的静态资源文件，比如视频，音频，图片等                                                                           |
| lib 目录                | armeabi-v7a基本通用所有android设备，arm64-v8a只适用于64位的android设备，x86常见用于android模拟器，其目录下的.so文件是c或c++编译的动态链接库文件 |
| META-INF目录            | 保存应用的签名信息，签名信息可以验证APK文件的完整性，相当于APK的身份证(验证文件是否又被修改)                                                 |
| res目录                 | res目录存放资源文件，包括图片，字符串等等，APK的脸蛋由他的layout文件设计                                                         |
| AndroidMainfest.xml文件 | APK的应用清单信息，它描述了应用的名字，版本，权限，引用的库文件等等信息                                                              |
| classes.dex文件         | classes.dex是java源码编译后生成的java字节码文件，APK运行的主要逻辑                                                       |
| resources.arsc文件      | resources.arsc是编译后的二进制资源文件，它是一个映射表，映射着资源和id，通过R文件中的id就可以找到对应的资源                                    |
APK（Android Package） 文件的本质就是压缩包，通过改后缀 zip 或者直接用解压软件打开即可看到文件的结构：

![](blogs/Android逆向工程基础/image-20240313183258639.png)

一般逆向的话就是关注代码逻辑部分，Java 代码可以通过工具得到，而 C++ 层的代码就在 lib 文件夹下，有时会有针对不同架构编译的不同 .so 文件。

注意这里架构带来的影响，一般来说 PC 上的 Android 模拟器都是 x86 架构的，如果使用 IDA Pro 远程调试，dbg_server 也要选 x86 架构，而 armeabi-v7a 和 arm64-v8a 都是 ARM 架构的，在模拟器上虽然可能是通过转译或者其他途径跑起来了，但是由于没有 x86 架构的链接库，是不好调试的，具体效果可以自行尝试。

除此之外，有时也涉及一些元数据的修改，也是从上面的帖子里直接拿来了一张表：

|属性|定义|
|---|---|
|versionCode|版本号，主要用来更新，例如:12|
|versionName|版本名，给用户看的，例如:1.2|
|package|包名，例如：com.zj.52pj.demo|
|uses-permission android:name=""|应用权限，例如：android.permission.INTERNET 代表网络权限|
|android:label="@string/app_name"|应用名称|
|android:icon="@mipmap/ic_launcher"|应用图标路径|
|android:debuggable="true"|应用是否开启debug权限|
用的比较多的就是包名是否可调试了。


## 0x3 Activity及其生命周期
APK 文件的结构可以认为是 APK 的外观，能让我们对它有一些初步的判断，而其大名鼎鼎的四大组件就是它的四肢，各司其职相互配合从而满足用户的需求。

本文主要是讲 Android 逆向相关的内容，开发方面的常识只简单说说，而且碍于笔者学习进度堪忧，也讲不深刻。

所谓四大组件即 Activity，Service, ContentProvider, BroadcastReceiver

如果学习安卓开发的话，首先就能接触到 Activity，这是四大组件中最重要的一个。一个 Activity 可以认为是一个窗口，而一般地，APK 会有一个 MainActivity 类，约等于 C 语言编程里的 main 函数，一些简单的逆向题的逻辑也全在这个类里了，当然，它也可以叫其他名字。

在 AndroidManifest.xml 文件中找到如下标签：
```xml
<intent-filter>
	<action android:name="android.intent.action.MAIN"/>
	<category android:name="android.intent.category.LAUNCHER"/>
</intent-filter>
```
拥有这个标签的即为启动活动。

如下是 Activity 的生命周期：

![](blogs/Android逆向工程基础/image-20240313194429285.png)

关于 Activity 的生命周期，可以看[简书](https://www.jianshu.com/p/fb44584daee3)上的这篇文章，作为逆向者我们需要知道，这其中的每一个 `OnXxx()` 函数，都是回调函数，在生命周期发生变化时即会被调用，这成为了 APP 逻辑运转的骨架。比如 `OnCreate()` 函数里会涉及一些初始化操作。


## 0x4 Smali代码
Java 运行在 JVM 上，而 Android 系统中，为了解决其的某些痛点，引入了 ART 虚拟机，将源代码进行预编译，以加快运行速度，类比地看，Java 源文件相当 C 源文件，dex 包相当于编译好的可执行 PE 或者 ELF 文件， 而 Smali 就相当于 Android 上的汇编语言。

Smali 代码的可读性很高，其基于寄存器的机制上也不复杂，初学者通过查阅文档就能做到很快理解，所以笔者不在此赘述 Smali 代码的语法，而且实际场景中，阅读 Smali 代码的情况很少，大多数情况下，我们的工具可以直接提供可读性非常高的几乎源码级别的 Java 代码，然而当我们需要对代码做 patch 时，还是需要在 Smali 上修改，这是学习 Smali 的最重要的原因。

学习 Smali 和其他的编程语言一样，关注它的数据类型，语法关键词，控制流实现，函数和类的实现，最后关于逆向的——关注它怎么使用寄存器（p 寄存器和 v 寄存器有何区别）。

## 0x5 APK开发流程

apk 有自己的前端和后端，前端由 xml 写成，后端由 Java 或者 Kotlin 写成，有时还会利用 JNI 写原生 C++，所以攻击者也可以从这些方面入手，比如根据前端的按钮追踪负责这一部分的代码的逻辑。

一般地，建立前后端之间联系的方法是通过：`findViewById`方法。
```java
mTrueButton = (Button) findViewById(R.id.true_button);
```

以这种形式去把视图绑定到 Java 变量上，接下来可以通过代码逻辑更改视图的属性，或者注册事件监听函数：
```java
mTrueButton.setOnClickListener(new View.OnClickListener() { //这里使用了匿名内部类
	@Override
	public void onClick(View view) {
		checkAnswer(true);
	}
});
```
如前文所说，这段代码实现在`MainActivity`的`OnCreate`方法中，简单的 apk 基本都是如此。

而在前端部分，按钮是这样写的：
```xml
<Button
	android:id="@+id/true_button"
	android:layout_width="wrap_content"
	android:layout_height="wrap_content"
	android:text="@string/true_button" />
```

注意 Button 的属性`id`，这是用于唯一标识视图的，经过编译后，所有的 id 都会存放在 R 类中，表现形式为：
```java
public static final int true_button = 1145141919810;
public static final int background_grey = 2131034143;
public static final int black = 2131034146;
public static final int blue_200 = 2131034147;
public static final int blue_500 = 2131034148;
public static final int blue_700 = 2131034149;
public static final int green = 2131034209;
public static final int purple = 2131034307;
public static final int red = 2131034309;
public static final int teal_200 = 2131034322;
public static final int teal_700 = 2131034323;
public static final int white = 2131034328;
public static final int yellow = 2131034329;
public static final int abc_background_cache_hint_selector_material_dark = 2131034112;
public static final int abc_background_cache_hint_selector_material_light = 2131034113;
```

这之间形成一个“View-Id-Variable”的关系，同时也可以注意到 Button 的`text`属性，也用到了类似的思路去绑定资源和 View。

除了前后端开发之间的联系，还存在 Java 层和 Native 层之间的联系，一般地，如果一个符号`native`修饰，那么就存在这种联系：
```java
package com.example.myapp;

public class MyActivity extends AppCompatActivity {
    // 声明本机函数
    public native void nativeFunction();

    static {
        // 加载本机库
        System.loadLibrary("native-lib");
    }

    // 在某个方法中调用本机函数
    public void callNativeFunction() {
        nativeFunction();
    }
}
```

而 Native 层的写法是这样：
```c
#include <jni.h>
#include <stdio.h>

extern "C" JNIEXPORT void JNICALL
Java_com_example_myapp_MyActivity_nativeFunction(JNIEnv *env, jobject obj) {
    printf("Hello from native function!\n");
}
```

注意到两个类`JNIEnv`和`jobject`，包括`jclass`和`jstring`，都是在 IDA Pro 中有预设的，在 arm 架构下的 so 库中可以之间将函数中的形参`a1`的类型改为`JNIEnv*`，这将大大增加可读性。

## 0x6 总结

在本文中，笔者提到了 Android 逆向的诸多常识（其中可能有说的不准确的），有很多细节没有讲到，也不便讲。本文为理清个人思路而作，讲的比较抽象，如果即使这样读者能够完全读懂，说明已经有了足够的先验知识，可以对付 CTF 中的一些入门级甚至中档的 Android 题了。

