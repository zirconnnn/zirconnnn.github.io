---
layout: post
title: "Retrofit 2.0的一些用法"
modified: 2016-01-25 17:00:32 +0800
tags: [android,retrofit,2.0,network,library,square,volley,上传,upload,下载,download]
image:
  feature: abstract-6.jpg
  credit:
  creditlink:
comments: true
share: true
---
### 前言
最近在项目中使用了Square家的[Retrofit](http://square.github.io/retrofit/)网络库，主要是为了配合[RxJava](https://github.com/ReactiveX/RxJava)，使用下来感觉还不错，这里稍微记录下。

我使用时最新版本是`compile 'com.squareup.retrofit2:retrofit:2.0.0-beta2'`，按照`Jake Wharton`的说法虽然还是beta版但是接口已经相对稳定了，所以我们可以在项目中依赖它。由于我并没有在项目中使用过1.x版本，所以不会过多的与1.x版本比较。至于与其他网络库的对比，可自行Google，还是有很多文章的，比如stackoverflow上的[这篇](http://stackoverflow.com/questions/16902716/comparison-of-android-networking-libraries-okhttp-retrofit-volley#)。

### 初始化Retrofit实例
{% highlight java %}
public static String BASE_URL = "http://example.com/";

OkHttpClient okHttpClient = new OkHttpClient();
HttpLoggingInterceptor loggingInterceptor = new HttpLoggingInterceptor();
loggingInterceptor.setLevel(HttpLoggingInterceptor.Level.BODY);
okHttpClient.interceptors().add(loggingInterceptor);

Retrofit retrofit = new Retrofit.Builder().baseUrl(BASE_URL).client(okHttpClient)
                .addConverterFactory(FastJsonConvertFactory.create())
                .addCallAdapterFactory(RxJavaCallAdapterFactory.create())
                .build();
{% endhighlight %}

Retrofit 2.0强制依赖[okhttp](https://github.com/square/okhttp)，且不可替换。我们可以通过[okhttp Interceptors](https://github.com/square/okhttp/wiki/Interceptors)对request和response进行一些监视或者处理，比如打印request和response，加入公用的http headers等。`Interceptors`是链式的，我们可以串接多个。

`Converter`会对request body和response body进行序列化和反序列化，它也可以添加多个，按照添加顺序，如果前面的`Converter`不能处理该种类型，则传递到下一个。一般我们服务端要求的请求和返回数据都是`json`格式，所以我们要添加一个`JsonConverter`进行对象的序列化和反序列化，Retrofit默认有几个写好的`Converter`，请参照官网。由于我在项目中采用[fastjson](https://github.com/alibaba/fastjson)解析`json`，而且需要对数据进行`base64`编码，所以需要自己写一个`Converter`。注意：`json`相关的`Converter`要放在最后，因为无法通过什么确切条件来判断一个对象是否是json对象，所以它会让所有类型的对象都通过。

`CallAdapter`原理跟`Converter`较像，默认一个请求方法会返回`Call`对象(可以参照源码中的`DefaultCallAdapter`)。我们也可以返回其他对象，比如返回`RxJava`的`Observable`对象，这时就需要添加一个自定义的`CallAdapter`，而Retrofit提供了对`RxJava`的支持。

### 定义请求接口
Retrofit使用特殊的注解来映射请求参数及请求方法等。

{% highlight java %}
public interface Api {
	@GET
	Observable<QueryInfoResponse> queryInfo(@Url String queryInfoRequest);
	
	@GET("group/{id}/users")
  	Call<List<User>> groupList(@Path("id") int groupId, @Query("sort") String sort);
	
	@POST("login")
  	Observable<LoginResponse> login(@Body LoginRequest loginRequest);
}
{% endhighlight %}

关于Url的拼接，盗张图说明下吧：

![url](/images/postimgs/urljoint.png)

所以建议是：

- Base URL：总是以/结尾
- @GET/POST等：不要以/开头

返回类型决定了使用哪一个`CallAdapter`。

### 请求
{% highlight java %}
Api api = retrofit.create(Api.class);
api.queryInfo("...");
{% endhighlight %}

### 下载
下载时ResponseBody返回的是一个流，相应接口如下：
{% highlight java %}
public interface DownloadApi {
	@GET
	Observable<ResponseBody> downloadFile(@Url String downloadRequest);
}

downloadApi.downloadFile(url).map(new Func1<ResponseBody, File>() {
  @Override
  public File call(ResponseBody responseBody) {
      FileOutputStream outputStream = null;
      try {
          byte[] bytes = responseBody.bytes();
          File file = new File(context.getExternalFilesDir(null), "fileName");
          outputStream = new FileOutputStream(file);
          outputStream.write(bytes);
          return file;
      } catch (IOException e) {
          e.printStackTrace();
      } finally {
          if (outputStream != null) {
              try {
                  outputStream.close();
              } catch (IOException e) {
                  e.printStackTrace();
              }
          }
      }
      return null;
  }
}).subscribeOn(Schedulers.io()).observeOn(AndroidSchedulers.mainThread());
{% endhighlight %}

### 上传文件
上传文件时是`multipart`请求，相应接口如下：
{% highlight java %}
public interface UploadApi {
	@Multipart
  	@POST(Constants.ADD)
  	Observable<BlankDataResponse> add(@PartMap Map<String, RequestBody> 	params);
}
{% endhighlight %}

`@PartMap`代表由`boundary`分隔的每一部分，对于`text/plain`类型的part，可用下面的方法生成`Part`：

{% highlight java %}
Map<String, RequestBody> params = new HashMap<>();
RequestBody requestBody = RequestBody.create(MediaType.parse("text/plain"), "token");
params.put("token", requestBody);
{% endhighlight %}

对于图片类型则可以如下：

{% highlight java %}
File file = new File(path);
RequestBody requestBody = RequestBody.create(MediaType.parse("image/jpeg"), file);
params.put("AttachmentKey\"; filename=\"" + image.getFileName(), requestBody);
{% endhighlight %}

上面的代码对应于Http RequesBody内容如下：

{% highlight text %}
--88fc3b38-77d8-4ec3-b057-35600d14f0b3
Content-Disposition: form-data; name="AttachmentKey"; filename="example.jpg"
Content-Transfer-Encoding: binary
Content-Type: image/jpeg
Content-Length: 169913
...
{% endhighlight %}

### 其他
- 如果你想在Http Headers里加入公用的Header，可以如下做：

{% highlight java %}
// 自定义okhttp的Interceptor
public class RequestInterceptor implements Interceptor {
    @Override
    public Response intercept(Chain chain) throws IOException {
        Request request = chain.request().newBuilder().addHeader("version", 				"1.0.0").build();
        return chain.proceed(request);
    }
}

// 加入链中
RequestInterceptor requestInterceptor = new RequestInterceptor();
okHttpClient.interceptors().add(requestInterceptor);
{% endhighlight %}






