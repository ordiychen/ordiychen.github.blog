---
layout:  post
title:  Apache Http-Client5 使用
date:  2020-01-01 00:00 +08:00
categories:  [tech]
tags:  [Java,JavaWeb,Http,Apache]
toc: true
---

HttpClient提供一个高效，最新且功能丰富的软件包来实现这一空白，以实现最新HTTP标准和建议的客户端。
`Apache http client 5.0`支持了`HTTP/2`的大部分特性，并对之前版本进行了一些列优化。
<!--more-->

# Apache http client 5.0 简介
`Apache http client 5.0`支持了`HTTP/2`的大部分特性，并对之前版本进行了一些列优化，具体如下：
```txt
Notable changes and features included in the 5.0 series are:

* Support for the HTTP/2 protocol and conformance to requirements and
  recommendations of the latest HTTP/2 protocol specification documents
  (RFC 7540, RFC 7541.)

  Supported features:

    ** HPACK header compression
    ** Stream multiplexing (client and server)
    ** Flow control
    ** Response push
    ** Message trailers
    ** Expect-continue handshake
    ** Connection validation (ping)
    ** Application-layer protocol negotiation (ALPN)
    ** TLS 1.2 security features

* Improved conformance to requirements and recommendations of the latest HTTP/1.1 protocol
  specification documents (RFC 7230, RFC 7231.)

* New connection pool implementation with lax connection limit guarantees and better
  performance under higher concurrency due to absence of a global pool lock.

* Support for Reactive Streams API [http://www.reactive-streams.org/]

* Package name space changed to 'org.apache.hc.client5'.

* Maven group id changed to 'org.apache.httpcomponents.client5'.

HttpClient 5.0 releases can be co-located with earlier major versions on the same classpath
due to the change in package names and Maven module coordinates.
```

# 基本使用用法

### 表单登录获取cookie

```java
    public String admin_userName="admin";
    public String admin_userPwd="admin123";
    String formUserFiledName ="j_username";
    String formPwdFiledName="j_password";
    BasicCookieStore cookieStore = new BasicCookieStore();

    //调用该方法注意对 CloseableHttpClient 在调用完成后执行 close()方法 || 可以复用
    public CloseableHttpClient buildCookieHttpClient(String loginUrl) throws IOException, URISyntaxException {
         CloseableHttpClient httpClient = HttpClients.custom()
                .setDefaultCookieStore(cookieStore).build();

            //step1~2 cookie and login
            initCookie(httpClient, cookieStore,loginUrl);
            int statusCode = loginRanger(httpClient, cookieStore, loginUrl);
            if (statusCode != 200) {
                // login failed
                log.warn(" ranger-admin login failed ,status_code:{}", statusCode);
                throw new RuntimeException("login ranger-admin failed.");
            }
        return httpClient;
    }
        private int loginRanger(CloseableHttpClient httpClient,BasicCookieStore cookieStore,String rangerLoginUrl) throws URISyntaxException, IOException {
        log.info("==> loginRanger userName:{}",admin_userName);
         ClassicHttpRequest login = ClassicRequestBuilder.post()
                .setUri(new URI(rangerLoginUrl))
                .addParameter(formUserFiledName, admin_userName)
                .addParameter(formPwdFiledName, admin_userPwd)
                .build();
        try (final CloseableHttpResponse loginResponse = httpClient.execute(login)) {
            final HttpEntity entity = loginResponse.getEntity();
            log.debug("Login form get: " + loginResponse.getCode() + " " + loginResponse.getReasonPhrase());
            EntityUtils.consume(entity);

            log.debug("====> Post login cookies:");
            final List<Cookie> cookies = cookieStore.getCookies();
            if (cookies.isEmpty()) {
                log.debug("===> Cookie None");
            } else {
                for (int i = 0; i < cookies.size(); i++) {
                    log.debug("- " + cookies.get(i).toString());
                }
            }
            return loginResponse.getCode();
        }
    }

     private void initCookie(CloseableHttpClient httpClient, BasicCookieStore cookieStore,String hostUrl)throws IOException {
         HttpGet httpget = new HttpGet(hostUrl);
        try (final CloseableHttpResponse response1 = httpClient.execute(httpget)) {
            final HttpEntity entity = response1.getEntity();

            log.debug("Login form get: " + response1.getCode() + " " + response1.getReasonPhrase());
            EntityUtils.consume(entity);

            log.debug("Initial set of cookies:");
            final List<Cookie> cookies = cookieStore.getCookies();
            if (cookies.isEmpty()) {
                log.warn(" get user cookies is None");
            } else {
                for (int i = 0; i < cookies.size(); i++) {
                    log.debug("get user cookies - " + cookies.get(i).toString());
                }
            }
        }
    }
```
官方Demo [Apache-Client ClientFormLogin.java](https://hc.apache.org/httpcomponents-client-5.0.x/httpclient5/examples/ClientFormLogin.java)


### `POST request`使用示例
`POST request`使用示例
```java
     //默认的header
    List<Header> defaultHeaderArr = Arrays.asList(new BasicHeader("Content-Type", "application/json"),
                new BasicHeader("Accept-Encoding", " gzip, deflate"),
                new BasicHeader("Accept", "application/json, text/javascript, */*; q=0.01"),
                new BasicHeader("Connection", "keep-alive"),
                new BasicHeader("X-Requested-With", "XMLHttpRequest"));

    String jsonStr ="{}"
    HttpPost request = new HttpPost(url);
    headerList.stream().forEach(request::addHeader);
    request.setEntity(jsonStr);
    httpClient.execute(request);

```


### `Get request` 使用示例
`Get request` 使用demo:
```java
                 //build get request with paramer
                 String url =hostUrl + "/service/plugins/policies/service" + "/" + serviceId;
                 HttpGet getRequest  = new HttpGet(url);
                 getRequest.addHeader(HttpHeaders.ACCEPT,"application/json, text/javascript, */*; q=0.01");
                 URIBuilder uriBuilder = new URIBuilder(getRequest.getUri());
                 URI uri = uriBuilder
                         .addParameter("page", "0")
                         .addParameter("pageSize", "2000")
                         .addParameter("startIndex", "0")
                         .build();
                 getRequest.setUri(uri);
                 httpClient.execute(getRequest);
```
官方Demo [GetQuickStart.java](https://hc.apache.org/httpcomponents-client-5.0.x/httpclient5/examples/QuickStart.java)


### 拦截器特性
可以通过给client request增加 interceptor可以增加`request id`和异常拦截等功能
```
public static void main(final String[] args) throws Exception {
        try (final CloseableHttpClient httpclient = HttpClients.custom()

                // Add a simple request ID to each outgoing request

                .addRequestInterceptorFirst(new HttpRequestInterceptor() {

                    private final AtomicLong count = new AtomicLong(0);

                    @Override
                    public void process(
                            final HttpRequest request,
                            final EntityDetails entity,
                            final HttpContext context) throws HttpException, IOException {
                        request.setHeader("request-id", Long.toString(count.incrementAndGet()));
                    }
                })

                // Simulate a 404 response for some requests without passing the message down to the backend

                .addExecInterceptorAfter(ChainElement.PROTOCOL.name(), "custom", new ExecChainHandler() {

                    @Override
                    public ClassicHttpResponse execute(
                            final ClassicHttpRequest request,
                            final ExecChain.Scope scope,
                            final ExecChain chain) throws IOException, HttpException {

                        final Header idHeader = request.getFirstHeader("request-id");
                        if (idHeader != null && "13".equalsIgnoreCase(idHeader.getValue())) {
                            final ClassicHttpResponse response = new BasicClassicHttpResponse(HttpStatus.SC_NOT_FOUND, "Oppsie");
                            response.setEntity(new StringEntity("bad luck", ContentType.TEXT_PLAIN));
                            return response;
                        } else {
                            return chain.proceed(request, scope);
                        }
                    }

                })
                .build()) {

            for (int i = 0; i < 20; i++) {
                final HttpGet httpget = new HttpGet("http://httpbin.org/get");

                System.out.println("Executing request " + httpget.getMethod() + " " + httpget.getUri());

                try (final CloseableHttpResponse response = httpclient.execute(httpget)) {
                    System.out.println("----------------------------------------");
                    System.out.println(response.getCode() + " " + response.getReasonPhrase());
                    System.out.println(EntityUtils.toString(response.getEntity()));
                }
            }
        }
    }
```
参照官方Demo [ClientInterceptors.java](https://hc.apache.org/httpcomponents-client-5.0.x/httpclient5/examples/ClientInterceptors.java)

### 线程池
使用`PoolingHttpClientConnectionManager`可以实现多线程并发访问，在并发场景下推荐使用异步模式。
```
// Create an HttpClient with the PoolingHttpClientConnectionManager.
        // This connection manager must be used if more than one thread will
        // be using the HttpClient.
        final PoolingHttpClientConnectionManager cm = new PoolingHttpClientConnectionManager();
        cm.setMaxTotal(100);

        try (final CloseableHttpClient httpclient = HttpClients.custom()
                .setConnectionManager(cm)
                .build()) {
            // create an array of URIs to perform GETs on
            final String[] urisToGet = {
                    "http://hc.apache.org/",
                    "http://hc.apache.org/httpcomponents-core-ga/",
                    "http://hc.apache.org/httpcomponents-client-ga/",
            };

            // create a thread for each URI
            final GetThread[] threads = new GetThread[urisToGet.length];
            for (int i = 0; i < threads.length; i++) {
                final HttpGet httpget = new HttpGet(urisToGet[i]);
                threads[i] = new GetThread(httpclient, httpget, i + 1);
            }

            // start the threads
            for (final GetThread thread : threads) {
                thread.start();
            }

            // join the threads
            for (final GetThread thread : threads) {
                thread.join();
            }
```

### 发送多个部分的参数
使用`MultipartEntityBuilder`可以实现发送多个不同的种类的请求参数：
```
 if (args.length != 1)  {
            System.out.println("File path not given");
            System.exit(1);
        }
        try (final CloseableHttpClient httpclient = HttpClients.createDefault()) {
            final HttpPost httppost = new HttpPost("http://localhost:8080" +
                    "/servlets-examples/servlet/RequestInfoExample");

            final FileBody bin = new FileBody(new File(args[0]));
            final StringBody comment = new StringBody("A binary file of some kind", ContentType.TEXT_PLAIN);

            final HttpEntity reqEntity = MultipartEntityBuilder.create()
                    .addPart("bin", bin)
                    .addPart("comment", comment)
                    .build();


            httppost.setEntity(reqEntity);

            System.out.println("executing request " + httppost);
            try (final CloseableHttpResponse response = httpclient.execute(httppost)) {
                System.out.println("----------------------------------------");
                System.out.println(response);
                final HttpEntity resEntity = response.getEntity();
                if (resEntity != null) {
                    System.out.println("Response content length: " + resEntity.getContentLength());
                }
                EntityUtils.consume(resEntity);
            }
        }

```


### 异步请求
基于NIO的`HttpAsyncClients` 很好的实现了异步请求。
```java
 final IOReactorConfig ioReactorConfig = IOReactorConfig.custom()
                .setSoTimeout(Timeout.ofSeconds(5))
                .build();

        final CloseableHttpAsyncClient client = HttpAsyncClients.custom()
                .setIOReactorConfig(ioReactorConfig)
                .build();
        client.start();

        final String requestUri = "http://httpbin.org/post";
        final AsyncRequestProducer requestProducer = AsyncRequestBuilder.post(requestUri)
                .setEntity(new StringAsyncEntityProducer("some stuff", ContentType.TEXT_PLAIN))
                .build();
        final Future<SimpleHttpResponse> future = client.execute(requestProducer, SimpleResponseConsumer.create(),
                new FutureCallback<SimpleHttpResponse>() {

                    @Override
                    public void completed(final SimpleHttpResponse response) {
                        System.out.println(requestUri + "->" + response.getCode());
                        System.out.println(response.getBody());
                    }

                    @Override
                    public void failed(final Exception ex) {
                        System.out.println(requestUri + "->" + ex);
                    }

                    @Override
                    public void cancelled() {
                        System.out.println(requestUri + " cancelled");
                    }

                });
        try {
            future.get();
        } catch (InterruptedException e) {
            e.printStackTrace();
        } catch (ExecutionException e) {
            e.printStackTrace();
        }

        System.out.println("Shutting down");
        client.close(CloseMode.GRACEFUL);
```

### `http-client5`日志配置
`http-client5`使用了`slf4j`输出日志，如果需要配置为`log4j2`需要进行一下配置：
```xml
<dependency>
            <groupId>org.apache.logging.log4j</groupId>
            <artifactId>log4j-core</artifactId>
            <version>2.13.2</version>
        </dependency>
        <dependency>
            <groupId>org.apache.logging.log4j</groupId>
            <artifactId>log4j-api</artifactId>
            <version>2.13.2</version>
        </dependency>
        <dependency>
            <groupId>org.apache.logging.log4j</groupId>
            <artifactId>log4j-slf4j-impl</artifactId>
            <version>2.13.2</version>
        </dependency>
```
`log4j2.xml`配置：
```xml
<Configuration>
    <Appenders>
        <Console name="STDOUT" target="SYSTEM_OUT" >
            <PatternLayout pattern="%d{HH:mm:ss.SSS} [%thread] %-5level %logger{36}.%M\(%line\) - %msg%n" />
        </Console>

    </Appenders>
    <Loggers>
        <Logger name="org.apache.hc.client5.http" level="INFO">
            <AppenderRef ref="Console"/>
        </Logger>
        <Logger name="org.apache.hc.client5.http.wire" level="DEBUG">
            <AppenderRef ref="Console"/>
        </Logger>
        <Root level="DEBUG">
            <AppenderRef ref="STDOUT" />
        </Root>
    </Loggers>
</Configuration>
```
官方配置说明 [logger guide](https://hc.apache.org/httpcomponents-client-5.0.x/logging.html)

## 参考
[apache HttpComponents](https://hc.apache.org/httpcomponents-client-5.0.x/quickstart.html)
[httpcomponents-jackson ](https://ok2c.github.io/httpcomponents-jackson/)
