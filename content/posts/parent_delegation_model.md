+++
date = '2025-06-19T16:01:25+08:00'
draft = false
title = '双亲委派机制'
tags = ['java基础']
categories = ['技术']
+++
## 双亲委派机制
### 什么是双亲委派机制
双亲委派机制是java一种加载类的机制，流程如下：
1. 检查本地缓存，避免重复加载
2. 是否有父加载器，如果有则委托给父加载器
3. 父加载器重复1，2步骤
4. 如果父加载器无法加载，则子加载器尝试加载

### java中的类加载器层级
- BootstrapClassLoader 启动类加载器，加载核心类如String类等
- ExtensionClassLoader 扩展类加载器，加载`jre/lib/ext`包下的类
- Application ClassLoader 应用类加载器，加载项目中自定义的类
- Custom ClassLoader 自定义类加载器

### 双亲委派机制的好处
1. 避免重复加载类
2. 避免用户自定义覆盖核心类，如避免用户自己实现的`java.lang.String`类

### 如何打破双亲委派机制
1. 继承ClassLoader，重写loadClass方法，完全打破双亲委派
2. 重写findClass方法，仍然会尝试先由父类加载
3. 通过`Thread Context ClassLoader`获取类加载器进行加载，通常与SPI联合使用
```java
// java.sql.DriverManager
public class DriverManager {
    private static void loadInitialDrivers() {
        // 通过SPI查找所有Driver实现
        ServiceLoader<Driver> loadedDrivers = ServiceLoader.load(Driver.class);
        // 遍历加载驱动
        Iterator<Driver> driversIterator = loadedDrivers.iterator();
        // ...
    }
    
    // ServiceLoader.load()的核心实现
    public static <S> ServiceLoader<S> load(Class<S> service) {
        ClassLoader cl = Thread.currentThread().getContextClassLoader();
        return load(service, cl);
    }
}
```
一般热部署以及SPI等都会要求打破双亲委派，因为SPI定义的接口使用启动类加载器，而用户自定义的实现没办法用启动类加载器加载，会导致找不到实现类，所以就需要用TCCL以及SPI来打破双亲委派机制。