---
layout: post
title: "Android DataBinding实践"
modified: 2016-01-27 17:00:32 +0800
tags: [android,databinding,MVVM,MVP]
image:
  feature: abstract-5.jpg
  credit:
  creditlink:
comments: true
share: true
---
## 前言
最近在新项目中采用了`MVVM + DataBinding + RxJava`的架构，后面会把一些知识点都记录下来，这篇先来说说[DataBinding](https://developer.android.com/intl/zh-cn/tools/data-binding/guide.html)。

之前使用`MVP`模式时，`View`和`Presenter`之间总要定义一个介面`IView`，`Presenter`必须通过这个`IView`来操作`View`。但是现在使用了`MVVM + DataBinding`，`ViewModel`通过`DataBinding`机制将`Model`塞给`View`，无需这个`IView`了，而且再也不需要`findViewById()`或者[ButterKnife](http://jakewharton.github.io/butterknife/)这样的注入框架了，使用下来确实少写很多代码。

DataBinding允许你直接在XML布局中绑定对象，并且可以监听对象的变化以便及时更新UI，但是现在只能单向绑定，比如你给`EditText`绑定了一个`String`，该`String`的变化会及时显示在`EditText`，但是你在`EditText`中输入的内容不会直接赋值到`String`中。现在IDE对DataBinding支持一般，没有代码提示，毕竟推出不久，估计`Android Studio 2.0`之后会好些。

## 示例
我用V2EX的[API](http://v2ex.com/p/7v9TEc53)写了个简单的[项目](https://github.com/zirconnnn/V2EXTest)用来示例`DataBinding`的一些用法，采用了`MVVM + DataBinding + RxJava`。

### 集成
在项目中集成DataBinding很容易，只要你的Android Studio版本>=1.3以及Android的Gradle插件版本>=1.5，然后在`build.gradle`中加入如下配置即可：

{% highlight groovy %}
android {
    ....
    dataBinding {
        enabled = true
        // version = 1.1 // 可选，建议最好不加，使用Gradle插件默认的版本
    }
}
{% endhighlight %}

gradle插件会帮你下载DataBinding相关的依赖包，1.5版本的gradle插件默认集成的是`1.0-rc5`版本的DataBinding，`version`选项经实验最好不加，使用当前Gradle插件默认的DataBinding版本即可，否则会有些莫名奇妙的编译问题，比如我遇到[这个问题](https://code.google.com/p/android/issues/detail?id=195178&q=databinding%20StringIndexOutOfBoundsException&colspec=ID%20Status%20Priority%20Owner%20Summary%20Stars%20Reporter%20Opened)。可以从[jcenter](https://bintray.com/android/android-tools/com.android.databinding.compilerCommon/view)上查看到DataBinding的最新版本。

## 一些用法

### Layout相关

- 除了`java.lang.*`包下的类，其他的类要么使用全名，要么需要`import`：

{% highlight xml %}
<import type="android.view.View"/>
{% endhighlight %}

- 可以在xml中使用静态方法：

{% highlight xml %}
android:text="@{Html.fromHtml(mainItem.content_rendered), default=无内容}"
{% endhighlight %}

- 自定义Binding类名字，默认根据xml的名字生成

{% highlight xml %}
<data class="MainListBinding">
    ...
</data>
{% endhighlight %}

- Include

如果xml中包含`include`标签，可以如下方式传递：

{% highlight xml %}
// activity_main.xml
<include
	id="@+id/empty_layout"
 	layout="@layout/empty_content"
 	app:isShowContent="@{viewModel.isShowContent}" />
 	
// empty_content.xml
<data>
   <variable
       name="isShowContent"
       type="boolean"/>
</data>
{% endhighlight %}

如果你需要引用被`include`布局中`view`元素，`include`标签的`id`属性不可少。

- xml中带`id`属性的views都会在binding类中生成一个`public final`类型的引用，引用名即`id`属性值：

{% highlight xml %}
<TextView
	android:id="@+id/tv_username"
	android:layout_width="wrap_content"
	android:layout_height="wrap_content"/>
	
// 生成 public final android.widget.TextView tvUsername;
{% endhighlight %}

这样我们代码中就不需要`findViewById()`了，直接`binding.tvUsername`即可引用。

- 空指针问题

{% highlight xml %}
android:text="@{mainItem.title, default=无标题}"/>
{% endhighlight %}

如果`mainItem`为空，不会报错，text的值是`mainItem.title`的默认值`null`。

### Observable
要想让对象的变化及时通知到view，则该对象必须实现[Observable](https://developer.android.com/intl/zh-cn/reference/android/databinding/Observable.html)接口或者是一个[ObservableFields](https://developer.android.com/reference/android/databinding/ObservableField.html)。

### Adapter中的绑定
因为`RecyclerView`或者`ListView`等每个item的布局不一定相同，可能会产生多个Binding类，这种情况可以使用Binding的基类`ViewDataBinding`，然后在bind data到每个view时，如果data相同，则可使用`setVariable()`，如果data不同，可以根据每个view type的不同，强制转换`ViewDataBinding`至具体的Binding类，再调用其`setXxx`方法。可参见示例。

### 绑定变量的变化会自动调用属性的setter方法
比如`TextView`的`android:text`其实是去调用了其`setText(String)`方法，注意参数类型的匹配，因为可能有多个重载方法。而且哪怕一个控件并没有相应的属性，但是有setter方法，我们仍然可以这么写，比如示例中：

{% highlight xml %}
<android.support.v4.widget.SwipeRefreshLayout
  android:id="@+id/refresh_layout"
  android:layout_width="match_parent"
  android:layout_height="match_parent"
  app:refreshing="@{viewModel.isRefresh}"/>
{% endhighlight %}

`SwipeRefreshLayout`控件并没有`refreshing`这个属性，但是却有`setRefresh(boolean)`方法，用来控制刷新状态。可以点击菜单上的刷新来测验这个写法。

### @BindingAdapter
因为并不是所有属性都有setter方法，这时候就需要用这个注解来自定义属性的逻辑。比如示例中：

{% highlight xml %}
<android.support.v7.widget.RecyclerView
	android:id="@+id/list_hot"
	android:layout_width="match_parent"
	android:layout_height="match_parent"
	app:contentList="@{viewModel.contentList}"/>
{% endhighlight %}

`RecyclerView`并没有`contentList`这个属性而且没有`setContentList()`方法，所以需要我们自定义：

{% highlight java %}
@BindingAdapter("bind:contentList")
public static void setContentList(RecyclerView view, List<Topic> list) {
   MainAdapter adapter = (MainAdapter) view.getAdapter();
   adapter.addRows(list);
}
{% endhighlight %}

这样每次刷新时只要将返回数据set给`viewModel.contentList`就会调用上面的方法刷新adapter了。官网文档还有更详尽的`@BindingAdapter`例子。

## 总结
详细的文档还是直接参考官网吧，有的写法比如`@BindingConversion`，我的示例中并未用到，后面想到使用场景时再用。我发这篇博客时，最新版本已经到2.0.0-alpha9了，想必会和Android studio2.0一起出正式版，DataBinding能方便的实现`MVVM`模型，又有官方支持，应该会慢慢流行起来，所以建议可以在项目中尝试了。











