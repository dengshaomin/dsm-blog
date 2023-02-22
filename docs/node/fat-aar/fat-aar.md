[fat-aar-android plugin](https://github.com/kezong/fat-aar-android)

[fat-aar-android gradle](https://github.com/adwiv/android-fat-aar)

[fat-aar原理分析](https://juejin.cn/post/6844903634518409224)

flutter混合开发项目过程中为了提升工程化效率需要将flutter相关代码单独打成aar集成到项目中，该方案有以下两个优点：
* 不需要所有开发都安装flutter环境
* 每次编译、打包都不需要编译fluter

通过fat-aar将fluter application工程进行改造打出aar，并将fluter sdk以及flutter依赖的plugin集成到改aar中
* ###### 1. 修改flutter-android-`build.gradle`，增加library模式
```
buildscript {
    //模式切换开关
    ext.isLibrary = true
    dependencies {
        //fat-aar依赖
        classpath 'com.github.kezong:fat-aar:1.3.8'
    }
}
```
* ###### 2.修改flutter-android-app-`build.gradle`
```
def localProperties = new Properties()
def localPropertiesFile = rootProject.file('local.properties')
if (localPropertiesFile.exists()) {
    localPropertiesFile.withReader('UTF-8') { reader ->
        localProperties.load(reader)
    }
}
def flutterRoot = localProperties.getProperty('flutter.sdk')
if (flutterRoot == null) {
    throw new GradleException("Flutter SDK not found. Define location with flutter.sdk in the local.properties file.")
}
def flutterVersionCode = localProperties.getProperty('flutter.versionCode')
if (flutterVersionCode == null) {
    flutterVersionCode = '1'
}
def flutterVersionName = localProperties.getProperty('flutter.versionName')
if (flutterVersionName == null) {
    flutterVersionName = '1.0'
}
//控制模式切换
if (isLibrary) {
    apply plugin: 'com.android.library'
} else {
    apply plugin: 'com.android.application'
}
apply plugin: 'kotlin-android'
apply from: "$flutterRoot/packages/flutter_tools/gradle/flutter.gradle"
if (isLibrary) {
    //添加fat-aar插件
    apply plugin: 'com.kezong.fat-aar'
}
android {
    compileSdkVersion flutter.compileSdkVersion
    ndkVersion flutter.ndkVersion

    compileOptions {
        sourceCompatibility JavaVersion.VERSION_1_8
        targetCompatibility JavaVersion.VERSION_1_8
    }

    kotlinOptions {
        jvmTarget = '1.8'
    }

    sourceSets {
        main.java.srcDirs += 'src/main/kotlin'
    }

    defaultConfig {
        // TODO: Specify your own unique Application ID (https://developer.android.com/studio/build/application-id.html).
        if (!isLibrary) {
            //library模式去除applicationid
            applicationId "com.niopower.flutte.module.pe_flutter_app"
        }
        // You can update the following values to match your application needs.
        // For more information, see: https://docs.flutter.dev/deployment/android#reviewing-the-build-configuration.
        minSdkVersion flutter.minSdkVersion
        targetSdkVersion flutter.targetSdkVersion
        versionCode flutterVersionCode.toInteger()
        versionName flutterVersionName
    }

    buildTypes {
        release {
            // TODO: Add your own signing config for the release build.
            // Signing with the debug keys for now, so `flutter run --release` works.
            signingConfig signingConfigs.debug
        }
    }
}
flutter {
    source '../..'
}
dependencies {
    implementation fileTree(dir: 'libs', include: ['*.jar', '*.aar'])
    implementation "org.jetbrains.kotlin:kotlin-stdlib-jdk7:$kotlin_version"
    if (isLibrary) {
//        embed fileTree(dir:'libs',include:['*.jar','*.aar'])
        // 添加 flutter_embedding.jar 
        embed "io.flutter:flutter_embedding_release:1.0.0-3316dd8728419ad3534e3f6112aa6291f587078a"
        //添加flutter 平台相关jar
        embed "io.flutter:arm64_v8a_release:1.0.0-3316dd8728419ad3534e3f6112aa6291f587078a"
        embed "io.flutter:armeabi_v7a_release:1.0.0-3316dd8728419ad3534e3f6112aa6291f587078a"
        //TODO 添加fat-aar处理flutter打包成aar中三方依赖
        def flutterProjectRoot = rootProject.projectDir.parentFile.toPath()
        def plugins = new Properties()
        def pluginsFile = new File(flutterProjectRoot.toFile(), '.flutter-plugins')
        if (pluginsFile.exists()) {
            pluginsFile.withReader('UTF-8') { reader -> plugins.load(reader) }
        }
        plugins.each { name, _ ->
            println name
            embed project(path: ":$name", configuration: 'default')
        }
    }
}
```


###### * 3.修改 /your flutter path/flutter/packages/flutter_tools/gradle/flutter.gradle

![logo](0.png ':size=WIDTHxHEIGHT')

进入flutter-andorid，运行 `gradlew assembleRelease` ,生成的aar目录：flutter-build-outputs-aar。


