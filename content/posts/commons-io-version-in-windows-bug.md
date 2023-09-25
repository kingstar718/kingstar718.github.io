---
author: "kingstar718"
title: 'commons-io版本变动在windows环境下引发的NTFS ADS separator问题'
date: 2023-09-22T03:33:54Z
tags: ["java","bug"]
draft: false
---
## 起因

因业务需求，项目中引入了一个对方的业务jar包，但是发现代码却启动不起来了，报错：

```
org.springframework.beans.PropertyBatchUpdateException; nested PropertyAccessExceptions (1) are:
PropertyAccessException 1: org.springframework.beans.MethodInvocationException: Property 'locations' threw exception; nested exception is java.lang.IllegalArgumentException: NTFS ADS separator (':') in file name is forbidden.
```

尽管本地环境（windows）启动不起来，但是打包后放入Linux容器中是正常运行的。

随即进入debug分析，定位异常位置，发现是在使用 `disconf`的jar中会调用到 `commons-io`的 `FilenameUtils`的 `getExtension`方法，来获取一个文件的后缀，具体调用如下：

```java
String extension = FilenameUtils.getExtension(fileName);
```

比如 `fileName`为log4j2.xml，方法返回的就是 `xml`。

由于业务方引入的jar包中包含的commons-io的2.8.0的新包，IDEA启动时默认会使用该包，实际上原项目commons-io使用的是2.5版本。

## 分析

disconf使用的是 `org.apache.commons.io`下的 `FilenameUtils`，我们先看下2.5版本的实现：

```java
public static String getExtension(final String filename) {  
    if (filename == null) {  
        return null;  
    }  
    final int index = indexOfExtension(filename);  
    if (index == NOT_FOUND) {  
        return "";  
    } else {  
        return filename.substring(index + 1);  
    }  
}
```

关注下indexOfExtension的实现：

```java
public static int indexOfExtension(final String filename) {  
    if (filename == null) {  
        return NOT_FOUND;  
    }  
    final int extensionPos = filename.lastIndexOf(EXTENSION_SEPARATOR);  
    final int lastSeparator = indexOfLastSeparator(filename);  
    return lastSeparator > extensionPos ? NOT_FOUND : extensionPos;  
}
```

而2.8.0的commons-io的实现如下：

```java
public static String getExtension(final String fileName) throws IllegalArgumentException {  
    if (fileName == null) {  
        return null;  
    }  
    final int index = indexOfExtension(fileName);  
    if (index == NOT_FOUND) {  
        return EMPTY_STRING;  
    }  
    return fileName.substring(index + 1);  
}
```

```java
public static int indexOfExtension(final String fileName) throws IllegalArgumentException {  
    if (fileName == null) {  
        return NOT_FOUND;  
    }  
    if (isSystemWindows()) {  
        // Special handling for NTFS ADS: Don't accept colon in the fileName.  
        final int offset = fileName.indexOf(':', getAdsCriticalOffset(fileName));  
        if (offset != -1) {  
            throw new IllegalArgumentException("NTFS ADS separator (':') in file name is forbidden.");  
        }  
    }  
    final int extensionPos = fileName.lastIndexOf(EXTENSION_SEPARATOR);  
    final int lastSeparator = indexOfLastSeparator(fileName);  
    return lastSeparator > extensionPos ? NOT_FOUND : extensionPos;  
}
```

问题就出在这里，方法多了一个 `IllegalArgumentException`异常，并且是特定在windows环境下才抛出 `NTFS ADS separator (':') in file name is forbidden.`，这也就解释了为什么服务打包后在Linux环境下没有问题。

该方法的注释上也写了为什么会做这个改变：

> `<b>`Note:`</b>` This method used to have a hidden problem for names like "foo.exe:bar.txt".  In this case, the name wouldn't be the name of a file, but the identifier of an  alternate data stream (bar.txt) on the file foo.exe. The method used to return  ".txt" here, which would be misleading. Commons IO 2.7, and later versions, are throwing  an {@link IllegalArgumentException} for names like this.

翻译过来就是，在windows上类似 `foo.exe:bar.txt`的文件，`bar.txt`不是文件的名称，而是文件 `foo.exe`上的备用数据流 `bar.txt`的标识符。

> NTFS中的备用数据流（Alternate Data Stream，ADS）允许将一些元数据嵌入文件或是目录，而不需要修改其原始功能或内容。

这个改动刚好是 `commons-io`的2.7版本，命中了项目因为引入业务jar包导致的2.5->2.8.0的升级迭代。

## 解决

后续决定还是在业务jar中移除commons-io的2.8.0版本，沿用之前项目中的2.5版本。

## 总结

项目使用 `disconf`来监听配置文件的变更，实际上传入的是类似 `classpath:log4j2.xml`这样的fileName，导致的问题。

项目初步启动时，直接报错的堆栈中并没有关于 `commons-io`的堆栈信息，只知道是 `disconf`的bean无法启动，可以先从这个bean入手，逐步定位到是 `com.baidu.disconf.client.addons.properties.ReloadablePropertiesFactoryBean`成员变量 `locations`的问题，进而从 `setLocations`这个方法定位到使用了 `FilenameUtils`中的 `getExtension`，从而得知是哪里的异常。

进而直接使用 `Dependency Analyzer`，看到业务jar包引入了更新版本的 `commons-io`，并通过对比两个版本的 `FilenameUtils`的 `getExtension`方法的区别，了解本次异常的原因。

这是一次较为常见的组件变动引起的启动异常，但是实际上问题的表征并不能直接定位原因，合理利用debug，在项目启动时，添加断点，逐步分析，是开发者需要掌握的技能。

这个问题直接搜索是不太能得到答案的，但是分析问题的过程，是初学者逐步向熟练者进阶的必备技能，特此记录，以作参考。

## 参考

1. [NTFS ADS（备用数据流）](https://www.cnblogs.com/zUotTe0/p/13455971.html)
