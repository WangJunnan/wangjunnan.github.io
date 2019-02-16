---
title: Tomcat源码解析二 启动过程
tags:
  - Tomcat
categories:
  - web容器
  - Tomcat
date: 2019-02-12 16:40:29
---

## 引言

上文介绍了`Tomcat`设计的整体架构，对于接下来的源码分析非常重要。接下来我们将从Tomcat的入口启动开始分析Tomcat的源码

`Tomcat`的启动入口我们可以从其启动脚本`catalina.sh`中发现，是由`org.apache.catalina.startup.Bootstrap.java`的main方法启动的。现在我们就从`Bootstrap.java`开始分析`tomcat`的源码

## Bootstrap

在看Main函数之前，我们先看下静态块中的代码

```
static {
    String userDir = System.getProperty("user.dir");

    // 获取 catalina.home 的配置目录
    String home = System.getProperty(Globals.CATALINA_HOME_PROP);
    File homeFile = null;

    if (home != null) {
        File f = new File(home);
        try {
            homeFile = f.getCanonicalFile();
        } catch (IOException ioe) {
            homeFile = f.getAbsoluteFile();
        }
    }
		
		// 如果未配置 则取bootstrap.jar所在目录的上一级
    if (homeFile == null) {
        // First fall-back. See if current directory is a bin directory
        // in a normal Tomcat install
        File bootstrapJar = new File(userDir, "bootstrap.jar");

        if (bootstrapJar.exists()) {
            File f = new File(userDir, "..");
            try {
                homeFile = f.getCanonicalFile();
            } catch (IOException ioe) {
                homeFile = f.getAbsoluteFile();
            }
        }
    }
    if (homeFile == null) {
        // Second fall-back. Use current directory
        File f = new File(userDir);
        try {
            homeFile = f.getCanonicalFile();
        } catch (IOException ioe) {
            homeFile = f.getAbsoluteFile();
        }
    }

    catalinaHomeFile = homeFile;
    System.setProperty(
            Globals.CATALINA_HOME_PROP, catalinaHomeFile.getPath());

    // 未配置catalina.base的情况下，catalina.base的目录和catalina.home一致
    String base = System.getProperty(Globals.CATALINA_BASE_PROP);
    if (base == null) {
        catalinaBaseFile = catalinaHomeFile;
    } else {
        File baseFile = new File(base);
        try {
            baseFile = baseFile.getCanonicalFile();
        } catch (IOException ioe) {
            baseFile = baseFile.getAbsoluteFile();
        }
        catalinaBaseFile = baseFile;
    }
    System.setProperty(
            Globals.CATALINA_BASE_PROP, catalinaBaseFile.getPath());
}
```

上面的代码设置了Tomcat的安装目录和实际工作目录
* catalina.base tomcat工作目录 指`conf`、`logs`、`temp`、`webapps`和`work`文件夹的父目录
* catalina.home 安装目录 其实是指`bin`和`lib`文件夹的父目录

**利用这个特性，我们可以不必复制多个tomcat来启动多个实例**

### Main 函数入口

继续往下看，就是Tomcat启动的最终入口了

```
public static void main(String args[]) {
		
	// 为何这里要加锁？？
    synchronized (daemonLock) {
        if (daemon == null) {
            // Don't set daemon until init() has completed
            Bootstrap bootstrap = new Bootstrap();
            try {
            	  // 执行初始化方法
                bootstrap.init();
            } catch (Throwable t) {
                handleThrowable(t);
                t.printStackTrace();
                return;
            }
            daemon = bootstrap;
        } else {
            // When running as a service the call to stop will be on a new
            // thread so make sure the correct class loader is used to
            // prevent a range of class not found exceptions.
            Thread.currentThread().setContextClassLoader(daemon.catalinaLoader);
        }
    }

    try {
        String command = "start";
        if (args.length > 0) {
        		 // 获取命令参数
            command = args[args.length - 1];
        }
        if (command.equals("startd")) {
            args[args.length - 1] = "start";
            daemon.load(args);
            daemon.start();
        } else if (command.equals("stopd")) {
            args[args.length - 1] = "stop";
            daemon.stop();
        } else if (command.equals("start")) {
        		 // 启动
            daemon.setAwait(true);
            daemon.load(args);
            daemon.start();
            if (null == daemon.getServer()) {
                System.exit(1);
            }
        } else if (command.equals("stop")) {
            daemon.stopServer(args);
        } else if (command.equals("configtest")) {
            daemon.load(args);
            if (null == daemon.getServer()) {
                System.exit(1);
            }
            System.exit(0);
        } else {
            log.warn("Bootstrap: command \"" + command + "\" does not exist.");
        }
    } catch (Throwable t) {
        ....
    }

}
```

上面的代码是整个tomcat的统一入口。可以看到，在Main方法中，执行了`init`方法，那么在`init`方法中肯定做了很多事情

```
public void init() throws Exception {
		
	  // 初始化类加载器
    initClassLoaders();

    Thread.currentThread().setContextClassLoader(catalinaLoader);

    SecurityClassLoad.securityClassLoad(catalinaLoader);

    // Load our startup class and call its process() method
    if (log.isDebugEnabled())
        log.debug("Loading startup class");
        // 获取容器类 org.apache.catalina.startup.Catalina
    Class<?> startupClass = catalinaLoader.loadClass("org.apache.catalina.startup.Catalina");
    // 实例化一个Servlet容器
    Object startupInstance = startupClass.getConstructor().newInstance();
    if (log.isDebugEnabled())
        log.debug("Setting startup class properties");
    String methodName = "setParentClassLoader";
    Class<?> paramTypes[] = new Class[1];
    paramTypes[0] = Class.forName("java.lang.ClassLoader");
    Object paramValues[] = new Object[1];
    paramValues[0] = sharedLoader;
    Method method =
        startupInstance.getClass().getMethod(methodName, paramTypes);
    method.invoke(startupInstance, paramValues);
    catalinaDaemon = startupInstance;

}
```

在`init`方法中初始化了类加载器，并且初始化了一个`org.apache.catalina.startup.Catalina`实例，`main`方法中调用`Catalina`的`start`方法来启动容器。

### start

```
public void start() {
	// 如果 Server为空，则初始化一个
    if (getServer() == null) {
        load();
    }

    if (getServer() == null) {
        log.fatal(什么.getString("catalina.noServer"));
        return;
    }

    long t1 = System.nanoTime();

    // Start the new server
    try {
    		// 启动Server
        getServer().start();
    } catch (LifecycleException e) {
               try {
            getServer().destroy();
        } catch (LifecycleException e1) {
            log.debug("destroy() failed for failed Server ", e1);
        }
        return;
    }

    long t2 = System.nanoTime();
    if(log.isInfoEnabled()) {    
    }
    if (await) {
        await();
        stop();
    }
}
```

可以看到`Start`方法中依赖调用了`Server`的`start`方法，看到这里我们可以知道为何Tomcat容器这么多的组件为何可以统一启动，统一销毁，其实这些组件都是`org.apache.catalina
.Lifecycle`的实现类，`Lifecycle`接口包含了`start`启动，`init`初始化，`destory`销毁，`stop`停止等方法，并且都继承了抽象类`LifecycleBase`，其中具体的`initInternal()`、`startInternal()` 和`stopInternal()` 方法，交由子类自己实现通过实现该接口，达到统一启动关闭的效果，这里其实体现了模板方法的设计模式。

再回到上面的方法，我们着重看一下`load`方法，该方法初始化了一个新的`Server`

```
public void load() {

    if (loaded) {
        return;
    }
    loaded = true;

    long t1 = System.nanoTime();

    initDirs();

    // Before digester - it may be needed
    initNaming();

    // Set configuration source
    ConfigFileLoader.setSource(new CatalinaBaseConfigurationSource(Bootstrap.getCatalinaBaseFile(), getConfigFile()));
    File file = configFile();

    // 解析配置文件 server.xml ，创建一个解析器
    Digester digester = createStartDigester();

    try (ConfigurationSource.Resource resource = ConfigFileLoader.getSource().getServerXml()) {
        InputStream inputStream = resource.getInputStream();
        InputSource inputSource = new InputSource(resource.getURI().toURL().toString());
        inputSource.setByteStream(inputStream);
        digester.push(this);
        digester.parse(inputSource);
    } catch (Exception e) {
        if  (file == null) {
            log.warn(什么.getString("catalina.configFail", getConfigFile() + "] or [server-embed.xml"), e);
        } else {
            log.warn(什么.getString("catalina.configFail", file.getAbsolutePath()), e);
            if (file.exists() && !file.canRead()) {
                log.warn(什么.getString("catalina.incorrectPermissions"));
            }
        }
        return;
    }

    getServer().setCatalina(this);
    getServer().setCatalinaHome(Bootstrap.getCatalinaHomeFile());
    getServer().setCatalinaBase(Bootstrap.getCatalinaBaseFile());

    // Stream redirection
    initStreams();

    // Start the new server
    try {
        getServer().init();
    } catch (LifecycleException e) {
        if (Boolean.getBoolean("org.apache.catalina.startup.EXIT_ON_INIT_FAILURE")) {
            throw new java.lang.Error(e);
        } else {
            log.error(什么.getString("catalina.initError"), e);
        }
    }

    long t2 = System.nanoTime();
    if(log.isInfoEnabled()) {
        log.info(什么.getString("catalina.init", Long.valueOf((t2 - t1) / 1000000)));
    }
}
```

`load`方法其实做的主要事情就是解析`server.xml`文件，`server.xml`是`tomcat`关键配置文件，上篇文章我们聊过`Tomcat`的整体架构，整个架构体系其实非常完整的体现在了`server.xml`文件上。
这里解析`server.xml`文件用的xml解析工具是`SAX`，是一种以事件驱动的XMl API，具体解析的机制这里就不说了
这里其实依次通过解析配置文件创建了 `StandardServer`、`StandardService`、`StandardEngine`、`StandardHost`。
然后调用`StandardServer`的`init()`方法初始化Tomcat容器的一系列组件，容器初始化的的时候，都会调用其子容器的init()方法，初始化它的子容器。上面提到的是`init`的调用过程，`start`方法的启动过程其实也是类似，由父容器启动子容器层层调用。


## 总结

本文分析了`Tomcat`启动部分的源码，这是阅读其源码的入口



