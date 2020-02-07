---
title: Slf4j源码分析
date: 2020-02-06 16:53:45
categories: 源码分析
tags: [SLF4J, 源码分析]
---
众所周知，SLF4J是非常流行的Java日志规范框架，使用门面模式来抽象日志接口，未提供具体的接口实现，通常需要配合log4j、logback等日志框架来使用。本文主要是为了研究SLF4J的源码，来剖析SLF4J的实现原理。

文章以以下方式组织：

1. SLF4J使用样例，结合logback来介绍一些常见用法；
2. 探究SLF4J抽象出来的几个主要接口及其实现原理；
3. 结合SLF4J抽象出来的接口，实现一个简单的日志框架（这里以`slf4j-simple`为例）；
4. 对于log4j框架，SLF4J需要一个适配层来使用log4j，探究该适配层如何实现。

## SLF4J常见用法
在SLF4J中，将日志分为5个等级：`error`，`warn`，`info`，`debug`，`trace`。这里使用SLF4J和logback来介绍常见用法。[项目源码](https://github.com/BitInit/learning/slf4j-test)，Maven依赖如下：

``` java
<dependencies>
  <dependency>
    <groupId>org.slf4j</groupId>
    <artifactId>slf4j-api</artifactId>
    <version>1.7.25</version>
  </dependency>
  <dependency>
    <groupId>ch.qos.logback</groupId>
    <artifactId>logback-classic</artifactId>
    <version>1.2.3</version>
  </dependency>
</dependencies>
```
### 日志打印
首先，来看一个简单的Hello World代码：

``` java
public static void main(String[] args) {
    Logger logger = LoggerFactory.getLogger("test");
    logger.info("Hello World");
}

// 结果
18:12:06.200 [main] INFO test - Hello World
```
SLF4J通过扫描classpath寻找SLF4J的实现（在1.7.25版本中是通过类查找机制，在2.0.0版本中使用的是SPI机制，下文分析源码时会详细说明），如果在Maven中没有添加`logback-classic`等任何实现依赖，这时，应用会打印：

```
SLF4J: Failed to load class "org.slf4j.impl.StaticLoggerBinder".
SLF4J: Defaulting to no-operation (NOP) logger implementation
SLF4J: See http://www.slf4j.org/codes.html#StaticLoggerBinder for further details.
```

可以通过`{}`占位符来打印日志：

``` java
public static void main(String[] args) {
    Logger logger = LoggerFactory.getLogger("test");
    String name = "john";
    int age = 20;
    logger.info("User name:{} age:{}", name, age);
}

// 结果
18:18:45.978 [main] INFO test - User name:john age:20
```

在**SLF4J 2.0版本**（现在2.0还是Snapshot）中，还支持通过编码来指定参数：

``` java
public static void main(String[] args) {
        Logger logger = LoggerFactory.getLogger("test");
        String name = "john";
        int age = 20;
        logger.atInfo().log("User name:{} age:{}", name, age);
        logger.atInfo().addArgument(name).addArgument(age).log("User name:{} age:{}");
        logger.atInfo().addArgument(name).log("User name:{} age:{}", age);
        logger.atInfo().addArgument(() -> name).log("User name:{} age:{}", age);
}
    
// 结果
[main] INFO test - User name:john age:20
[main] INFO test - User name:john age:20
[main] INFO test - User name:john age:20
[main] INFO test - User name:john age:20
```

### Marker
Marker中文翻译“标记”、“记号”。在SLF4J中，可以理解成对某条或某些条日志添加一个记号，就相当于我们对每个博客或物品添加一些标签。

``` java
public static void main(String[] args) {
    Logger logger = LoggerFactory.getLogger("test");

    Marker notifyMarker = MarkerFactory.getMarker("NOTIFY_ADMIN");
    logger.error(notifyMarker, "notify admin {}", new IllegalStateException("system error"));
}

// 结果
18:52:47.010 [main] ERROR test - notify admin {}
java.lang.IllegalStateException: system error
	at site.bitinit.slf4j.MarkerTests.main(MarkerTests.java:18)
```
根据[stackoverflow best-practices-for-using-markers-in-slf4j-logback](https://stackoverflow.com/questions/4165558/best-practices-for-using-markers-in-slf4j-logback/10231016#10231016)，Marker有两种作用：

1. trigger：在[http://logback.qos.ch/manual/appenders.html#OnMarkerEvaluator](http://logback.qos.ch/manual/appenders.html#OnMarkerEvaluator)中详细描述了一个应用场景，当系统中出现某些日志，例如严重的系统错误时，系统应该向管理员报警。配置一个发送报警email的appender，当出现某个日志含有上例中notifyMarker时，就通过Email Appender向管理员发送一封邮件。
2. filter：根据marker筛选日志。例如我只想在系统的console上打印含有上例中notifyMarker的日志，而其他日志我输出到文件中。

### MDC
MDC 全称 Mapped Diagnostic Context，映射调试上下文。logback官网对MDC有详细的使用说明，这里就不过多说明了。[http://logback.qos.ch/manual/mdc.html](http://logback.qos.ch/manual/mdc.html)

## SLF4J源码解析
本节对SLF4J源码解析是基于SLF4J 1.7.25版本，源码下载地址 [github slf4j-1.7.25](https://github.com/qos-ch/slf4j/archive/v_1.7.25.tar.gz)。

![slf4j 源码结构图](http://image.55555.io/blog_slf4j_structure.png)

上面SLF4J的源代码结构中有几个最重要的模块：`slf4j-api` 是 SLF4J 的核心模块，所有的日志抽象接口都放在这里面；`slf4j-simple` 是一种 SLF4J 日志规范的实现，类似于 logback、log4j； `slf4j-log4j12` 和 `slf4j-jdk14` 是通过适配器的方式使得 log4j 和 JDK logger 满足 SLF4J 日志规范，而 logback 是天生支持 SLF4J 规范，所以不再需要适配器。

本节主要分析 `slf4j-api` 里的源码。

![slf4j-api](http://image.55555.io/blog_slf4j_api_structure.png)

这里先提一下，包`org.slf4j.impl`下的类在 slf4j-api 打包成 jar 过程中会被过滤掉，具体原因会在下面说明。

在 slf4j-api 中，最主要的几个抽象接口便是 `Logger.class`、`ILoggerFactory.class`、`Marker.class`、`IMarkerFactory.class`。

首先看看 `Logger`和`Marker`抽象接口方法：

``` java
public interface Logger {
	String ROOT_LOGGER_NAME = "ROOT";
	String getName();
	
	boolean isTraceEnabled();
	void trace(String msg);
	void trace(String format, Object arg);
	void trace(String format, Object arg1, Object arg2);
	void trace(String format, Object... arguments);
	void trace(String msg, Throwable t);
	boolean isTraceEnabled(Marker marker);
	void trace(Marker marker, String msg);
	void trace(Marker marker, String format, Object arg);
	void trace(Marker marker, String format, Object arg1, Object arg2);
	void trace(Marker marker, String format, Object... argArray);
	void trace(Marker marker, String msg, Throwable t);
	
	// 下面还有 debug、info、warn、error 相关方法，其方法定义形式和 trace 一样
}

public interface Marker {
	String ANY_MARKER = "*";
	String ANY_NON_NULL_MARKER = "+";
	String getName();
	
	void add(Marker reference);
	boolean remove(Marker reference);
    // 方法过时，被 hasReferences() 取代
	boolean hasChildren();
	boolean hasReferences();
	Iterator<Marker> iterator();
	boolean contains(Marker other);
	boolean contains(String name);
	boolean equals(Object o);
	int hashCode();
}
```

对于 `Logger`和`Marker`都定义了一个工厂接口，用于创建`Logger`和`Marker`。

``` java
public interface ILoggerFactory {
	Logger getLogger(String name);
}

public interface IMarkerFactory {
	Marker getMarker(String name);
	boolean exists(String name);
	boolean detachMarker(String name);
	Marker getDetachedMarker(String name);
}
```

`Logger`与`ILoggerFactory`、`Marker`与`IMarkerFactory`，这是典型的工厂方法设计模式的应用。如果用户要基于 slf4j 日志规范自己开发一个 logger，就需要实现 `Logger`和`ILoggerFactory`接口。

> 在进行面向对象编程的时候，我们并不是一上来就去思考，如何将复杂的流程拆解为一个一个方法，而是采用曲线救国的策略，先去思考如何给业务建模，如何将需求翻译为类，如何给类之间建立交互关系，而完成这些工作完全不需要考虑错综复杂的处理流程。当我们有了类的设计之后，然后再像搭积木一样，按照处理流程，将类组装起来形成整个程序。这种开发模式、思考问题的方式，能让我们在应对复杂程序开发的时候，思路更加清晰。

现在我们已经有了基本抽象接口，接下来就需要探究如何把这些抽象的接口和用户自定义接口实现按照某种规则组织起来，并且向 slf4j 使用者提供一个简单、清晰的 API。

我们都知道，在使用 slf4j 时，我们是这样 `Logger logger = LoggerFactory.getLogger(HelloWorld.class)` 来获取一个 Logger 的，这儿 `LoggerFactory` 其实就是一个处理流程组装类，并为 slf4j 用户提供一个简单、清晰的API。它自动查找应用程序 classpath 中 `Logger` 和 `ILoggerFactory` 的实现，并初始化一个 `ILoggerFactory` 对象。

接下来就来解析 `LoggerFactory.getLogger(String name)`的实现：

``` java
public static Logger getLogger(String name) {
    ILoggerFactory iLoggerFactory = getILoggerFactory();
    return iLoggerFactory.getLogger(name);
}
```
通过调用 `getILoggerFactory()` 方法得到 `ILoggerFactory` 对象，`ILoggerFactory` 对象来获取 `Logger`。

``` java
public static ILoggerFactory getILoggerFactory() {
    if (INITIALIZATION_STATE == UNINITIALIZED) {
        synchronized (LoggerFactory.class) {
            if (INITIALIZATION_STATE == UNINITIALIZED) {
                INITIALIZATION_STATE = ONGOING_INITIALIZATION;
                performInitialization(); // 1
            }
        }
    }
    switch (INITIALIZATION_STATE) {
    case SUCCESSFUL_INITIALIZATION:
        return StaticLoggerBinder.getSingleton().getLoggerFactory(); // 2
    case NOP_FALLBACK_INITIALIZATION:
        return NOP_FALLBACK_FACTORY;
    case FAILED_INITIALIZATION:
        throw new IllegalStateException(UNSUCCESSFUL_INIT_MSG);
    case ONGOING_INITIALIZATION:
        // support re-entrant behavior.
        // See also http://jira.qos.ch/browse/SLF4J-97
        return SUBST_FACTORY;
    }
    throw new IllegalStateException("Unreachable code");
}
```

这儿有2个重点(上面代码注释为1、2处)，1处是为了加载并初始化全局唯一的 `StaticLoggerBinder` 对象(使用的是单例模式中的饿汉模式)，`StaticLoggerBinder` 可以理解成是对 `ILoggerFactory` 的包装，`performInitialization()` 方法里面会调用 `build()` 来构建 `StaticLoggerBinder`。

``` java
private final static void performInitialization() {
    bind();
    if (INITIALIZATION_STATE == SUCCESSFUL_INITIALIZATION) {
        versionSanityCheck();
    }
}

private final static void bind() {
     // 删除了异常处理等非核心代码
    Set<URL> staticLoggerBinderPathSet = null;
    // skip check under android, see also
    // http://jira.qos.ch/browse/SLF4J-328
    if (!isAndroid()) {
        // 在classpath中查找 org/slf4j/impl/StaticLoggerBinder.class
        staticLoggerBinderPathSet = findPossibleStaticLoggerBinderPathSet();
        reportMultipleBindingAmbiguity(staticLoggerBinderPathSet);
    }
    // the next line does the binding
    StaticLoggerBinder.getSingleton();
    INITIALIZATION_STATE = SUCCESSFUL_INITIALIZATION;
    reportActualBinding(staticLoggerBinderPathSet);
    fixSubstituteLoggers();
    replayEvents();
    // release all resources in SUBST_FACTORY
    SUBST_FACTORY.clear();
}
```

在具体的 slf4j 日志规范的实现框架中(logback、slf4j-simple、slf4j-log4j)都会有 `org/slf4j/impl/StaticLoggerBinder.class` 类，该类一般会实现 `org.slf4j.ILoggerFactory.LoggerFactoryBinder` 接口：

``` java
public interface LoggerFactoryBinder {
    ILoggerFactory getLoggerFactory();
    String getLoggerFactoryClassStr();
}
```

在进行类加载时，应用就会加载 logback 等框架里面的 `org/slf4j/impl/StaticLoggerBinder.class` 类，并且该类必须有一个 `public static final StaticLoggerBinder getSingleton()` 方法。在 slf4j-api 模块 `org.slf4j.impl` 包下也有一个 `StaticLoggerBinder` 类，该类主要是为了 slf4j-api 模块开发的便利，这也是为什么上文提到的把 slf4j-api 打包成 jar 包时会过滤掉该包下所有的类。在 slf4j 2.0.0版本中，是通过 SPI 机制来实现，个人认为 SPI 实现机制要比 `StaticLoggerBinder.class` 优雅很多。

在 `build()` 方法中调用 `StaticLoggerBinder.getSingleton()` 就完成了 LoggerFactory 的绑定。根据 `StaticLoggerBinder` 就能得到 `ILoggerFactory` 单例对象。

`Marker` 的实现原理和 `Logger` 类似，这里就不过多赘述。下一节会通过具体的例子(slf4j-simple)来探究SLF4J的设计思想。

## 实现一个简易的SLF4J规范日志框架
本节以 slf4j-simple 为原型来讨论的，并且只讨论 Logger 的实现，Marker 和 MDC 暂不讨论。

如果要实现一个 slf4j 日志规范框架，首先需要对 `Logger` 接口和它的工厂 `ILoggerFactory` 接口实现。在 slf4j-simple 中，对应的实现类是 `SimpleLogger` 和 `SimpleLoggerFactory`。

![SimpleLogger](http://image.55555.io/blog_slf4j_SimpleLogger.png)

`MarkerIgnoringBase` 和 `NamedLoggerBase` 是 `org.slf4j.helpers` 包下的一些辅助类。`MarkerIgnoringBase` 为了方便子类忽略 Logger 中含有 Marker 的这些方法，`MarkerIgnoringBase` 中某个 trace 方法：

``` java
public void trace(Marker marker, String format, Object arg) {
    trace(format, arg);
}
```

`SimpleLoggerFactory` 是 `ILoggerFactory` 的实现：

``` java
public class SimpleLoggerFactory implements ILoggerFactory {
    // 存储所有的 Logger
    ConcurrentMap<String, Logger> loggerMap;

    public SimpleLoggerFactory() {
        loggerMap = new ConcurrentHashMap<String, Logger>();
        // SimpleLogger 中提供了一个静态初始化方法，为了初始化系统参数
        SimpleLogger.lazyInit();
    }

    /**
     * Return an appropriate {@link SimpleLogger} instance by name.
     */
    public Logger getLogger(String name) {
        Logger simpleLogger = loggerMap.get(name);
        if (simpleLogger != null) {
            return simpleLogger;
        } else {
            Logger newInstance = new SimpleLogger(name);
            Logger oldInstance = loggerMap.putIfAbsent(name, newInstance);
            return oldInstance == null ? newInstance : oldInstance;
        }
    }

    /**
     * Clear the internal logger cache.
     *
     * This method is intended to be called by classes (in the same package) for
     * testing purposes. This method is internal. It can be modified, renamed or
     * removed at any time without notice.
     *
     * You are strongly discouraged from calling this method in production code.
     */
    void reset() {
        loggerMap.clear();
    }
}
```
现在有了 `SimpleLogger` 和 `SimpleLoggerFactory` 类了，如何把这两个类组织起来为 slf4j 服务呢？这时就要用到上节提到的 `org.slf4j.impl.StaticLoggerBinder` 类了：

``` java
public class StaticLoggerBinder implements LoggerFactoryBinder {
    // 利用饿汉模式创建一个 StaticLoggerBinder 对象
    private static final StaticLoggerBinder SINGLETON = new StaticLoggerBinder();

    public static final StaticLoggerBinder getSingleton() {
        return SINGLETON;
    }

    /**
     * Declare the version of the SLF4J API this implementation is compiled against. 
     * The value of this field is modified with each major release. 
     */
    // to avoid constant folding by the compiler, this field must *not* be final
    public static String REQUESTED_API_VERSION = "1.6.99"; // !final

    private static final String loggerFactoryClassStr = SimpleLoggerFactory.class.getName();

    private final ILoggerFactory loggerFactory;
    // 创建一个 LoggerFactory，该 LoggerFactory 的生命周期和 StaticLoggerBinder 单例对象一致
    private StaticLoggerBinder() {
        loggerFactory = new SimpleLoggerFactory();
    }

    public ILoggerFactory getLoggerFactory() {
        return loggerFactory;
    }

    public String getLoggerFactoryClassStr() {
        return loggerFactoryClassStr;
    }
}
```

当应用调用 `LoggerFactory.getLogger(HelloWorld.class)` 时，slf4j-api 会找到 slf4j-simple 里面的 `org.slf4j.impl.StaticLoggerBinder` 类，该 `StaticLoggerBinder` 类会实例化一个 `SimpleLoggerFactory` 对象，通过该工厂对象便可以得到或创建一个 `Logger` 对象。

到现在为止，基本上已经完成了 slf4j 规范日志的实现，`Logger` 里面的方法，例如 `info(String msg)` 等就不讲述了。

## SLF4J适配层
在 [slf4j 用户手册中](http://www.slf4j.org/manual.html) 有这么一张图：

![slf4j adapter](http://image.55555.io/blog_slf4j_adapter.png)

如果要使用 log4j 和 java.util.logging 作为 slf4j 的实现，就需要一个适配层，在 slf4j 的项目中，`slf4j-log4j12` 和 `slf4j-jdk14` 就是该适配层。其实该适配层的核心思想就是设计模式中的适配器模式。本节以 `slf4j-log4j12` 模块来说明。

![slf4j-log4j12](http://image.55555.io/blog_slf4j_log4j_adapter.png)

其代码和 `slf4j-simle` 类似，对于 Logger 而言，核心类就是 `Log4jLoggerAdapter`、`Log4jLoggerFactory` 和 `StaticLoggerBinder` 三个。`Log4jLoggerFactory`、`StaticLoggerBinder` 和上节说的 `SimpleLoggerFactory`、`StaticLoggerBinder`基本一样。而 `Log4jLoggerAdapter` 就是一个适配器：

``` java
public final class Log4jLoggerAdapter extends MarkerIgnoringBase implements LocationAwareLogger, Serializable {
	private static final long serialVersionUID = 6182834493563598289L;

    final transient org.apache.log4j.Logger logger;

    /**
     * Following the pattern discussed in pages 162 through 168 of "The complete
     * log4j manual".
     */
    final static String FQCN = Log4jLoggerAdapter.class.getName();

    // Does the log4j version in use recognize the TRACE level?
    // The trace level was introduced in log4j 1.2.12.
    final boolean traceCapable;
    
    Log4jLoggerAdapter(org.apache.log4j.Logger logger) {
        this.logger = logger;
        this.name = logger.getName();
        traceCapable = isTraceCapable();
    }

    private boolean isTraceCapable() {
        try {
            logger.isTraceEnabled();
            return true;
        } catch (NoSuchMethodError e) {
            return false;
        }
    }
    
    public boolean isTraceEnabled() {
        if (traceCapable) {
            return logger.isTraceEnabled();
        } else {
            return logger.isDebugEnabled();
        }
    }
    
    public void trace(String msg) {
        logger.log(FQCN, traceCapable ? Level.TRACE : Level.DEBUG, msg, null);
    }
    
    // 下面就是 Logger 中 trace/debug/info/warn/error 方法
}
```

该 `Log4jLoggerAdapter` 就是使用**组合 + 委托**方式来适配 `Logger` 接口。

## 总结
最近突发奇想看看 SLF4J 实现原理，整个思想体系不算复杂，从顶层看，SLF4J就是一种典型门面模式的应用，为了扩展性，定义了几个抽象接口，灵活应用工厂方法模式来实例化 `Logger`、`Marker`对象。`LoggerFactory` 需要获取 `ILoggerFactory` 的实现，在 SLF4J 1.7.25版本中使用的是 `StaticLoggerBinder` 类来初始化，个人感觉这种方式并不优雅，好在在 SLF4J 2.0.0版本中采用的是 SPI 机制，只需要通过配置文件来指导接口的实现类，这样不再需要维护一个 `StaticLoggerBinder` 类，简洁了很多。

