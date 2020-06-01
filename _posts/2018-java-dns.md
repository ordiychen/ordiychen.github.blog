---
layout:  post
title:  Java DNS解析问题-以Socket Client为例
date:  2017-01-01 00:00 +08:00
categories:  [tech]
tags:  [CoreJava,Java,Net,JDK]
---

* content
{:toc}

## 问题描述
域名映射关系
```text
127.0.0.1 test.lan-gl-42.dev
```
java应用程序
```
-Dsun.net.inetaddr.ttl=15
```
Java Socket Client Code:
```java
 @Test
    public void testSocket() {
        String host = "test.lan-gl-42.dev";
        int port = 9876;
        try (Socket socket = new Socket(host, port);
             ObjectOutputStream oos = new ObjectOutputStream(socket.getOutputStream())) {
            //write to socket using ObjectOutputStream
            System.out.println("Sending request to Socket Server,socket remote:" + socket.getRemoteSocketAddress().toString());
            while (true){
                Thread.sleep(10000);
                System.out.println("Sending request to Socket Server,socket remote:" + socket.getRemoteSocketAddress().toString());
            }
            //Thread.sleep(10000);
        } catch (IOException | InterruptedException  e) {
            e.printStackTrace();
        }catch (Exception e){
            e.printStackTrace();
        }
    }
 ```
 执行日志：
 ```
 Sending request to Socket Server,socket remote:test.lan-gl-42.dev/127.0.0.1:9876
Sending request to Socket Server,socket remote:test.lan-gl-42.dev/127.0.0.1:9876
Sending request to Socket Server,socket remote:test.lan-gl-42.dev/127.0.0.1:9876
```
修改域名的映射关系为：
```text
10.224.17.250 test.lan-gl-42.dev
```
** socket连接的地址依然是 127.0.0.1 **



### 原因
Java进程及OS DNS解析过程：
![](https://raw.githubusercontent.com/ordiychen/study_notes/master/res/image/node_image/blog_20200601180658.png)
[图片来源](https://medium.com/@maheshsenni/host-name-resolution-in-java-80301fea465a)

![](https://raw.githubusercontent.com/ordiychen/study_notes/master/res/image/node_image/blog_20200601172248.png)
因为`InetSocketAddress`被设计了为一个不可变的对象（immutable object ）,`Socket` 对象实例的InetSocketAddress 对象，只在实例化对象时解析hostName与hostAddress的映射关系。



### 修复方法
可以借鉴 zookeeper 的 [`StaticHostProvider.java`](https://github.com/apache/zookeeper/blob/master/zookeeper-server/src/main/java/org/apache/zookeeper/client/StaticHostProvider.java)
调用`next()`方法在解析一次域名与IP地址的映射关系。
```
public InetSocketAddress next(long spinDelay) {
        boolean needToSleep = false;
        InetSocketAddress addr;
        synchronized(this) {
            if (this.reconfigMode) {
                addr = this.nextHostInReconfigMode();
                if (addr != null) {
                    this.currentIndex = this.serverAddresses.indexOf(addr);
                    return this.resolve(addr);
                }

                this.reconfigMode = false;
                needToSleep = spinDelay > 0L;
            }

            ++this.currentIndex;
            if (this.currentIndex == this.serverAddresses.size()) {
                this.currentIndex = 0;
            }

            addr = (InetSocketAddress)this.serverAddresses.get(this.currentIndex);
            needToSleep = needToSleep || this.currentIndex == this.lastIndex && spinDelay > 0L;
            if (this.lastIndex == -1) {
                this.lastIndex = 0;
            }
        }

        if (needToSleep) {
            try {
                Thread.sleep(spinDelay);
            } catch (InterruptedException var7) {
                LOG.warn("Unexpected exception", var7);
            }
        }

        return this.resolve(addr);
    }

private InetSocketAddress resolve(InetSocketAddress address) {
        try {
            String curHostString = address.getHostString();
            List<InetAddress> resolvedAddresses = new ArrayList(Arrays.asList(this.resolver.getAllByName(curHostString)));
            if (resolvedAddresses.isEmpty()) {
                return address;
            } else {
                Collections.shuffle(resolvedAddresses);
                return new InetSocketAddress((InetAddress)resolvedAddresses.get(0), address.getPort());
            }
        } catch (UnknownHostException var4) {
            LOG.error("Unable to resolve address: {}", address.toString(), var4);
            return address;
        }
    }
```

