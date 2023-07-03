SPI 全称为Service Provider Interface，是JDK内置的一种服务提供发现机制。简单来说，它是一种动态替换发现机制。例如，在设计组件化路由框架的时候就会使用到 SPI 设计思想。ServiceLoader 会从指定的路径下解析文件来获取到指定的子类信息，这些子类的生成过程原理：主要是利用：编译期注解、注解处理器。

过程主要分为三步：

**开发阶段**：对要关联的类使用编译期注解。

**编译阶段**：通过注解处理器去遍历查找被指定注解修饰的类的信息，然后收集到后写入到 META-INF/services 文件夹下的文件中。

**使用阶段**：ServiceLoader 去 META-INF/services 文件夹下查找指定的文件，并解析文件内容，获取到要加载的子类信息。

####    引入依赖
```
kapt 'com.google.auto.service:auto-service:1.0.1'
implementation 'com.google.auto.service:auto-service:1.0.1'
```

####    定义组建接口
```
interface IComponent {
    fun getName(): String
    fun action(context: Context, actionName: String, params: MutableMap<String, Any>): Any?
}
```

####    定义组建常量
```
object ComponentConstants {
    object ComponentA {
        const val COMPONENT_A = "COMPONENT_A"
        const val METHOD_0 = COMPONENT_A + "METHOD_0"
    }
}
```
####    定义组件ComponentA

```
@AutoService(IComponent::class)
class ComponentA : IComponent {
    override fun getName(): String {
        return ComponentConstants.ComponentA.COMPONENT_A
    }

    override fun action(context: Context,actionName: String, params: MutableMap<String, Any>): Any? {
        when(actionName){
            ComponentConstants.ComponentA.METHOD_0-> return "result"
        }
        return null
    }
}
```

####    组件构造类
```
class ComponentBuilder(
    val context: Context,
    val componentName: String,
    val action: String,
    val params: MutableMap<String, Any> = mutableMapOf()
) {

}
```

####    定义组件池
```
object ComponentManager {
    private var components = mutableMapOf<String, IComponent>()

    init {
        loadComponents()
    }

    private fun loadComponents() {
        val list = ServiceLoader.load(IComponent::class.java, javaClass.classLoader)
        list.toList().map {
            components.put(it.getName(), it)
        }
    }

    fun action(componentBuilder: ComponentBuilder): Any? {
        components[componentBuilder.componentName]?.apply {
            return this.action(
                componentBuilder.context,
                componentBuilder.action,
                componentBuilder.params
            )
        }
        return null
    }
}
```
####    组件调用
```
val result = ComponentManager.action(
            ComponentBuilder(
                this,
                ComponentConstants.ComponentA.COMPONENT_A,
                ComponentConstants.ComponentA.METHOD_0,
                mutableMapOf<String, Any>().apply {
                    put("key", "aaa")
                })
        )
```

```
https://blog.csdn.net/Love667767/article/details/106422787
```


