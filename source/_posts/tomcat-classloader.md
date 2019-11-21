---
title: 那些有趣的代码(二)--偏不听父母话的 Tomcat 类加载器
date: 2019-10-27 23:32:30
tags: [java,tomcat,classloader]
---

看 Tomcat 的源码越看越有趣。Tomcat 的代码总有一种处处都有那么一点调皮的感觉。今天就聊一聊 Tomcat 的类加载机制。

了解过 JVM 的类加载一定知道，JVM 类加载的双亲委派机制。但是 Tomcat 却打破了 JVM 固有的双亲委派加载机制。

<!--more-->

## JVM 的类加载

首先需要明确一下类加载是什么？

- Java虚拟机把描述类的数据从Class文件加载到内存，并对数据进行校验、转换解析和初始化，最终形成可以被虚拟机直接使用的Java类型，这就是虚拟机的加载机制。

JVM 预定义的三个加载器:

- **启动类加载器（Bootstrap ClassLoader）**：是用本地代码实现的类装入器，它负责将 `<Java_Runtime_Home>/lib`下面的类库加载到内存中（比如`rt.jar`）。由于引导类加载器涉及到虚拟机本地实现细节，开发者无法直接获取到启动类加载器的引用，所以不允许直接通过引用进行操作。
- **标准扩展类加载器（Extension ClassLoader）**：是由 Sun 的 `ExtClassLoader（sun.misc.Launcher$ExtClassLoader）`实现的。它负责将`< Java_Runtime_Home >/lib/ext`或者由系统变量 `java.ext.dir`指定位置中的类库加载到内存中。开发者可以直接使用标准扩展类加载器。
- **应用程序类加载器（Application ClassLoader）**：是由 Sun 的 `AppClassLoader（sun.misc.Launcher$AppClassLoader）`实现的。它负责将系统类路径（`CLASSPATH`）中指定的类库加载到内存中。开发者可以直接使用系统类加载器。

双亲委派机制：

所谓双亲委派机制，这里要指出的是，其实双亲委派来源于英文的 ”parents delegate“，仅仅表示的只是”父辈“，可见翻译的人不但英文是半吊子，而且也不了解 JVM 的类加载策略，造成了很大的误解。尤其是这个”双“字在初学的时候给我造成了极大的干扰。所以换个说法，应该是”父辈代理“。

类加载的时候，把加载的这个动作递归的委托给父辈，由父辈代劳，只有父辈无法加载时，才会由自己加载。

双亲委派加载模型：

![parents](https://img.xilidou.com/img/parents.jpg)

这里需要特别注意的是加载器的关系并非是继承的关系。我们看代码：

```java
static class ExtClassLoader extends URLClassLoader{
    ... ...
}
static class AppClassLoader extends URLClassLoader{
    ... ...
}
```

二者同时继承了 URLClassLoader ，继承关系如下：

![appClassLoader](https://img.xilidou.com/img/Jietu20191027-235532.jpg)

怎么实现委托机制呢？在 ClassLoader 里面有几处比较重要的代码：

```java
public abstract class ClassLoader {
      // The parent class loader for delegation
      // Note: VM hardcoded the offset of this field, thus all new fields
      // must be added *after* it.
    private final ClassLoader parent;

    protected Class<?> loadClass(String name, boolean resolve)
          throws ClassNotFoundException
    {
        synchronized (getClassLoadingLock(name)) {
            // First, check if the class has already been loaded
            Class<?> c = findLoadedClass(name);
            if (c == null) {
                long t0 = System.nanoTime();
                try {
                    if (parent != null) {
                    // 尝试使用 父辈的 loadClass 方法
                        c = parent.loadClass(name, false);
                    } else {
                    // 如果没有 父辈的 classLoader 就使用 bootstrap classLoader
                        c = findBootstrapClassOrNull(name);
                    }
                } catch (ClassNotFoundException e) {
                    // ClassNotFoundException thrown if class not found
                    // from the non-null parent class loader
                }

                if (c == null) {
                    // If still not found, then invoke findClass in order
                    // to find the class.
                    long t1 = System.nanoTime();
                    // 父辈没法加载这个 class，就自己尝试加载
                    c = findClass(name);
                    // this is the defining class loader; record the stats
                    sun.misc.PerfCounter.getParentDelegationTime().addTime(t1 - t0);
                    sun.misc.PerfCounter.getFindClassTime().addElapsedTimeFrom(t1);
                    sun.misc.PerfCounter.getFindClasses().increment();
                }
            }
            if (resolve) {
                resolveClass(c);
            }
                return c;
        }
    }

    // 根据类名 寻找 class。我们在之前我们讲过，不通过的 classLoader 加载的 class 的位置不同。
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        return defineClass(name, res);
    }

}
```

1. 首先在初始化 ClassLoader 的时候需要指定自己的 parent 是谁？（这很重要）
2. 先检查类有没被加载，如果类已经被加载了，直接返回。
3. 如果没有被加载，则通过 parent 的 loadClass 来尝试加载类。（双亲委派的核心逻辑）
4. 找不到 parent 的时候使用 bootstrap ClassLoader 进行加载。
5. 如果委托的 parent 没法加载类，那就自己加载。

## Tomcat 的类加载

Tomcat 自己实现了自己的类加载器 WebAppClassLoader。类图关系图如下：

![WebAppClassLoader](https://img.xilidou.com/img/Jietu20191027-212824.jpg)

我们就来看看 Tomcat 的类加载器是怎么打破双亲委派的机制的。我们先看代码：

### findClass

```java
    @Override
    public Class<?> findClass(String name) throws ClassNotFoundException {
        // Ask our superclass to locate this class, if possible
        // (throws ClassNotFoundException if it is not found)
        Class<?> clazz = null;

        // 先在自己的 Web 应用目录下查找 class
        clazz = findClassInternal(name);

        // 找不到 在交由父类来处理
        if ((clazz == null) && hasExternalRepositories) {  
            clazz = super.findClass(name);
        }
        if (clazz == null) {
             throw new ClassNotFoundException(name);
        }
        return clazz;
    }
```

对于 Tomcat 的类加载的 findClass 方法:

- 首先在 web 目录下查找。（重要）
- 找不到再交由父类的 findClass 来处理。
- 都找不到，那就抛出 ClassNotFoundException。

### loadClass 方法

```java
    public Class<?> loadClass(String name, boolean resolve) throws ClassNotFoundException {
        synchronized (getClassLoadingLock(name)) {
            Class<?> clazz = null;
            //1. 先在本地cache查找该类是否已经加载过
            clazz = findLoadedClass0(name);
            if (clazz != null) {
                if (resolve)
                    resolveClass(clazz);
                return clazz;
            }
            //2. 从系统类加载器的cache中查找是否加载过
            clazz = findLoadedClass(name);
            if (clazz != null) {
                if (resolve)
                    resolveClass(clazz);
                return clazz;
            }
           // 3. 尝试用ExtClassLoader类加载器类加载
            ClassLoader javaseLoader = getJavaseClassLoader();
            try {
                clazz = javaseLoader.loadClass(name);
                if (clazz != null) {
                    if (resolve)
                        resolveClass(clazz);
                    return clazz;
                }
            } catch (ClassNotFoundException e) {
                // Ignore
            }
            // 4. 尝试在本地目录搜索class并加载
            try {
                clazz = findClass(name);
                if (clazz != null) {
                    if (resolve)
                        resolveClass(clazz);
                    return clazz;
                }
            } catch (ClassNotFoundException e) {
                // Ignore
            }
            // 5. 尝试用系统类加载器(也就是AppClassLoader)来加载
                try {
                    clazz = Class.forName(name, false, parent);
                    if (clazz != null) {
                        if (resolve)
                            resolveClass(clazz);
                        return clazz;
                    }
                } catch (ClassNotFoundException e) {
                    // Ignore
                }
           }
        //6. 上述过程都加载失败，抛出异常
        throw new ClassNotFoundException(name);
    }
```

总结一下加载的步骤：

1. 先在本地cache查找该类是否已经加载过，看看 Tomcat 有没有加载过这个类。
2. 如果Tomcat 没有加载过这个类，则从系统类加载器的cache中查找是否加载过。
3. 如果没有加载过这个类，尝试用ExtClassLoader类加载器类加载，重点来了，这里并没有首先使用 AppClassLoader 来加载类。这个Tomcat 的 WebAPPClassLoader 违背了双亲委派机制，直接使用了 ExtClassLoader来加载类。这里注意 ExtClassLoader 双亲委派依然有效，ExtClassLoader 就会使用 Bootstrap ClassLoader 来对类进行加载，保证了 Jre 里面的核心类不会被重复加载。 比如在 Web 中加载一个 Object 类。WebAppClassLoader → ExtClassLoader → Bootstrap ClassLoader，这个加载链，就保证了 Object 不会被重复加载。
4. 如果 BoostrapClassLoader，没有加载成功，就会调用自己的 findClass 方法由自己来对类进行加载，findClass 加载类的地址是自己本 web 应用下的 class。
5. **加载依然失败，才使用 AppClassLoader 继续加载。**
6. 都没有加载成功的话，抛出异常。

总结一下以上步骤，WebAppClassLoader 加载类的时候，故意打破了JVM 双亲委派机制，绕开了 AppClassLoader，直接先使用 ExtClassLoader 来加载类。

- 保证了基础类不会被同时加载。
- 由保证了在同一个 Tomcat 下不同 web 之间的 class 是相互隔离的。

## more

准备把有趣的代码这个系列慢慢写下去，发现编程的乐趣：

[那些有趣的代码(一)--有点萌的 Tomcat 的线程池](https://xilidou.com/2019/10/15/tomcat-threadpool/)
