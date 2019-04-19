---
title: 使用Java NIO实现一个HTTP服务器
tags:
  - NIO
  - HTTP
categories:
  - NIO
  - HTTP
date: 2019-04-19 11:04:30
---

## 引言

一直以来都想写一个自己的http服务器，这次趁着稍微空闲了一些，赶紧码了一个`mini`版的。在这里跟大家分享一下编写的整个思路，总体来说，整个应用非常简单，目前也只是实现了最基本的静态资源访问，但对于想学习`Http`协议的同学来说，应该还是有所帮助

## HTTP协议

先简单介绍一下`HTTP`协议，`HTTP`协议是构建于`TCP/IP`协议之上的一个应用层协议，并且是无连接无状态的。

### http请求报文

http的请求报文由三部分组成

* 状态行  `<method> <request-URL> <version>` method包含`GET`，`POST`，`PUT`，`DELETE`等

* 请求头

* 消息主体

下面是典型的http请求报文示例

```

GET /static/home.html HTTP/1.1

Host: localhost:8080

Connection: keep-alive

Cache-Control: max-age=0

Upgrade-Insecure-Requests: 1

User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_14_0) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/73.0.3683.103 Safari/537.36

Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3

Accept-Encoding: gzip, deflate, br

Accept-Language: zh-CN,zh;q=0.9

```

### http响应报文

响应报文跟请求报文类似，也由三部分组成

*  状态行  这里需要关注下响应的各个状态所代表的含义，以及浏览器识别这些状态码会相应的做哪些事情

*  响应头

*  响应正文

```

HTTP/1.1 200 OK

Server: cloud http v1.0

Content-Type: text/html;charset=UTF-8

<html>...

```

**##** **动手**

前面简单介绍了一下http协议的基本信息，下面就是源码实现了，我这里直接使用了Java NIO作为底层通信支撑，在实际的生产代码中，大多会选择封装良好的`netty`来作为稳定的底层通信

这里先放上项目源码 [[CloudHttp](https://github.com/WangJunnan/cloudhttp)](https://github.com/WangJunnan/cloudhttp)

### 定义Request和Response

* Request

```java

public class Request {

 /**

 * method

 */

 private String method;

 /**

 * http协议版本

 */

 private String httpVersion;

 /**

 * 请求uri

 */

 private String uri;

 /**

 * 请求相对路径

 */

 private String path;

 /**

 * 请求头信息

 */

 private Map<String, String> headers;

 /**

 * 请求参数

 */

 private Map<String, String> attribute;

}

```

* Response

```java

public class Response {

 private Integer code;

 private String protocol = "HTTP/1.1";

 private String msg;

 private Map<String, String> headers;

 private ByteArrayOutputStream outPutStream = new ByteArrayOutputStream();

 public Response() {

 this.code = HttpCode.STATUS_200.getCode();

 this.msg = HttpCode.STATUS_200.getMsg();

 headers = new HashMap<>();

 headers.put("Content-Type", "text/html;charset=UTF-8");

 headers.put("Server", "cloud http v1.0");

 }

}

```

*  定义请求响应解析类  `HttpParser`

```java

public class HttpParser {

 private final static Logger logger = LoggerFactory.getLogger(HttpParser.class);

 /**

 * <p>解析http请求体</p>

 *

 * @param buffers

 * @return

 */

 public static Request decodeReq(byte [] buffers) {

 Request request = new Request();

 if (buffers != null) {

 String resString = new String(buffers);

 logger.info(resString);

 String[] headers = resString.trim().split("\r\n");

 if (headers.length > 0) {

 String firstline = headers[0];

 // 按空格分割字符串

  // 解析 method uri 协议版本

 String mainInfo [] = firstline.split("\\s+");

 request.setMethod(mainInfo[0]);

 try {

 request.setUri(URLDecoder.decode(mainInfo[1], "UTF-8"));

 } catch (UnsupportedEncodingException e) {

 logger.error("error_HttpParser_URLDecode, uri = {}", mainInfo[1], e);

 }

 request.setHttpVersion(mainInfo[2]);

 // 解析header

 Map<String, String> headersMap = new HashMap<>();

 for (int i = 1; i < headers.length; i++) {

 String entryStr = headers[i];

 String entry [] = entryStr.trim().split(":");

 headersMap.put(entry[0].trim(), entry[1].trim());

 }

 request.setHeaders(headersMap);

 // 解析参数

 String uri = request.getUri();

 Map<String, String> attribute = new HashMap<>();

 request.setPath(uri);

 if (StringUtils.isNotEmpty(uri)) {

 int indexOfParam = uri.indexOf("?");

 if (indexOfParam > 0) {

 // 设置path

 request.setPath(uri.substring(0, indexOfParam));

 String queryString = uri.substring(indexOfParam + 1, uri.length());

 String paramEntrys [] = queryString.split("&");

 for (String paramEntry : paramEntrys) {

 String [] entry = paramEntry.split("=");

 if (entry.length > 0) {

 String key = entry[0];

 String value = entry.length > 1 ? entry[1] : "";

 attribute.put(key, value);

 }

 }

 }

 }

 request.setAttribute(attribute);

 }

 }

 return request;

 }

 /**

 * <p>返回http响应字节流</p>

 *

 * @param response

 * @return

 */

 public static byte[] encodeResHeader(Response response) {

 StringBuilder resBuild = new StringBuilder();

 resBuild.append(response.getProtocol() + " " + response.getCode() + " " + response.getMsg());

 resBuild.append("\r\n");

 Map<String, String> headers = response.getHeaders();

 headers.entrySet().forEach(entry -> {

 resBuild.append(entry.getKey());

 resBuild.append(": ");

 resBuild.append(entry.getValue());

 resBuild.append("\r\n");

 });

 resBuild.append("\r\n");

 String resString = resBuild.toString();

 byte [] bytes = null;

 try {

 bytes = resString.getBytes("UTF-8");

 } catch (UnsupportedEncodingException e) {

 logger.error("error_encodeResHeader", e);

 }

 return bytes;

 }

}

```

### 使用Java nio 启动服务器

* NioServer

```java

public class NioServer implements Server, Runnable {

 private final static Logger logger = LoggerFactory.getLogger(NioServer.class);

 private Thread serverThread;

 private Integer port;

  private  boolean running = false;

 private static volatile NioServer server;

 private CloudService cloudService;

 private ServerSocketChannel serverSocketChannel;

 private ByteBuffer readBuffer = ByteBuffer.allocate(8192);

 public static final ExecutorService requestWork = Executors.newCachedThreadPool();

 public static NioServer getServerInstance () {

 if (server == null) {

 synchronized (NioServer.class) {

 if (server == null) {

 server = new NioServer();

 }

 }

 }

 return server;

 }

 public NioServer() {

 this.cloudService = new CloudService();

 port = Integer.valueOf(CloudHttpConfig.getValue("port", "8080"));

 }

 @Override

  public  synchronized  void start() {

 if (running) {

 logger.info("服务器已经启动");

 return;

 }

 serverThread = new Thread(this);

 serverThread.start();

 this.running = true;

 }

 @Override

 public void stop() {

 try {

 serverSocketChannel.close();

 serverThread.stop();

 } catch (IOException e) {

 logger.error("error_NioServer_stop", e);

 }

 }

 @Override

 public void run() {

 try {

  //打开ServerSocketChannel通道

 serverSocketChannel = ServerSocketChannel.open();

  //得到ServerSocket对象

 serverSocketChannel.socket().bind(new InetSocketAddress(port));

 Selector selector = Selector.open();

 serverSocketChannel.configureBlocking(false);

 serverSocketChannel.register(selector, SelectionKey.OP_ACCEPT);

 while (true) {

 selector.select();

 Iterator<SelectionKey> selectedKeys = selector.selectedKeys().iterator();

 SelectionKey key = null;

 while (selectedKeys.hasNext()) {

 key = selectedKeys.next();

 selectedKeys.remove();

 if (!key.isValid()) {

 continue;

 }

 if (key.isAcceptable()) {

 accept(key);

 }

 if (key.isReadable()) {

 read(key);

 }

 if (key.isWritable()) {

 write(key);

 }

 }

 }

 } catch (IOException e) {

 logger.error("error_NioServer_run", e);

 }

 }

 private void accept(SelectionKey key) {

 try {

 ServerSocketChannel serverSocketChannel = (ServerSocketChannel)key.channel();

 SocketChannel socketChannel = serverSocketChannel.accept();

 socketChannel.configureBlocking(false);

 socketChannel.register(key.selector(), SelectionKey.OP_READ);

 } catch (IOException e) {

 logger.error("error_NioServer_accept", e);

 }

 }

 private Request read(SelectionKey key) {

 Request request = null;

 SocketChannel socketChannel = (SocketChannel) key.channel();

 try {

 int readNum = socketChannel.read(readBuffer);

 if (readNum == -1) {

 socketChannel.close();

 key.cancel();

 return null;

 }

 readBuffer.flip();

 byte [] buffers = new byte[readBuffer.limit()];

 readBuffer.get(buffers);

 readBuffer.clear();

 request = HttpParser.decodeReq(buffers);

 requestWork.execute(new RequestWorker(request, key, cloudService));

 } catch (IOException e) {

 logger.error("error_NioServer_read", e);

 }

 return request;

 }

 private void write(SelectionKey key) throws IOException {

 ByteBuffer buffer = (ByteBuffer) key.attachment();

 if(buffer == null || !buffer.hasRemaining()) {

 return;

 }

 SocketChannel socketChannel = (SocketChannel) key.channel();

 socketChannel.write(buffer);

 if(!buffer.hasRemaining()){

 key.interestOps(SelectionKey.OP_READ);

 buffer.clear();

 }

 socketChannel.close();

 }

}

```

### 处理请求的 RequestWork类

对于每一个进入的请求。都会单独启动一个线程来进行处理

* RequestWorker

```java

public class RequestWorker implements Runnable {

 private final static Logger logger = LoggerFactory.getLogger(RequestWorker.class);

 private Request request;

 private SelectionKey key;

 private SocketChannel channel;

 private CloudService cloudService;

 public RequestWorker(Request request, SelectionKey key, CloudService cloudService) {

 this.request = request;

 this.key = key;

 this.channel = (SocketChannel) key.channel();

 this.cloudService = cloudService;

 }

 @Override

 public void run() {

 Response response = new Response();

 try {

 cloudService.doService(request, response);

 } catch (ViewNotFoundException e) {

 response.setCode(HttpCode.STATUS_404.getCode());

 response.setMsg(HttpCode.STATUS_404.getMsg());

 ByteArrayOutputStream outputStream = response.getOutPutStream();

 InputStream inputStream = this.getClass().getClassLoader().getResourceAsStream("404.html");

 byte bytes [] = new byte [1024];

 int len;

 try {

 while ((len = inputStream.read(bytes)) > -1) {

 outputStream.write(bytes, 0, len);

 }

 } catch (IOException e1) {

 logger.error("error_requestWork_write404", e);

 }

 }

 byte [] resHeader = HttpParser.encodeResHeader(response);

 byte [] body = response.getOutPutStream().toByteArray();

 ByteBuffer byteBuffer = ByteBuffer.allocate(resHeader.length + body.length);

 byteBuffer.put(resHeader);

 byteBuffer.put(body);

 byteBuffer.flip();

  // 将输出流绑定至附件

 key.attach(byteBuffer);

  // 注册写事件

 key.interestOps(SelectionKey.OP_WRITE);

 key.selector().wakeup();

 }

}

```

### 定义Servlet接口

* CloudServlet

```java

public interface CloudServlet {

 /**

 * 判断当前请求是否匹配 handler

 *

 * @param request

 * @return

 */

 boolean match(Request request);

 /**

 * 执行初始化

 */

 void init(Request request, Response response);

 /**

 * 执行请求

 *

 * @param request

 * @param response

 */

 void doService(Request request, Response response);

}

```

* StaticViewServlet 定义一个处理静态资源的`StaticViewServlet`实现`CloudServlet`接口

该静态资源处理器会自动拦截`static`路径下的请求

```java

public class StaticViewServlet implements CloudServlet {

 private final static Logger logger = LoggerFactory.getLogger(StaticViewServlet.class);

 private String staticRootPath;

 public static Pattern p = Pattern.compile("^/static/\\S+");

 @Override

 public boolean match(Request request) {

 String path = request.getPath();

 Matcher matcher = p.matcher(path);

 return matcher.matches();

 }

 @Override

 public void init(Request request, Response response) {

 staticRootPath = CloudHttpConfig.getValue("static.resource.path");

 }

 @Override

 public void doService(Request request, Response response) {

 String path = request.getPath();

 String fileRelativePath = path.substring(8);

 String absolutePath = staticRootPath + "/" + fileRelativePath;

 RandomAccessFile randomAccessFile = null;

 try {

 randomAccessFile = new RandomAccessFile(absolutePath, "r");

 FileChannel fileChannel = randomAccessFile.getChannel();

 ByteBuffer htmBuffer = ByteBuffer.allocate((int)fileChannel.size());

 fileChannel.read(htmBuffer);

 htmBuffer.flip();

 byte [] htmByte = new byte[htmBuffer.limit()];

 htmBuffer.get(htmByte);

 response.getOutPutStream().write(htmByte);

 } catch (FileNotFoundException e) {

 throw new ViewNotFoundException();

 } catch (IOException e) {

 logger.error("error_StaticViewServlet_doService 异常", e);

 }

 }

}

```

### 定义 CloudService

`CloudService`是用于盛放`Servlet`的容器

```java

public class CloudService {

 private List<CloudServlet> cloudServlets = new ArrayList<>();

 public CloudService() {

 cloudServlets.add(new StaticViewServlet());

 cloudServlets.add(new MappingUrlServlet());

 }

 public void doService(Request request, Response response) {

 CloudServlet servlet = doSelect(request);

 servlet.init(request, response);

 servlet.doService(request, response);

 }

 private CloudServlet doSelect(Request request) {

 for (CloudServlet cloudServlet : cloudServlets) {

 if (cloudServlet.match(request)) {

 return cloudServlet;

 }

 }

 return new MappingUrlServlet();

 }

}

```

### 最后  编写启动类

```java

public class CloudHttpServer {

 /**

 * 启动服务器

 */

 public static void startServer() {

 NioServer.getServerInstance().start();

 }

 public static void main(String[] args) {

 startServer();

 }

}

```

## 测试

1.  启动服务器

2.  浏览器访问  `http://localhost:8080/static/home.html`

响应页面

![image.png](https://upload-images.jianshu.io/upload_images/2717496-aa1ea0a3a1d1f266.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


## 尾言

本文只是一个示例，仅仅实现了静态资源访问。做为自娱自乐的项目 +——=

