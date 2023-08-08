随便来一段crash代码

![](0.png)

!>  留意一下这里导致crash的堆栈行号应该是第10、15行，后面retrace的时候会提到

####    UncaughtExceptionHandler捕获全局异常

```
class CrashHandler
private constructor() : UncaughtExceptionHandler {
  private var mDefaultHandler: UncaughtExceptionHandler? = null

  private var mContext: Context? = null

  //用来存储设备信息和异常信息
  private val infos: MutableMap<String, String> = HashMap()

  private val formatter: DateFormat = SimpleDateFormat("yyyy-MM-dd HH-mm-ss")
  private var cacheDir: File? = null
  fun init(context: Context) {
    mContext = context
    cacheDir = context.cacheDir
    //获取系统默认的UncaughtException处理器
    mDefaultHandler = Thread.getDefaultUncaughtExceptionHandler()
    //设置该CrashHandler为程序的默认处理器
    Thread.setDefaultUncaughtExceptionHandler(this)
  }

  /**
   * 当UncaughtException发生时会转入该函数来处理
   */
  override fun uncaughtException(thread: Thread, ex: Throwable) {
    handleException(ex)
    if (mDefaultHandler != null) {
      //如果用户没有处理则让系统默认的异常处理器来处理
      mDefaultHandler!!.uncaughtException(thread, ex)
    }
    //else {
    try {
      Thread.sleep(3000)
    } catch (e: InterruptedException) {
      Log.e(TAG, "error : ", e)
    }
    //退出程序
    //android.os.Process.killProcess(android.os.Process.myPid());
    //System.exit(1);
    //}
  }

  /**
   * 自定义错误处理,收集错误信息 发送错误报告等操作均在此完成.
   *
   * @return true:如果处理了该异常信息;否则返回false.
   */
  private fun handleException(ex: Throwable?): Boolean {
    if (ex == null) {
      return false
    }
    //使用Toast来显示异常信息
    //new Thread() {
    //  @Override
    //  public void run() {
    //    Looper.prepare();
    //    Toast.makeText(mContext, "很抱歉,程序出现异常,即将退出.", Toast.LENGTH_LONG).show();
    //    Looper.loop();
    //  }
    //}.start();
    //收集设备参数信息
    collectDeviceInfo(mContext)
    //保存日志文件
    saveCrashInfo2File(ex)
    return true
  }

  /**
   * 收集设备参数信息
   */
  fun collectDeviceInfo(ctx: Context?) {
    try {
      val pm = ctx!!.packageManager
      val pi = pm.getPackageInfo(ctx.packageName, PackageManager.GET_ACTIVITIES)
      if (pi != null) {
        val versionName = if (pi.versionName == null) "null" else pi.versionName
        val versionCode = pi.versionCode.toString() + ""
        infos["versionName"] = versionName
        infos["versionCode"] = versionCode
      }
    } catch (e: NameNotFoundException) {
      Log.e(TAG, "an error occured when collect package info", e)
    }
    val fields = Build::class.java.declaredFields
    for (field in fields) {
      try {
        field.isAccessible = true
        infos[field.name] = field[null].toString()
        Log.d(TAG, field.name + " : " + field[null])
      } catch (e: Exception) {
        Log.e(TAG, "an error occured when collect crash info", e)
      }
    }
  }

  /**
   * 保存错误信息到文件中
   *
   * @return 返回文件名称, 便于将文件传送到服务器
   */
  private fun saveCrashInfo2File(ex: Throwable): String? {
    val sb = StringBuffer()
    for ((key, value) in infos) {
      sb.append("$key=$value\n")
    }
    val writer: Writer = StringWriter()
    val printWriter = PrintWriter(writer)
    ex.printStackTrace(printWriter)
    var cause = ex.cause
    while (cause != null) {
      cause.printStackTrace(printWriter)
      cause = cause.cause
    }
    printWriter.close()
    val result = writer.toString()
    sb.append(result)
    try {
      val timestamp = System.currentTimeMillis()
      val time = formatter.format(Date())
      val fileName = "crash-$time.log"
      val fos = FileOutputStream(cacheDir!!.absolutePath + File.separator + fileName)
      fos.write(sb.toString().toByteArray())
      fos.close()
      return fileName
    } catch (e: Exception) {
      Log.e(TAG, "an error occured while writing file...", e)
    }
    return null
  }

  companion object {
    const val TAG = "CrashHandler"

    /** 获取CrashHandler实例 ,单例模式  */
    //CrashHandler实例
    val instance: CrashHandler by lazy(mode = LazyThreadSafetyMode.SYNCHRONIZED) {
      CrashHandler()
    }
  }
}
```
####    防止生成多份重复日志

上方code 26行处，大部分网文会加个判断如果自己处理失败了才交给系统默认的handler来处理，这样实践下来基本上自己处理都是成功的并不会走到交由默认处理这一步，通过runtime的日志发现很多ROM上会重启Application这样会导致生成了多份重复日志，所以上述代码中会写成自己处理的同时交由默认handler处理，应该是defaultHander中有抑制重启的代码调用；

```
if (!handleException(ex) && mDefaultHandler != null) {
      //如果用户没有处理则让系统默认的异常处理器来处理
      mDefaultHandler!!.uncaughtException(thread, ex)
    }
```
####    通过retrace.jar对堆栈进行反混淆

google对retrace的说明：https://developer.android.com/studio/build/shrink-code?hl=zh-cn#decode-stack-trace

为确保对堆栈轨迹进行轨迹还原时清楚明确，您应将以下规则添加到模块的 proguard-rules.pro 文件中：

```
-keepattributes LineNumberTable,SourceFile
-renamesourcefileattribute SourceFile
```

如果没有在 proguard-rules.pro 文件中添加，堆栈信息如下

![](1.png)

通过retrace.jar解析出来的堆栈信息行号、类名、方法名可能都不对

![](2.png)

加上之后的crash stack

![](3.png)

正确解析出来的堆栈信息

![](4.png)

####    R8对混淆的影响

在gradle 3.4之前是没有使用R8规则进行优化的，所以默认使Android/sdk/tools/proguard/bin下对应的retrace是能解析出来的，在3.5之后默认开启了R8的优化这时候需要是用R8对应的retrace才能正常解析出来了；

google对R8 retrace的说明：https://developer.android.com/studio/command-line/retrace?hl=zh-cn

r8 retrace 相关工具下载：https://android.googlesource.com/platform/prebuilts/r8/+/refs/heads/master ，点击tgz即可下载
如果不想使用R8可以在项目根目录的 gradle.properties 中写上 android.enableR8=false 即可，禁用过后编译、打包过程可能会报错，在 proguard-rules.pro 中写上 -ignorewarnings可以规避