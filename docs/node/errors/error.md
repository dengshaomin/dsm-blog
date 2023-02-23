* #### <font color="red"> 1. android依赖 flutter module报错 </font>

```
FAILURE: Build failed with an exception.
* Where:
Script '/Users/shaomin.deng/flutter/packages/flutter_tools/gradle/flutter.gradle' line: 173
* What went wrong:
A problem occurred evaluating script.
> Failed to apply plugin class 'FlutterPlugin'.
   > Build was configured to prefer settings repositories over project repositories but repository 'maven' was added by plugin class 'FlutterPlugin'
```
修改setting.gradle：https://github.com/realm/realm-java/issues/7374

```
Replace the line:
repositoriesMode.set(RepositoriesMode.FAIL_ON_PROJECT_REPOS)
with
repositoriesMode.set(RepositoriesMode.PREFER_SETTINGS)

That's a gradle setting which allows this to work.
```



