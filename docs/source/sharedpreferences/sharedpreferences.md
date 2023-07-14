***
Android 9
***

### 概述
注：本文基于Android 9源码，为了文章的简洁性，引用源码的地方可能有所删减。

SharedPreference 是 Android 提供的一种简单易用的轻量级存储方式，本质是一个以 key-value 方式保存数据的xml文件，存储路径为: /data/data/$pkg/shared_prefs。但是 SharedPreference 存在着一些缺陷，如存在多进程的安全问题，以及 ANR(即使 apply 也可能会 ANR) 等。目前可供替换的方案有腾讯的第三方库–MMKV，以及官方的 Jetpack DataStore 组件。

### 创建实例
```
// ContextImpl
public SharedPreferences getSharedPreferences(String name, int mode) {
    // ...
    File file;
    synchronized (ContextImpl.class) {
        if (mSharedPrefsPaths == null) {
            mSharedPrefsPaths = new ArrayMap<>();
        }
        file = mSharedPrefsPaths.get(name);
        if (file == null) {
            file = getSharedPreferencesPath(name);
            mSharedPrefsPaths.put(name, file);
        }
    }
    // 根据 name 对应的 xml 文件获取 SP 实例
    return getSharedPreferences(file, mode);
}

public File getSharedPreferencesPath(String name) {
    // /data/data/$pkg/shared_prefs/$name.xml
    return makeFilename(getPreferencesDir(), name + ".xml");
}

public SharedPreferences getSharedPreferences(File file, int mode) {
    SharedPreferencesImpl sp;
    synchronized (ContextImpl.class) {
        // 通过 packageName 获取/创建 cache 实例
        final ArrayMap<File, SharedPreferencesImpl> cache = getSharedPreferencesCacheLocked();
        sp = cache.get(file);
        if (sp == null) {
            // ...
            sp = new SharedPreferencesImpl(file, mode);
            cache.put(file, sp);
            return sp;
        }
    }
    // ...
    return sp;
}
```
SharedPreferences 只是一个接口，具体实现是 SharedPreferencesImpl, 我们看看 SharedPreferencesImpl 的构造方法：
```
SharedPreferencesImpl(File file, int mode) {
    mFile = file;
    mBackupFile = makeBackupFile(file); // 根据file创建一个.bak的File对象
    mMode = mode;
    mLoaded = false;
    mMap = null;
    mThrowable = null;
    startLoadFromDisk();
}

private void startLoadFromDisk() {
    synchronized (mLock) { // 加锁
        mLoaded = false;
    }
    new Thread("SharedPreferencesImpl-load") {
        public void run() {
            loadFromDisk();
        }
    }.start();
}

private void loadFromDisk() {
    synchronized (mLock) {
        if (mLoaded) {
            return; // 已经加载完毕则直接返回
        }
        if (mBackupFile.exists()) {
            // 如果备份文件存在，则删除源文件，并将备份文件重命名为源文件
            mFile.delete();
            mBackupFile.renameTo(mFile);
        }
    }

    Map<String, Object> map = null;
    StructStat stat = null;
    Throwable thrown = null;
    try {
        stat = Os.stat(mFile.getPath()); // 获取文件信息
        // 将 xml 中的数据存入 map
    } catch (ErrnoException e) {
    } catch (Throwable t) {
        thrown = t;
    }

    synchronized (mLock) {
        mLoaded = true; // 标记已经加载过了
        mThrowable = thrown;
        try {
            // 将 map 赋值给 mMap，并更新文件访问时间和文件大小
        } catch (Throwable t) {
            mThrowable = t;
        } finally {
            mLock.notifyAll(); // notify所有等待 mLock 锁的线程
        }
    }
}
```
SP 实例化时会将 xml 文件中的数据放入内存中的 mMap 中，这样以后的每次读数据都是从这个 Map 中读取，而不需要每次都读文件。然而当 xml 中数据量比较大时，可能会产生高内存占用的风险，然而实际上，如果 SP 中保存的只是一些基础类型数据如 int, boolean 等，则很难有这么大的数据量，而如果存储的是一些复杂数据序列化后的字符串，当然容易占用过高的内存，但这也违背了 SharedPreferences 设计的初衷 – 轻量级存储，这种大容量的数据本身就不应该用 SP 来持久化！

### 读操作

以 getString 为例：
```
public String getString(String key,  String defValue) {
    synchronized (mLock) {
        awaitLoadedLocked();
        String v = (String)mMap.get(key);
        return v != null ? v : defValue;
    }
}

private void awaitLoadedLocked() {
    while (!mLoaded) {
        try {
            mLock.wait(); // 等待 mLock 被 notify
        } catch (InterruptedException unused) {
        }
    }
    if (mThrowable != null) { // mThrowable不为null则抛出异常
        throw new IllegalStateException(mThrowable);
    }
}
```
awaitLoadedLocked 方法作用是在同步块中等待 SP 加载完成，那么如果 xml 文件很大，加载 SP 比较慢，那么 getXXX 方法就会一直阻塞当前线程！

### 写操作
首先看下 SP.edit 方法：
```
public Editor edit() {
    synchronized (mLock) {
        awaitLoadedLocked(); // 等待加载完成
    }
    return new EditorImpl();
}
```

然后看下 EditorImpl.putString/remove/clear 方法：
```
public Editor putString(String key,  String value) {
    synchronized (mEditorLock) {
        mModified.put(key, value);
        return this;
    }
}


public Editor remove(String key) {
    synchronized (mEditorLock) {
        mModified.put(key, this); // put this 对象
        return this;
    }
}


public Editor clear() {
    synchronized (mEditorLock) {
        mClear = true;
        return this;
    }
}
```
####    commitToMemory
在看 apply 和 commit 方法之前先看看 commitToMemory 方法，这个方法加了两级锁: SharedPreferencesImpl.mLock 和 EditorImpl.mEditorLock 锁，因此在 commit 或 apply 时任何 getXXX 方法都会 block。

```
private MemoryCommitResult commitToMemory() {
    synchronized (SharedPreferencesImpl.this.mLock) {// mLock 锁
        // 当有多个写操作等待执行时深拷贝一份map
        if (mDiskWritesInFlight > 0) {
            mMap = new HashMap<String, Object>(mMap);
        }
        mapToWriteToDisk = mMap;
        mDiskWritesInFlight++;// 自增表示多了一个未完成的操作

        boolean hasListeners = mListeners.size() > 0;
        if (hasListeners) {
            keysModified = new ArrayList<String>();
            listeners = new HashSet<OnSharedPreferenceChangeListener>(mListeners.keySet());
        }

        synchronized (mEditorLock) { // mEditorLock 锁
            boolean changesMade = false;
            if (mClear) { // 处理调用 clear 的情况
                if (!mapToWriteToDisk.isEmpty()) {
                    changesMade = true;
                    mapToWriteToDisk.clear();
                }
                mClear = false;
            }
            for (Map.Entry<String, Object> e : mModified.entrySet()) { // 遍历 mModified
                String k = e.getKey();
                Object v = e.getValue();
                // "this" 是一个特殊的 value，上面remove会将其put进来
                if (v == this || v == null) {
                    if (!mapToWriteToDisk.containsKey(k)) {
                        continue;
                    }
                    mapToWriteToDisk.remove(k);
                } else {
                    if (mapToWriteToDisk.containsKey(k)) {
                        Object existingValue = mapToWriteToDisk.get(k);
                        if (existingValue != null && existingValue.equals(v)) {
                            continue;
                        }
                    }
                    mapToWriteToDisk.put(k, v);
                }
                changesMade = true; // 表示有更新产生
                if (hasListeners) {
                    keysModified.add(k);
                }
            }
            mModified.clear(); // apply 或 commit 执行完后清空 mModified, 准备接下来的put操作
            if (changesMade) { // 有更新产生则自增
                mCurrentMemoryStateGeneration++;
            }
            memoryStateGeneration = mCurrentMemoryStateGeneration;
        }
    }
    return new MemoryCommitResult(memoryStateGeneration, keysModified, listeners, mapToWriteToDisk);
}
```
####    apply
```
public void apply() {
    final long startTime = System.currentTimeMillis();
    final MemoryCommitResult mcr = commitToMemory();
    final Runnable awaitCommit = new Runnable() {
            
            public void run() {
                try { // block等待写操作完成
                    mcr.writtenToDiskLatch.await();
                } catch (InterruptedException ignored) {
                }
            }
        };
    QueuedWork.addFinisher(awaitCommit); // 将 awaitCommit 加入 QueuedWork

    Runnable postWriteRunnable = new Runnable() {
            
            public void run() {
                awaitCommit.run();// 执行 awaitCommit 并从 QueueWork 中移除
                QueuedWork.removeFinisher(awaitCommit);
            }
        };
    SharedPreferencesImpl.this.enqueueDiskWrite(mcr, postWriteRunnable);// 准备将 mcr 写到磁盘中
    notifyListeners(mcr);
}
```
####    commit
```
public boolean commit() {
    long startTime = 0;
    MemoryCommitResult mcr = commitToMemory();

    SharedPreferencesImpl.this.enqueueDiskWrite(mcr, null /* sync write on this thread okay */);
    try {// block等待写操作完成，如果是UI线程则可能会阻塞UI
        mcr.writtenToDiskLatch.await();
    } catch (InterruptedException e) {
        return false;
    } finally {
    }
    notifyListeners(mcr);
    return mcr.writeToDiskResult;
}
```
####    enqueueDiskWrite
```
private void enqueueDiskWrite(final MemoryCommitResult mcr, final Runnable postWriteRunnable) {
    final boolean isFromSyncCommit = (postWriteRunnable == null); // 根据 postWriteRunnable 判断是否异步
    final Runnable writeToDiskRunnable = new Runnable() {
            
            public void run() {
                synchronized (mWritingToDiskLock) { // 第三把锁保护写操作
                    writeToFile(mcr, isFromSyncCommit);
                }
                synchronized (mLock) { // 加 mLock 锁
                    mDiskWritesInFlight--; // 表示一个写操作完成
                }
                if (postWriteRunnable != null) postWriteRunnable.run();
            }
        };
    if (isFromSyncCommit) { // commit
        boolean wasEmpty = false;
        synchronized (mLock) {
            wasEmpty = mDiskWritesInFlight == 1;
        }
        if (wasEmpty) { // 直接在当前线程执行
            writeToDiskRunnable.run();
            return;
        }
    }
    // apply - 异步执行
    QueuedWork.queue(writeToDiskRunnable, !isFromSyncCommit);
}

private void writeToFile(MemoryCommitResult mcr, boolean isFromSyncCommit) {
    boolean fileExists = mFile.exists();
    if (fileExists) {
        // 根据 memoryStateGeneration 判断是否有改动需要写文件，不需要则直接返回
        // ...
        boolean backupFileExists = mBackupFile.exists();
        if (!backupFileExists) {// 备份文件不存在则备份，防止本次写失败破坏文件
            if (!mFile.renameTo(mBackupFile)) {
                mcr.setDiskWriteResult(false, false);
                return;
            }
        } else {// 备份文件存在则删除 mFile, 因此本次会写入新的 mFile
            mFile.delete();
        }
    }

    try {
        FileOutputStream str = createFileOutputStream(mFile);
        if (str == null) {
            mcr.setDiskWriteResult(false, false);
            return;
        }
        // 写文件，写成功则删除备份文件
        // ...
        return;
    } catch (XmlPullParserException e) {
    } catch (IOException e) {
    }
    // 运行到这里表示写失败，删除 mFile 并标记false
    mcr.setDiskWriteResult(false, false);
}

// MemoryCommitResult
void setDiskWriteResult(boolean wasWritten, boolean result) {
    this.wasWritten = wasWritten;
    writeToDiskResult = result;
    writtenToDiskLatch.countDown(); // 释放等待锁
}
```

####    小结
当每次调用 putXXX 时，都是将数据存入到内存里的 Map 中，等到调用 apply 或 commit 的时候，才会将 Map 中的数据持久化到 xml 中。另外，如果开发者只调用 putXXX 方法往 Map 中存入数据，而没有调用 apply/commit 方法持久化，为了防止 Map 中的数据被 getXXX 方法获取，设计者在 Editor 类中也使用了一个 Map 对象来存储这些 putXXX 存入的值，直到 apply/commit 被调用后，才将 Editor 中的 Map 合入 SP 中的 Map, 然后持久化。

当由于一些原因导致 SP 的写操作异常终止时，xml 文件可能已经被破坏了，因此 SP 采用了文件备份机制来处理这种情况：SharedPreferences 写操作执行之前会对文件进行备份(.bak)，当写操作执行成功后会删除这个 bak 文件，反之若发生异常，则在下次实例化 SP 时会检查是否存在 bak 文件，若存在则直接将备份文件重命名为原文件。

### 线程安全
为了保证线程安全，SharedPreferences 一共使用了三把锁：
```
final class SharedPreferencesImpl implements SharedPreferences {
    // 加锁顺序:
    //  - acquire SharedPreferencesImpl.mLock before EditorImpl.mLock
    //  - acquire mWritingToDiskLock before EditorImpl.mLock
    private final Object mLock = new Object(); // 第一把
    private final Object mWritingToDiskLock = new Object(); // 第二把

    // 使用 GuardedBy 注解标记该对象应该持有哪把锁
    
    private Map<String, Object> mMap;

    
    private long mDiskStateGeneration;

    public final class EditorImpl implements Editor {
        private final Object mEditorLock = new Object(); // 第三把

        
        private final Map<String, Object> mModified = new HashMap<>();
    }
}
```
####    读操作的锁：

读操作的原理是读取内存中 mMap 的值并返回，因此只需要加一把锁保证 mMap 的线程安全即可：
```
public String getString(String key,  String defValue) {
    synchronized (mLock) {
        awaitLoadedLocked();
        String v = (String)mMap.get(key);
        return v != null ? v : defValue;
    }
}
```
####    写操作的锁：

首先对于 Editor 的 put 操作而言，需要一把锁来保证线程安全：
```
public Editor putString(String key,  String value) {
    synchronized (mEditorLock) {
        mModified.put(key, value);
        return this;
    }
}
```
然后在执行 apply 时，需要加锁保证 mEditorMap 和 mMap 合并的安全：
```
private MemoryCommitResult commitToMemory() {
    synchronized (SharedPreferencesImpl.this.mLock) {
        // handle mMap, mDiskWritesInFlight ...
        synchronized (mEditorLock) {
            // merge
        }
    }
}
```
最后将 mMap 写入 xml 文件也需要加锁保证安全：
```
private void enqueueDiskWrite(final MemoryCommitResult mcr, final Runnable postWriteRunnable) {
    // ...
    synchronized (mWritingToDiskLock) {
        writeToFile(mcr, isFromSyncCommit);
    }
}
```
####    进程安全
SharedPreferences 是进程不安全的，对于如何保证 SharedPreferences 的进程安全性，可以通过一些方法：

*   文件锁：java.nio.channels.FileLock 可以获取文件指定部分的锁(独占或共享)。读文件时使用共享锁，写文件时使用独占锁。参考 https://blog.csdn.net/qq_27512671/article/details/101445642 ；
*   使用 ContentProvider 实现 SharedPreferences 跨进程共享数据，可以用 ContentProvider 的 update 或 insert 方法实现 putXXX 方法，用 delete 实现 clean 和 remove 方法，用 getType 或者 query 实现 get 和 getAll 方法；
*   等等

### apply引起的ANR

####    ANR产生原因
在阅读 SharedPreferences apply 写操作时，有以下代码：
```
public void apply() {
    final long startTime = System.currentTimeMillis();
    final MemoryCommitResult mcr = commitToMemory();
    final Runnable awaitCommit = new Runnable() {
            
            public void run() {
                try { // block等待写操作完成
                    mcr.writtenToDiskLatch.await();
                } catch (InterruptedException ignored) {
                }
            }
        };
    QueuedWork.addFinisher(awaitCommit); // 将 awaitCommit 加入 QueuedWork

    Runnable postWriteRunnable = new Runnable() {
            
            public void run() {
                awaitCommit.run();// 执行 awaitCommit 并从 QueueWork 中移除
                QueuedWork.removeFinisher(awaitCommit);
            }
        };
    SharedPreferencesImpl.this.enqueueDiskWrite(mcr, postWriteRunnable);// 准备将 mcr 写到磁盘中
    notifyListeners(mcr);
}
```
这里将 awaitCommit 添加到了 QueuedWork 的 Finishers 中，然后在 enqueueDiskWrite 方法中有如下代码(只留下关键代码)：
```
private void enqueueDiskWrite(final MemoryCommitResult mcr, final Runnable postWriteRunnable) {
    final Runnable writeToDiskRunnable = new Runnable() {
        
        public void run() {
            synchronized (mWritingToDiskLock) {
                // 在这个方法中会执行 writtenToDiskLatch.countDown 释放锁
                writeToFile(mcr, isFromSyncCommit);
            }
            if (postWriteRunnable != null) {
                postWriteRunnable.run();
            }
        }
    };
    QueuedWork.queue(writeToDiskRunnable, !isFromSyncCommit);
}
```
然后我们看一下 QueuedWork 在 queue 后是怎么工作的：
```
public class QueuedWork {
    private static final long DELAY = 100;
    private static boolean sCanDelay = true;

    public static void queue(Runnable work, boolean shouldDelay) {
        Handler handler = getHandler(); // 获取名为 queued-work-looper 的线程的 handler

        synchronized (sLock) {
            sWork.add(work); // 将 work 添加到任务列表
            if (shouldDelay && sCanDelay) {
                handler.sendEmptyMessageDelayed(QueuedWorkHandler.MSG_RUN, DELAY);
            } else {
                handler.sendEmptyMessage(QueuedWorkHandler.MSG_RUN);
            }
        }
    }
}
```
queued-work-looper 线程 handler 接收到 MSG_RUN 后会调用 processPendingWork 方法来依次执行 sWork 中的任务。按照这个逻辑，handler处理了消息后，执行 writeToFile 写入操作完成后会释放 writtenToDiskLatch 锁，然后调用 writtenToDiskLatch.await 的等待操作不会阻塞当前线程(queued-work-looper)，也不会产生 ANR。

注意到上面 QueuedWork.addFinisher(awaitCommit) 将 awaitCommit 加入了 Finishers 列表，那么这个 Finisher 是什么时候会被执行呢？
```
public static void addFinisher(Runnable finisher) {
    synchronized (sLock) {
        sFinishers.add(finisher);
    }
}
```
接着看 QueuedWork 的源码，发现 sFinishers 中的任务被执行是在 QueuedWork.waitToFinish 方法中，这个方法的注释中有如下片段: Is called from the Activity base class's onPause(), after BroadcastReceiver's onReceive, after Service command handling, etc. (so async work is never lost)。即在上面这些场景中，会回调 waitToFinish 方法，我们看下它的源码：
```
public static void waitToFinish() {
    boolean hadMessages = false;
    Handler handler = getHandler();

    synchronized (sLock) {
        if (handler.hasMessages(QueuedWorkHandler.MSG_RUN)) {
            // Delayed work will be processed at processPendingWork() below
            handler.removeMessages(QueuedWorkHandler.MSG_RUN);
        }
        // We should not delay any work as this might delay the finishers
        sCanDelay = false;
    }
    processPendingWork();

    try {
        while (true) {
            Runnable finisher;
            synchronized (sLock) {
                finisher = sFinishers.poll();
            }
            if (finisher == null) {
                break;
            }
            finisher.run();
        }
    } finally {
        sCanDelay = true;
    }
}
```
可以在 ActivityThread 中看到在 handleServiceArgs, handleStopService, handlePauseActivity, handleStopActivity, handleSleeping 中都有调用到 waitToFinish 方法，而这些方法都会分别调用到 Service.onStartCommand, Service.onDestroy, Activity.onPause, Activity.onStop 等生命周期。再根据 waitToFinish 方法的其它注释，大概可以猜到这样设计的原因是当 APP 发生崩溃等异常时，尽可能保证数据持久化成功, waitToFinish 方法会在当前线程立即执行 sWork 中的任务，然后依次执行 sFinishers 中的任务。即在主线程会执行 waitToFinish 方法，自然会有产生 ANR 的可能！

####    避免ANR
从上面 ANR 的原因分析可以知道，此类 ANR 都是在主线程调用 QueuedWork.waitToFinish() 触发的，因此可以在调用此函数之前，将 QueuedWork 中的 sFinishers 列表清空。

结合 Android-Activity启动源码解读, Android-Service启动源码解读, Android-Broadcast机制原理, Android-ContentProvider源码解读 可以知道，上面 handlePauseActivity, handleStopActivity 等都是通过 ClientTransaction机制 调用的，可以从这里出发，Hook 相关的逻辑，在调用 handlePauseActivity 等方法之前清除 sFinishers 队列。剖析SharedPreference apply引起的ANR问题-字节跳动技术团队 上的解决方法在 Android 9 上已经过时了，Android 9 上 AMS 回调 Activity 生命周期不再通过 H 类来直接调用。

示例如下：
```
fun hook() {
    val atCls = Class.forName("android.app.ActivityThread")
    atCls.getDeclaredMethod("currentActivityThread")?.let { currentActivityThread ->
        currentActivityThread.invoke(null)?.let { at ->
            val mH = atCls.getDeclaredField("mH")
            mH.isAccessible = true
            val cb = Handler::class.java.getDeclaredField("mCallback")
            cb.isAccessible = true
            cb.set(mH.get(at), object : Handler.Callback {
                override fun handleMessage(msg: Message): Boolean {
                    when (msg.what) {
                        159 -> {
                            handler(msg.obj)
                        }
                    }
                    return false
                }
            })
        }
    }
}

private fun handler(transaction: Any) {
    val transCls = Class.forName("android.app.servertransaction.ClientTransaction")
    val lifecycleStateRequestM = transCls.getDeclaredMethod("getLifecycleStateRequest")
    val lifecycleStateRequest = lifecycleStateRequestM.invoke(transaction)
    val pauseActivityItem = Class.forName("android.app.servertransaction.PauseActivityItem")
    // onPause
    if (pauseActivityItem.isAssignableFrom(lifecycleStateRequest.javaClass)) {
        // do clear work
    }
    // onStop ...
}
```
### 总结
SharedPreferences 的坑：

*   getXXX 方法可能会导致主线程阻塞
*   不能保证类型安全(血泪…)
*   加载的数据一直存在内存中
*   apply 方法可能造成 ANR
*   跨进程不安全

***
https://ljd1996.github.io/2020/10/29/Android-SharedPreferences%E6%BA%90%E7%A0%81%E8%A7%A3%E8%AF%BB/
***