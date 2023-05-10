##### 当前Flutter环境
```
Flutter 3.3.10 • channel stable • https://github.com/flutter/flutter.git
Framework • revision 135454af32 (3 months ago) • 2022-12-15 07:36:55 -0800
Engine • revision 3316dd8728
Tools • Dart 2.18.6 • DevTools 2.15.0
```

##### 背景
线上flutter开发的页面出现严重bug时需要修复：示例实现从左侧页面到右侧页面的动态加载

![logo](0.png ':size=300x600')  ![logo](1.png ':size=300x600')

##### 源码查看
###### io.flutter.embedding.engine.FlutterEngine
FlutterEngine的构造函数中最终会判断flutterjni是否挂载，第一次进入时会转入flutterloader中进行相关初始化步骤
```
public FlutterEngine(@NonNull Context context, @Nullable FlutterLoader flutterLoader, @NonNull FlutterJNI flutterJNI, @NonNull PlatformViewsController platformViewsController, @Nullable String[] dartVmArgs, boolean automaticallyRegisterPlugins, boolean waitForRestorationData) {
        ...
        if (!flutterJNI.isAttached()) {
            flutterLoader.startInitialization(context.getApplicationContext());
            flutterLoader.ensureInitializationComplete(context, dartVmArgs);
        }
        ...
    }

```
##### io.flutter.embedding.engine.loader.FlutterLoader
* startInitialization获取`FlutterApplicationInfo`，该类中`aotSharedLibraryName`即libapp.so，这就是flutter打包时业务代码最终的产物，这是一个hook点
* ensureInitializationComplete中shellArgs包含了一些启动Dart虚拟机的参数，其中`--aot-shared-library-name=`包含了加载业务so的名称：libapp.so，以及libapp.so的路径；最终调用到FlutterJNI.init这也是一个hook点

```
public void startInitialization(@NonNull Context applicationContext, @NonNull Settings settings) {
        if (this.settings == null) {
            if (Looper.myLooper() != Looper.getMainLooper()) {
                throw new IllegalStateException("startInitialization must be called on the main thread");
            } else {
                TraceSection.begin("FlutterLoader#startInitialization");

                try {
                    ...
                    this.flutterApplicationInfo = ApplicationInfoLoader.load(appContext);
                    ...
                } finally {
                    TraceSection.end();
                }

            }
        }
    }

public void ensureInitializationComplete(@NonNull Context applicationContext, @Nullable String[] args) {
        if (!this.initialized) {
            if (Looper.myLooper() != Looper.getMainLooper()) {
                throw new IllegalStateException("ensureInitializationComplete must be called on the main thread");
            } else if (this.settings == null) {
                throw new IllegalStateException("ensureInitializationComplete must be called after startInitialization");
            } else {
                TraceSection.begin("FlutterLoader#ensureInitializationComplete");

                try {
                    InitResult result = (InitResult)this.initResultFuture.get();
                    List<String> shellArgs = new ArrayList();
                    shellArgs.add("--icu-symbol-prefix=_binary_icudtl_dat");
                    shellArgs.add("--icu-native-lib-path=" + this.flutterApplicationInfo.nativeLibraryDir + File.separator + "libflutter.so");
                    if (args != null) {
                        Collections.addAll(shellArgs, args);
                    }

                    String kernelPath = null;
                    shellArgs.add("--aot-shared-library-name=" + this.flutterApplicationInfo.aotSharedLibraryName);
                    shellArgs.add("--aot-shared-library-name=" + this.flutterApplicationInfo.nativeLibraryDir + File.separator + this.flutterApplicationInfo.aotSharedLibraryName);
                    ...
                    this.initialized = true;
                } catch (Exception var20) {
                    Log.e("FlutterLoader", "Flutter initialization failed.", var20);
                    throw new RuntimeException(var20);
                } finally {
                    TraceSection.end();
                }

            }
        }
    }

```
##### 获取补丁so
* 修改main.dart将页面title修改成:Flutter Demo Home Page FIX
* flutter buidl apk --release编译出对应apk，解压apk获取对应对应abi下的libapp.so库，将libapp.so修改成libapp-fix.so表示是修复完的so
* 通过adb push libapp-fix.so到/sdcard
* app启动时将libapp-fixl.so拷贝到内部目录fileDirs

##### [Aspectjx](https://github.com/HujiangTechnology/gradle_plugin_android_aspectjx) hook相关函数
* hook ApplicationInfoLoader.load 将FlutterApplicationInfo的`aotSharedLibraryName`修改成libapp-fix.so
* hook FlutterJNI.init的入参进而修改启动Dart虚拟机的入参，该参数位于index为 2和3
![logo](2.png ':size=WIDTHxHEIGHT') 

```
@Aspect
public class AspectTarget {
    public static final String DYNAMIC_APP_SO = "libapp-fix.so";

    @Around("execution(* io.flutter.embedding.engine.loader.ApplicationInfoLoader.load(..))")
    public Object hookApplicationInfoLoader(ProceedingJoinPoint joinPoint) throws Throwable {
        Context context = (Context) joinPoint.getArgs()[0];
        ApplicationInfo appInfo = context.getApplicationInfo();
        //修改函数返回值
        return new FlutterApplicationInfo(DYNAMIC_APP_SO, getString(appInfo.metaData, PUBLIC_VM_SNAPSHOT_DATA_KEY), getString(appInfo.metaData, PUBLIC_ISOLATE_SNAPSHOT_DATA_KEY), getString(appInfo.metaData, PUBLIC_FLUTTER_ASSETS_DIR_KEY), getNetworkPolicy(appInfo, context), appInfo.nativeLibraryDir, getBoolean(appInfo.metaData, "io.flutter.automatically-register-plugins", true));
    }

    @Around("execution(* io.flutter.embedding.engine.FlutterJNI.init(..))")
    public void hookFlutterJNI(ProceedingJoinPoint joinPoint) throws Throwable {
        //修改函数入参
        Context context = (Context) joinPoint.getArgs()[0];
        String[] params = ((String[]) joinPoint.getArgs()[1]);
        params[2] = params[2].replace("libapp.so", DYNAMIC_APP_SO);
        params[3] = "--aot-shared-library-name=" + context.getFilesDir() + File.separator + DYNAMIC_APP_SO;
        joinPoint.proceed();
    }

    //以下代码从ApplicationInfoLoader拷贝出
    private static boolean getBoolean(Bundle metadata, String key, boolean defaultValue) {
        return metadata == null ? defaultValue : metadata.getBoolean(key, defaultValue);
    }

    
    private static String getNetworkPolicy(ApplicationInfo appInfo, Context context) {
        Bundle metadata = appInfo.metaData;
        if (metadata == null) {
            return null;
        } else {
            int networkSecurityConfigRes = metadata.getInt("io.flutter.network-policy", 0);
            if (networkSecurityConfigRes <= 0) {
                return null;
            } else {
                JSONArray output = new JSONArray();

                try {
                    XmlResourceParser xrp = context.getResources().getXml(networkSecurityConfigRes);
                    xrp.next();

                    for (int eventType = xrp.getEventType(); eventType != 1; eventType = xrp.next()) {
                        if (eventType == 2 && xrp.getName().equals("domain-config")) {
                            parseDomainConfig(xrp, output, false);
                        }
                    }
                } catch (XmlPullParserException | IOException var7) {
                    return null;
                }

                return output.toString();
            }
        }
    }

    private static void parseDomainConfig(XmlResourceParser xrp, JSONArray output, boolean inheritedCleartextPermitted) throws IOException, XmlPullParserException {
        boolean cleartextTrafficPermitted = xrp.getAttributeBooleanValue((String) null, "cleartextTrafficPermitted", inheritedCleartextPermitted);

        while (true) {
            while (true) {
                int eventType = xrp.next();
                if (eventType == 2) {
                    if (xrp.getName().equals("domain")) {
                        parseDomain(xrp, output, cleartextTrafficPermitted);
                    } else if (xrp.getName().equals("domain-config")) {
                        parseDomainConfig(xrp, output, cleartextTrafficPermitted);
                    } else {
                        skipTag(xrp);
                    }
                } else if (eventType == 3) {
                    return;
                }
            }
        }
    }

    private static void parseDomain(XmlResourceParser xrp, JSONArray output, boolean cleartextPermitted) throws IOException, XmlPullParserException {
        boolean includeSubDomains = xrp.getAttributeBooleanValue((String) null, "includeSubdomains", false);
        xrp.next();
        if (xrp.getEventType() != 4) {
            throw new IllegalStateException("Expected text");
        } else {
            String domain = xrp.getText().trim();
            JSONArray outputArray = new JSONArray();
            outputArray.put(domain);
            outputArray.put(includeSubDomains);
            outputArray.put(cleartextPermitted);
            output.put(outputArray);
            xrp.next();
            if (xrp.getEventType() != 3) {
                throw new IllegalStateException("Expected end of domain tag");
            }
        }
    }

    private static void skipTag(XmlResourceParser xrp) throws IOException, XmlPullParserException {
        String name = xrp.getName();

        for (int eventType = xrp.getEventType(); eventType != 3 || xrp.getName() != name; eventType = xrp.next()) {
        }

    }

    private static String getString(Bundle metadata, String key) {
        return metadata == null ? null : metadata.getString(key, (String) null);
    }
}
```
重新启动app，页面已经变成libapp-fix.so中的了。

