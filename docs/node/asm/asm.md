本文以kotlin为语言基础，白手做一个组件化插件，选择kotlin一是因为groovy语言编写起来不太友好，二是顺便学习一下kotlin；选择做一个插件，是因为插件可以hook编译、打包的各个过程，达到给组件解耦（ARouter）；优化资源(McImage)，同时学习asm字节码注入。

### 如何设计？

正常组件之间调用，组件 A调用组件B的方法，要达到解耦那就不能直接调用，那就写个中间层吧，组件池：CRouteFactory，A、B通过组件池间接调用达到解耦目的

![](0.png ':size=500')

各个组件分别把自己注入CRouteFactory，表明自己的身份（name）以及自己的能干什么（action），以及完成每个功能所需要的参数(params),这样CRouteFactory就相当于一个组件池

![](1.png ':size=500')

当组件之间需要通信时，当A需要调用B时，A将自己的意图（name,action,params ）告知CRouteFactory，组件池通过name找到需要调用哪个组件，同时透传action和 params给目标组件，让目标组件内部自己处理，这里需要每个组件都继承同一个接口以提供给组件池调用，定义这个接口为IRoute，这样组件化框架的雏形就基本成型了

![](2.png ':size=500')

### 设计组件池CRouteFactory

RoutePool:组件池，所有组件注入的地方，并根据name找到目标组件，执行调用
```
class RoutePool {
    companion object {
        //组件列表
        private var pools: MutableMap<String, IRoute> = mutableMapOf()
        //组件注入方法
        fun registerComponent(name: String) {
            if (name.isNullOrEmpty()) {
                return
            }
            if (pools.containsKey(name)) {
                throw Exception("has register same components,please change other name")
            }
            var name = name.replace(File.separator, ".")
            try {
                //获取真正的组件名称
                name = name.substring(0, name.length - 6)
                //反射获取组件入口实例
                val cls = Class.forName(name)
                val iRoute = cls.newInstance() as IRoute
                val componentName: String = iRoute.getRouteName()
                if (TextUtils.isEmpty(componentName)) {
                    return
                }
                //添加到组件池子
                pools.put(componentName, iRoute)
            } catch (e: ClassNotFoundException) {
                e.printStackTrace()
            } catch (e: IllegalAccessException) {
                e.printStackTrace()
            } catch (e: InstantiationException) {
                e.printStackTrace()
            }
        }
        
        fun excute(cBuilder: CBuilder):Any {
            if (!pools.keys.contains(cBuilder.name)) {
                return false
            }
            //寻找目标组件
            var iRoute = pools.getValue(cBuilder.name!!)
            //调用组件方法
            return iRoute.excute(cBuilder.context!!, cBuilder.action!!, cBuilder.params)
        }
    }
}
```

**IRoute**：组件入口，每个需要成为单独组件的module都需要有一个入口，并注入到CRouteFactory，需要一个getName获取当前组件名称，以及一个excute，组件执行入口

```
interface IRoute {
    fun getRouteName(): String
    fun excute(context: Context, action: String, params: MutableMap<String, String>):Any
}
```
**CBuilder：**组件调用数据构建类,使用链式调用，组件之间一般涉及到activity跳转，所以构建时需要传入context
```
class CBuilder {
    var context: Context? = null
    var name: String? = null
    var action: String? = null
    var params: MutableMap<String, String> = mutableMapOf()

    companion object {
        fun create(context: Context): CBuilder {
            var cBuilder = CBuilder()
            cBuilder.context = context
            return cBuilder
        }
    }

    fun params(params: MutableMap<String, String>): CBuilder {
        if (!params.isNullOrEmpty())
            this.params.putAll(params)
        return this
    }

    fun name(name: String): CBuilder {
        this.name = name
        return this
    }

    fun action(action: String): CBuilder {
        this.action = action
        return this
    }

    fun build(): Any {
        return RoutePoolJava.excute(this)
    }
}
```

####    创建一个组件常量池RouteConstants

每个组件之间通信都是通过 name,action,params相互通信，这样每个组件就应该有一个固定的name,以及固定的action，所以这里创建一个组件常量模块，给所有需要组件通信的模块使用,每个组件建议建立一个单独的class表明自己的身份以及能力

```
class ComponentA {
    companion object{
        val name = ComponentA::class.java.simpleName
        val action1 = "startActivity"
        val action2 = "getResult"
        val p1 = "p1"
        val p2 = "p2"
    }
}
```

####    设计几个Component

上面说了，每个组件都必须有一个入口继承IRoute，并注入到RoutePool，这样才能让组件池找到自己，并调用自己的入口

```
class ComponentA : IRoute {
    override fun getRouteName(): String {
        return ComponentA.name
    }

    override fun excute(context: Context, action: String, params: MutableMap<String, String>): Any {
        when (action) {
            ComponentA.action1 -> {
                context.startActivity(Intent(context, ComponentAActivity::class.java))
                return true
            }
            ComponentA.action2 -> {
                return ComponentAActivity.getResult(params)
            }
        }
        return true
    }
}
```
到这里差不多所有的就设计完了，总共一下几个moule

**app**：主app
**CRouteFactory** :组件池
**RouteConstants**:组件常量基础库
**Components：**若干组件


####    各组件如何注入到组件池？
CRouteFactory提供组件注入入口registerComponent，让各个子组件依赖CRouteFactory，直接调用入口函数注入？如果是这样的话，各个子组件就需要依赖组件池，就达不到组件化解耦的目的了。看上图中的设计，原本就没想让子组件依赖组件池，那怎么注入呢 ？


可以通过自定义gradle插件，在transform时遍历input，将继承至IRoute的class通过ASM注入到RoutePool中

####    自定义gradle插件
1.  新建一个android library，取名crregister
2.  保留src目录、build.gradle，其余全部删除
3.  修改build.gradle，引入插件开发需要的库，并支持发布到本地repo

```
apply plugin: 'groovy'
apply plugin: 'maven'
apply plugin: 'kotlin'
dependencies {
    implementation gradleApi()
    implementation localGroovy()
    implementation 'com.android.tools.build:gradle:3.5.0'
}
buildscript {
    ext.kotlin_version = '1.3.72'
    repositories {
        jcenter()
        google()
        mavenCentral()
    }

    dependencies {
        classpath "org.jetbrains.kotlin:kotlin-gradle-plugin:$kotlin_version"
    }
}

allprojects {
    repositories {
        jcenter()
        google()
    }
}
uploadArchives {
    //发布到本地repo仓库，以提供给app使用
    repositories {
        mavenDeployer {
            pom.groupId = 'com.code.crregister'
            pom.artifactId = 'crregister'
            pom.version = '1.0.0'
            repository(url: uri('../repo'))
        }
    }
}
```

4.  build project，此时会在gradle面板中多出upload task: uploadArchives

![](3.png ':size=500')

5.  main目录下新建resouces/META-INF/gradle-plugins完整目录，并在该目录下新建ccregister.properties文件
6.  在java/packagename目录下新建kotlin class,CRoutePlugin
7.  ccregister.properties中增加以下代码，指向步骤6中的插件类
```
implementation-class=com.code.crregister.CRoutePlugin
```
8.  完善字节码注入过程的所有代码，此处不做详细接受，文章结尾会有源代码传送门

![](4.png ':size=500')


9.  gradle面板，uploadArchives，发布插件到本地repo仓库
10. 项目根目录build.gradle引入发布的插件

```
repositories {
//增加本地仓依赖
        maven {
            url uri('repo')
        }
    }
    dependencies {
        //引入发布的crregister
        classpath 'com.code.crregister:crregister:1.0.0'
       ...
    }
```

11.在app module目录下的build.gradle引用插件

```
apply plugin: 'crregister'
```

12.运行app查看build log

```
********CRoutePlugin  start
isIncremental:true
components:[com/code/componentb/ComponentB.class, com/code/componenta/ComponentA.class]
pool path:/Users/balance/git-project/CRoute/app/build/intermediates/transforms/com.code.crregister.CRoutePlugin/debug/33.jar
inject finish const:1334
********CRoutePlugin  end
```

创建的组件ComponentA、ComponentB都已经扫描到了，并注入到RoutePool，并且打印出了组件池所在的jar路径，通过jd-gui查看字节码注入后的jar

![](5.png ':size=500')

代码注入的很完美，但是有个问题，正如图中所标示的：kotlin的静态方法是通过comopanion伴生对象访问的在static里面应该调用不到，在主APP里通过以下代码跳转ComponentA中的Actiivty

```
CBuilder.create(this)
                .name(ComponentA.name)
                .action(ComponentA.action1)
                .params(mutableMapOf())
                .build()
```

果不其然，crash了，找不到静态方法resisterComponent

![](6.png ':size=500')

asm字节码注入对kotlin还是不太友好，先不折腾了，把RoutePool改用java重新写一遍，再次通过jd-gui查看注入的代码


![](7.png ':size=500')

从主APP通过CBuilder跳转ComponentA中的activity也成功了

感谢大家看到这里，最后附上源码：[GitHub CRoute](https://github.com/dengshaomin/CRoute)

***
https://www.jianshu.com/p/dfc4681f8090
***