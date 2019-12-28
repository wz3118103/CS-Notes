# 1.slf4j是怎样找到并加载logback
定义了实现类的class地址，然后通过类加载器加载指定的文件。
```
private static String STATIC_LOGGER_BINDER_PATH = "org/slf4j/impl/StaticLoggerBinder.class";
```
<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/slf4j-api.jpg" width="520px" > </div><br>

logback-classic中有实现类org.slf4j.impl.StaticLoggerBinder：
<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/logback-classic.jpg" width="520px" > </div><br>

## 1.1 源码流程
```
@Service
public class UserServiceImpl implements UserService {

	Logger logger = LoggerFactory.getLogger(UserServiceImpl.class);
	
	@Resource(name="tUserMapper")
	private TUserMapper userMapper;

	@Override
	public TUser getUserById(Integer id) {
		logger.info("start");
		return userMapper.selectByPrimaryKey(id);
	}
}
```
为什么要传入UserServiceImpl.class？
* 因为每个类使用的日志处理实现可能不同，iLoggerFactory中也是根据名字来判断一个类的实现方式的。

```
    public static Logger getLogger(Class<?> clazz) {
        Logger logger = getLogger(clazz.getName());
        if (DETECT_LOGGER_NAME_MISMATCH) {
            Class<?> autoComputedCallingClass = Util.getCallingClass();
            if (autoComputedCallingClass != null && nonMatchingClasses(clazz, autoComputedCallingClass)) {
                Util.report(String.format("Detected logger name mismatch. Given name: \"%s\"; computed name: \"%s\".", logger.getName(),
                                autoComputedCallingClass.getName()));
                Util.report("See " + LOGGER_NAME_MISMATCH_URL + " for an explanation");
            }
        }
        return logger;
    }
```
```
    public static Logger getLogger(String name) {
        ILoggerFactory iLoggerFactory = getILoggerFactory();
        return iLoggerFactory.getLogger(name);
    }
```
在getLogger方法中，通过getLoggerFactory获取工厂，然后获取日志对象。
```
    public static ILoggerFactory getILoggerFactory() {
        if (INITIALIZATION_STATE == UNINITIALIZED) {
            synchronized (LoggerFactory.class) {
                if (INITIALIZATION_STATE == UNINITIALIZED) {
                    INITIALIZATION_STATE = ONGOING_INITIALIZATION;
                    performInitialization();
                }
            }
        }
        switch (INITIALIZATION_STATE) {
        case SUCCESSFUL_INITIALIZATION:
            return StaticLoggerBinder.getSingleton().getLoggerFactory();
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
该方法分为两个步骤：
* step1.判断是否进行初始化，如果没有初始化，则修改状态，进入performIntialization初始化
* step2.检查初始化状态，如果初始化成功，则通过StaticLoggerBinder获取日志工厂


```
    private final static void performInitialization() {
        bind();
        if (INITIALIZATION_STATE == SUCCESSFUL_INITIALIZATION) {
            versionSanityCheck();
        }
    }
```

```
    private final static void bind() {
        try {
            Set<URL> staticLoggerBinderPathSet = null;
            // skip check under android, see also
            // http://jira.qos.ch/browse/SLF4J-328
            if (!isAndroid()) {
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
        } catch (NoClassDefFoundError ncde) {
            String msg = ncde.getMessage();
            if (messageContainsOrgSlf4jImplStaticLoggerBinder(msg)) {
                INITIALIZATION_STATE = NOP_FALLBACK_INITIALIZATION;
                Util.report("Failed to load class \"org.slf4j.impl.StaticLoggerBinder\".");
                Util.report("Defaulting to no-operation (NOP) logger implementation");
                Util.report("See " + NO_STATICLOGGERBINDER_URL + " for further details.");
            } else {
                failedBinding(ncde);
                throw ncde;
            }
        } catch (java.lang.NoSuchMethodError nsme) {
            String msg = nsme.getMessage();
            if (msg != null && msg.contains("org.slf4j.impl.StaticLoggerBinder.getSingleton()")) {
                INITIALIZATION_STATE = FAILED_INITIALIZATION;
                Util.report("slf4j-api 1.6.x (or later) is incompatible with this binding.");
                Util.report("Your binding is version 1.5.5 or earlier.");
                Util.report("Upgrade your binding to version 1.6.x.");
            }
            throw nsme;
        } catch (Exception e) {
            failedBinding(e);
            throw new IllegalStateException("Unexpected initialization failure", e);
        }
    }
```

这个方法的主要功能就是寻找日志实现类，如果找不到或者指定的方法不存在，都会报错提示！

```
    private static String STATIC_LOGGER_BINDER_PATH = "org/slf4j/impl/StaticLoggerBinder.class";

    static Set<URL> findPossibleStaticLoggerBinderPathSet() {
        // use Set instead of list in order to deal with bug #138
        // LinkedHashSet appropriate here because it preserves insertion order
        // during iteration
        Set<URL> staticLoggerBinderPathSet = new LinkedHashSet<URL>();
        try {
            ClassLoader loggerFactoryClassLoader = LoggerFactory.class.getClassLoader();
            Enumeration<URL> paths;
            if (loggerFactoryClassLoader == null) {
                paths = ClassLoader.getSystemResources(STATIC_LOGGER_BINDER_PATH);
            } else {
                paths = loggerFactoryClassLoader.getResources(STATIC_LOGGER_BINDER_PATH);
            }
            while (paths.hasMoreElements()) {
                URL path = paths.nextElement();
                staticLoggerBinderPathSet.add(path);
            }
        } catch (IOException ioe) {
            Util.report("Error getting resources from path", ioe);
        }
        return staticLoggerBinderPathSet;
    }
```
logback-classic中有实现类org.slf4j.impl.StaticLoggerBinder：
<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/pathset.jpg" width="620px" > </div><br>
```
SLF4J: Class path contains multiple SLF4J bindings.
SLF4J: Found binding in [jar:file:/D:/Program%20Files/apache-maven-3.5.4/repository/ch/qos/logback/logback-classic/1.2.3/logback-classic-1.2.3.jar!/org/slf4j/impl/StaticLoggerBinder.class]
SLF4J: Found binding in [jar:file:/D:/Program%20Files/apache-maven-3.5.4/repository/org/slf4j/slf4j-log4j12/1.7.25/slf4j-log4j12-1.7.25.jar!/org/slf4j/impl/StaticLoggerBinder.class]

```

每一种加载器加载指定的class文件会得到不同的类，因此为了能够使用，这里必须要保证LoggerFactory的类加载器与StaticLoggerBinder的类加载是相同的。

```
getResource("java/lang/Class.class");
```
其实上面的方式就是常用的类加载的方式。

StaticLoggerBinder.getSingleton();会进入logback-classic中类StaticLoggerBinder的初始化。然后获取LoggerFactory：
```
    public ILoggerFactory getLoggerFactory() {
        if (!initialized) {
            return defaultLoggerContext;
        }

        if (contextSelectorBinder.getContextSelector() == null) {
            throw new IllegalStateException("contextSelector cannot be null. See also " + NULL_CS_URL);
        }
        return contextSelectorBinder.getContextSelector().getLoggerContext();
    }
```

# 2.Logback是怎样找到logback.xml配置文件的？
第一次调用StaticLoggerBinder.getSingleton();会进入StaticLoggerBinder的静态初始化代码块：
```
public class StaticLoggerBinder implements LoggerFactoryBinder {

    /**
     * Declare the version of the SLF4J API this implementation is compiled
     * against. The value of this field is usually modified with each release.
     */
    // to avoid constant folding by the compiler, this field must *not* be final
    public static String REQUESTED_API_VERSION = "1.7.16"; // !final

    final static String NULL_CS_URL = CoreConstants.CODES_URL + "#null_CS";

    /**
     * The unique instance of this class.
     */
    private static StaticLoggerBinder SINGLETON = new StaticLoggerBinder();

    private static Object KEY = new Object();

    static {
        SINGLETON.init();
    }
```
```
    /**
     * Package access for testing purposes.
     */
    void init() {
        try {
            try {
                new ContextInitializer(defaultLoggerContext).autoConfig();
            } catch (JoranException je) {
                Util.report("Failed to auto configure default logger context", je);
            }
            // logback-292
            if (!StatusUtil.contextHasStatusListener(defaultLoggerContext)) {
                StatusPrinter.printInCaseOfErrorsOrWarnings(defaultLoggerContext);
            }
            contextSelectorBinder.init(defaultLoggerContext, KEY);
            initialized = true;
        } catch (Exception t) { // see LOGBACK-1159
            Util.report("Failed to instantiate [" + LoggerContext.class.getName() + "]", t);
        }
    }
```
autoConfig方法：
```
    public void autoConfig() throws JoranException {
        StatusListenerConfigHelper.installIfAsked(loggerContext);
        URL url = findURLOfDefaultConfigurationFile(true);
        if (url != null) {
            configureByResource(url);
        } else {
            Configurator c = EnvUtil.loadFromServiceLoader(Configurator.class);
            if (c != null) {
                try {
                    c.setContext(loggerContext);
                    c.configure(loggerContext);
                } catch (Exception e) {
                    throw new LogbackException(String.format("Failed to initialize Configurator: %s using ServiceLoader", c != null ? c.getClass()
                                    .getCanonicalName() : "null"), e);
                }
            } else {
                BasicConfigurator basicConfigurator = new BasicConfigurator();
                basicConfigurator.setContext(loggerContext);
                basicConfigurator.configure(loggerContext);
            }
        }
    }
```
findURLOfDefaultConfigurationFile按顺序查找配置文件：
```
    final public static String GROOVY_AUTOCONFIG_FILE = "logback.groovy";
    final public static String AUTOCONFIG_FILE = "logback.xml";
    final public static String TEST_AUTOCONFIG_FILE = "logback-test.xml";
    final public static String CONFIG_FILE_PROPERTY = "logback.configurationFile";
```
* step1.从logback.configurationFile指定的位置查找
* step2.查找logback-test.xml
* step3.查找logback.groovy
* step4.查找logback.xml
* step5.根据上面的autoConfig()，如果还没有找到，就使用BasicConfigurator
```
    public URL findURLOfDefaultConfigurationFile(boolean updateStatus) {
        ClassLoader myClassLoader = Loader.getClassLoaderOfObject(this);
        URL url = findConfigFileURLFromSystemProperties(myClassLoader, updateStatus);
        if (url != null) {
            return url;
        }

        url = getResource(TEST_AUTOCONFIG_FILE, myClassLoader, updateStatus);
        if (url != null) {
            return url;
        }

        url = getResource(GROOVY_AUTOCONFIG_FILE, myClassLoader, updateStatus);
        if (url != null) {
            return url;
        }

        return getResource(AUTOCONFIG_FILE, myClassLoader, updateStatus);
    }
```
其中findConfigFileURLFromSystemProperties查找属性CONFIG_FILE_PROPERTY=logback.configurationFile
```
    private URL findConfigFileURLFromSystemProperties(ClassLoader classLoader, boolean updateStatus) {
        String logbackConfigFile = OptionHelper.getSystemProperty(CONFIG_FILE_PROPERTY);
        if (logbackConfigFile != null) {
            URL result = null;
            try {
                result = new URL(logbackConfigFile);
                return result;
            } catch (MalformedURLException e) {
                // so, resource is not a URL:
                // attempt to get the resource from the class path
                result = Loader.getResource(logbackConfigFile, classLoader);
                if (result != null) {
                    return result;
                }
                File f = new File(logbackConfigFile);
                if (f.exists() && f.isFile()) {
                    try {
                        result = f.toURI().toURL();
                        return result;
                    } catch (MalformedURLException e1) {
                    }
                }
            } finally {
                if (updateStatus) {
                    statusOnResourceSearch(logbackConfigFile, classLoader, result);
                }
            }
        }
        return null;
    }
```

```
    private URL getResource(String filename, ClassLoader myClassLoader, boolean updateStatus) {
        URL url = Loader.getResource(filename, myClassLoader);
        if (updateStatus) {
            statusOnResourceSearch(filename, myClassLoader, url);
        }
        return url;
    }

```
将logback.xml放在resources下面的时候，获得的url为：
```
file:/E:/wz/projects/java/target/classes/logback.xml
```

# 3.logback.xml配置信息放在哪个数据结构中？

<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/Logger各种属性.png" width="520px" > </div><br>


这些配置信息是在doConfigure()的interpreter.getEventPlayer().play(eventList);也即play方法中进行处理的
* AppenderRefAction.begin()方法调用appenderAttachable.addAppender(appender)将appender加入到logback的Logger.aai中去
* <file>${appName}.log</file>调用NestedBasicPropertyIA.body()方法，最终调用RollingFileAppender.setFile()方法
* <rollingPolicy> 调用NestedComplexPropertyIA.end()方法，最终调用RollingFileAppender.setRollingPolicy
```
    public void autoConfig() throws JoranException {
        StatusListenerConfigHelper.installIfAsked(loggerContext);
        URL url = findURLOfDefaultConfigurationFile(true);
        if (url != null) {
            configureByResource(url);
```
```
    public void configureByResource(URL url) throws JoranException {
        if (url == null) {
            throw new IllegalArgumentException("URL argument cannot be null");
        }
        final String urlString = url.toString();
        if (urlString.endsWith("groovy")) {
            if (EnvUtil.isGroovyAvailable()) {
                // avoid directly referring to GafferConfigurator so as to avoid
                // loading groovy.lang.GroovyObject . See also http://jira.qos.ch/browse/LBCLASSIC-214
                GafferUtil.runGafferConfiguratorOn(loggerContext, this, url);
            } else {
                StatusManager sm = loggerContext.getStatusManager();
                sm.add(new ErrorStatus("Groovy classes are not available on the class path. ABORTING INITIALIZATION.", loggerContext));
            }
        } else if (urlString.endsWith("xml")) {
            JoranConfigurator configurator = new JoranConfigurator();
            configurator.setContext(loggerContext);
            configurator.doConfigure(url);
        } else {
            throw new LogbackException("Unexpected filename extension of file [" + url.toString() + "]. Should be either .groovy or .xml");
        }
    }
```
```
    public final void doConfigure(URL url) throws JoranException {
        InputStream in = null;
        try {
            informContextOfURLUsedForConfiguration(getContext(), url);
            URLConnection urlConnection = url.openConnection();
            // per http://jira.qos.ch/browse/LBCORE-105
            // per http://jira.qos.ch/browse/LBCORE-127
            urlConnection.setUseCaches(false);

            in = urlConnection.getInputStream();
            doConfigure(in, url.toExternalForm());
        } catch (IOException ioe) {
            String errMsg = "Could not open URL [" + url + "].";
            addError(errMsg, ioe);
            throw new JoranException(errMsg, ioe);
        } finally {
            if (in != null) {
                try {
                    in.close();
                } catch (IOException ioe) {
                    String errMsg = "Could not close input stream";
                    addError(errMsg, ioe);
                    throw new JoranException(errMsg, ioe);
                }
            }
        }
    }

```
```
    public final void doConfigure(InputStream inputStream, String systemId) throws JoranException {
        InputSource inputSource = new InputSource(inputStream);
        inputSource.setSystemId(systemId);
        doConfigure(inputSource);
    }
```
```
    public final void doConfigure(final InputSource inputSource) throws JoranException {

        long threshold = System.currentTimeMillis();
        // if (!ConfigurationWatchListUtil.wasConfigurationWatchListReset(context)) {
        // informContextOfURLUsedForConfiguration(getContext(), null);
        // }
        SaxEventRecorder recorder = new SaxEventRecorder(context);
        recorder.recordEvents(inputSource);
        doConfigure(recorder.saxEventList);
        // no exceptions a this level
        StatusUtil statusUtil = new StatusUtil(context);
        if (statusUtil.noXMLParsingErrorsOccurred(threshold)) {
            addInfo("Registering current configuration as safe fallback point");
            registerSafeConfiguration(recorder.saxEventList);
        }
    }
```
```
    public void doConfigure(final List<SaxEvent> eventList) throws JoranException {
        buildInterpreter();
        // disallow simultaneous configurations of the same context
        synchronized (context.getConfigurationLock()) {
            interpreter.getEventPlayer().play(eventList);
        }
    }
```
```
    public void play(List<SaxEvent> aSaxEventList) {
        eventList = aSaxEventList;
        SaxEvent se;
        for (currentIndex = 0; currentIndex < eventList.size(); currentIndex++) {
            se = eventList.get(currentIndex);

            if (se instanceof StartEvent) {
                interpreter.startElement((StartEvent) se);
                // invoke fireInPlay after startElement processing
                interpreter.getInterpretationContext().fireInPlay(se);
            }
            if (se instanceof BodyEvent) {
                // invoke fireInPlay before characters processing
                interpreter.getInterpretationContext().fireInPlay(se);
                interpreter.characters((BodyEvent) se);
            }
            if (se instanceof EndEvent) {
                // invoke fireInPlay before endElement processing
                interpreter.getInterpretationContext().fireInPlay(se);
                interpreter.endElement((EndEvent) se);
            }

        }
    }
```
<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/处理logback.jpg" width="620px" > </div><br>

进一步，addAppender是在play中解析出来的：
<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/addAppender是在play中解析出来.jpg" width="620px" > </div><br>

```
public class AppenderAttachableImpl<E> implements AppenderAttachable<E> {

    @SuppressWarnings("unchecked")
    final private COWArrayList<Appender<E>> appenderList = new COWArrayList<Appender<E>>(new Appender[0]);

    /**
     * Attach an appender. If the appender is already in the list in won't be
     * added again.
     */
    public void addAppender(Appender<E> newAppender) {
        if (newAppender == null) {
            throw new IllegalArgumentException("Null argument disallowed");
        }
        appenderList.addIfAbsent(newAppender);
    }

```

<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/appenderlist.jpg" width="520px" > </div><br>

也即将appender放在了logback的Logger的aai属性里面

<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/保存在contextSelector的defaultLoggerContext里面.jpg" width="620px" > </div><br>


最后获取LoggerFactory时,contextSelectorBinder.getContextSelector()获取的就是contextSelector：

```
    public static ILoggerFactory getILoggerFactory() {
        if (INITIALIZATION_STATE == UNINITIALIZED) {
            synchronized (LoggerFactory.class) {
                if (INITIALIZATION_STATE == UNINITIALIZED) {
                    INITIALIZATION_STATE = ONGOING_INITIALIZATION;
                    performInitialization();
                }
            }
        }
        switch (INITIALIZATION_STATE) {
        case SUCCESSFUL_INITIALIZATION:
            return StaticLoggerBinder.getSingleton().getLoggerFactory();
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

```
    public ILoggerFactory getLoggerFactory() {
        if (!initialized) {
            return defaultLoggerContext;
        }

        if (contextSelectorBinder.getContextSelector() == null) {
            throw new IllegalStateException("contextSelector cannot be null. See also " + NULL_CS_URL);
        }
        return contextSelectorBinder.getContextSelector().getLoggerContext();
    }
```

然后从中获取ILoggerFactory：
```
public class DefaultContextSelector implements ContextSelector {

    private LoggerContext defaultLoggerContext;

    public DefaultContextSelector(LoggerContext context) {
        this.defaultLoggerContext = context;
    }

    public LoggerContext getLoggerContext() {
        return getDefaultLoggerContext();
    }

    public LoggerContext getDefaultLoggerContext() {
        return defaultLoggerContext;
    }
```
<div align="center"> <img src="https://github.com/wz3118103/CS-Notes/blob/master/notes/pics/defaultLoggerContext.jpg" width="620px" > </div><br>

# 参考
* [使用ClassLoader加载资源详解](https://my.oschina.net/u/2474629/blog/1501101)
* [ClassLoader资源加载机制](https://plentymore.github.io/2019/01/12/ClassLoader%E8%B5%84%E6%BA%90%E5%8A%A0%E8%BD%BD%E6%9C%BA%E5%88%B6/)