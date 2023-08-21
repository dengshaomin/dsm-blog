### 环境
***
*   mac
*   agp：7.5
***

新建AndroidStudio项目，新建type为了java or kotlin library，这里取名为jiagu

![](0.png ':size=50%')

新建两个class，JiaGu.kt和JiaGu.java用来展示kotlin和java两种不同语言类型作为入口函数类的区别

### 使用java类作为主函数入口
```
package com.android.jiagu;

public class JiaGuJava {
    public static void main(String[] args){
        StringBuilder params = new StringBuilder();
        for(int i = 0;i< args.length;i++){
            params.append(args[i]);
            params.append("\n");
        }
        System.out.println(params);
    }
}
```
修改library目录下build.gradle，通过jar指定主class
```
jar{
    manifest {
        attributes 'Main-Class': 'com.android.jiagu.JiaGuJava'
    }
}
```
在gradle任务面板中执行jiagu目录下的jar task，会在jiagu\build\libs下生成jiagu.jar

![](1.png ':size=30%')

iTerm中运行jar并制定两个参数：
```
java -jar jiagu/build/libs/jiagu.jar -i=123 -o=234
```

结果输出如下：
```
-i=123
-o=234
```

!> gradle中的jar命令会自动生成jar配置文件MANIFEST.MF，解压jar包在META-INF文件下

### 使用Kotlin 类作为主函数入口
```
package com.android.jiagu

object JiaGu {
    @JvmStatic
    fun main(args: Array<String>) {
        val params = StringBuilder()
        for (i in args.indices) {
            params.append(args[i])
            params.append("\n")
        }
        println(params)
    }
}
```


修改library目录下build.gradle，通过jar指定主class
```
jar{
    manifest {
        attributes 'Main-Class': 'com.android.jiagu.JiaGu'
    }
    from { configurations.runtimeClasspath.collect { it.isDirectory() ? it : zipTree(it) } }
    duplicatesStrategy = DuplicatesStrategy.EXCLUDE
}
```
这里和java版的区别在于多了from和duplicatesStrategy的配置，如果没有from配置将会遇到下面这个问题：
```
Exception in thread "main" java.lang.NoClassDefFoundError: kotlin/jvm/internal/Intrinsics
	at com.android.jiagu.JiaGu.main(JiaGu.kt)
Caused by: java.lang.ClassNotFoundException: kotlin.jvm.internal.Intrinsics
	at java.net.URLClassLoader.findClass(URLClassLoader.java:387)
	at java.lang.ClassLoader.loadClass(ClassLoader.java:418)
	at sun.misc.Launcher$AppClassLoader.loadClass(Launcher.java:355)
	at java.lang.ClassLoader.loadClass(ClassLoader.java:351)
	... 1 more
```
反编译jiagu.jar看源码，Intrinsics这个class是Kotlin转java的时候自动生成的
```
package com.android.jiagu;

import kotlin.Metadata;
import kotlin.jvm.JvmStatic;
import kotlin.jvm.internal.Intrinsics;

/* compiled from: JiaGu.kt */
@Metadata(d1 = {"\u0000\u001e\n\u0002\u0018\u0002\n\u0002\u0010\u0000\n\u0002\b\u0002\n\u0002\u0010\u0002\n\u0000\n\u0002\u0010\u0011\n\u0002\u0010\u000e\n\u0002\b\u0002\bÆ\u0002\u0018\u00002\u00020\u0001B\u0007\b\u0002¢\u0006\u0002\u0010\u0002J\u001b\u0010\u0003\u001a\u00020\u00042\f\u0010\u0005\u001a\b\u0012\u0004\u0012\u00020\u00070\u0006H\u0007¢\u0006\u0002\u0010\b¨\u0006\t"}, d2 = {"Lcom/android/jiagu/JiaGu;", "", "()V", "main", "", "args", "", "", "([Ljava/lang/String;)V", "jiagu"}, k = 1, mv = {1, 8, 0}, xi = 48)
/* loaded from: classes2.dex */
public final class JiaGu {
    public static final JiaGu INSTANCE = new JiaGu();

    private JiaGu() {
    }

    @JvmStatic
    public static final void main(String[] args) {
        Intrinsics.checkNotNullParameter(args, "args");
        StringBuilder params = new StringBuilder();
        for (String str : args) {
            params.append(str);
            params.append("\n");
        }
        System.out.println(params);
    }
}
```
from那行是把kotlin相关依赖都打进jar包，duplicatesStrategy那行是去除jar包中的重复依赖，如果缺失在运行jar task会有以下错误：
```
Execution failed for task ':jiagu:jar'.
> Entry META-INF/versions/9/module-info.class is a duplicate but no duplicate handling strategy has been set. Please refer to https://docs.gradle.org/7.5/dsl/org.gradle.api.tasks.Copy.html#org.gradle.api.tasks.Copy:duplicatesStrategy for details.
```


