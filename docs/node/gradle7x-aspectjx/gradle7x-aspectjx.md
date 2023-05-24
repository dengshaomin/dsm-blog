##### 环境
* Android Studio Electric Eel | 2022.1.1 Patch 1 
* gradle：distributionUrl=https\://services.gradle.org/distributions/gradle-7.5-bin.zip
* plugins
```
plugins {
    id 'com.android.application' version '7.4.1' apply false
    id 'com.android.library' version '7.4.1' apply false
    id 'org.jetbrains.kotlin.android' version '1.8.0' apply false
}
```

#### 遇到的问题

* 将`setting.gradle repositoriesMode.set(RepositoriesMode.FAIL_ON_PROJECT_REPOS)`修改为 `repositoriesMode.set(RepositoriesMode.PREFER_SETTINGS)`

```
FAILURE: Build failed with an exception.

* Where:
Build file '/Users/shaomin.deng/gitproject/FatAarHost/app/build.gradle' line: 4

* What went wrong:
An exception occurred applying plugin request [id: 'android-aspectjx']
> Failed to apply plugin 'android-aspectjx'.
   > Build was configured to prefer settings repositories over project repositories but repository 'MavenLocal' was added by plugin 'android-aspectjx'

```

* 修改root的`build.gradle`
```
plugins {
    id 'com.android.application' version '7.4.1' apply false
    id 'com.android.library' version '7.4.1' apply false
    id 'org.jetbrains.kotlin.android' version '1.8.0' apply false
}
修改成
plugins {
    id 'com.android.application' version '7.1.3' apply false
    id 'com.android.library' version '7.1.3' apply false
    id 'org.jetbrains.kotlin.android' version '1.8.0' apply false
}
```

```

FAILURE: Build failed with an exception.

* Where:
Build file '/Users/shaomin.deng/gitproject/FatAarHost/app/build.gradle' line: 4

* What went wrong:
An exception occurred applying plugin request [id: 'android-aspectjx']
> Failed to apply plugin 'android-aspectjx'.
   > No such property: FD_INTERMEDIATES for class: com.android.builder.model.AndroidProject

* Try:
> Run with --info or --debug option to get more log output.
> Run with --scan to get full insights.

* Exception is:
org.gradle.api.plugins.InvalidPluginException: An exception occurred applying plugin request [id: 'android-aspectjx']
	at org.gradle.plugin.use.internal.DefaultPluginRequestApplicator.exceptionOccurred(DefaultPluginRequestApplicator.java:223)
	at org.gradle.plugin.use.internal.DefaultPluginRequestApplicator.applyPlugin(DefaultPluginRequestApplicator.java:205)
	at org.gradle.plugin.use.internal.DefaultPluginRequestApplicator.applyPlugin(DefaultPluginRequestApplicator.java:147)
	at org.gradle.plugin.use.internal.DefaultPluginRequestApplicator.access$200(DefaultPluginRequestApplicator.java:61)
	at org.gradle.plugin.use.internal.DefaultPluginRequestApplicator$1$1.lambda$add$1(DefaultPluginRequestApplicator.java:120)
	at org.gradle.plugin.use.internal.DefaultPluginRequestApplicator.lambda$applyPlugins$0(DefaultPluginRequestApplicator.java:143)
```
* 修改app目录下的`build.gradle`
```
android {
    ...
    aspectjx {
        exclude  'versions.9'
    }
}
```
```
Execution failed for task ':app:dexBuilderDebug'.
> java.util.zip.ZipException: zip file is empty
```

