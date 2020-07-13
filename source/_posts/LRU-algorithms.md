---
layout:  post
title:  常用缓存算法——LRU
date:  2017-01-01 00:00 +08:00
categories:  [tech]
tags:  [CoreJava,Java,Net,JDK]
---


LRU是一种常用的缓存算法，使用Java LinkedHashMap模拟实现。
<!-- more -->


### LRU算法
Least recently used（LRU）一种缓存算法,
LRU算法及其实现-以LinkedHashMap为例。

### LRU 使用`LinkedHashMap`代码实现
```
package com.github.ordiy.map;

import java.util.LinkedHashMap;
import java.util.Map;
/**
 * 经典的LRU 算法
 *  热点数据缓存
 * @Author:ordiy
 */
public class LRUCache<K,V> extends LinkedHashMap<K,V> {
    private int cacheSize;

    public LRUCache( int cacheSize) {
        super(cacheSize,(float)0.75,true);
        this.cacheSize = cacheSize;
    }
     @Override
    protected boolean removeEldestEntry(Map.Entry<K,V> eldest){
       // System.out.println(eldest);
        return size() > cacheSize;
    }
}
```
- 测试代码
```
    @Test
    public void testKey(){
        LRUCache<String,Integer> cache =  new LRUCache<>(5);
        //初始化数据
        for (int i = 0; i < 5; i++) {
            cache.put("key"+i,i);
        }
        // 模拟最近使用
        String key = "key3";
        cache.get(key);
        System.out.println("after get-->:"+ key);
        printMap(cache);

        //put data and remove Eldest Entry
        cache.put("key10",10);
        System.out.println("after put data and remove Eldest Entry");
        printMap(cache);
    }
```
- 输出结果
```
after get-->:key3
cache order:
key0=0 hashCode: 3288497 , key1=1 hashCode: 3288499 , key2=2 hashCode: 3288497 , key4=4 hashCode: 3288497 , key3=3 hashCode: 3288503 ,
3288497
after put data and remove Eldest Entry
cache order:
key1=1 hashCode: 3288499 , key2=2 hashCode: 3288497 , key4=4 hashCode: 3288497 , key3=3 hashCode: 3288503 , key10=10 hashCode: 101943476 ,
```

- 过程解析
![](https://raw.githubusercontent.com/ordiychen/study_notes/master/res/image/node_image/blog_20200602183106.png)


使用`LinkedHashMap`的记录数据顺序、accessOrder、`removeEldestEntry`特性实现了一个简单的LRU。在分布式场景下，需要使用类似redis,pika等软件进行实现。
**注意 因为`LinkedHashMap`并非线程安全的，多线程场景需要使用`Collections.synchronizedMap`进行包装**

### 深度解析




### 总结


参考
[https://juejin.im/post/5a4b433b6fb9a0451705916f](https://juejin.im/post/5a4b433b6fb9a0451705916f)

LinkedHashMap 源码详细分
https://segmentfault.com/a/1190000012964859
oracle doc LinkedHashMap
https://docs.oracle.com/javase/8/docs/api/java/util/LinkedHashMap.html

Java集合类 LinkedHashMap
https://juejin.im/post/5a4b433b6fb9a0451705916f
