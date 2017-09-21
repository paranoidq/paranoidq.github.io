title: 定制log4j中的RollingFileAppender
date: 2017-09-19 17:22:04
tags: [log4j, RollingFileAppender]
categories:
- log4j
---

## 场景描述
在开发中，一直使用的`RollingFileAppender`，但是存在几个问题：
- 这个appender是按照序号作为后缀来命名日志文件的，生产中日志较为频繁的情况下，根据发生时间定位问题日志需要全局grep，耗费时间并且对业务运行有一定的影响。
- rolling的时候会统一将所有已经存在的日志文件序号向后偏移，即原来是`test.log、test.log.1`，现在变成了`test.log(新建的空文件)、test.log.1、test.log.2`。这就导致可能之前查找的问题日志在`test.log.1`中，现在跑到`test.log.2`里面去了，那么想要保存日志以备后续分析就容易搞错。

因此为了解决上面的问题，决定采用下面的逻辑：
- 一旦日志满了，被backUp了，就不再改变文件名了，保持内容和文件名不动
- 用backUp的时间节点作为后缀，这样可以比较容易地根据时间来到对应的日志文件中查找，不需要全局grep了。

显然，需要对`RollingFileAppender`进行定制，更改其中的rollOver的逻辑。

## 具体方案
RollingFileAppender继承与FIleAppender，内部函数包括：
![Alt text](/img/log4j-rollingAppender/1.png)

大致可以知道，其实就是要修改`rollOver()`函数的逻辑。我们先看原有`rollOver()`函数相关的代码：

`subAppend`函数在当前文件写满之后，会调用rollOver函数，进行“滚动”操作
```java
protected void subAppend(LoggingEvent event) {
	super.subAppend(event);
	if(fileName != null && qw != null) {
		long size = ((CountingQuietWriter) qw).getCount();
		if (size >= maxFileSize && size >= nextRollover) {
		   rollOver();
		}
	}
}
```

`rollOver`函数做了几个事情：
1. 首先确保maxBackups值大于0，如果小于等于0则没有必要做滚动，直接重置当前文件即可；检查是否超过maxBackup值，超过了则删除最旧的日志备份文件，这个最旧的文件文件名是`fileName.maxBackupInex`。
2. 对当前已经存在的backUp后缀名进行逐个偏移，例如`test.log.1`变为`test.log.2`。
3. 将当前文件重命名为`test.log.1`，如果重命名没有成功，则恢复fileName为当前文件名，还是继续往后写。
4. 如果前面的重命名都成功了，那么关闭当前文件，系统在下次写日志时会重建一个新的空日志文件作为写入目标。

```java
public // synchronization not necessary since doAppend is alreasy synched
void rollOver() {
	File target;
    File file;

    if (qw != null) {
        long size = ((CountingQuietWriter) qw).getCount();
        LogLog.debug("rolling over count=" + size);
        //   if operation fails, do not roll again until
        //      maxFileSize more bytes are written
        nextRollover = size + maxFileSize;
    }
    LogLog.debug("maxBackupIndex="+maxBackupIndex);

    boolean renameSucceeded = true;
    // If maxBackups <= 0, then there is no file renaming to be done.
    if(maxBackupIndex > 0) {
	    ① ······················································
	    // Delete the oldest file, to keep Windows happy.
	    file = new File(fileName + '.' + maxBackupIndex);
		if (file.exists())
		    renameSucceeded = file.delete();

		② ······················································
	    // Map {(maxBackupIndex - 1), ..., 2, 1} to {maxBackupIndex, ..., 3, 2}
	    for (int i = maxBackupIndex - 1; i >= 1 && renameSucceeded; i--) {
		    file = new File(fileName + "." + i);
			if (file.exists()) {
				  target = new File(fileName + '.' + (i + 1));
				  LogLog.debug("Renaming file " + file + " to " + target);
				  renameSucceeded = file.renameTo(target);
			}
		}

	    ③ ······················································
	    if(renameSucceeded) {
	        // Rename fileName to fileName.1
            target = new File(fileName + "." + 1);

	        this.closeFile(); // keep windows happy.

	        file = new File(fileName);
	        LogLog.debug("Renaming file " + file + " to " + target);
	        renameSucceeded = file.renameTo(target);
	        //
	        //   if file rename failed, reopen file with append = true
	        //
            if (!renameSucceeded) {
		        try {
		            this.setFile(fileName, true, bufferedIO, bufferSize);
		        } catch(IOException e) {
	                if (e instanceof InterruptedIOException) {
	                    Thread.currentThread().interrupt();
	                }
	                LogLog.error("setFile("+fileName+", true) call failed.", e);
	            }
	        }
        }
    }

    //
    //   if all renames were successful, then
    //
    ④ ······················································
    if (renameSucceeded) {
	    try {
	      // This will also close the file. This is OK since multiple
	      // close operations are safe.
	        this.setFile(fileName, false, bufferedIO, bufferSize);
	        nextRollover = 0;
	     } catch(IOException e) {
	         if (e instanceof InterruptedIOException) {
	             Thread.currentThread().interrupt();
	         }
             LogLog.error("setFile("+fileName+", false) call failed.", e);
         }
     }
  }
```


根据以上的逻辑我们修改rollOver函数中的文件命名规则，在重命名时采用当前时间作为后缀即可。但是，还有一个问题需要解决：`如何判定最旧的文件名字`。显然，`RollingFileAppender`以序号作为后缀是很简单的，直接找最大序号即可；但是如果用时间作为后缀就需要做一些额外的处理。这里有两种思路：
- 按照后缀排序
- 将文件名存在一个FILO队列中，每次需要删除的时候取出队首的文件名进行删除

第二种思路相对比较简单，因此我采用了第二种方式。在实现过程中，还需要解决一个问题，一旦系统重启，FILO队列将丢失，如果不保存FILO队列到磁盘，那么下次系统启动的时候就无法知道已经有了哪些backUp日志文件，FILO就失去了作用，每次系统重启就会导致日志备份文件增长一倍。因此还需要做的一件事情是：
- 在改变FILO队列之后，立马将其序列化到磁盘中；然后在`Logger`初始化的时候加载FILO队列。这样可以还原系统之前的`Logger`备份文件的所有名称，当达到`maxBackUpIndex`的时候就能够正确找到文件名进行删除了。

基本的实现思路介绍完了，下面是修改的代码：

首先需要定义两个变量，一个是FILO队列，另一个是序列化FILO队列到磁盘时的文件后缀名，注意要与日志备份文件区分开。
```java
/**
 * 序列化existedLogFileNames队列的文件名后缀
 */
private static final String SERIALIZE_POSTFIX = ".restore";

/**
 * 存储backUp的文件名
 * existedLogFileNames.capacity == maxBackUp
 */
private LinkedBlockingDeque<String> existedLogFileNames;
```

然后定义一个函数，用当前时间和原始文件名构造新的backup文件名：
```java
/**
 * 将当前的日志文件重命名为backUp文件
 * 以当前时间作为后缀，方便查找
 *
 * @return
 */
private String renameAsBackup() {
    SimpleDateFormat format = new SimpleDateFormat("yyyyMMdd_HHmmss_SSS");
    String dateString = format.format(new Date(System.currentTimeMillis()));
    return fileName + "." + dateString;
}
```

然后我们改写rollOver函数：
1.  删除文件的时候，从FILO队列中poll头部文件名删除
2.  移除了滚动重命名的部分代码，只需要将当前已满的日志文件重命名，重命名的名字由`renameBackup()`函数根据当前时间来构造
3.  重命名成功之后，将重命名后的文件放入FILO队列中，同时立即序列化FILO队列到磁盘
```java
public // synchronization not necessary since doAppend is alreasy synched
void rollOver() {
    LogLog.debug("当前队列剩余空间: " + existedLogFileNames.remainingCapacity());
    System.out.println("当前队列剩余空间: " + existedLogFileNames.remainingCapacity());

    File file;
    if (qw != null) {
        long size = ((CountingQuietWriter) qw).getCount();
        LogLog.debug("rolling over count=" + size);
        //   if operation fails, do not roll again until
        //      maxFileSize more bytes are written
        nextRollover = size + maxFileSize;
    }
    LogLog.debug("maxBackupIndex="+maxBackupIndex);

    boolean deleteSuccess = true;
    // If maxBackups <= 0, then there is no file renaming to be done.
    if(maxBackupIndex > 0 && existedLogFileNames.size() >= maxBackupIndex) {
        // Delete the oldest file, to keep Windows happy.
        ① ······················································
        file = new File(existedLogFileNames.pollFirst());
        if (file.exists()) {
            deleteSuccess = file.delete();
        }
    }

    boolean renameSuccess = true;
    if (deleteSuccess) {
    ② ······················································
        // 将当前日志文件backUp重命名
        File target = new File(renameAsBackup());
        this.closeFile();
        file = new File(fileName);
        LogLog.debug("Renaming file " + file + " to " + target);
        renameSuccess = file.renameTo(target);

        if (renameSuccess) {
            // 将重命名后的文件名存入FILO队列
            ③ ······················································
            try {
                renameSuccess = existedLogFileNames.offerLast(target.getCanonicalPath());
            } catch (IOException e) {
                renameSuccess = false;
            }
            if (renameSuccess) {
                // 将FILO队列序列化到磁盘，以便系统重启时读取
                try {
                    File f = new File(fileName + SERIALIZE_POSTFIX);
                    f.createNewFile();
                    SerializationUtils.serialize(existedLogFileNames, new FileOutputStream(f, false));
                } catch (IOException e) {
                    LogLog.debug("Error serialize existed file queue", e);
                    renameSuccess = false;
                }
            }
        }

        //
        //   if file rename failed, reopen file with append = true
        //
        if (!renameSuccess) {
            try {
                this.setFile(fileName, true, bufferedIO, bufferSize);
            }
            catch(IOException e) {
                if (e instanceof InterruptedIOException) {
                    Thread.currentThread().interrupt();
                }
                LogLog.error("setFile("+fileName+", true) call failed.", e);
            }
        }
    }

    //
    //   if deletes were successful, then
    //
    if (renameSuccess) {
        try {
            // This will also close the file. This is OK since multiple
            // close operations are safe.
            this.setFile(fileName, false, bufferedIO, bufferSize);
            nextRollover = 0;
        }
        catch(IOException e) {
            if (e instanceof InterruptedIOException) {
                Thread.currentThread().interrupt();
            }
            LogLog.error("setFile("+fileName+", false) call failed.", e);
        }
    }
}
```

rollOver的逻辑写完了，还需要添加Logger初始化时从磁盘加载FILO队列的逻辑。首先将这段逻辑封装为一个函数：
```java
private void restoreBackUpsIfNull() {
    if (existedLogFileNames == null) {
        existedLogFileNames = new LinkedBlockingDeque<String>(getMaxBackupIndex());
        try {
            LinkedBlockingDeque<String> restore = (LinkedBlockingDeque<String>) SerializationUtils.deserialize(
                new FileInputStream(fileName + SERIALIZE_POSTFIX));
            existedLogFileNames.addAll(restore);
        } catch (FileNotFoundException e) {
            e.printStackTrace();
        }
    }
}
```
然后通过debug可以发现Logger初始化的时候会调用`setMaxBackUpIndex()`来将配置文件的值设置进来，因此重写`setMaxBackUpIndex()`函数，添加加载FILO队列的逻辑：（**这种写法实际上是存在BUG的，后面会分析到**）
```java
public void setMaxBackupIndex(int maxBackups) {
     this.maxBackupIndex = maxBackups;
     restoreBackUpsIfNull();
 }
```
以上就是定制`RollingFileAppender`的整个思路。

## 运行测试及BUG出现
`log4j.properties`配置文件如下：
```
log4j.rootLogger=DEBUG
log4j.logger.me.zex=DEBUG, file
log4j.logger.org.log4j=DEBUG, console

log4j.appender.console=org.apache.log4j.ConsoleAppender
log4j.appender.console.threshold=INFO
log4j.appender.console.layout=org.apache.log4j.PatternLayout
log4j.appender.console.layout.ConversionPattern=%d{yyyy-MM-dd HH:mm:ss} [%5p] - %c -%F(%L) -%m%n

log4j.appender.file=me.zex.util.logger.namedLogger.NamedRollingFileAppender
log4j.appender.file.File=/Users/paranoidq/test/test.log
log4j.appender.file.layout=org.apache.log4j.PatternLayout
log4j.appender.file.layout.ConversionPattern=%d{yyyy-MM-dd HH:mm:ss} [%5p] - %c -%F(%L) -%m%n
log4j.appender.file.MaxFileSize=2KB
log4j.appender.file.MaxBackupIndex=10
```

运行测试代码，包括了单线程写和多线程写：
```java
public class Test {
    private static org.apache.log4j.Logger logger = org.apache.log4j.Logger.getLogger(Test.class);
    public static void main(String[] args) throws Exception {
//        normalTest();
        parallelTest();
    }

    private static void normalTest() throws InterruptedException {
        while (true) {
            logger.info("test");
            Thread.sleep(100);
        }
    }

    private static void parallelTest() throws Exception {
        final CountDownLatch latch = new CountDownLatch(1);
        for (int i = 0; i < 2000; i++) {
            int finalI = i;
            new Thread(new Runnable() {
                @Override
                public void run() {
                    try {
                        latch.await();
                        for (int j = 0; j < 1; j++) {
                            logger.info("Thread-" + finalI + ": test log " + j);
//                            Thread.sleep(2000);
                        }
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            }).start();
        }
        latch.countDown();
    }
}
```

测试通过，能够正常写文件和删除文件。重启之后也没问题。
![Alt text](/img/log4j-rollingAppender/2.png)

但是在同事测试的时候却发现重启之后无法加载FILO队列，在加载FILO队列时抛出`FileNotFoundException`：
![Alt text](/img/log4j-rollingAppender/3.png)
从报错信息可以看出，在`setMaxBackUpIndex()`中调用`restoreBackUpsIfNull()`获取的fileName为null。经过一系列的排查，最终定位到同事配置文件的写法与我不同：
```
log4j.rootLogger=DEBUG
log4j.logger.me.zex=DEBUG, FILE
log4j.logger.org.log4j=DEBUG, console

log4j.appender.console=org.apache.log4j.ConsoleAppender
log4j.appender.console.threshold=INFO
log4j.appender.console.layout=org.apache.log4j.PatternLayout
log4j.appender.console.layout.ConversionPattern=%d{yyyy-MM-dd HH:mm:ss} [%5p] - %c -%F(%L) -%m%n

log4j.appender.FILE=me.zex.util.logger.namedLogger.NamedRollingFileAppender
log4j.appender.FILE.File=/Users/paranoidq/test/test.log
log4j.appender.FILE.layout=org.apache.log4j.PatternLayout
log4j.appender.FILE.layout.ConversionPattern=%d{yyyy-MM-dd HH:mm:ss} [%5p] - %c -%F(%L) -%m%n
log4j.appender.FILE.MaxFileSize=2KB
log4j.appender.FILE.MaxBackupIndex=10
```

我的配置是`log4j.appender.file`，而他的配置则是`log4j.appender.FILE`。难道仅仅是大小写的不同就造成了log4j在执行时的差别么？我又重新将`log4j.appender`改为了其他大写的名字，例如`XXX`，`DDD`，诡异的问题出现了，某些大写名称是可以正常读到`fileName`的，但是某些大写名称读到的`fileName`就为空。于是，为了解决这个BUG，我决定从源码入手，搞清楚log4j在加载Logger时的初始化流程。


## 源码级别的问题定位
通过断点一步步进行debug，log4j加载Logger的大致流程如下：

##### 1. `Logger`调用`LoggerManager.getLogger(clazz.getName())`：
![Alt text](/img/log4j-rollingAppender/4.png)

##### 2. `LoggerManager`中会从`LoggerRepository`中根据名称获取Logger
![Alt text](/img/log4j-rollingAppender/5.png)
而这个LoggerRepository则是在LoggerManager的static初始化块中进行一系列的初始化工作的。那么继续定位static初始化块。

##### 3. 初始化块做了几件事情：
1. 首选构造了一个`Hirarchy`对象，这个对象用于表示`Logger`之间的层级关系
2. 然后依次检查`log4j.configuration`是否配置、`log4j.xml`配置和`log4j.properties`配置文件，如果有就利用Loader加载。
3. 加载完成后，通过`OptionConverter.selectAndConfigure()`函数进行初始化工作。显然，下一步我们需要到这个函数中取一探究竟。
```java
static {
    // By default we use a DefaultRepositorySelector which always returns 'h'.
    Hierarchy h = new Hierarchy(new RootLogger((Level) Level.DEBUG));
    repositorySelector = new DefaultRepositorySelector(h);

    /** Search for the properties file log4j.properties in the CLASSPATH.  */
    String override =OptionConverter.getSystemProperty(DEFAULT_INIT_OVERRIDE_KEY,
						       null);

    // if there is no default init override, then get the resource
    // specified by the user or the default config file.
    if(override == null || "false".equalsIgnoreCase(override)) {

      String configurationOptionStr = OptionConverter.getSystemProperty(
							  DEFAULT_CONFIGURATION_KEY,
							  null);

      String configuratorClassName = OptionConverter.getSystemProperty(
                                                   CONFIGURATOR_CLASS_KEY,
						   null);

      URL url = null;

      // if the user has not specified the log4j.configuration
      // property, we search first for the file "log4j.xml" and then
      // "log4j.properties"
      if(configurationOptionStr == null) {
	url = Loader.getResource(DEFAULT_XML_CONFIGURATION_FILE);
	if(url == null) {
	  url = Loader.getResource(DEFAULT_CONFIGURATION_FILE);
	}
      } else {
	try {
	  url = new URL(configurationOptionStr);
	} catch (MalformedURLException ex) {
	  // so, resource is not a URL:
	  // attempt to get the resource from the class path
	  url = Loader.getResource(configurationOptionStr);
	}
      }

      // If we have a non-null url, then delegate the rest of the
      // configuration to the OptionConverter.selectAndConfigure
      // method.
      if(url != null) {
	    LogLog.debug("Using URL ["+url+"] for automatic log4j configuration.");
        try {
            OptionConverter.selectAndConfigure(url, configuratorClassName,
					   LogManager.getLoggerRepository());
        } catch (NoClassDefFoundError e) {
            LogLog.warn("Error during default initialization", e);
        }
      } else {
	    LogLog.debug("Could not find resource: ["+configurationOptionStr+"].");
      }
    } else {
        LogLog.debug("Default initialization of overridden by " +
            DEFAULT_INIT_OVERRIDE_KEY + "property.");
    }  
}
```

##### 4.  OptionConverter
```java
static public
  void selectAndConfigure(URL url, String clazz, LoggerRepository hierarchy) {
   Configurator configurator = null;
   String filename = url.getFile();

   if(clazz == null && filename != null && filename.endsWith(".xml")) {
     clazz = "org.apache.log4j.xml.DOMConfigurator";
   }

   if(clazz != null) {
     LogLog.debug("Preferred configurator class: " + clazz);
     configurator = (Configurator) instantiateByClassName(clazz,
							  Configurator.class,
							  null);
     if(configurator == null) {
   	  LogLog.error("Could not instantiate configurator ["+clazz+"].");
   	  return;
     }
   } else {
     configurator = new PropertyConfigurator();
   }

   configurator.doConfigure(url, hierarchy);
  }
```
在selectAndConfigure中，判断是采用哪一种`Configurator`解析配置文件，如果是xml就采用`org.apache.log4j.xml.DOMConfigurator`，如果不是，则采用`PropertyConfigurator`。为了扩展性，也可以自定义configurator来解析自定义格式的配置。最后调用`configurator.doConfigure(url, hierarchy)`进行解析。显然，这里调用的应该是`PropertyConfigurator.doConfigure()`实现。

##### 5. PropertyConfigurator
```java
public
  void doConfigure(Properties properties, LoggerRepository hierarchy) {
	repository = hierarchy;
    String value = properties.getProperty(LogLog.DEBUG_KEY);
    if(value == null) {
      value = properties.getProperty("log4j.configDebug");
      if(value != null)
	LogLog.warn("[log4j.configDebug] is deprecated. Use [log4j.debug] instead.");
    }

    if(value != null) {
      LogLog.setInternalDebugging(OptionConverter.toBoolean(value, true));
    }

      //
      //   if log4j.reset=true then
      //        reset hierarchy
    String reset = properties.getProperty(RESET_KEY);
    if (reset != null && OptionConverter.toBoolean(reset, false)) {
          hierarchy.resetConfiguration();
    }

    String thresholdStr = OptionConverter.findAndSubst(THRESHOLD_PREFIX,
						       properties);
    if(thresholdStr != null) {
      hierarchy.setThreshold(OptionConverter.toLevel(thresholdStr,
						     (Level) Level.ALL));
      LogLog.debug("Hierarchy threshold set to ["+hierarchy.getThreshold()+"].");
    }

    configureRootCategory(properties, hierarchy);
    configureLoggerFactory(properties);
    parseCatsAndRenderers(properties, hierarchy);

    LogLog.debug("Finished configuring.");
    // We don't want to hold references to appenders preventing their
    // garbage collection.
    registry.clear();
  }
```
前面的代码都是做一些debug性的工作，我们暂时忽略，关键的加载和初始化代码是三句话：
![Alt text](/img/log4j-rollingAppender/6.png)

###### 5.1 configureRootCategory
configureRootCategory主要负责解析根Logger相关的Category，Category在log4j中已经基本被Logger代替了。
这段根我们的appender没有太大的关系。
```java
void configureRootCategory(Properties props, LoggerRepository hierarchy) {
    String effectiveFrefix = ROOT_LOGGER_PREFIX;
    String value = OptionConverter.findAndSubst(ROOT_LOGGER_PREFIX, props);

    if(value == null) {
      value = OptionConverter.findAndSubst(ROOT_CATEGORY_PREFIX, props);
      effectiveFrefix = ROOT_CATEGORY_PREFIX;
    }

    if(value == null)
      LogLog.debug("Could not find root logger information. Is this OK?");
    else {
      Logger root = hierarchy.getRootLogger();
      synchronized(root) {
	parseCategory(props, root, effectiveFrefix, INTERNAL_ROOT_NAME, value);
      }
    }
  }
```
###### 5.2 configureLoggerFactory

这段从字面意思上值用于配置自定义的Logger工厂的，与我们的Appender也没有关系。
```java
 protected void configureLoggerFactory(Properties props) {
    String factoryClassName = OptionConverter.findAndSubst(LOGGER_FACTORY_KEY,
							   props);
    if(factoryClassName != null) {
      LogLog.debug("Setting category factory to ["+factoryClassName+"].");
      loggerFactory = (LoggerFactory)
	          OptionConverter.instantiateByClassName(factoryClassName,
							 LoggerFactory.class,
							 loggerFactory);
      PropertySetter.setProperties(loggerFactory, props, FACTORY_PREFIX + ".");
    }
  }
```

###### 5.3 parseCatsAndRenders
```java
protected
  void parseCatsAndRenderers(Properties props, LoggerRepository hierarchy) {
    Enumeration enumeration = props.propertyNames();
    while(enumeration.hasMoreElements()) {
      String key = (String) enumeration.nextElement();
      if(key.startsWith(CATEGORY_PREFIX) || key.startsWith(LOGGER_PREFIX)) {
	String loggerName = null;
	if(key.startsWith(CATEGORY_PREFIX)) {
	  loggerName = key.substring(CATEGORY_PREFIX.length());
	} else if(key.startsWith(LOGGER_PREFIX)) {
	  loggerName = key.substring(LOGGER_PREFIX.length());
	}
	String value =  OptionConverter.findAndSubst(key, props);
	Logger logger = hierarchy.getLogger(loggerName, loggerFactory);
	synchronized(logger) {
	  parseCategory(props, logger, key, loggerName, value);
	  parseAdditivityForLogger(props, logger, loggerName);
	}
      } else if(key.startsWith(RENDERER_PREFIX)) {
	String renderedClass = key.substring(RENDERER_PREFIX.length());
	String renderingClass = OptionConverter.findAndSubst(key, props);
	if(hierarchy instanceof RendererSupport) {
	  RendererMap.addRenderer((RendererSupport) hierarchy, renderedClass,
				  renderingClass);
	}
      } else if (key.equals(THROWABLE_RENDERER_PREFIX)) {
          if (hierarchy instanceof ThrowableRendererSupport) {
            ThrowableRenderer tr = (ThrowableRenderer)
                  OptionConverter.instantiateByKey(props,
                          THROWABLE_RENDERER_PREFIX,
                          org.apache.log4j.spi.ThrowableRenderer.class,
                          null);
            if(tr == null) {
                LogLog.error(
                    "Could not instantiate throwableRenderer.");
            } else {
                PropertySetter setter = new PropertySetter(tr);
                setter.setProperties(props, THROWABLE_RENDERER_PREFIX + ".");
                ((ThrowableRendererSupport) hierarchy).setThrowableRenderer(tr);

            }
          }
      }
    }
  }
```
这段代码主要是解析非root的配置元素
它首先会根据配置文件的前缀通过subString获取到loggerName：例如如果配置的是`log4j.logger.me.zex`，那么就会通过`loggerName = key.substring(LOGGER_PREFIX.length());`获取到loggerName为`me.zex`。这个loggerName会唯一确定一个Logger。
![Alt text](/img/log4j-rollingAppender/7.png)

获取Logger后，对Logger进行初始化，包括设置Category和Additivity。
![Alt text](/img/log4j-rollingAppender/8.png)

###### 5.4 parseCategory
```java
void parseCategory(Properties props, Logger logger, String optionKey,
		     String loggerName, String value) {

    LogLog.debug("Parsing for [" +loggerName +"] with value=[" + value+"].");
    // We must skip over ',' but not white space
    StringTokenizer st = new StringTokenizer(value, ",");

    // If value is not in the form ", appender.." or "", then we should set
    // the level of the loggeregory.

    if(!(value.startsWith(",") || value.equals(""))) {

      // just to be on the safe side...
      if(!st.hasMoreTokens())
	return;

      String levelStr = st.nextToken();
      LogLog.debug("Level token is [" + levelStr + "].");

      // If the level value is inherited, set category level value to
      // null. We also check that the user has not specified inherited for the
      // root category.
      if(INHERITED.equalsIgnoreCase(levelStr) ||
 	                                  NULL.equalsIgnoreCase(levelStr)) {
	if(loggerName.equals(INTERNAL_ROOT_NAME)) {
	  LogLog.warn("The root logger cannot be set to null.");
	} else {
	  logger.setLevel(null);
	}
      } else {
	logger.setLevel(OptionConverter.toLevel(levelStr, (Level) Level.DEBUG));
      }
      LogLog.debug("Category " + loggerName + " set to " + logger.getLevel());
    }

    // Begin by removing all existing appenders.
    logger.removeAllAppenders();

    Appender appender;
    String appenderName;
    while(st.hasMoreTokens()) {
      appenderName = st.nextToken().trim();
      if(appenderName == null || appenderName.equals(","))
	continue;
      LogLog.debug("Parsing appender named \"" + appenderName +"\".");
      appender = parseAppender(props, appenderName);
      if(appender != null) {
	logger.addAppender(appender);
      }
    }
  }
```
parseCategory函数会根据log4j.properties中读取的配置行提取出appender和level，并进行初始化工作。例如从`log4j.logger.me.zes=DEBUG, FILE`提取leve为`DEBUG`，而通过逗号分隔提取出appender。

appender的解析在parseAppender中进行，配置行中的逗号可以分隔多个appender，会因此进行解析。
![Alt text](/img/log4j-rollingAppender/9.png)

###### 5.5 parseAppender
```java
Appender parseAppender(Properties props, String appenderName) {
    Appender appender = registryGet(appenderName);
    if((appender != null)) {
      LogLog.debug("Appender \"" + appenderName + "\" was already parsed.");
      return appender;
    }
    // Appender was not previously initialized.
    String prefix = APPENDER_PREFIX + appenderName;
    String layoutPrefix = prefix + ".layout";

    appender = (Appender) OptionConverter.instantiateByKey(props, prefix,
					      org.apache.log4j.Appender.class,
					      null);
    if(appender == null) {
      LogLog.error(
              "Could not instantiate appender named \"" + appenderName+"\".");
      return null;
    }
    appender.setName(appenderName);

    if(appender instanceof OptionHandler) {
      if(appender.requiresLayout()) {
	Layout layout = (Layout) OptionConverter.instantiateByKey(props,
								  layoutPrefix,
								  Layout.class,
								  null);
	if(layout != null) {
	  appender.setLayout(layout);
	  LogLog.debug("Parsing layout options for \"" + appenderName +"\".");
	  //configureOptionHandler(layout, layoutPrefix + ".", props);
          PropertySetter.setProperties(layout, props, layoutPrefix + ".");
	  LogLog.debug("End of parsing for \"" + appenderName +"\".");
	}
      }
      final String errorHandlerPrefix = prefix + ".errorhandler";
      String errorHandlerClass = OptionConverter.findAndSubst(errorHandlerPrefix, props);
      if (errorHandlerClass != null) {
    		ErrorHandler eh = (ErrorHandler) OptionConverter.instantiateByKey(props,
					  errorHandlerPrefix,
					  ErrorHandler.class,
					  null);
    		if (eh != null) {
    			  appender.setErrorHandler(eh);
    			  LogLog.debug("Parsing errorhandler options for \"" + appenderName +"\".");
    			  parseErrorHandler(eh, errorHandlerPrefix, props, repository);
    			  final Properties edited = new Properties();
    			  final String[] keys = new String[] {
    					  errorHandlerPrefix + "." + ROOT_REF,
    					  errorHandlerPrefix + "." + LOGGER_REF,
    					  errorHandlerPrefix + "." + APPENDER_REF_TAG
    			  };
    			  for(Iterator iter = props.entrySet().iterator();iter.hasNext();) {
    				  Map.Entry entry = (Map.Entry) iter.next();
    				  int i = 0;
    				  for(; i < keys.length; i++) {
    					  if(keys[i].equals(entry.getKey())) break;
    				  }
    				  if (i == keys.length) {
    					  edited.put(entry.getKey(), entry.getValue());
    				  }
    			  }
    		      PropertySetter.setProperties(eh, edited, errorHandlerPrefix + ".");
    			  LogLog.debug("End of errorhandler parsing for \"" + appenderName +"\".");
    		}

      }
      //configureOptionHandler((OptionHandler) appender, prefix + ".", props);
      PropertySetter.setProperties(appender, props, prefix + ".");
      LogLog.debug("Parsed \"" + appenderName +"\" options.");
    }
    parseAppenderFilters(props, appenderName, appender);
    registryPut(appender);
    return appender;
  }
```
该函数负责根据appenderName解析对应的appender配置。
首先检查是否已经解析过了，如果已经解析过了，则直接返回。
![Alt text](/img/log4j-rollingAppender/10.png)
然后通过调用`OptionConveter.instantizeByKey`方法根据配置的appender类名来反射构造一个appender
![Alt text](/img/log4j-rollingAppender/11.png)

**--> --> -->**
![Alt text](/img/log4j-rollingAppender/12.png)

**--> --> -->**
![Alt text](/img/log4j-rollingAppender/13.png)

此时会进入到自定义的NamedRollingFileAppender的默认构造函数，创建一个对象，并执行初始化。
![Alt text](/img/log4j-rollingAppender/14.png)
但是需要注意的是，由于调用的是默认构造函数，因此除了显示指定的成员变量值外，其他成员变量没有赋值。**fileName此时也是null。**
![Alt text](/img/log4j-rollingAppender/15.png)


返回到parseAppender之后，NamedRollingFileAppender已经构造完成。此时需要进一步对他进行配置
![Alt text](/img/log4j-rollingAppender/16.png)

首先配置layout，通过`OptionConveter.instantizeByKey`反射构造layout对象，并set进appender中：
![Alt text](/img/log4j-rollingAppender/17.png)

然后通过`PropertySetter`类对layout中的属性进行设置。
![Alt text](/img/log4j-rollingAppender/18.png)

`PropertySetter类`利用了Java中的**内省机制**（java.beans.Instrospect），根据剔除前缀后的配置项的key来找到对应的对象中的成员变量，并将配置项的value设置到成员变量中。例如：对于layout的配置`log4j.appender.console.layout.ConversionPattern=%d{yyyy-MM-dd HH:mm:ss} [%5p] - %c -%F(%L) -%m%n`，`PropertySetter`会根据前缀`log4j.appender.console.layout`进行匹配，然后提取`ConversionPattern`这个key，通过`getPropertyDescriptor(Introspector.decapitalize(key))`来内省地获取已经构造的layout对象中的对应属性，并通过set函数完成设值。
![Alt text](/img/log4j-rollingAppender/19.png)

layout初始化完毕之后，进行appender的初始化。这里与layout的做法相同，也是通过`PropertySetter`将配置设置到对象中去。这些都完成后就一个Logger就正式初始化完毕，可以被使用了。
![Alt text](/img/log4j-rollingAppender/20.png)


## 问题分析

以上就是整个Logger从构造到初始化完成的大致源码流程。那么回到上面的BUG上，显然在进行最后一步appender的内省赋值之前，fileName都是为null的。内省赋值会根据配置项的key依次调用对应的set函数进行设置，比如配置了`log4j.appender.FILE.maxBackUpIndex`就会被反射调用`setMaxBackUpIndex`函数，配置了`log4j.appender.FILE.File`就会被反射地调用`setFile`函数。

那么问题的关键其实就是，**调用`setMaxBackUpIndex`函数时fileName没有被正确的设置，也就是`setMaxBackUpIndex`在`setFile`之前被反射调用了。**
根据`PropertySetter.setProperties()`的源码，我们不难分析出原因：


>由于`Properties`内部是以`HashTable`的方式存储配置项的，因此在遍历配置项的时候，是按照key值的所在的hash桶来顺序访问。而这个顺序是随着不同的配置项的名称而变化的。因此才会出现有些appender名称会先调用`setFile`函数，而有些appender名称会先调用`setMaxBackupIndex`函数，从而导致了fileName还没有被设值的情况发生。
![Alt text](/img/log4j-rollingAppender/21.png)

为了验证上面解释，我们在不同的appender名字下，debug观察Properties中HashTable的元素的顺序：
- 配置`log4j.appender.FILE`的情况下：`log4j.appender.FILE.MaxBackUpIndex`排在`log4j.appender.FILE.File`前面：
![Alt text](/img/log4j-rollingAppender/22.png)
- 配置配置`log4j.appender.file`的情况下：`log4j.appender.file.MaxBackUpIndex`排在`log4j.appender.file.File`后面：
![Alt text](/img/log4j-rollingAppender/23.png)


因此造成问题的根源就在于此。



## 如何FIX
从源码级别的分析可以知道，由于Properties内的HashTable不能保证顺序，因此我们不能将加载FILO队列的逻辑放到任何设值函数中。解决方案就是**在`rollOver()`开始处添加**，这样可以保证任何设置函数都已经执行完毕，`fileName`和`maxBackupIndex`都已经正确获得了配置值。

```java
public // synchronization not necessary since doAppend is alreasy synched
    void rollOver() {
        // 如果没有在log4j中配置maxBackupIndex，那么set函数不会被调用，因此要在这里检查existedLogFileNames队列是否成功初始化
        restoreBackUpsIfNull();
        ...
```

## 参考链接
- [http://blog.csdn.net/s464036801/article/details/22915963](http://blog.csdn.net/s464036801/article/details/22915963)
