---
title: 记一次排查线上CPU飚高的问题
date: 2018-11-22 22:08:23
tags: java
---
# 记一次排查线上机器CPU飙升的问题

最近我们的报表系统经常受到短信和邮件告警。

由于这个服务是专门用来做报表导出的、每天在跑各种报表任务、做XML解析，很多情况下还是多个任务在跑的，所以之前也没怎么关心，运维也反馈过告警有点频繁。

安排组里相关同学看了一下，也做了一些导出的限制，但是效果好像不是很好。

昨天早上系统疯狂告警、从8点钟开始告警，然后看了一下相关的问题。下面是具体的排查过程

### 查看机器相关状态

在监控系统中查看了一下机器的CPU和内存的状态:

机器整天的CPU占用状态达到了90%、初步判断有相关线程在进行相关耗资源的操作（可能是死循环、频繁IO等）

于是打算到机器上查看当前占用CPU资源的线程。

### 使用Arthas的`thread`命令查看线程占用资源状态

`arthas`是Alibaba开源的Java诊断利器、应该是今年上半年开源的，使用起来非常方便。这里贴出来github地址: [Arthas](https://github.com/alibaba/arthas/blob/master/README_CN.md)

使用`thread` 命令查看当前线程信息、查看线程的堆栈。

我装好arthas之后使用thread命令查看了当前最忙的5个线程并打印堆栈, 发现占用CPU资源最高的是这个线程`pool-9-thread-1`(开发竟然没对线程池进行命名orz...):

![](http://ww1.sinaimg.cn/large/e1417e4bgy1fxh6b6l22mj20we0o8do2.jpg)

这个线程竟然整整占了一个核心的资源（机器配置4C 4G）

同时我们能在打印的堆栈中看到如下信息

```java
- locked java.io.FileNotFoundException

---

Number of locked synchronizers = 1
```

说明当前线程阻塞住了其他线程、应用卡主了，通常是由于线程拿住了某个锁， 并且其他线程都在等待这把锁造成的。

### 定位相关代码、分析问题原因


![](http://ww1.sinaimg.cn/large/e1417e4bgy1fxh6cgg7vej20jh0ckmze.jpg)

找到问题出现的代码在这里：

```java
doc = DocumentHelper.parseText(message.getXml)
```

先不要吐槽代码写的好不好，当时看到这个就有点懵逼了，这只是一句正常的XML解析的逻辑，用了dom4j里面的解析文本的方法，怎么就能够把CPU跑满呢？

带着问题我看了一下里面的具体实现, 里面大体的逻辑是先去获取XMLReader, 即选取XML解析器，然后对XML进行解析。我们看到线程打印的堆栈最终是出现了`FileNotFoundException`, 那为什么会出现这个异常呢?

### JDK设计模式 - Service Provider机制（SPI）

![](http://ww1.sinaimg.cn/large/e1417e4bgy1fxh6cztd18j20cf0bc0sw.jpg)

为什么要说这个呢，因为在上面的选取XML解析器的过程中就用到了这个机制，抱着好奇的心态我debug了一下这个parseText代码, 在选取 XML 解析器的代码中:

![](http://ww1.sinaimg.cn/large/e1417e4bgy1fxh6ef2g5rj20mj054aao.jpg)

这个`FactoryFinder.find()`是寻找XML解析器的逻辑:

![](http://ww1.sinaimg.cn/large/e1417e4bgy1fxh6eugituj20e80a03yu.jpg)

```java

static Object find(String factoryId, String fallbackClassName)
        throws ConfigurationError
    {        
        dPrint("find factoryId =" + factoryId);
        
        // Use the system property first
        try {
            String systemProp = ss.getSystemProperty(factoryId);
            if (systemProp != null) {                
                dPrint("found system property, value=" + systemProp);
                return newInstance(systemProp, null, true);
            }
        } 
        catch (SecurityException se) {
            if (debug) se.printStackTrace();
        }

        // try to read from $java.home/lib/jaxp.properties
        try {
            String factoryClassName = null;
            if (firstTime) {
                synchronized (cacheProps) {
                    if (firstTime) {
                        String configFile = ss.getSystemProperty("java.home") + 
                        File.separator+
                            "lib" + File.separator + "jaxp.properties";
                        File f = new File(configFile);
                        firstTime = false;
                        if (ss.doesFileExist(f)) {
                            dPrint("Read properties file "+f);
                            cacheProps.load(ss.getFileInputStream(f));
                        }
                    }
                }
            }
            factoryClassName = cacheProps.getProperty(factoryId);            

            if (factoryClassName != null) {
                dPrint("found in $java.home/jaxp.properties, value=" + 
                factoryClassName);
                return newInstance(factoryClassName, null, true);
            }
        } 
        catch (Exception ex) {
            if (debug) ex.printStackTrace();
        }

        // Try Jar Service Provider Mechanism
        Object provider = findJarServiceProvider(factoryId);
        if (provider != null) {
            return provider;
        }
        if (fallbackClassName == null) {
            throw new ConfigurationError(
                "Provider for " + factoryId + " cannot be found", null);
        }

        dPrint("loaded from fallback value: " + fallbackClassName);
        return newInstance(fallbackClassName, null, true);
    }

```

总共是下面四步:

第一步： 检查系统属性（systemProperty）是否设置了javax.xml.parsers.SAXParserFactory  属性值，如果设置了，就使用设置的属性值来返回SAXParserFactory 对象。

``` java
String systemProp = ss.getSystemProperty(factoryId);         
if (systemProp != null) {                
       dPrint("found system property, value=" + systemProp);
       return newInstance(systemProp, null, true);
}
```

若没找到，进行第二步

第二步：检查java.home/lib/jaxp.properties 属性配置文件中是否有javax.xml.parsers.SAXParserFactory。

```java
// try to read from $java.home/lib/jaxp.properties
try {
    String factoryClassName = null;
    if (firstTime) {
        synchronized (cacheProps) {
            if (firstTime) {
                String configFile = ss.getSystemProperty("java.home") + File.separator +
                    "lib" + File.separator + "jaxp.properties";
                File f = new File(configFile);
                firstTime = false;
                if (ss.doesFileExist(f)) {
                    dPrint("Read properties file "+f);
                    cacheProps.load(ss.getFileInputStream(f));
                }
            }
        }
    }
    factoryClassName = cacheProps.getProperty(factoryId); 
```

有则返回，无则进入第三步

第三步：    使用jar包的Service Provider机制，加载工厂类。也就是说，查找所有加载的jar包中META-INF/services目录下的配置文件，文件名为 javax.xml.parsers.SAXParserFactory。文件的内容就是该jar包内提供的类实例。比如xercesImple-2.x.x.jar提供的org.apache.xerces.jaxp.SAXParserFactoryImpl.

``` java
// Try Jar Service Provider Mechanism
Object provider = findJarServiceProvider(factoryId);
if (provider != null) {
     return provider;
}
```

如果找不到，则进入最后第四步，使用默认的配置
第四步：使用系统默认的工厂类：

com.sun.org.apache.xerces.internal.jaxp.SAXParserFactoryImpl。

```java
if (fallbackClassName == null) {
    throw new ConfigurationError(
        "Provider for " + factoryId + " cannot be found", null);
}

dPrint("loaded from fallback value: " + fallbackClassName);
return newInstance(fallbackClassName, null, true);
```

在没有进行任何配置以及使用第三方jar包的情况下，最终程序都会走到第四步，返回系统默认的工厂类。

### 得出结论

> 刚好我们的代码就是走到了这里面的逻辑, 寻找XML解析器里面的逻辑上面四步都走了，然后返回默认的工厂类, 且每次都会去通过`SPI`到加载的jar包中寻找文件, 找不到便会 lock 住FileNotFoundException, 执行一次这个代码可能还好, 但是这段导报表的逻辑是需要循环差不多9w次！！！

![](http://ww1.sinaimg.cn/large/e1417e4bgy1fxh6fgbwomj212801i74w.jpg)


9w次不断的遍历jar包寻找文件，然后不断的抛出 FileNotFoundException, 怪不得会把CPU飙升到90%。

其实我也在想为什么遍历完一次之后这边的解析逻辑中为什么不会把解析后的逻辑缓存一份，下次不用再去寻找这个文件，有点困惑、、、

### 提出解决办法、进行尝试

找到问题之后问题就比较好解决了，主要是因为去使用SPI机制寻找jar文件的过程会比较消耗CPU，
我们只要不让他去寻找，早先设置好默认的工厂类就好了。

可选择的方法应该有如下四种:

1.在运行java时，通过设置系统属性
``` java
java -D javax.xml.parsers.SAXParserFactory=new com.sun.org.apache.xerces.internal.jaxp.SAXParserFactoryImpl
```

2.方法二:在程序中调用
``` java
System.setProperty(“javax.xml.parsers.SAXParserFactory”,” com.sun.org.apache.xerces.internal.jaxp.SAXParserFactoryImpl”)
```
  来设定实际的XML解析器.

3.方法三:编写一个jaxp.properties文件，在其中加入如下内容

```java
javax.xml.parsers.SAXParserFactory=com.sun.org.apache.xerces.internal.jaxp.SAXParserFactoryImpl
```
  再将此文件放入JAVA_HOME/lib/下

 4.方法四:在打得jar包下，在目录META-INF/下新建一个services目录，在此目录新建一个文件名为
 
`javax.xml.parsers.SAXParserFactory`

的文件，文件内容写上实际使用的解析器类，如写

`com.sun.org.apache.xerces.internal.jaxp.SAXParserFactoryImpl`

，这样在加载jar包之后，就会找到这个xml解析工厂。

最终我这边为了方便采取了第二种方式:

![](http://ww1.sinaimg.cn/large/e1417e4bgy1fxh6dhoowcj20np0bljsx.jpg)

### 上线验证结果

![](http://ww1.sinaimg.cn/large/e1417e4bgy1fxh6guj8cmj20rs086aas.jpg)


上面是上线后第二天观察到的机器load情况，可以看到有很明显的改善！

### 总结

通过这次排查问题，我对JAXP有一个比较深的了解，其实，JAVA中有许多这种思想（Service Provider）的做法，这种思想，，就是平台无关性，发展到不依赖于具体的实现。

如我们熟悉的JNDI，JDBC，JAXP等。JNDI是抽像各种目录服务操作的类库，因为目录服务器厂商太多了，如SUN公司的ldapsdk, 还有novell公司等等。JDBC是抽像各种数据库操作的类库，因为数据库厂商也太多了，如ORACLE，SQLSERVER，MYSQL，INFORMIX等等。JAXP就是抽像各种XML解析器和转换器产品的类库，因为XML解析器和转换器产品够多的了。sun公司自己实现的HttpServer也是基于Service Provide机制。
