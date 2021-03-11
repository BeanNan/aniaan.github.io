# flink的类加载机制

在研究flink中使用的类加载机制之前，需要先对jvm中的双亲委派模型有个大概了解。

## jvm的双亲委派模型

在jvm中，想要加载一个类，加载的过程中涉及到的classloader并不只有一个，而是由使用的classloader以及它的parent classloader协作完成，当一个classloader收到一个加载类的请求之后，它会先去判断这个类是不是已经加载过了，如果没有，则提交给他的parent classloader来进行加载，同理，parent classloader也会做相同的处理，先提交给它的parent classloader来加载，这样的过程会一直持续到classloader这条链路的最顶端，只有说最顶端的parent classloader加载不了，才会逐级下降，让child classloader来判断是否能加载，加载不了继续下降让下一级的classloader加载，这样一个协作的过程，叫双亲委派模型。

一个classloader中有个关键属性，叫classpath，顾名思义，这代表着这个classloader能加载的类的范围，如果加载的类不在这个classloader的这个范围内，那肯定加载不了的。

jvm启动的时候已经帮我们提供了好了3个classloader,

bootstrap class loader <- extension class loader <- application class loader

1. bootstrap classloader 启动类加载器负责加载最为基础、最为重要的类，比如存放在 JRE 的 lib 目录下 jar 包中的类（以及由虚拟机参数 -Xbootclasspath 指定的类
2. extension classloader 扩展类加载器的父类加载器是启动类加载器。它负责加载相对次要、但又通用的类，比如存放在 JRE 的 lib/ext 目录下 jar 包中的类（以及由系统变量 java.ext.dirs 指定的类
3. application classloader 应用类加载器的父类加载器则是扩展类加载器。它负责加载应用程序路径下的类。（这里的应用程序路径，便是指虚拟机参数 -cp/-classpath、系统变量 java.class.path 或环境变量 CLASSPATH 所指定的路径。）默认情况下，应用程序中包含的类便是由应用类加载器加载的。

上述就是jvm中使用到的双亲委派模型，为什么要用这种机制，而不是直接用一个classloader来完成，这个原因事后也比较容易想清楚

1. 安全性, 可以避免人为的去写一些java.lang.String这种类，即便你写了，最终也用的jvm自己的。
2. 对修改关闭，对扩展开放，有个学名叫开闭原则，这是一个很重要的原则。双亲委派可以方便我们自定义的去扩展类加载器，而不是去修改原先的classpath，扩展性强。

## flink中使用的类加载机制

了解完jvm的类加载机制之后，来看一下flink中使用到的类加载机制，在flink-conf.yaml配置文件中，有这么一行配置`classloader.resolve-order: child-first`, 这个参数有两个可选值，`child-first`和`parent-first`，默认值为`child-first`, 这两个值主要是针对flink session模式来说的，下面依次介绍下不同值的含义。

1. parent-first: flink官方给的解释是`Java default`，也就是说parent-first其实就是jvm中的双亲委派模型
2. child-first: 与parent-first相反，打破了双亲委派模型，不再是parent classloader先加载了，而是child classloader先加载，加载不到才会让parent classloader加载。

下面具体说一下两者在flink中使用的适用场景:

### child-first

在session模式中，我们可以在在集群中放置我们的多个usercode jar包，然后可以根据需要去run指定的jar包，这里先大概介绍一下flink的run api的大概处理流程。

1. 收到run请求后，建立一个ChildFirstClassLoader，这个classloader的classpath为将要运行的这个jar包的地址，他的parent classloader为flink jobmangager这个jvm的application classloader， application classloader主要的classpath就是为`$FLINK_HOME/lib`目录下的jar包。

2. 将当前threade的context classloader设置为ChildFirstClassLoader，然后对jar包的主类进行反射调用。在反射这个jar包的过程中，涉及到了类加载，那么ChildFirstClassLoader重写了默认的loadClass方法，会先去加载他自己的jar里面的类，加载不到才会让parent classloader加载。当然在这个过程中，也会有一些特殊处理，如果要加载的类是flink和hadoop相关类，那么始终会让parent classloader来进行加载。

**优点**

资源隔离，如果让flink parent classloader来统一加载这些jar的包，那么势必是会出现class冲突的问题，使用child-first是资源隔离的，每次只会去加载自己jar包中依赖，而不会用到别的jar的依赖。这里有个小知识点，class被加载到内存中之后，会以Class对象的形式存在jvm方法区中，Class对象在方法区的唯一性是由classloader + Class类名决定的，也就是说，不同的classloader加载同一个类，在方法区中，他们是不同的Class, 那么在run请求中，每次都会建立一个新的ChildFirstClassLoader，由它来加载自己的类, 就保证了，在每次run的过程中，使用到的类都是自己的，不用担心会与其他jar发生冲突。

**适用场景**

如果在session模式下，你有多个不同的jar包，那么采用这种方式是OK。

### parent-first

和child-first处理流程差不多，唯一的区别在于每次建立的不是ChildFirstClassLoader，而是ParentFirstClassloader，ParentFirstClassloader的parent classloader也是flink application classloader，只不过ParentFirstClassloader没有去重写默认的loadClass方法，还是用的双亲委派。

**适用场景**

在session模式下，如果你只有一个jar包，那么就用parent-first吧，同时把你的这个jar放到flink的lib目录下，为什么要用这个，child-first有个开销是，每次run都会去加载一次用到的类，如果有多个jar包，这样是不可避免的，毕竟要资源隔离，但是你只有一个jar包，每次run都重新加载一次，有点占用资源了，如果你invoke jar的速度是远快于jvm回收方法区的速度的，也就是说，你同时并发一上来，同时重复加载类，很容易直接该方法区out of memory，

用parent-first还有一个需要注意的点是，我们编译jar用的某个外部依赖版本可能会和flink本身用的这个外部依赖版本不一致，导致run的时候报错，解决这个问题，就是二者的依赖版本同步一下就好了，都用同一个。

### thread context classloader

上文讲到，在反射jar包之前，需要将thread context classloader设置为ChildFirstClassLoader，不设置会有什么后果，那当然是找不到对应jar包的类了，因为flink application classloader的classpath里面是没有我们自己的jar包的，设置了之后，反射就能成功，反推一下，反射用的classloader是thread context classloader，，我当时看这段代码的时候，也是很蒙，后续反过来一想，就明白了，代码如下

```java

try {
    Thread.currentThread().setContextClassLoader(userCodeClassLoader);

    LOG.info(
            "Starting program (detached: {})",
            !configuration.getBoolean(DeploymentOptions.ATTACHED));

    ContextEnvironment.setAsContext(
            executorServiceLoader,
            configuration,
            userCodeClassLoader,
            enforceSingleJobExecution,
            suppressSysout);

    StreamContextEnvironment.setAsContext(
            executorServiceLoader,
            configuration,
            userCodeClassLoader,
            enforceSingleJobExecution,
            suppressSysout);

    try {
        // invoke jar
        program.invokeInteractiveModeForExecution();
    } finally {
        ContextEnvironment.unsetAsContext();
        StreamContextEnvironment.unsetAsContext();
    }
} finally {
    Thread.currentThread().setContextClassLoader(contextClassLoader);
}

```

## 引用

[极客时间-深入拆解Java虚拟机](https://time.geekbang.org/column/article/11523)