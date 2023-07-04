

***
android 9
***

### SystemServer#main
由Android-init-zygote可知，分析AMS的启动应该从SystemServer#main方法开始：
```
// SystemServer.java
public final class SystemServer {
    ...
    public static void main(String[] args) {
        //先初始化SystemServer对象，再调用对象的run()方法
        new SystemServer().run();
    }
}

private void run() {
    Looper.prepareMainLooper();// 准备主线程looper

    //加载android_servers.so库，该库包含的源码在frameworks/base/services/目录下
    System.loadLibrary("android_servers");

    createSystemContext(); //初始化系统上下文

    //创建系统服务管理
    mSystemServiceManager = new SystemServiceManager(mSystemContext);
    LocalServices.addService(SystemServiceManager.class, mSystemServiceManager);

    //启动各种系统服务
    try {
        startBootstrapServices(); // 启动引导服务
        startCoreServices();      // 启动核心服务
        startOtherServices();     // 启动其他服务
    } catch (Throwable ex) {
        throw ex;
    }

    //一直循环执行
    Looper.loop();
    throw new RuntimeException("Main thread loop unexpectedly exited");
}
```

### 启动AMS服务
####    startBootstrapServices
```
private void startBootstrapServices() {
    mActivityManagerService = mSystemServiceManager.startService(
            ActivityManagerService.Lifecycle.class).getService();
    mActivityManagerService.setSystemServiceManager(mSystemServiceManager);
    mActivityManagerService.setInstaller(installer);
    mActivityManagerService.initPowerManagement();
    // Set up the Application instance for the system process and get started.
    mActivityManagerService.setSystemProcess();
}

// SystemServiceManager.startService
public <T extends SystemService> T startService(Class<T> serviceClass) {
    final T service;
    Constructor<T> constructor = serviceClass.getConstructor(Context.class);
    service = constructor.newInstance(mContext);
    startService(service);
    return service;
}

public void startService( final SystemService service) {
    // Register it.
    mServices.add(service);
    // Start it.
    long time = SystemClock.elapsedRealtime();
    try {
        service.onStart();
    } catch (RuntimeException ex) {
    }
}
```
mSystemServiceManager.startService方法主要便是创建ActivityManagerService.Lifecycle实例，然后调用其onStart方法。

####    AMS.Lifecycle
```
public static final class Lifecycle extends SystemService {
    private final ActivityManagerService mService;

    public Lifecycle(Context context) {
        super(context);
        mService = new ActivityManagerService(context);
    }

    
    public void onStart() {
        mService.start();
    }

    
    public void onBootPhase(int phase) {
        mService.mBootPhase = phase;
        if (phase == PHASE_SYSTEM_SERVICES_READY) {
            mService.mBatteryStatsService.systemServicesReady();
            mService.mServices.systemServicesReady();
        }
    }

    
    public void onCleanupUser(int userId) {
        mService.mBatteryStatsService.onCleanupUser(userId);
    }

    public ActivityManagerService getService() {
        return mService;
    }
}
```
####    AMS构造函数
```
public class ActivityManagerService extends IActivityManager.Stub
        implements Watchdog.Monitor, BatteryStatsImpl.BatteryCallback {

    // TAG_WITH_CLASS_NAME = false
    // static final String TAG_AM = "ActivityManager";
    private static final String TAG = TAG_WITH_CLASS_NAME ? "ActivityManagerService" : TAG_AM;

    public ActivityManagerService(Context systemContext) {
        mContext = systemContext;
        mSystemThread = ActivityThread.currentActivityThread();
        mUiContext = mSystemThread.getSystemUiContext();
        // 创建名为"ActivityManager"的前台线程
        mHandlerThread = new ServiceThread(TAG,
                THREAD_PRIORITY_FOREGROUND, false /*allowIo*/);
        mHandlerThread.start();
        mHandler = new MainHandler(mHandlerThread.getLooper());
        // 通过UiThread类，创建名为"android.ui"的线程
        mUiHandler = mInjector.getUiHandler(this);
        // 创建名为"ActivityManager:procStart"的前台线程
        mProcStartHandlerThread = new ServiceThread(TAG + ":procStart",
                THREAD_PRIORITY_FOREGROUND, false /* allowIo */);
        mProcStartHandlerThread.start();
        mProcStartHandler = new Handler(mProcStartHandlerThread.getLooper());

        if (sKillHandler == null) {
            // // 创建名为"ActivityManager:kill"的后台线程
            sKillThread = new ServiceThread(TAG + ":kill",
                    THREAD_PRIORITY_BACKGROUND, true /* allowIo */);
            sKillThread.start();
            sKillHandler = new KillHandler(sKillThread.getLooper());
        }
        // 创建目录/data/system
        File dataDir = Environment.getDataDirectory();
        File systemDir = new File(dataDir, "system");
        systemDir.mkdirs();

        mProcessStats = new ProcessStatsService(this, new File(systemDir, "procstats"));
        mAppOpsService = mInjector.getAppOpsService(new File(systemDir, "appops.xml"), mHandler);
        mGrantFile = new AtomicFile(new File(systemDir, "urigrants.xml"), "uri-grants");

        mStackSupervisor = createStackSupervisor();
        mStackSupervisor.onConfigurationChanged(mTempConfig);
        mRecentTasks = createRecentTasks();
        mStackSupervisor.setRecentTasks(mRecentTasks);
        mLifecycleManager = new ClientLifecycleManager();
        // 创建名为"CpuTracker"的线程
        mProcessCpuThread = new Thread("CpuTracker") {
            
            public void run() {
                // ...
            }
        };

        // WatchDog监控
        Watchdog.getInstance().addMonitor(this);
        Watchdog.getInstance().addThread(mHandler);
    }
}
```
####    AMS.start
```
private void start() {
    removeAllProcessGroups();   // 移除所有的进程组
    mProcessCpuThread.start();  // 启动CpuTracker线程

    mBatteryStatsService.publish(); // 启动电池统计服务
    mAppOpsService.publish(mContext);
    // 创建LocalService，并添加到LocalServices
    LocalServices.addService(ActivityManagerInternal.class, new LocalService());
    // ...
}

// LocalServices.addService
public static <T> void addService(Class<T> type, T service) {
    synchronized (sLocalServiceObjects) {
        if (sLocalServiceObjects.containsKey(type)) {
            throw new IllegalStateException("Overriding service registration");
        }
        sLocalServiceObjects.put(type, service);
    }
}

// LocalService
final class LocalService extends ActivityManagerInternal {
}

public abstract class ActivityManagerInternal {
}
```

####    AMS.setSystemProcess
```
public void setSystemProcess() {
    try {
        ServiceManager.addService(Context.ACTIVITY_SERVICE, this, /* allowIsolated= */ true,
                DUMP_FLAG_PRIORITY_CRITICAL | DUMP_FLAG_PRIORITY_NORMAL | DUMP_FLAG_PROTO);
        ServiceManager.addService(ProcessStats.SERVICE_NAME, mProcessStats);
        ServiceManager.addService("meminfo", new MemBinder(this), /* allowIsolated= */ false,
                DUMP_FLAG_PRIORITY_HIGH);
        ServiceManager.addService("gfxinfo", new GraphicsBinder(this));
        ServiceManager.addService("dbinfo", new DbBinder(this));
        if (MONITOR_CPU_USAGE) {
            ServiceManager.addService("cpuinfo", new CpuBinder(this),
                    /* allowIsolated= */ false, DUMP_FLAG_PRIORITY_CRITICAL);
        }
        ServiceManager.addService("permission", new PermissionController(this));
        ServiceManager.addService("processinfo", new ProcessInfoService(this));

        ApplicationInfo info = mContext.getPackageManager().getApplicationInfo(
                "android", STOCK_PM_FLAGS | MATCH_SYSTEM_ONLY);
        mSystemThread.installSystemApplicationInfo(info, getClass().getClassLoader());

        synchronized (this) {
            ProcessRecord app = newProcessRecordLocked(info, info.processName, false, 0);
            app.persistent = true;  // 设置为persistent进程
            app.pid = MY_PID;
            app.maxAdj = ProcessList.SYSTEM_ADJ;
            app.makeActive(mSystemThread.getApplicationThread(), mProcessStats);
            synchronized (mPidsSelfLocked) {
                mPidsSelfLocked.put(app.pid, app);
            }
            updateLruProcessLocked(app, false, null);
            updateOomAdjLocked();
        }
    } catch (PackageManager.NameNotFoundException e) {
        throw new RuntimeException(
                "Unable to find android system package", e);
    }
    // ...
}

// ActivityThread.java
public void installSystemApplicationInfo(ApplicationInfo info, ClassLoader classLoader) {
    synchronized (this) {
        // 调用了ContextImpl.installSystemApplicationInfo，然后调用了LoadedApk.installSystemApplicationInfo
        getSystemContext().installSystemApplicationInfo(info, classLoader);
        getSystemUiContext().installSystemApplicationInfo(info, classLoader);

        // give ourselves a default profiler
        mProfiler = new Profiler();
    }
}

void installSystemApplicationInfo(ApplicationInfo info, ClassLoader classLoader) {
    assert info.packageName.equals("android");
    mApplicationInfo = info; // 将包名为"android"的应用信息保存到mApplicationInfo
    mClassLoader = classLoader;
    mAppComponentFactory = createAppFactory(info, classLoader);
}
```
该方法主要工作是注册各种服务。

####    startOtherServices
```
private void startOtherServices() {
    mActivityManagerService.installSystemProviders();
    mActivityManagerService.setWindowManager(wm);
    mActivityManagerService.systemReady(new Runnable() {
        public void run() {
            // ...
        }
    });
}
```
####    AMS.installSystemProviders
```
public final void installSystemProviders() {
    List<ProviderInfo> providers;
    synchronized (this) {
        ProcessRecord app = mProcessNames.get("system", SYSTEM_UID);
        providers = generateApplicationProvidersLocked(app);
        if (providers != null) {
            for (int i=providers.size()-1; i>=0; i--) {
                ProviderInfo pi = (ProviderInfo)providers.get(i);
                if ((pi.applicationInfo.flags&ApplicationInfo.FLAG_SYSTEM) == 0) {
                    // 移除非系统的provider
                    providers.remove(i);
                }
            }
        }
    }
    if (providers != null) {
        // 安装所有的系统provider
        mSystemThread.installSystemProviders(providers);
    }
    // 创建一些Observer，用于监控Settings的改变
    mCoreSettingsObserver = new CoreSettingsObserver(this);
    mFontScaleSettingObserver = new FontScaleSettingObserver();
    mDevelopmentSettingsObserver = new DevelopmentSettingsObserver();
}
```

####    AMS.systemReady

AMS.systemReady()方法可以简单地分为三个部分：

*   before goingCallback;：进行一些准备工作，使系统和进程都处于ready状态；
*   goingCallback.run();：启动WebView，启动一些systemui等服务；
*   after goingCallback;：回调所有SystemService的onStartUser()方法；启动home Activity;发送广播USER_STARTED和USER_STARTING；恢复栈顶Activity;发送广播USER_SWITCHED等。

### 总结

AMS是系统的引导服务，应用进程的启动、切换和调度、四大组件的启动和管理都需要AMS的支持，AMS的功能比较繁多，本文只是简要介绍了一下AMS的启动流程，具体的可以结合Activity等组件启动流程分析。

***
https://ljd1996.github.io/2020/04/01/Android-AMS%E5%90%AF%E5%8A%A8%E5%8E%9F%E7%90%86/
***