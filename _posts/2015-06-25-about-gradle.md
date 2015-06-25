---
layout: post
title: "Gradle的一些总结"
modified: 2015-06-25 17:00:32 +0800
tags: [android,android studio,gradle,tool]
image:
  feature: abstract-3.jpg
  credit: 
  creditlink: 
comments: true
share: true
---
## 简介

Android Studio默认使用Gradle作为构建工具，不像Ant那样基于XML，取而代之的是采用基于Groovy的DSL（Domain Specific Language）。

Gradle构建工具是任务驱动型的构建工具，并且可以通过各种Plugin扩展功能以适应各种构建任务。

采用约定优于配置的原则，最简单方式是使用一个默认的目录结构。当然目录结构是可以自己修改的。

## Gradle Build Files

新建一个Android Studio项目时，会默认生成以下Gradle相关文件。

### Project级别的build.gradle文件

```text
buildscript {
 
//编译Gradle脚本过程中需要的依赖关系//
 
    repositories {
 
        jcenter()
    }
 
//选择JCenter repository.//
 
    dependencies {
 
   classpath 'com.android.tools.build:gradle:1.2.3'
 
//脚本依赖Android plugin for Gradle 1.2.3// 
 
    }
}
 
allprojects {
 
//你应用的依赖关系//
 
    repositories {
        jcenter()
    }
}
 
//应用依赖jCenter repository.//
```


### Module级别的build.gradle文件

```text
apply plugin: 'com.android.application'
 
//引入Android App插件.//
 
android {
 
//下面的部分配置Android App相关的信息.//
 
    compileSdkVersion 21
 
//编译的SDK版本.// 
 
    buildToolsVersion "21.1.1"
 
//Build Tools版本，最好选择版本号大于或等于compileSdkVersion的.//
 
    defaultConfig {
 
        applicationId "com.example.jessica.myapplication"
 
//application’s ID. 旧版本是 ‘packageName’.//
 
        minSdkVersion 16
 
//需要的最小API版本.//
 
        targetSdkVersion 21
 
//应用运行的API版本.//
         
        versionCode 1
 
        versionName "1.0"
    }
 
    buildTypes {
        release {
 
//‘BuildTypes’ 控制你App如何编译打包. 如果你想创建你自己的 build variants, 添加到这里.//
 
            minifyEnabled true
 
//是否进行混淆.//
 
            proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
 
//混淆配置.// 
 
        }
    }
}
 
dependencies {
 
//当然Module需要的依赖关系.//
 
   compile fileTree(dir: 'libs', include: ['*.jar'])
 
//依赖编译app/libs.// 
 
   compile 'com.android.support:appcompat-v7:21.0.3'
 
//依赖编译远端库.//
 
}
```

### Other Gradle Files

#### gradle-wrapper.properties (Gradle Version)

wrapper相关配置文件：当其他人编译你的项目时，即使他们没有安装gradle，使用`./gradlew`相关命令时，这个文件会检查正确版本的gradle是否被安装，如有必要会帮你下载正确的版本。下面会详细讲wrapper。

```text
distributionBase=GRADLE_USER_HOME
 
//决定解压后的Gradle包是存储到工程目录下, 还是存储到Gradle user home directory. 
//(对于Unix machines, Gradle user home directory默认在 ~/.gradle/wrapper.)
//如果是PROJECT则zip会解压到该工程目录下
 
distributionPath=wrapper/dists
 
//解压后的Gradle包存放的具体路径.//
 
zipStoreBase=GRADLE_USER_HOME
 
zipStorePath=wrapper/dists

//下载的Gradle zip包所放置的目录//
 
distributionUrl=https\://services.gradle.org/distributions/gradle-2.2.1-all.zip
 
//Gradle的下载地址.//
```

#### settings.gradle

该文件组合你所有想构建的module。

```text
include ':app', ':ActionBar-PullToRefresh'
```

#### gradle.properties (Project Properties)

这个文件包含整个project的配置信息。默认是空文件，你可以添加各种属性值到这个文件，build.gradle中可以引用之。

#### local.properties (SDK Location)

这个文件告诉Android Gradle plugin你的Android SDK安装在哪：

```text
sdk.dir=/Users/jessica/Library/Android/sdk
```

由于这是你本地的SDK路径，所以这个文件不应该加入版本控制中。

## The Gradle Wrapper

Android Studio新建项目时都建议使用Gradle Wrapper来构建，这不是必须的，直接指定你本地的安装的Gradle也行。

用wrapper的好处是当别人参与到该项目时，无须事先本地安装Gradle或者Android Studio，当运行`./gradlew`相关的命令时，wrapper会检查本地相应目录下是否已经有对应版本的Gradle，如果没有会帮你下载正确的版本。还有对于一些持续集成的测试服务器，不一定是安装了Gradle的，这时候使用wrapper就能避免重新配置服务器。

如果你想将Gradle项目转换成基于Gradle Wrapper的，只需要简单的跑一下`gradle wrapper`即可，也可以加上`--gradle-version 2.4`选项来指定一个gradle的版本，`--gradle-distribution-url`选项指定了从哪儿下载gradle，如果没有任何选项，则会从Gradle repository下载你运行`wrapper`task的gradle版本。你也可以将`wrapper`这个task写入你的`build.gradle`文件：

```text
task wrapper(type: Wrapper) {
    gradleVersion = '2.4'
}
```

`wrapper`task会在你的项目根目录下生成如下文件：

```text
sample/
  gradlew
  gradlew.bat
  gradle/wrapper/
    gradle-wrapper.jar
    gradle-wrapper.properties
```
这些文件是都应该加入到版本控制中的。

如果你想更改Gradle版本，可以直接修改`gradle-wrapper.properties`文件：

```text
distributionUrl=https\://services.gradle.org/distributions/gradle-2.4-all.zip
```
或者再跑一下`wrapper`task重新生成这些文件，因为`wrapper`脚本可能会有更新。

使用wrapper的话会忽略你本机已经安装的gradle。

## 关于Build variants

Build variants是product flavors和build types的组合。

```text
buildTypes {
    debug {
        debuggable true
    }
    release {
        minifyEnabled true
        proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
    }
}
productFlavors {
    demo {
        applicationId "me.zircon.test.demo"
        versionName "1.0-demo"
    }
    full {
        applicationId "me.zircon.test.full"
        versionName "1.0-full"
    }
}
```
上面的片段会组合成4种variants：`demoDebug`，`demoRelease`，`fullDebug`，`fullRelease`。

Android Studio正常左下角有一个Build Variants的面板，里面可以选择你最终想编译的版本，当选择run一个模块时，就是编译的这里选择的版本。也可以通过命令行`./gradlew assemble<flavor><buildtype>`来编译，编译出的apk可以在`app/build/outputs/apk`里找到，格式是`app-<flavor>-<buildtype>.apk`。
也有利用productFlavors来打渠道包的例子。可见[这篇文章](http://www.stormzhang.com/devtools/2015/01/15/android-studio-tutorial6/)。

### Source Sets

我们可以在src下建立多个source set：

![sourceSet](/images/postimgs/sourceSet.png)

每一个set代表一个flavor或者build type，main set所有variants都会用到。在编译某个variants时会选择相应set中的src和res。比如`assembleDemoDebug`，会组合main，demo和debug set。对于res和manifest文件，内容会合并，如果有重名字段的，其合并优先级从低到高是：`libraries/dependencies -> main src -> productFlavor -> buildType`。对于src，组合成variants的set中不能存在重名文件，即demo和Debug中不能同时存在一个A.java，但是Debug和Release中可以同时有。
当然我们也可以通过脚本中的sourceSets{ }配置目录结构。
特别对于非约定目录结构的工程（比如Eclipse工程），通过sourceSets{ }加以配置以便Gradle识别：

```text
sourceSets {
        main {
            manifest.srcFile 'AndroidManifest.xml'
            java.srcDirs = ['src']
            aidl.srcDirs = ['src']
            renderscript.srcDirs = ['src']
            res.srcDirs = ['res']
            assets.srcDirs = ['assets']
        }
        demo {
        		 ...
        }
        instrumentTest.setRoot('tests')
}
```

## 关于dependencies

dependencies一般有三种形式：

- Module Dependencies

> compile project(":lib")

- Local Dependencies

> compile fileTree(dir: 'libs', include: ['*.jar'])

- Remote Dependencies

> compile 'com.android.support:appcompat-v7:19.0.1'

对于Remote Dependencies，书写的格式一般是`group:name:version`，[这个网址](http://gradleplease.appspot.com/)可以帮你在[Maven Central](http://search.maven.org/)搜索lib并生成dependencies。

### jcenter 和 Maven Central

Gradle会自动从repositories{...}帮你下载编译Remote Dependencies。

旧版本的Android Studio默认生成的repositories{ }是Maven Central，而新版本已经变成jcenter。这两者都是相互独立的lib仓库，由不同的厂商托管，相较之下，jcenter有以下优势：

- jcenter对开发者是友好的，上传自己的lib到jcenter很方便
- jcenter利用CDN加速，所以下载lib更快
- jcenter类库更全，可以认为是Maven Central的超类

并不是所有类库都host在jcenter或者Maven Central上，有些是host在自己的Maven仓库中，比如我们项目中所使用的twitter的crashlytics库，它host在twitter自己的仓库中：

```text
repositories {
    maven { url 'https://maven.fabric.io/public' }
}
```
关于如何将自己的类库上传到jcenter可以参考[这篇文章](http://inthecheesefactory.com/blog/how-to-upload-library-to-jcenter-maven-central-as-dependency/en)。

## Gradle的一些概念

我们回过头来看看Gradle的一些概念。

Gradle脚本基于Groovy。每一个Gradle脚本执行时都会配置一种类型的对象。比如执行`build.gradle`会建立并配置一个`Project`类型的对象，而`settings.gradle`则配置`Settings`对象。相应的这个类型中的属性和方法可以直接用在脚本中，如：`file(...)`返回路径的File对象；`println name`则打印该project的name。

脚本里可以使用任何Groovy/Java的语法。比如解析AndroidManifest文件取得VersionName：

```text
def manifestVersionName() {
    def manifestFile = file(project.projectDir.absolutePath + '/src/main/AndroidManifest.xml')
    def ns = new groovy.xml.Namespace("http://schemas.android.com/apk/res/android", "android")
    def xml = new XmlParser().parse(manifestFile)
    return xml.attributes()[ns.versionName].toString()
}
```

### Task

Gradle是基于Task的，每一个Task代表了一连串原子性的操作。在Android Studio右边的Gradle面板或者输入命令`./gradlew tasks`都能查看当前project所有的Task。
我们也可以在脚本中新建一个task：

```text
task myTask
task myTask { configure closure }
task myType << { task action }
task myTask(type: SomeType)
task myTask(type: SomeType) { configure closure }
```

### Plugin

添加Plugin到Gradle中其实就是添加了一些新的task，域对象(如SourceSet)，约定(如Java source默认放在src/main/java下)，同时也会扩展一些已经存在的类型。
Plugin分两种：脚本插件`apply from: 'other.gradle'`和二进制插件`apply plugin: 'java'`。

## Refs
1. [https://docs.gradle.org/current/userguide/userguide.html](https://docs.gradle.org/current/userguide/userguide.html)
2. [http://tools.android.com/tech-docs/new-build-system](http://tools.android.com/tech-docs/new-build-system)
3. [http://developer.android.com/sdk/installing/studio-build.html](http://developer.android.com/sdk/installing/studio-build.html)

