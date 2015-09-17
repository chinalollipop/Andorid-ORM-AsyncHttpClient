# Andorid-ORM-AsyncHttpClient
android-async-http的学习和使用

Andorid ORM之AsyncHttpClient技术分享





时间：2015年9月17日


目录
Andorid ORM之AsyncHttpClient技术分享	1
前沿	3
1、简介	4
2、特性	4
3、使用方法	4
4、建议使用静态的Http Client对象	4
5、AsyncHttpClient,RequestParams,AsyncHttpResponseHandler简单范例	5
        (1)AsyncHttpClient 		7
(2)RequestParams 		7
(3)AsyncHttpResponseHandler	8
6、利用PersistentCookieStore持久化存储cookie	11
7、利用RequestParams上传文件		12
8、用BinaryHttpResponseHandler下载二进制数据	13
9、核心类简介	13
10、请求流程UML简介	15
11、官方推荐网络请求和取消的原则和使用概要	20
网络请求	20
网络取消	23
总结	26
1、优点	26
2、缺点	29







前沿
写这篇文章的初衷和目的是为了了解android-async-http这个网络框架，根据项目，挑选合适的网络框架来编写高质量的代码，并且会提高代码的可读性和可维护性。前期自己编写的网络框架模块多多少少在并发性能上和网络取消这一块考虑不够，考虑不太全面和细腻，当然也是学习为主了，因为以前对android-async-http了解不够透彻，只了解有这么一个网络框架，具体没有深究过。再加上前段时间面试比较多，看到很多面试者，在不同程度上都说到使用到一些网络框架，如Volley，Afinal，XUtils以及android-async-http  等等。今天在这里总结一下，也方便自己学习和以后具体使用。
Volley框架这里就暂且不说了，因为我们目前项目中使用Volley比较多，并且是谷歌自己开源的项目，有很多优点就不谈了，这里主要说一下android-async-http的使用以及源码深度解析。
前几天抽空看了一下xUtils框架，然后编写了一个文档，内部分享了，今天在这里对android-async-http进行深度解析，做一篇可读性高的技术文档，也方便自己以后学习、回顾和巩固。
其实XUtils和android-async-http 的原理都是一样，都是使用Apache HTTP Client，然后使用线程池来进行异步、同步网络请求和缓存机制，都是在线程池里面执行，然后发消息通知给UI main线程进行刷新View。不同的是，xUtils直接调用Apache的类使用，而android-async-http 是对他进行重新的封装和优化，优点就是android-async-http 他目前一直在更新和维护，xUtils从发布至今，再也没有看到新的版本和更新了。其实你可以理解android-async-http 是xUtils对httpUtils网络的升级版
android-async-http和xUtils httpUtils部分的比较，有比较，才有针对性进行详细的介绍，为什么是他的升级版呢？
首先，我们来查看源码可以分析得到,他们都使用线程池ExecutorService 来控制网络请求。android-async-http 是默认使用缓存线程池Executors.newCachedThreadPool()，当然你也可以设定线程池的方式，此处扩展性比较好，而xUtils 使用Executors.newFixedThreadPool() new一个固定线程池，如果没有可执行线程，依然保持60秒的唤醒等待，之后关闭。至于这两个线程池之间的具体区别和优缺点可以找谷姐。她的解析更加全面，这里不再阐述了。
其次xUtils httpUtils是对AsyncTask<...>一次重新封装和回调，其实开发过安卓应用的人都知道，AsyncTask有很多弊处，比如每次发送请求都需要new一个AsyncTask，还有取消这一块也是硬伤,不知道啥时候是具体取消这次请求的，有点GC[java垃圾回收机制]的感觉。暂时不能做到单例模式，安卓官方文档也不推荐使用了。如果线程池里面监听到有消息，它主要是通过Handler发消息和Looper.getMainLooper()，在UI线程中去显示，然而android-async-http 直接使用线程池+Looper或者线程池+MainLooper对消息进行处理和回调，主要看你请求的方式，最后对应到在子线程数据保存和UI mian线程中去执行刷新View。
不同之处是在于android-async-http多了一次选择，是可以在子线程中去处理这些消息，而xUtils httpUtils是不可以的。如果你使用的是默认UI线程去发送网络请求，你直接就可以在UI线程刷新界面，这点跟HttpUtils类似；但是，如果你是通过HandlerThread来初始化网络请求，你可以在子线程去监听消息，比如监听到有数据之后插入数据库或者缓存到本地[APP缓存]，如果还需要刷新UI的话，需要发消息给Handler来通知UI main线程更新View。这是他的一点区别，但是他们还是有一些弊端，后面有详细的阐述。
1.简介
Android中网络请求一般使用Apache HTTP Client或者采用HttpURLConnect，但是直接使用这两个类库需要写大量的代码才能完成网络post和get请求，而使用android-async-http这个库可以大大的简化操作，它是基于Apache’s HttpClient ，所有的请求都是独立在UI主线程之外，通常是请求放在线程池里面，然后请求有结果之后返回到主线程去回调，通过回调方法处理请求结果，采用android  Handler message 机制传递信息。
2.特性
	(1)采用异步http请求，并通过匿名内部类处理回调结果
	(2)http请求独立在UI主线程之外
	(3)采用线程池来处理并发请求
	(4)采用RequestParams类创建GET/POST参数
	(5)不需要第三方包即可支持Multipart file文件上传
	(6)文件很小，只有104kb
	(7)自动为各种移动电话处理连接断开时请求重连[重定向]
	(8)超快的自动gzip响应解码支持
	(9)使用BinaryHttpResponseHandler类下载二进制文件(如图片)
	(10) 使用JsonHttpResponseHandler类可以自动将响应结果解析为json格式
	(11)持久化cookie存储，可以将cookie保存到你的应用程序的SharedPreferences中

3.使用方法
	(1)到官网http://loopj.com/android-async-http/下载最新的android-async-http-1.4.8.jar，
如果你使用的Eclipse。请将此jar包添加进Android应用程序 libs文件夹,，
如果你使用的是Android Studio，请直接从Git上的网址https://github.com/AsyncHttpClient/async-http-client上，下载下来，做为一个Library库文件导入到Android studio中,
当然如果你习惯使用Android Studio并使用现成的jar文件导入Android Studio即可，网上也有现成的Android studio导入库文件方法。
	(2)通过import com.loopj.android.http.*;引入相关类,当然你也可以跟着需要导入需要的class文件。
	(3)创建异步请求
[java] 简单的请求过程
1.AsyncHttpClient client = new AsyncHttpClient();  
2.client.get("http://www.google.com", new AsyncHttpResponseHandler() {  
3.    @Override  
4.    public void onSuccess(String response) {  
5.        System.out.println(response);  
6.    }  
7.});  
4.建议使用静态的Http Client对象
	在下面这个例子，我们创建了静态的http client对象，原因是声明为Static的方法和函数变量先初始化，并且不会重复初始化，使其很容易连接到Twitter的API
[java] 
1.import com.loopj.android.http.*;  
2.  
3.public class TwitterRestClient {  
4.  private static final String BASE_URL = "http://api.twitter.com/1/";  
5.  
6.  private static AsyncHttpClient client = new AsyncHttpClient();  
7.  
8.  public static void get(String url, RequestParams params, AsyncHttpResponseHandler responseHandler) {  
9.      client.get(getAbsoluteUrl(url), params, responseHandler);  
10.  }  
11.  
12.  public static void post(String url, RequestParams params, AsyncHttpResponseHandler responseHandler) {  
13.      client.post(getAbsoluteUrl(url), params, responseHandler);  
14.  }  
15.  
16.  private static String getAbsoluteUrl(String relativeUrl) {  
17.      return BASE_URL + relativeUrl;  
18.  }  
19.}  
然后我们可以很容易的在代码中操作Twitter的API
[java] 
1.import org.json.*;  
2.import com.loopj.android.http.*;  
3.  
4.class TwitterRestClientUsage {  
5.    public void getPublicTimeline() throws JSONException {  
6.        TwitterRestClient.get("statuses/public_timeline.json", null, new JsonHttpResponseHandler() {  
7.            @Override  
8.            public void onSuccess(JSONArray timeline) {  
9.                // Pull out the first event on the public timeline   
10.                JSONObject firstEvent = timeline.get(0);  
11.                String tweetText = firstEvent.getString("text");  
12.  
13.                // Do something with the response   
14.                System.out.println(tweetText);  
15.            }  
16.        });  
17.    }  
18.}  

5.AsyncHttpClient,RequestParams,AsyncHttpResponseHandler简单范例
	(1)AsyncHttpClient
	public class AsyncHttpClient extends java.lang.Object
 该类通常用在android应用程序中创建异步GET, POST, PUT,PATCH和DELETE HTTP请求，请求参数通过RequestParams实例创建，响应通过重写匿名内部类 ResponseHandlerInterface的方法处理。
例子：
[java] 
1.AsyncHttpClient client = new AsyncHttpClient();  
2. client.get("http://www.google.com", new ResponseHandlerInterface() {  
3.     @Override  
4.     public void onSuccess(String response) {  
5.         System.out.println(response);  
6.     }  
7. });    
(2)RequestParams
	public class RequestParams extends java.lang.Object  implements Serializable
用于创建AsyncHttpClient实例中的请求参数(包括字符串或者文件)的集合
例子：
[java] 
1.RequestParams params = new RequestParams();  
2. params.put("username", "james");  
3. params.put("password", "123456");  
4. params.put("email", "my@email.com");  
5. params.put("profile_picture", new File("pic.jpg")); // Upload a File   
6. params.put("profile_picture2", someInputStream); // Upload an InputStream   
7. params.put("profile_picture3", new ByteArrayInputStream(someBytes)); // Upload some bytes   
8.  
9. Map<String, String> map = new HashMap<String, String>();  
10. map.put("first_name", "James");  
11. map.put("last_name", "Smith");  
12. params.put("user", map); // url params: "user[first_name]=James&user[last_name]=Smith"   
13.  
14. Set<String> set = new HashSet<String>(); // unordered collection   
15. set.add("music");  
16. set.add("art");  
17. params.put("like", set); // url params: "like=music&like=art"   
18.  
19. List<String> list = new ArrayList<String>(); // Ordered collection   
20. list.add("Java");  
21. list.add("C");  
22. params.put("languages", list); // url params: "languages[]=Java&languages[]=C"   
23.  
24. String[] colors = { "blue", "yellow" }; // Ordered collection   
25. params.put("colors", colors); // url params: "colors[]=blue&colors[]=yellow"   
26.  
27. List<Map<String, String>> listOfMaps = new ArrayList<Map<String, String>>();  
28. Map<String, String> user1 = new HashMap<String, String>();  
29. user1.put("age", "30");  
30. user1.put("gender", "male");  
31. Map<String, String> user2 = new HashMap<String, String>();  
32. user2.put("age", "25");  
33. user2.put("gender", "female");  
34. listOfMaps.add(user1);  
35. listOfMaps.add(user2);  
36. params.put("users", listOfMaps); // url params: "users[][age]=30&users[][gender]=male&users[][age]=25&users[][gender]=female"   
37.  
38. AsyncHttpClient client = new AsyncHttpClient();  
39. client.post("http://myendpoint.com", params, responseHandler);  
(3)AsyncHttpResponseHandler
public class AsyncHttpResponseHandler extends java.lang.Object implements ResponseHandlerInterface
用于拦截和处理由AsyncHttpClient创建的请求。在匿名类AsyncHttpResponseHandler中的重写 onSuccess(int, org.apache.http.Header[], byte[])方法用于处理响应成功的请求。此外，你也可以重写 onFailure(int, org.apache.http.Header[], byte[], Throwable), onStart(), onFinish(), onRetry() 和onProgress(int, int)等方法
	1、字节流传递例子：
[java] 
1.AsyncHttpClient client = new AsyncHttpClient();  
2. client.get("http://www.google.com", new AsyncHttpResponseHandler() {  
3.     @Override  
4.     public void onStart() {  
5.         // Initiated the request   
6.     }  
7.  
8.     @Override  
9.     public void onSuccess(int statusCode, Header[] headers, byte[] responseBody) {  
10.         // Successfully got a response   
11.     }  
12.  
13.     @Override  
14.     public void onFailure(int statusCode, Header[] headers, byte[] responseBody, Throwable error)  
15. {  
16.         // Response failed :(   
17.     }  
18.  
19.     @Override  
20.     public void onRetry() {  
21.         // Request was retried   
22.     }  
23.  
24.     @Override  
25.     public void onProgress(int bytesWritten, int totalSize) {  
26.         // Progress notification   
27.     }  
28.  
29.     @Override  
30.     public void onFinish() {  
31.         // Completed the request (either success or failure)   
32.     }  
33. });  
 
上面的例子是返回的response直接是原生字节流的情况，如果你需要把返回的结果当一个String对待，这时只需要匿名实现一个TextHttpResponseHandler就行，其继承自AsyncHttpResponse，并将原生的字节流根据指定的encoding转化成了string对象，
2、字符串传递代码如下：
AsyncHttpClient client = new AsyncHttpClient();RequestParams params = new RequestParams();params.put("key", "value");params.put("more", "data");client.get("http://www.google.com", params, new    TextHttpResponseHandler() {        @Override        public void onSuccess(int statusCode, Header[] headers, String response） {            System.out.println(response);        }        @Override        public void onFailure(int statusCode, Header[] headers, String responseBody, Throwable error） {            Log.d("ERROR", error);        }        });

同样的方式，你可以发送json请求，
3、Json传递例子如下：

String url = "https://ajax.googleapis.com/ajax/services/search/images";AsyncHttpClient client = new AsyncHttpClient();RequestParams params = new RequestParams();params.put("q", "android");params.put("rsz", "8");client.get(url, params, new JsonHttpResponseHandler() {                @Override    public void onSuccess(int statusCode, Header[] headers, JSONObject response) {       // Handle resulting parsed JSON response here    }    @Override    public void onSuccess(int statusCode, Header[] headers, JSONArray response) {      // Handle resulting parsed JSON response here    }});

看到了没，返回的response已经自动转化成JSONObject了，当然也支持JSONArray类型，override你需要的那个版本就行。
　　有了AsyncHttpClient，要实现这些功能是不是很简单呢？当然这里只是很初级的介绍和使用，剩下的还需要开发者自己参考官方文档、源码（官方甚至提供了一个Sample使用的集合），在实际项目中实践。
 
6.利用PersistentCookieStore持久化存储cookie
PersistentCookieStore类用于实现Apache HttpClient的CookieStore接口，可以自动的将cookie保存到Android设备的SharedPreferences中，如果你打算使用cookie来管理验证会话，这个非常有用，因为用户可以保持登录状态，不管关闭还是重新打开你的app
(1)首先创建 AsyncHttpClient实例对象
[java] 
1.AsyncHttpClient myClient = new AsyncHttpClient(); 
(2)将客户端的cookie保存到PersistentCookieStore实例对象,带有activity或者应用程序context的构造方法
[java] 
1.PersistentCookieStore myCookieStore = new PersistentCookieStore(this);  
2.myClient.setCookieStore(myCookieStore);  
(3)任何从服务器端获取的cookie都会持久化存储到myCookieStore中，添加一个cookie到存储中，只需要构造一个新的cookie对象，并且调用addCookie方法
[java] 
1.BasicClientCookie newCookie = new BasicClientCookie("cookiesare", "awesome");  
2.newCookie.setVersion(1);  
3.newCookie.setDomain("mydomain.com");  
4.newCookie.setPath("/");  
5.myCookieStore.addCookie(newCookie);   

7.利用RequestParams上传文件
	RequestParams类支持multipart file 文件上传
	(1)在RequestParams 对象中添加InputStream用于上传
[java] 
1.InputStream myInputStream = blah;  
2.RequestParams params = new RequestParams();  
3.params.put("secret_passwords", myInputStream, "passwords.txt");  
(2)添加文件对象用于上传
[java] 
1.File myFile = new File("/path/to/file.png");  
2.RequestParams params = new RequestParams();  
3.try {  
4.    params.put("profile_picture", myFile);  
5.} catch(FileNotFoundException e) {}  
(3)添加字节数组用于上传
[java] view plaincopy
1.byte[] myByteArray = blah;  
2.RequestParams params = new RequestParams();  
3.params.put("soundtrack", new ByteArrayInputStream(myByteArray), "she-wolf.mp3");  

8.用BinaryHttpResponseHandler下载二进制数据

[java] 
1.BinaryHttpResponseHandler用于获取二进制数据如图片和其他文件  
2.AsyncHttpClient client = new AsyncHttpClient();  
3.String[] allowedContentTypes = new String[] { "image/png", "image/jpeg" };  
4.client.get("http://example.com/file.png", new BinaryHttpResponseHandler(allowedContentTypes) {  
5.    @Override  
6.    public void onSuccess(byte[] fileData) {  
7.        // Do something with the file   
8.    }  
9.});  


9、核心类简介
AsyncHttpRequest
继承自Runnabler，被submit至线程池执行网络请求并发送start，success，faliture等消息
AsyncHttpResponseHandler
接收请求结果，一般重写onSuccess及onFailure接收请求成功或失败的消息，还有onStart，onFinish等消息
TextHttpResponseHandler
继承自AsyncHttpResponseHandler，只是重写了AsyncHttpResponseHandler的onSuccess和onFailure方法，将请求结果由byte数组转换为String
JsonHttpResponseHandler
继承自TextHttpResponseHandler，同样是重写onSuccess和onFailure方法，将请求结果由String转换为JSONObject或JSONArray
BaseJsonHttpResponseHandler
继承自TextHttpResponseHandler，是一个泛型类，提供了parseResponse方法，子类需要提供实现，将请求结果解析成需要的类型，子类可以灵活地使用解析方法，可以直接原始解析，使用gson等。
RequestParams
请求参数，可以添加普通的字符串参数，并可添加File，InputStream上传文件
前面有说明范例
AsyncHttpClient
核心类，使用HttpClient执行网络请求，提供了get，put，post，patch，delete，head等请求方法，使用起来很简单，只需以url及RequestParams调用相应的方法即可，还可以选择性地传入Context，官方推荐传递上下文Content，用于取消Content相关的请求。同时必须提供ResponseHandlerInterface（AsyncHttpResponseHandler继承自ResponseHandlerInterface）的实现类，一般为AsyncHttpResponseHandler的子类，AsyncHttpClient内部有一个线程池，当使用AsyncHttpClient执行网络请求时，最终都会调用sendRequest方法，在这个方法内部将请求参数封装成AsyncHttpRequest（继承自Runnable）交由内部的线程池执行。
SyncHttpClient
继承自AsyncHttpClient，同步执行网络请求，AsyncHttpClient把请求封装成AsyncHttpRequest后提交至线程池，SyncHttpClient把请求封装成AsyncHttpRequest后直接调用它的run方法。
10、请求流程UML简介

1.调用AsyncHttpClient的get或post等方法发起网络请求
2.所有的请求都走了sendRequest，在sendRequest中把请求封装为了AsyncHttpRequest，并添加到线程池执行
3.当请求被执行时（即AsyncHttpRequest的run方法），执行AsyncHttpRequest的makeRequestWithRetries方法执行实际的请求，当请求失败时可以重试。并在请求开始，结束，成功或失败时向请求时传的ResponseHandlerInterface实例发送消息
4.基本上使用的都是AsyncHttpResponseHandler的子类，调用其onStart，onSuccess等方法返回请求结果
5、使用一张清晰的图来概括请求和回调过程

6、重发处理机制
RetryHandler类中维护了一个黑白名单，黑白名单中存储了异常类型，每次请求发送失败都会抛出一个异常，根据这个黑白名单判断是否重发请求。

调用retryRequest()方法传入一个异常对象和一个整数，异常对象是在进行HTTP访问时抛出的，整数表示请求重发的次数。在RetryHandler类的实现中，retryRequest()方法根据重发的次数和抛出的异常还有其它一些属性，响应重发请求。

在makeRequestWithRetries()方法的实现中，利用一个while循环实现请求重发的机制。
请求重发的流程图如下：

7、处理HTTP响应事件
ResponseHandlerInterface接口用于发送消息通知客户端代码处理请求执行过程发送的事件（请求的过程中会发送很多事件，包括开始事件、结束事件、成功事件、失败事件、重发事件、取消事件、更新进度事件），它是一个处理HTTP响应的回调接口。AsyncHttpResponseHandler类是这个接口的直接实现，这个类实现了最基本的事件处理逻辑，还有很多其它具体的处理类都是基于这个类的包装了几个接口，以满足一些特殊的需求。
HTTP请求事件的处理有两种方式：同步和异步，同步方式会在执行请求的线程中直接调用回调方法，而异步方式则会在创建这个处理对象的线程中调用回调方法。
ResponseHandlerInterface接口中定义了一系列sendXXX()方法，这些方法在请求执行过程中被调用，用于发送事件，在AsyncHttpResponseHandler的实现中，这些sendXXX()方法调用了sendMessage()方法，sendMessage()方法中封装了选择同步或异步处理的逻辑。

protected void sendMessage(Message msg) {
    if (getUseSynchronousMode() || handler == null) {
        handleMessage(msg);
    } else if (!Thread.currentThread().isInterrupted()) { // do not send messages if request has been cancelled
        AssertUtils.asserts(handler != null, "handler should not be null!");
        handler.sendMessage(msg);
    }
}
handleMessage()方法模仿了Handler中的处理方法，在这个方法里面实现了具体的事件分发，根据事件类型调用相应的回调接口。
同步方式直接调用handleMessage()方法在当前线程中执行处理；异步方式使用了Handler，在构造方法中使用一个构造线程的Looper对象初始化了这个Handler，事件的处理委托给Handler进行。
处理请求事件的流程图如下：
8、详细使用方法

官方建议使用一个静态的AsyncHttpClient，像下面的这样：

public class TwitterRestClient {    private static final String BASE_URL = "http://api.twitter.com/1/";    private static AsyncHttpClient client = new AsyncHttpClient();    public static void get(String url, RequestParams params, AsyncHttpResponseHandler responseHandler) {        client.get(getAbsoluteUrl(url), params, responseHandler);    }    public static void post(String url, RequestParams params, AsyncHttpResponseHandler responseHandler) {        client.post(getAbsoluteUrl(url), params, responseHandler);    }    private static String getAbsoluteUrl(String relativeUrl) {        return BASE_URL + relativeUrl;    }}
封装的方法建议都加上Context参数，方便在Activity pause或stop时取消掉没用的请求[回收网络请求]。官方也推荐这样使用。
详细使用方法这里不阐述了。具体可以查看官方文档
https://loopj.com/android-async-http/doc/

11、官方推荐网络请求和取消的原则和使用概要
网络请求
首先，我先来讲网络请求过程，你看AsyncHttpRequest里面的方法大多数put、get、post、delete、patch等等，但是他们最终的执行都是sendRequest()方法，当然，他里面的主要方法就是sendRequest请求了。我随便挑一个吧，我就拿delete方法来进行分析吧。
跟踪delete网络请求的源码
//**
     * Perform a HTTP DELETE request.
     *
     * @param context         the Android Context which initiated the request.
     * @param url             the URL to send the request to.
     * @param headers         set one-time headers for this request
     * @param params          additional DELETE parameters or files to send along with request
     * @param responseHandler the response handler instance that should handle the response.
     * @return RequestHandle of future request process
     */
    public RequestHandle delete(Context context, String url, Header[] headers, RequestParams params, ResponseHandlerInterface responseHandler) {
        HttpDelete httpDelete = new HttpDelete(getUrlWithQueryString(isUrlEncodingEnabled, url, params));
        if (headers != null) httpDelete.setHeaders(headers);
        return sendRequest(httpClient, httpContext, httpDelete, null, responseHandler, context);
    }
然后跟踪sendRequest方法，其他的代码注掉了，看核心代码
........
 if (context != null) {
     List<RequestHandle> requestList;
     // Add request to request map
     synchronized (requestMap) {
         requestList = requestMap.get(context);
         if (requestList == null) {
             requestList = Collections.synchronizedList(new LinkedList<RequestHandle>());
             requestMap.put(context, requestList);
         }
     }

     requestList.add(requestHandle);

     Iterator<RequestHandle> iterator = requestList.iterator();
     while (iterator.hasNext()) {
         if (iterator.next().shouldBeGarbageCollected()) {
             iterator.remove();
         }
     }
 }
对上下文进行了为空判断，为什么呢？在跟踪一下AsyncHttpClient的初始化过程
在public AsyncHttpClient(SchemeRegistry schemeRegistry)方法里。
此方法里面有一个重要初始化过程，就是
requestMap = Collections.synchronizedMap(new WeakHashMap<Context, List<RequestHandle>>());
然后在跟踪requestMap ，
private final Map<Context, List<RequestHandle>> requestMap;
requestMap 是一个集合，保存所有Context的集合列表。
这里有什么用呢？下面再说，先我们回到AsyncHttpClient的delete方法，跟踪源码，最终的请求还是AsyncHttpRequest.java文件,AsyncHttpRequest是实现了Runnable接口，所以最重要的方法还是在Run()方法中。
 @Override
    public void run() {
        if (isCancelled()) {
            return;
        }

        // Carry out pre-processing for this request only once.
        if (!isRequestPreProcessed) {
            isRequestPreProcessed = true;
            onPreProcessRequest(this);
        }

        if (isCancelled()) {
            return;
        }

        responseHandler.sendStartMessage();

        if (isCancelled()) {
            return;
        }

        try {
            makeRequestWithRetries();
        } catch (IOException e) {
            if (!isCancelled()) {
                responseHandler.sendFailureMessage(0, null, null, e);
            } else {
                AsyncHttpClient.log.e("AsyncHttpRequest", "makeRequestWithRetries returned error", e);
            }
        }

        if (isCancelled()) {
            return;
        }

        responseHandler.sendFinishMessage();

        if (isCancelled()) {
            return;
        }

        // Carry out post-processing for this request.
        onPostProcessRequest(this);

        isFinished = true;
    }
你可以看出，在run()方法中，标红的代码出现了5次之多，执行之平频繁，这里面还牵扯到其他的回调对是否cancel的判断，所以具体什么时候会取消，在哪里去取消，不得而知了。因为你的取消请求有可能在这几处之中的任何一个中。
网络取消
其次，接下来说取消网络请求有下面几种方式。有了上一步网络请求的过程，接下来说取消网络部分会轻松的多。


在AsyncHttpClient中有三个方法：cancelAllRequest(boolean)、cancelRequests(Context, booolean)，cancelRequestsByTAG(Object TAG, boolean mayInterruptIfRunning)前一个方法取消系统中所有的请求，后一个方法取消一个Context下的所有请求，最后一个是取消当前标记的所有请求。
分别粘贴源码进行解析
/**
     * Cancels any pending (or potentially active) requests associated with the passed Context.
     * <p>&nbsp;</p> <b>Note:</b> This will only affect requests which were created with a non-null
     * android Context. This method is intended to be used in the onDestroy method of your android
     * activities to destroy all requests which are no longer required.
     *
     * @param context               the android Context instance associated to the request.
     * @param mayInterruptIfRunning specifies if active requests should be cancelled along with
     *                              pending requests.
     */
    public void cancelRequests(final Context context, final boolean mayInterruptIfRunning) {
        if (context == null) {
            log.e(LOG_TAG, "Passed null Context to cancelRequests");
            return;
        }

        final List<RequestHandle> requestList = requestMap.get(context);
        requestMap.remove(context);

        if (Looper.myLooper() == Looper.getMainLooper()) {
            Runnable runnable = new Runnable() {
                @Override
                public void run() {
                    cancelRequests(requestList, mayInterruptIfRunning);
                }
            };
            threadPool.submit(runnable);
        } else {
            cancelRequests(requestList, mayInterruptIfRunning);
        }
    }
/**
     * Cancels any pending (or potentially active) requests associated with the passed Context.
     * <p>&nbsp;</p> <b>Note:</b> This will only affect requests which were created with a non-null
     * android Context. This method is intended to be used in the onDestroy method of your android
     * activities to destroy all requests which are no longer required.
     *
     * @param context               the android Context instance associated to the request.
     * @param mayInterruptIfRunning specifies if active requests should be cancelled along with
     *                              pending requests.
     */
    public void cancelRequests(final Context context, final boolean mayInterruptIfRunning) {
        if (context == null) {
            log.e(LOG_TAG, "Passed null Context to cancelRequests");
            return;
        }

        final List<RequestHandle> requestList = requestMap.get(context);
        requestMap.remove(context);

        if (Looper.myLooper() == Looper.getMainLooper()) {
            Runnable runnable = new Runnable() {
                @Override
                public void run() {
                    cancelRequests(requestList, mayInterruptIfRunning);
                }
            };
            threadPool.submit(runnable);
        } else {
            cancelRequests(requestList, mayInterruptIfRunning);
        }
    }
private void cancelRequests(final List<RequestHandle> requestList, final boolean mayInterruptIfRunning) {
        if (requestList != null) {
            for (RequestHandle requestHandle : requestList) {
                requestHandle.cancel(mayInterruptIfRunning);
            }
        }
    }
其实很好理解，如果你没有传递这个值的，直接返回了。如果你传递此值，在初始化时，就对Content的对象进行初始化操作了，前面提到过，然后如他的对象并加到hashMap中，直接取出来，cancel即可。
这里有一个有意思的事，就是红色标注的地方，如果当前的looper是主线程，就放在线程池里面执行，如果是子线程的话，直接执行，是不是感觉设计的很好啊，不会暂用UI main线程的操作。
总结
1、优点
Android-Async-Http的使用非常简单，通过AsyncHttpClient发起请求就可以了，如果需要添加参数，直接传一个RequestParams过去，而且参数可以是Json、String、File和InputStream等等可以很方便地上传文件。
每个请求都需要传一个ResponseHandlerInterface的实例用以接收请求结果或请求失败，请求结束等通知，一般是AsyncHttpResponseHandler的子类。
通过BinaryHttpResponseHandler可以发起二进制请求，如请求图片。
通过TextHttpResponseHandler可以发起返回结果为字符串的请求，一般这个使用较多。
也可以使用它的子类JsonHttpResponseHandler，返回结果是一个JSONObject或JSONArray。不过感觉这个类作用不大，一是有另一个类BaseJsonHttpResponseHandler，可以直接解析返回的JSON数据，二是JsonHttpResponseHandler的方法太复杂了，有太多的onSuccess和onFailure方法，都不知道重写哪个了。

如上图所示，每个子类有太多的onSuccess和onFailure了，尤其是JsonHttpResponseHandler，这应该算是这个类库的不足吧。所以平时使用时基本不使用JsonHttpResponseHandler，而是直接使用TextHttpResponseHandler，当然也可以使用BaseJsonHttpResponseHandler。

支持Https校验，跟踪源码：
/**
     * Creates a new AsyncHttpClient with default constructor arguments values
     */
    public AsyncHttpClient() {
        this(false, 80, 443);
    }

    /**
     * Creates a new AsyncHttpClient.
     *
     * @param httpPort non-standard HTTP-only port
     */
    public AsyncHttpClient(int httpPort) {
        this(false, httpPort, 443);
    }

    /**
     * Creates a new AsyncHttpClient.
     *
     * @param httpPort  non-standard HTTP-only port
     * @param httpsPort non-standard HTTPS-only port
     */
    public AsyncHttpClient(int httpPort, int httpsPort) {
        this(false, httpPort, httpsPort);
    }

    /**
     * Creates new AsyncHttpClient using given params
     *
     * @param fixNoHttpResponseException Whether to fix issue or not, by omitting SSL verification
     * @param httpPort                   HTTP port to be used, must be greater than 0
     * @param httpsPort                  HTTPS port to be used, must be greater than 0
     */
    public AsyncHttpClient(boolean fixNoHttpResponseException, int httpPort, int httpsPort) {
        this(getDefaultSchemeRegistry(fixNoHttpResponseException, httpPort, httpsPort));
    }
/**
     * Returns default instance of SchemeRegistry
     *
     * @param fixNoHttpResponseException Whether to fix issue or not, by omitting SSL verification
     * @param httpPort                   HTTP port to be used, must be greater than 0
     * @param httpsPort                  HTTPS port to be used, must be greater than 0
     */
    private static SchemeRegistry getDefaultSchemeRegistry(boolean fixNoHttpResponseException, int httpPort, int httpsPort) {
        if (fixNoHttpResponseException) {
            log.d(LOG_TAG, "Beware! Using the fix is insecure, as it doesn't verify SSL certificates.");
        }

        if (httpPort < 1) {
            httpPort = 80;
            log.d(LOG_TAG, "Invalid HTTP port number specified, defaulting to 80");
        }

        if (httpsPort < 1) {
            httpsPort = 443;
            log.d(LOG_TAG, "Invalid HTTPS port number specified, defaulting to 443");
        }

        // Fix to SSL flaw in API < ICS
        // See https://code.google.com/p/android/issues/detail?id=13117
        SSLSocketFactory sslSocketFactory;
        if (fixNoHttpResponseException) {
            sslSocketFactory = MySSLSocketFactory.getFixedSocketFactory();
        } else {
            sslSocketFactory = SSLSocketFactory.getSocketFactory();
        }

        SchemeRegistry schemeRegistry = new SchemeRegistry();
        schemeRegistry.register(new Scheme("http", PlainSocketFactory.getSocketFactory(), httpPort));
        schemeRegistry.register(new Scheme("https", sslSocketFactory, httpsPort));

        return schemeRegistry;
    }
请看红色标注的地方，都是对用户的秘钥进行验证，如果你不需要进行校验的话，避开验证的话，需要在初始化AsyncHttpClient(boolean fixNoHttpResponseException, int httpPort, int httpsPort) {
的时候fixNoHttpResponseException设置为true
2、缺点
这个类库还有一点不足，就是onSuccess等方法一般会在主线程执行，其实这么说不太严谨，还是跟踪源码吧：

/**
 * Creates a new AsyncHttpResponseHandler
 */
public AsyncHttpResponseHandler() {
    this(null);
}

/**
 * Creates a new AsyncHttpResponseHandler with a user-supplied looper. If
 * the passed looper is null, the looper attached to the current thread will
 * be used.
 *
 * @param looper The looper to work with
 */
public AsyncHttpResponseHandler(Looper looper) {
    this.looper = looper == null ? Looper.myLooper() : looper;

    // Use asynchronous mode by default.
    setUseSynchronousMode(false);

    // Do not use the pool's thread to fire callbacks by default.
    setUsePoolThread(false);
}

可以看到，内部使用了Handler，当新建AsyncHttpResponseHandler的实例的时候会获取当前线程的Looper，如果为空就启用同步模式，即所有的回调都会在执行请求的线程中执行，当在一个普通的后台线程时这样执行是正常的，而我们一般都会在主线程发请请求，结果就是所有的回调都会在主线程中执行，这就限制了我们在onSuccess中执行耗时操作，比如请求成功后将数据持久化到数据库。
不过可以看到创建Handler的时候使用了Looper对象，所以我们就可以改进一下其构造函数，添加一个Looper参数（同步修改子类），这样所有的回调就都会在Looper所在线程执行，这样我们只需要开启一个HandlerThread就行了。
弊端：但这样和Looper为空时一样有一个弊端，如果要更新UI操作的话，还需要向一个主线程的Handler发送消息让UI更新。
还有第二个弊端，所有回调都在同一个HandlerThread中执行，如果一个处理耗时太久会阻塞后面的请求结果处理，如果只是简单地写个数据库影响应该不大，如果真耗时太久，为这个耗时处理再开个线程了。
第三个弊端，就是取消当前的网络请求，不知具体是哪一个时间段取消的，官方推荐取消网络请求时，最好添加一些标签TAG或者当前Activity的Content上下文，通过绑定Context，可以对一个Context相关的所有请求执行一些操作，例如，可以在Activity的onDestroy()方法中可以取消与在Activity中创建的所有请求。主要方便在当前取消网络请求做判断。为了更好的理解这种种设计理念，我们还是查看源码，前面已经阐述过了。因为请求和取消是想对应的。


