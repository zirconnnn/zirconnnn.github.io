---
layout: post
title: "Android插件化框架VirtualAPK资源处理分析"
modified: 2018-05-15 14:00:32 +0800
tags: [android,VirtualAPK,插件化,RePlugin,Small,热修复,hotfix,alibaba,didi]
image:
  feature: abstract-11.jpg
  credit:
  creditlink:
comments: true
share: true
---
## 前言

[VirtualAPK](https://github.com/didi/VirtualAPK)是didi出品的插件化框架，最近读了读源码，简单做些总结，先针对资源的处理这一块，dodola的[这篇文章](https://www.notion.so/VirtualAPK-1fce1a910c424937acde9528d2acd537)对原理阐述的很详尽，建议大家先看一下，我这边对文章中几个点再阐述下以及对gradle插件所做的事做一些总结。所在分支是dev。

## 资源的处理

一、先看下`ResourcesManager.java`的`createResources`方法片段：

```java
	if (Build.VERSION.SDK_INT < Build.VERSION_CODES.LOLLIPOP) {
	    assetManager = AssetManager.class.newInstance();
	    ReflectUtil.invoke(AssetManager.class, assetManager, "addAssetPath", hostContext.getApplicationInfo().sourceDir);
	} else {
	    assetManager = hostResources.getAssets();
	}
	ReflectUtil.invoke(AssetManager.class, assetManager, "addAssetPath", apk);
```
	
这边针对资源的加载做了兼容处理，差异主要是在`AssetManager.cpp`的`addAssetPath`方法，可以比较下[4.4](https://android.googlesource.com/platform/frameworks/base/+/android-4.4.2_r2.0.1/libs/androidfw/AssetManager.cpp#168)和[Android O](https://android.googlesource.com/platform/frameworks/base/+/oreo-release/libs/androidfw/AssetManager.cpp#160)的代码，会发现Android O中会将插件apk的resources.arsc所代表的Asset对象add到现有的索引表ResTable中，而4.4中只是将该apk的path加入到path列表里，所以它这边选择了new一个AssetManager。

文章中还提到了淘宝发布的《深入探索 Android 热修复技术原理》中的一种少兼容少hook的方案，简单理解就是重构native端的AssetManager，添加所有插件的path，接着`AssetManager::getResTable`时去生成新的索引表ResTable，而Java层的Resource对象地址是不变的，也就省去了兼容和hook。

二、Activity 启动过程中对资源的处理

`VAInstrumentation.java`的`newActivity`方法片段：
	
``` java
	Activity activity = mBase.newActivity(plugin.getClassLoader(), targetClassName, intent);
	activity.setIntent(intent);
	try {
		// for 4.1+
		ReflectUtil.setField(ContextThemeWrapper.class, activity, "mResources", plugin.getResources());
	} catch (Exception ignored) {
		// ignored.
	}
```
	
文章中提到此处需要重设`mResources`的原因在[ActivityThrea的performLaunchActivity方法](https://android.googlesource.com/platform/frameworks/base/+/oreo-release/core/java/android/app/ActivityThread.java#2683)（以Android O版本为例）
	
``` java
	private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
		...
		ContextImpl appContext = createBaseContextForActivity(r);
		activity = mInstrumentation.newActivity(cl, component.getClassName(), r.intent);
		...
		int theme = r.activityInfo.getThemeResource();
		if (theme != 0) {
			activity.setTheme(theme);
		}
		...
		if (r.isPersistable()) {
			mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);
		} else {
			mInstrumentation.callActivityOnCreate(activity, r.state);
		}
	}
```
	
> 在 createBaseContextForActivity 方法中创建出来的 ContextImpl appContext 使用的是宿主的Resources，如果不进行处理紧接着Activity会走入onCreate的生命周期中，此时插件加载资源的时候还是使用的宿主的资源，而不是我们特意为插件所创建出来的Resources对象，则会发生找不到资源的问题
	
这里说的有点问题，在走到`onCreate`之前，框架是hook了`Instrumentation.callActivityOnCreate`并且重设了`mResources`等一系列值，其实是可以找到插件的资源的，真正的问题出在上面的几句设置theme的代码，我们知道框架hook了`mInstrumentation.newActivity`此时返回的已经是插件的Activity实例，而此时给这个插件Activity`setTheme`这个theme是哪个Activity的theme，如果不做任何操作，肯定是占位Activity也就是宿主Application中的默认theme，而框架把这个theme已经换掉了，代码在`VAInstrumentation`的`handleMessage`方法中：
	
``` java
	public boolean handleMessage(Message msg) {
		if (msg.what == LAUNCH_ACTIVITY) {
			// ActivityClientRecord r
			Object r = msg.obj;
			try {
				Intent intent = (Intent) ReflectUtil.getField(r.getClass(), r, "intent");
				intent.setExtrasClassLoader(VAInstrumentation.class.getClassLoader());
				ActivityInfo activityInfo = (ActivityInfo) ReflectUtil.getField(r.getClass(), r, "activityInfo");
				if (PluginUtil.isIntentFromPlugin(intent)) {
					int theme = PluginUtil.getTheme(mPluginManager.getHostContext(), intent);
					if (theme != 0) {
						Log.i(TAG, "resolve theme, current theme:" + activityInfo.theme + "  after :0x" + Integer.toHexString(theme));
						activityInfo.theme = theme;
					}
				}
			} catch (Exception e) {
				e.printStackTrace();
			}
		}
		return false;
	}
```
	
所以如果此时上面的`newActivity`hook中不去重设`mResources`，`setTheme`方法就会报错。
	
文章中还提到：

> 另：如果采用上述所说的AssetManager销毁的方法，则无需在创建Activity后设置Resources对象，因为此处全局都是宿主+插件的资源。
	
这里简单解释下，虽然每次new一个新的Activity的时候都会产生一个新的ContextImpl，但是这个ContextImpl去生成Resource实例的时候首先会有个取缓存的过程，这个缓存的key一般情况下都是相同的，所以底层也就共用一个AssetManager。可以跟踪下`ResourcesManager.java`的`createBaseActivityResources`方法看下。

## gradle插件中对资源的处理

一、VAHostPlugin

宿主的这个plugin是为插件plugin服务的，作用是生成一些文件。与资源处理相关的是备份了宿主的R.txt和生成宿主的依赖列表versions.txt，放在`宿主module/build/VAHost`下
	
二、VAPlugin

插件的这个plugin针对资源处理相关的操作主要是hook了app编译过程中几个Task，如`Merge[Variant]Assets`，`Process[Variant]Resources`等以达到剔除与宿主共享资源，重设插件中资源id，裁剪resources.arsc等目的，下面我们具体看下这些hook。
	
- PrepareDependenciesHooker

hook了`pre[Variant]Build`task，主要目的是过滤插件依赖(aar or jar)，将其与宿主共享的依赖和其独有的依赖分开放到若干集合中，用到了上面说到的宿主依赖列表versions.txt。
	
- MergeAssetsHooker

hook了`Merge[Variant]Assets`task，这个task是用于merge插件依赖的各个aar中的assets，然后会交由AAPT处理，上一步过滤出了插件与宿主共享的aar，那么这些aar中的assets是不需要插件AAPT再处理了，所以需要此hook剔除这些assets。
	
- ProcessResourcesHooker

hook了`Process[Variant]Resources`task，这个task最终是调用AAPT去编译资源生成R文件和`resources-[Variant].ap_`，这个.ap_文件位置在`build/intermediates/res`下，可以解压出来包含了`resources.arsc`，编译过的`AndroidManifest.xml`和资源。这个hook的主要作用就是去修改.ap_文件和R文件。
	
我们看下`ProcessResourcesHooker.groovy`中的`repackage`方法里的一些关键点：
	
``` groovy
	void repackage(ProcessAndroidResources par, File apFile) {
		...
		resourceCollector = new ResourceCollector(project, par)
		resourceCollector.collect() // ①
		...
		def aapt = new Aapt(resourcesDir, rSymbolFile, androidConfig.buildToolsRevision)
		//Delete host resources, must do it before filterPackage
		aapt.filterResources(retainedTypes, filteredResources)  // ②
		//Modify the arsc file, and replace ids of related xml files
		aapt.filterPackage(retainedTypes, retainedStylealbes, virtualApk.packageId, resIdMap, libRefTable, updatedResources) // ③
		...
		/*
		 * Delete filtered entries and then add updated resources into resources-${variant.name}.ap_
		 */
		com.didi.virtualapk.utils.ZipUtil.with(apFile).deleteAll(filteredResources + updatedResources) // ④
		project.exec {
		    executable par.buildTools.getPath(BuildToolInfo.PathId.AAPT)
		    workingDir resourcesDir
		    args 'add', apFile.path
		    args updatedResources
		    standardOutput = System.out
		    errorOutput = System.err
		} // ⑤
		updateRJava(aapt, par.sourceOutputDir) // ⑥
		...
	}
```
	
①
解析宿主和插件的`R.txt`按照ResType收集资源，并从收集的插件资源集合中过滤出与宿主同名的资源(资源类型和名称相同，参见`ResourceEntry.groovy`的`equals()`)。补充下，`R.txt`也是AAPT的产物，除了自身的资源也包括了其依赖的aar中的资源，输出位置一般在`build/intermediates/symbols`下。接着重设资源ID，与宿主共享部分的资源，其ID需要设置成宿主中的值；插件独有的资源需要重新设置，0XPPTTEEEE，PP段设置成我们在插件gradle文件中设置的值，TT和EEEE段按顺序重新赋值。这一步主要生成几个集合供下面的几步使用。
	
②
遍历从`.ap_`文件中解压出来的所有资源，将与宿主共享的资源全部删除。
	
③
开始修改与裁剪`resources.arsc`各个chunk段的值，只留下插件独有的资源索引。关于arsc的格式，可以去参照[老罗的博客](https://blog.csdn.net/luoshengyang/article/details/8744683)，下面图片来自`ArscEditor.groovy`中：
	
![arsc](/images/postimgs/arsc.jpg)

接着将从`.ap_`文件中解压的已经编译过的xml文件中的引用的资源ID替换成上面新生成的ID值。编译的xml文件格式也可以参考[老罗的博客](https://blog.csdn.net/luoshengyang/article/details/8744683)，所有修改的文件(包括arsc，各种xml)都会放到一个集合中。下面便是打包新的`.ap_`文件。
	
④
删除`.ap_`文件中所有与宿主共享的和所有修改过的资源的原始版本
	
⑤
通过`aapt add`命令将所有修改的资源重新打包到`.ap_`文件，`.ap_`文件的修改全部结束。
	
⑥
下面便是更新插件的R.java包括其依赖的aar的R.java，将其中的所有ID值更新，这样便能正确索引arsc文件了。这一步还做了一个操作，将插件独有的资源写到一个单独的R.java中，作用会在下面说。到此为止，AAPT的产物都已经修改裁剪完毕，通过重设资源ID的PP段以达到避免ID冲突的问题，这种方式相比修改AAPT的源码兼容性相对好些，VirtualAPK的这种方式参照了[Small的实现](https://github.com/wequick/Small)。
	
- DxTaskHooker

hook了dex这个Transform，作用是在dex操作之前去覆盖插件app的R.class(不包括其依赖的aar的R.class)，只留下插件独有的资源ID，能这么操作是因为插件app编译后的class文件中对R中资源ID的引用都已经编译成对一个常量的引用，如下图（而aar中则是编译成对一个变量的引用，因为aar中ID值在编译时会和主module融合导致其值变化）：
	
![app_R](/images/postimgs/app_R.jpg)

而为了兼容宿主反射调用插件中的独有资源，则要留下插件独有的资源ID。这个hooker我觉得是可以去掉的，不去覆盖R.class也不会有什么影响。

## TL;DR

大体VirtualAPK针对资源的处理都梳理了，简单总结下，编译阶段，针对插件apk重设资源ID，运行阶段，宿主加载插件，将插件和宿主的apk添加到同一个AssetManager中，接着构建一个新的Resources去替换原来的Resources，这样通过Context去取资源的时候就能在一个完整的resources.arsc里索引。


	


