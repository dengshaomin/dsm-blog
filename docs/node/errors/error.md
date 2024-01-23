* #### <font color="red">  公司电脑git操作无反应，可能是host问题 </font>
修改host:  140.82.114.4    github.com
https://blog.csdn.net/ozhengsfo/article/details/121983929


* #### <font color="red">  flutter web部署github白屏不显示 </font>
新建workflow的时候将 flutter build web 改成：flutter build web --release --web-renderer html --base-href /your_repository_name/；


* #### <font color="red">  flutter腾讯地图插件，地图不显示 </font>

从 flutter v3 开始，AndroidView 的默认实现从 VirtualDisplay 改为 TextureLayer， 解决了部分性能问题，但与 SurfaceView 不兼容，而 TextureMapView 的性能是不能接受的。 目前我能找到最好的方案是 hybrid composition。值得注意的是，目前 flutter 官方文档 Android platform-views 是过时的，且不说 VirtualDisplay 已被移除，里面展示的代码已经不能实现 hybrid composition。 因为 PlatformViewsService.initSurfaceAndroidView 已被改成用于实现 TextureLayer，正确的用法是 `PlatformViewsService.initExpensiveAndroidView`。
https://github.com/qiuxiang/flutter-tencent-map

* #### <font color="red">  android依赖 flutter module报错 </font>

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



