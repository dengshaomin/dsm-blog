
### 什么会触发 ANR？

通常，当应用无法响应用户输入时，系统即会显示 ANR。例如，如果应用在界面线程中屏蔽了某些 I/O 操作（通常是网络访问），导致系统无法处理传入的用户输入事件。或者，应用在界面线程中花费太多时间构建复杂的内存结构或计算游戏的下一个走法。确保高效的计算始终至关重要，但即使最高效的代码仍然需要时间来运行。

在您的应用面临任何可能需要执行冗长的操作的情况下，您不应在界面线程中执行这些操作，而是应该创建工作线程并在其中执行大部分操作。这样即可让界面线程（用于驱动界面事件循环）保持运行，并阻止系统断定您的代码已卡住。由于这种线程通常是在类级别完成的，因此您可以将响应能力视为一种类问题。（可将其与基本代码性能进行比较，后者是方法级问题。）

在 Android 中，应用响应性由 Activity 管理器和窗口管理器系统服务监控。当 Android 检测到以下某一项条件时，便会针对特定应用显示 ANR 对话框：

*   在 5 秒内对输入事件（例如按键或屏幕轻触事件）没有响应。
*   在 10 秒内尚未执行完毕。
*   Service 在20秒内尚未执行完毕。（官方文档并没有这一条）

官方文档：​https://developer.android.com/training/articles/perf-anr?hl=zh-cn

### ANR的表现形式
![](0.png ':size=500')

比较好奇为什么没有说Service的ANR，本篇记录就从Service启动过程源码分析发生ANR的前因后果

### Service启动源码分析
写一个简单的Service使用demo，并在AndroidManifest.xml中注册该 service
```
class MainActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)
        bindService(Intent(this, AnrService::class.java), object : ServiceConnection {
            override fun onServiceDisconnected(name: ComponentName?) {
            }
​
            override fun onServiceConnected(name: ComponentName?, service: IBinder?) {
            }
        }, Context.BIND_AUTO_CREATE)
    }
}
```
#### 1.ContextWrapper.bindService
```
@Override
    public boolean bindService(Intent service, ServiceConnection conn, int flags) {
        warnIfCallingFromSystemProcess();
        return bindServiceCommon(service, conn, flags, null, mMainThread.getHandler(), null,
                getUser());
    }
```
#### 2.ContextImpl.bindService
```
@Override
    public boolean bindService(Intent service, ServiceConnection conn, int flags) {
        warnIfCallingFromSystemProcess();
        return bindServiceCommon(service, conn, flags, null, mMainThread.getHandler(), null,
                getUser());
    }
```
#### 3.ContentImpl.bindServiceCommon
```
private boolean bindServiceCommon(Intent service, ServiceConnection conn, int flags,
            String instanceName, Handler handler, Executor executor, UserHandle user) {
        try {
            ...
            //跳转 AMS
            int res = ActivityManager.getService().bindIsolatedService(
                mMainThread.getApplicationThread(), getActivityToken(), service,
                service.resolveTypeIfNeeded(getContentResolver()),
                sd, flags, instanceName, getOpPackageName(), user.getIdentifier());
            ...
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    }
```
#### 4.AMS.bindIsolatedService
```
#ActivityManagerService.java
public int bindIsolatedService(IApplicationThread caller, IBinder token, Intent service,
            String resolvedType, IServiceConnection connection, int flags, String instanceName,
            String callingPackage, int userId) throws TransactionTooLargeException {
        ...
        synchronized(this) {
            return mServices.bindServiceLocked(caller, token, service,
                    resolvedType, connection, flags, instanceName, callingPackage, userId);
        }
    }
```
####    5.ActivityServices.bindServiceLocked
```
#ActivityServices.java
int bindServiceLocked(IApplicationThread caller, IBinder token, Intent service,
            String resolvedType, final IServiceConnection connection, int flags,
            String instanceName, String callingPackage, final int userId)
            throws TransactionTooLargeException {
            ...
            bringUpServiceLocked(s, service.getFlags(), callerFg, false,
                        permissionsReviewRequired)
            ...
            
}
```
####    6.ActivityServices.bringUpServiceLocked
```
 private String bringUpServiceLocked(ServiceRecord r, int intentFlags, boolean execInFg,
            boolean whileRestarting, boolean permissionsReviewRequired)
            throws TransactionTooLargeException {
            
            realStartServiceLocked(r, app, execInFg);
}
```
####    7.ActivityServices.realStartServiceLocked
```
private final void realStartServiceLocked(ServiceRecord r,
            ProcessRecord app, boolean execInFg) throws RemoteException {
        //发送service_timeout延迟消息
        bumpServiceExecutingLocked(r, execInFg, "create");
       ...
        try {
            //回调oncreate
            app.thread.scheduleCreateService(r, r.serviceInfo,
                    mAm.compatibilityInfoForPackage(r.serviceInfo.applicationInfo),
                    app.getReportedProcState());
            r.postNotification();
            created = true;
        } 
        ...
        //此函数中间接回调取消超时消息
        sendServiceArgsLocked(r, execInFg, true);
    }
```
#####   #bindServiceExecutingLocked
```
private final void bumpServiceExecutingLocked(ServiceRecord r, boolean fg, String why) {
        scheduleServiceTimeoutLocked(r.app);
    }
```
#####   #scheduleServiceTimeoutLocked
```
void scheduleServiceTimeoutLocked(ProcessRecord proc) {
        if (proc.executingServices.size() == 0 || proc.thread == null) {
            return;
        }
        Message msg = mAm.mHandler.obtainMessage(
                ActivityManagerService.SERVICE_TIMEOUT_MSG);
        msg.obj = proc;
        //发送timeout delay消息，此消息如果在
        mAm.mHandler.sendMessageDelayed(msg,
                proc.execServicesFg ? SERVICE_TIMEOUT : SERVICE_BACKGROUND_TIMEOUT);
    }
```
这个函数发送了SERVICE_TIMEOUT消息，并根据不同状态延迟消息时间不一样，延迟时间总共3种，分别是20s、200s、10s
```
// How long we wait for a service to finish executing.
    static final int SERVICE_TIMEOUT = 20*1000;

    // How long we wait for a service to finish executing.
    static final int SERVICE_BACKGROUND_TIMEOUT = SERVICE_TIMEOUT * 10;

    // How long the startForegroundService() grace period is to get around to
    // calling startForeground() before we ANR + stop it.
    static final int SERVICE_START_FOREGROUND_TIMEOUT = 10*1000;
```
当service在指定的时间内如果完成了所有的任务那么回去将这个timeout消息移除掉，否则在规定时间内未完成将触发service_timeout即ANR，这种延迟消息策略在android framwork层比较常见，比如说onClick事件中longClick的触发

#ApplicationThread.scheduleCreatedService

AMS通过ApplicationThreadProxy与主线程进行通信

```
public final void scheduleCreateService(IBinder token,
                ServiceInfo info, CompatibilityInfo compatInfo, int processState) {
            updateProcessState(processState, false);
            CreateServiceData s = new CreateServiceData();
            s.token = token;
            s.info = info;
            s.compatInfo = compatInfo;
            //给H发送create_service消息
            sendMessage(H.CREATE_SERVICE, s);
        }
```

#####   #H.handleMessage
```
 public void handleMessage(Message msg) {
            switch (msg.what) {
                
                case CREATE_SERVICE:
                    if (Trace.isTagEnabled(Trace.TRACE_TAG_ACTIVITY_MANAGER)) {
                        Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER,
                                ("serviceCreate: " + String.valueOf(msg.obj)));
                    }
                    handleCreateService((CreateServiceData)msg.obj);
                    break;
}
```

#####   #ActivityThread.handlerCreateService
在此函数中回调onCreate

```
@UnsupportedAppUsage
    private void handleCreateService(CreateServiceData data) {
        
        try {
            //创建 service
            ContextImpl context = ContextImpl.createAppContext(this, packageInfo);
            Application app = packageInfo.makeApplication(false, mInstrumentation);
            java.lang.ClassLoader cl = packageInfo.getClassLoader();
            service = packageInfo.getAppFactory()
                    .instantiateService(cl, data.info.name, data.intent);
            ...
            //attatch操作
            service.attach(context, this, data.info.name, data.token, app,
                    ActivityManager.getService());
            //调用oncreate
            service.onCreate();
            mServices.put(data.token, service);
            ...
        } catch (Exception e) {
            ...
        }
    }
```

#####   #sendServiceArgsLocked
```
private final void sendServiceArgsLocked(ServiceRecord r, boolean execInFg,
            boolean oomAdjusted) throws TransactionTooLargeException {
        ...
                serviceDoneExecutingLocked(r, inDestroying, inDestroying);
            ...
        }
    }
```
#####   #serviceDoneExecutingLocked
```
private void serviceDoneExecutingLocked(ServiceRecord r, boolean inDestroying,
            boolean finishing) {
        //移除超时消息
        mAm.mHandler.removeMessages(ActivityManagerService.SERVICE_TIMEOUT_MSG, r.app);             
    }
```

####    8.AMS.MainHandler
如果在延迟发送service timeout消息时间内service相关任务没有执行完成，那消息将会发送到AMS中的MainHandler中
```
final class MainHandler extends Handler {
        public MainHandler(Looper looper) {
            super(looper, null, true);
        }

        @Override
        public void handleMessage(Message msg) {
            switch (msg.what) {
            case SERVICE_TIMEOUT_MSG: {
                mServices.serviceTimeout((ProcessRecord)msg.obj);
                }
}
```
####    9.ActivityServices.serviceTimeout

处理ANR
```
void serviceTimeout(ProcessRecord proc) {
        ...
        if (anrMessage != null) {
        //响应anr
            proc.appNotResponding(null, null, null, null, false, anrMessage);
        }
    }
```

大致流程都分析完了，有一篇更详细的分析文章：http://gityuan.com/2016/03/06/start-service/

BroardCast、ContentProvider的ANR机制也应该大概如此。

### 快速定位anr

1.  如果是ANR问题 ， 则搜索“ANR”关键词 。快速定位到关键事件信息 。
2.  如果是ForceClosed(程序强制关闭) 和其它异常退出信息，则搜索"Fatal" 关键词， 快速定位到关键事件信息 。

***
https://juejin.cn/post/7171684761327370277#heading-5
***