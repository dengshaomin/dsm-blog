### View Binding的配置

ButterKnife和kotlin-android-extensions已经被弃用，并且都推荐使用View Binding。
```
The 'kotlin-android-extensions' Gradle plugin is deprecated. Please use this migration guide (https://goo.gle/kotlin-android-extensions-deprecation) to start working with View Binding (https://developer.android.com/topic/libraries/view-binding)
```

如何配置
```
// Android Studio 3.6 
android { 
   viewBinding { enabled = true } 
} 

// Android Studio 4.0 
android { 
   buildFeatures { viewBinding = true } 
}
```

注意：
1、视图绑定是逐模块启用的，因此，如果您具有多模块项目设置，则需要在每个 build.gradle 文件中添加以上代码。
2、如果要在特定的布局禁用 view binding，则需要在布局文件的根视图中添加 tools：viewBindingIgnore = “true”。
3、启用后，我们可以立即开始使用它，并且当您完成同步 build.gradle 文件时，默认情况下会生成所有绑定类。
4、它通过将XML布局文件名转换为驼峰式大小写并在其末尾添加 Binding 来生成绑定类。 例如，如果您的布局文件名为 activity_splash，则它将生成绑定类为 ActivitySplashBinding。所以类中看不到R.layout.xxx类似的代码。
### View Binding的使用
####    Activity中使用
XML文件名为activity_main，文件中只有一个TextView，代码如下：
```
<?xml version="1.0" encoding="utf-8"?>
<androidx.constraintlayout.widget.ConstraintLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context=".MainActivity">

    <TextView
        android:id="@+id/tv"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="文本"
        android:textSize="28sp"
        app:layout_constraintLeft_toLeftOf="parent"
        app:layout_constraintTop_toTopOf="parent" />

</androidx.constraintlayout.widget.ConstraintLayout>
```
在Activity中使用
```
class MainActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        val binding: ActivityMainBinding = ActivityMainBinding.inflate(layoutInflater)  //ViewBinding自动生成了ActivityMainBinding类
        setContentView(binding.root)

        binding.tv.setOnClickListener {  //通过binding直接引用对象
           //TODO
        }
    }
}
```
####    2、在Fragment中使用
XML文件名为fragment_test,文件内容同Activity的xml内容。在Fragment中使用：
```
class TestFragment : Fragment() {

    private var fragmentTestBinding:FragmentTestBinding? = null
    private val binding get() = fragmentTestBinding!!

    override fun onCreateView(inflater: LayoutInflater, container: ViewGroup?, savedInstanceState: Bundle?): View? {
        fragmentTestBinding =FragmentTestBinding.inflate(inflater,container,false)
        return binding.root
    }

    override fun onActivityCreated(savedInstanceState: Bundle?) {
        super.onActivityCreated(savedInstanceState)
        binding.tv.setOnClickListener {
            //TODO
        }
    }

    override fun onDestroyView() {
        super.onDestroyView()
        fragmentTestBinding=null
    }
}
```
使用了两个不同的变量 fragmentTestBinding 和 binding，并且 fragmentTestBinding 变量在 onDestroyView() 中设置为null。

这是因为该 fragment 的生命周期与 activity 的生命周期不同，并且该fragment 可以超出其视图的生命周期，因此如果不将其设置为null，则可能会发生内存泄漏。

另一个变量通过 !! 使一个变量为可空值而使另一个变量为非空值避免了空检查。
####    3、在RecyclerView adapter中使用
XML文件名为adapter_test，文件内容同Activity的xml内容。在RecyclerView adapter中使用：
```
class TestRecycleAdapter(private val list:List<String>):RecyclerView.Adapter<TestRecycleAdapter.TestHolder>() {

    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): TestHolder {
        val itemBinding = AdapterTestBinding.inflate(LayoutInflater.from(parent.context), parent, false)
        return TestHolder(itemBinding)
    }

    override fun onBindViewHolder(holder: TestHolder, position: Int) {
        val value = list[position]
        holder.bind(value)  //绑定item的内容，binding对象应该是给ViewHolder持有
    }

    override fun getItemCount(): Int = list.size

    class TestHolder(private val itemBinding: AdapterTestBinding) : RecyclerView.ViewHolder(itemBinding.root) {
        //设置值
        fun bind(str: String) {
            itemBinding.tv.text = str
        }
    }
}
```
####    4、处理<include>标签
①<inlude> 不带 <merge>标签

activity_main.xml内容：

```
<?xml version="1.0" encoding="utf-8"?> <androidx.constraintlayout.widget.ConstraintLayout 
xmlns:android="http://schemas.android.com/apk/res/android" 
xmlns:app="http://schemas.android.com/apk/res-auto" 
android:layout_width="match_parent"
android:layout_height="match_parent"> 

<include 
    android:id="@+id/appbar"   //inclue的id
    layout="@layout/app_bar"   //inclue的布局文件
    app:layout_constraintTop_toTopOf="parent" /> 

</androidx.constraintlayout.widget.ConstraintLayout>
```

app_bar.xml内容：
```
<?xml version="1.0" encoding="utf-8"?> <androidx.constraintlayout.widget.ConstraintLayout 
xmlns:android="http://schemas.android.com/apk/res/android" 
xmlns:app="http://schemas.android.com/apk/res-auto" 
android:layout_width="match_parent" 
android:layout_height="wrap_content"> 

<androidx.appcompat.widget.Toolbar 
      android:id="@+id/toolbar"   //Toolbar设置id
      android:layout_width="0dp" 
      android:layout_height="?actionBarSize" 
      android:background="?colorPrimary" 
      app:layout_constraintEnd_toEndOf="parent" 
      app:layout_constraintStart_toStartOf="parent" 
      app:layout_constraintTop_toTopOf="parent" /> 

</androidx.constraintlayout.widget.ConstraintLayout>
```

在Activity中获取Toolbar：
```
class MainActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        val binding: ActivityMainBinding = ActivityMainBinding.inflate(layoutInflater)
        setContentView(binding.root)

        binding.appbar.toolbar.setOnClickListener {  //设置ID可以直接调用到
           //TODO
        }
    }
}
```

②<inlude> 带 <merge>标签
activity_main.xml内容：

```
<androidx.constraintlayout.widget.ConstraintLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <include 
        android:id="@+id/icd"        //inclue的id
        layout="@layout/placeholder" />  //inclue的布局文件

</androidx.constraintlayout.widget.ConstraintLayout>
```

placeholder.xml内容：
```
<?xml version="1.0" encoding="utf-8"?>
<merge xmlns:android="http://schemas.android.com/apk/res/android">

   <TextView
       android:id="@+id/tv"
       android:layout_width="match_parent"
       android:layout_height="wrap_content" />
       
</merge>
```

在Activity中调用：
```
 binding.icd.tv.setOnClickListener {
     Toast.makeText(this,"显示文本",Toast.LENGTH_LONG).show()
```

会发现运行就直接报错Missing required view with ID: com.cad.studymotionlayout:id/icd，找不到id。那么应该如何使用：

删掉include设置的id，加上就会报上面的错误：
```
<androidx.constraintlayout.widget.ConstraintLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <include 
        layout="@layout/placeholder" />  //删掉ID

</androidx.constraintlayout.widget.ConstraintLayout>
```

placeholder.xml的内容不变，在Activity中调用代码如下：
```
class MainActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        val binding: ActivityMainBinding = ActivityMainBinding.inflate(layoutInflater)
        //placeholder有自己生成的ViewBinding类PlaceholderBinding,将PlaceholderBinding与Activity的布局绑定就可以使用了
        val placeholderBinding = PlaceholderBinding.bind(binding.root)
        setContentView(binding.root)

        placeholderBinding.tv.setOnClickListener {
            //TODO
        }
    }
}
```

### 三、View Binding的封装
####    1、不依托基类的封装
无需继承什么基类，泛用性更强，移植代码更加容易。会用到一些 Kotlin 的特性，不适用于 Java。

①Activity的封装

有二种封装形式，用反射或者不用反射，封装时二选一。

```
import android.view.LayoutInflater
import androidx.activity.ComponentActivity
import androidx.databinding.ViewDataBinding
import androidx.viewbinding.ViewBinding


//使用反射的代码如下：
//使用反射获取inflate方法
inline fun <reified VB : ViewBinding> ComponentActivity.binding() = lazy {
    inflateBinding<VB>(layoutInflater).also { binding ->
        setContentView(binding.root)
        if (binding is ViewDataBinding) binding.lifecycleOwner = this
    }
}

inline fun <reified VB : ViewBinding> inflateBinding(layoutInflater: LayoutInflater) =
    VB::class.java.getMethod("inflate", LayoutInflater::class.java).invoke(null, layoutInflater) as VB
    
//不使用反射的代码如下:
//将inflate函数作为参数传递
fun <VB : ViewBinding> ComponentActivity.binding(inflate: (LayoutInflater) -> VB) = lazy {
    inflate(layoutInflater).also { binding ->
        setContentView(binding.root)
        if (binding is ViewDataBinding) binding.lifecycleOwner = this
    }
}
```
在Avtivity中使用：

```
class MainActivity : AppCompatActivity() {
    //绑定(使用反射)
    private val binding: ActivityMainBinding by binding()
    
    //绑定(不使用反射)
    private val binding by binding(ActivityMainBinding::inflate)

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        //使用
        binding.tv.text="测试"
    }
}
```
②Fragment的封装

有二种封装形式，用反射或者不用反射，封装时二选一。

其中反射也有二种方式，第一种是反射获取Layoutinflater的inflate()方法，另一种是反射获取View的bind()方法，为了避免内容太过复杂，这里就写出反射获取bind()的方式。可以点此深入github项目查看，感谢作者!

使用反射的代码：

```
import androidx.databinding.ViewDataBinding
import androidx.fragment.app.Fragment
import androidx.lifecycle.DefaultLifecycleObserver
import androidx.lifecycle.LifecycleOwner
import androidx.viewbinding.ViewBinding
import kotlin.properties.ReadOnlyProperty
import kotlin.reflect.KProperty
import android.view.View

//Fragment扩展方法
inline fun <reified VB : ViewBinding> Fragment.binding() =
    FragmentBindingProperty(VB::class.java)


//属性委托类
class FragmentBindingProperty<VB : ViewBinding>(private val clazz: Class<VB>) :
    ReadOnlyProperty<Fragment, VB> {

    private var binding: VB? = null

    @Suppress("UNCHECKED_CAST")
    override fun getValue(fragment: Fragment, property: KProperty<*>): VB {
        if (binding == null) {
            try {
                //反射获取View的bind方法
                binding = (clazz.getMethod("bind", View::class.java)
                    .invoke(null, fragment.requireView()) as VB)
                    .also { binding ->
                        if (binding is ViewDataBinding) binding.lifecycleOwner =
                            fragment.viewLifecycleOwner
                    }
            } catch (e: IllegalStateException) {
                throw IllegalStateException("The property of ${property.name} has been destroyed.")
            }
            //Fragment摧毁时将binding置位null避免内存泄露
            fragment.viewLifecycleOwner.lifecycle.addObserver(object : DefaultLifecycleObserver {
                override fun onDestroy(owner: LifecycleOwner) {
                    binding = null
                }
            })
        }
        return binding!!
    }
}
```

使用反射的代码在Fragment中使用：

```
//特别注意一定要加上布局文件,很容易忘掉
class TestFragment : Fragment(R.layout.fragment_test) {

    //绑定(使用反射)
    private val binding: FragmentTestBinding by binding()

    override fun onViewCreated(view: View,savedInstanceState: Bundle?) {
        super.onViewCreated(view,savedInstanceState)
        binding.tv.text = "测试"
    }
    
}
```
不使用反射的方法与Activity的原理类似，代码如下：
```
//将函数当做参数传递
fun <VB : ViewBinding> Fragment.binding(bind: (View) -> VB) = FragmentBindingDelegate(bind)

//属性委托
class FragmentBindingDelegate<VB : ViewBinding>(private val bind: (View) -> VB) :
    ReadOnlyProperty<Fragment, VB> {

    private var binding: VB? = null

    override fun getValue(thisRef: Fragment, property: KProperty<*>): VB {
        binding = try {
            thisRef.requireView().getBinding(bind).also { binding ->
                if (binding is ViewDataBinding) binding.lifecycleOwner = thisRef.viewLifecycleOwner
            }
        } catch (e: IllegalStateException) {
            throw IllegalStateException("The property of ${property.name} has been destroyed.")
        }
        thisRef.viewLifecycleOwner.lifecycle.addObserver(object : DefaultLifecycleObserver {
            override fun onDestroy(owner: LifecycleOwner) {
                //fragment销毁时置为null,避免内存泄露
                binding = null
            }
        })

        return binding!!
    }
}

@Suppress("UNCHECKED_CAST")
fun <VB : ViewBinding> View.getBinding(bind: (View) -> VB): VB = bind(this)

不使用反射的使用：
js复制代码//注意加上布局文件,特别容易漏
class TestFragment : Fragment(R.layout.fragment_test) {
    //绑定
    private val binding by binding(FragmentTestBinding::bind)

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        binding.tv.text = "测试"
    }
}
```

③Adapter的封装

使用反射

```
inline fun <reified VB : ViewBinding> BindingViewHolder(parent: ViewGroup) =
    BindingViewHolder(inflateBinding<VB>(parent))

class BindingViewHolder<VB : ViewBinding>(val binding: VB) : RecyclerView.ViewHolder(binding.root) {
    constructor(parent: ViewGroup, inflate: (LayoutInflater, ViewGroup, Boolean) -> VB) :
            this(inflate(LayoutInflater.from(parent.context), parent, false))
}

inline fun <reified VB : ViewBinding> inflateBinding(parent: ViewGroup) =
    inflateBinding<VB>(LayoutInflater.from(parent.context), parent, false)


inline fun <reified VB : ViewBinding> inflateBinding(
    layoutInflater: LayoutInflater, parent: ViewGroup?, attachToParent: Boolean) =
    VB::class.java.getMethod("inflate", LayoutInflater::class.java, ViewGroup::class.java, Boolean::class.java)
        .invoke(null, layoutInflater, parent, attachToParent) as VB
```
在Adapter中使用
```
class TestRecycleAdapter(private val list:List<String>):RecyclerView.Adapter<BindingViewHolder<AdapterTestBinding>>() {

    override fun onCreateViewHolder(parent: ViewGroup,viewType: Int): BindingViewHolder<AdapterTestBinding> =
        BindingViewHolder<AdapterTestBinding>(parent)

    override fun onBindViewHolder(holder: BindingViewHolder<AdapterTestBinding>, position: Int) {
        holder.binding.tv.text = list[position]
    }

    override fun getItemCount(): Int=list.size

}
```
不使用反射
```
fun <VB : ViewBinding> RecyclerView.ViewHolder.withBinding(bind: (View) -> VB, block: VB.(RecyclerView.ViewHolder) -> Unit) = apply {
    block(getBinding(bind), this@withBinding)
}

fun <VB : ViewBinding> BindingViewHolder<VB>.withBinding(block: VB.(BindingViewHolder<VB>) -> Unit) = apply {
    block(binding, this@withBinding)
}

fun <VB : ViewBinding> RecyclerView.ViewHolder.getBinding(bind: (View) -> VB) = itemView.getBinding(bind)

class BindingViewHolder<VB : ViewBinding>(val binding: VB) : RecyclerView.ViewHolder(binding.root) {
    constructor(parent: ViewGroup, inflate: (LayoutInflater, ViewGroup, Boolean) -> VB) :
            this(inflate(LayoutInflater.from(parent.context), parent, false))
}
```
在Adapter中使用
```
class TestRecycleAdapter(private val list: List<String>) :
    RecyclerView.Adapter<BindingViewHolder<AdapterTestBinding>>() {

    override fun onCreateViewHolder(parent: ViewGroup,viewType: Int): BindingViewHolder<AdapterTestBinding> =
        BindingViewHolder(parent, AdapterTestBinding::inflate)
        /*BindingViewHolder(parent, AdapterTestBinding::inflate).withBinding {
            //DO Something
            tv.setOnClickListener {
            }
        }*/

    override fun onBindViewHolder(holder: BindingViewHolder<AdapterTestBinding>, position: Int) {
        holder.binding.tv.text = list[position]
    }

    override fun getItemCount(): Int = list.size

}
```

④Dialog的封装，与Activity类似

使用反射

```
inline fun <reified VB : ViewBinding> Dialog.binding() = lazy {
  inflateBinding<VB>(layoutInflater).also { setContentView(it.root) }
}

inline fun <reified VB : ViewBinding> inflateBinding(layoutInflater: LayoutInflater) =
  VB::class.java.getMethod("inflate", LayoutInflater::class.java).invoke(null, layoutInflater) as VB

```
不使用反射

```
fun <VB : ViewBinding> Dialog.binding(inflate: (LayoutInflater) -> VB) = lazy {
  inflate(layoutInflater).also { setContentView(it.root) }
}
```

⑤PopupWindow的封装

使用反射
```
//Activity
inline fun <reified VB : ViewBinding> Activity.popupWindow(
  width: Int = ViewGroup.LayoutParams.WRAP_CONTENT,
  height: Int = ViewGroup.LayoutParams.WRAP_CONTENT,
  focusable: Boolean = false,
  crossinline block: VB.() -> Unit
) =
  lazy {
    PopupWindow(inflateBinding<VB>(layoutInflater).apply(block).root, width, height, focusable)
  }

//Fragment
inline fun <reified VB : ViewBinding> Fragment.popupWindow(
  width: Int = ViewGroup.LayoutParams.WRAP_CONTENT,
  height: Int = ViewGroup.LayoutParams.WRAP_CONTENT,
  focusable: Boolean = false,
  crossinline block: VB.() -> Unit
) =
  lazy {
    PopupWindow(inflateBinding<VB>(layoutInflater).apply(block).root, width, height, focusable)
  }
  
inline fun <reified VB : ViewBinding> inflateBinding(layoutInflater: LayoutInflater) =
  VB::class.java.getMethod("inflate", LayoutInflater::class.java).invoke(null, layoutInflater) as VB

```

不使用反射

```
fun <VB : ViewBinding> Activity.popupWindow(
  inflate: (LayoutInflater) -> VB,
  width: Int = ViewGroup.LayoutParams.WRAP_CONTENT,
  height: Int = ViewGroup.LayoutParams.WRAP_CONTENT,
  focusable: Boolean = false,
  block: VB.() -> Unit) = lazy {
    PopupWindow(inflate(layoutInflater).apply(block).root, width, height, focusable)
  }

fun <VB : ViewBinding> Fragment.popupWindow(
  inflate: (LayoutInflater) -> VB,
  width: Int = ViewGroup.LayoutParams.WRAP_CONTENT,
  height: Int = ViewGroup.LayoutParams.WRAP_CONTENT,
  focusable: Boolean = false,
  block: VB.() -> Unit) = lazy {
    PopupWindow(inflate(layoutInflater).apply(block).root, width, height, focusable)
  }
```

####    2、依托基类的情况

依托基类就是依托常写的Base类，同样区分使用反射和不使用反射的情况。

下面这个工具类是反射用到的工具类，下面Avtivity，Fragment和Dialog都有用到：
```
object ViewBindingUtil {

  @JvmStatic
  fun <VB : ViewBinding> inflateWithGeneric(genericOwner: Any, layoutInflater: LayoutInflater): VB =
    withGenericBindingClass<VB>(genericOwner) { clazz ->
      //反射
      clazz.getMethod("inflate", LayoutInflater::class.java).invoke(null, layoutInflater) as VB
    }.also { binding ->
      if (genericOwner is ComponentActivity && binding is ViewDataBinding) {
        binding.lifecycleOwner = genericOwner
      }
    }

  @JvmStatic
  fun <VB : ViewBinding> inflateWithGeneric(genericOwner: Any, parent: ViewGroup): VB =
    inflateWithGeneric(genericOwner, LayoutInflater.from(parent.context), parent, false)

  @JvmStatic
  fun <VB : ViewBinding> inflateWithGeneric(genericOwner: Any, layoutInflater: LayoutInflater, parent: ViewGroup?, attachToParent: Boolean): VB =
    withGenericBindingClass<VB>(genericOwner) { clazz ->
      clazz.getMethod("inflate", LayoutInflater::class.java, ViewGroup::class.java, Boolean::class.java)
        .invoke(null, layoutInflater, parent, attachToParent) as VB
    }.also { binding ->
      if (genericOwner is Fragment && binding is ViewDataBinding) {
        binding.lifecycleOwner = genericOwner.viewLifecycleOwner
      }
    }

  @JvmStatic
  fun <VB : ViewBinding> bindWithGeneric(genericOwner: Any, view: View): VB =
    withGenericBindingClass<VB>(genericOwner) { clazz ->
      clazz.getMethod("bind", View::class.java).invoke(null, view) as VB
    }.also { binding ->
      if (genericOwner is Fragment && binding is ViewDataBinding) {
        binding.lifecycleOwner = genericOwner.viewLifecycleOwner
      }
    }

  private fun <VB : ViewBinding> withGenericBindingClass(genericOwner: Any, block: (Class<VB>) -> VB): VB {
    var genericSuperclass = genericOwner.javaClass.genericSuperclass
    var superclass = genericOwner.javaClass.superclass
    while (superclass != null) {
      if (genericSuperclass is ParameterizedType) {
        genericSuperclass.actualTypeArguments.forEach {
          try {
            return block.invoke(it as Class<VB>)
          } catch (e: NoSuchMethodException) {
          } catch (e: ClassCastException) {
          } catch (e: InvocationTargetException) {
            throw e.targetException
          }
        }
      }
      genericSuperclass = superclass.genericSuperclass
      superclass = superclass.superclass
    }
    throw IllegalArgumentException("There is no generic of ViewBinding.")
  }
}
```

①Activity的封装

使用反射

```
abstract class BaseBindingActivity<VB : ViewBinding> : AppCompatActivity() {

  lateinit var binding: VB

  override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    binding = ViewBindingUtil.inflateWithGeneric(this, layoutInflater)
    setContentView(binding.root)
  }
}
```

在Avtivity中使用

```
class MainActivity : BaseBindingActivity<ActivityMainBinding>() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        binding.tv.text = "测试"
    }

}
```

不使用反射

```
abstract class BaseBindingActivity<VB : ViewBinding>(private val inflate: (LayoutInflater) -> VB) : AppCompatActivity() {

  lateinit var binding: VB

  override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    binding = inflate(layoutInflater)
    setContentView(binding.root)
  }
}
```
在Activity中使用

```
class MainActivity : BaseBindingActivity<ActivityMainBinding>(ActivityMainBinding::inflate) {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        binding.tv.text = "测试"
    }

}
```

②Fragment的封装

使用反射

```
abstract class BaseBindingFragment<VB : ViewBinding> : Fragment() {

  private var _binding: VB? = null
  val binding: VB get() = _binding!!

  override fun onCreateView(inflater: LayoutInflater, container: ViewGroup?, savedInstanceState: Bundle?): View {
    _binding = ViewBindingUtil.inflateWithGeneric(this, inflater, container, false)
    return binding.root
  }

  override fun onDestroyView() {
    super.onDestroyView()
    _binding = null
  }
}
```

在Fragment中使用
```
class TestFragment : BaseBindingFragment<FragmentTestBinding>() {
    
    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        binding.tv.text = "测试"
    }
    
}
```

不使用反射

```
abstract class BaseBindingFragment<VB : ViewBinding>(private val inflate: (LayoutInflater, ViewGroup?, Boolean) -> VB) : Fragment() {

  private var _binding: VB? = null
  val binding: VB get() = _binding!!

  override fun onCreateView(inflater: LayoutInflater, container: ViewGroup?, savedInstanceState: Bundle?): View {
    _binding = inflate(inflater, container, false)
    return binding.root
  }

  override fun onDestroyView() {
    super.onDestroyView()
    _binding = null
  }
}
```
在Fragment中使用

```
class TestFragment : BaseBindingFragment<FragmentTestBinding>(FragmentTestBinding::inflate) {

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        binding.tv.text = "测试"
    }

}
```

③Dialog的封装

使用反射

```
abstract class BaseBindingDialog<VB : ViewBinding>(context: Context, themeResId: Int = 0)
  : Dialog(context, themeResId) {

  lateinit var binding: VB

  override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    binding = ViewBindingUtil.inflateWithGeneric(this, layoutInflater)
    setContentView(binding.root)
  }
}
```

在Dialog中使用

```
class TestDialog(context: Context):BaseBindingDialog<DialogTestBinding>(context) {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        binding.tv.text = "测试"
    }
}
```

不使用反射

```
abstract class BaseBindingDialog<VB : ViewBinding>(context: Context,themeResId: Int,private val inflate: (LayoutInflater) -> VB
) : Dialog(context, themeResId) {

  lateinit var binding: VB

  constructor(context: Context, inflate: (LayoutInflater) -> VB) : this(context, 0, inflate)

  override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    binding = inflate(layoutInflater)
    setContentView(binding.root)
  }
}
```
在Dialog中使用

```
class TestDialog(context: Context): BaseBindingDialog<DialogTestBinding>(context,0,DialogTestBinding::inflate) {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        binding.tv.text = "测试"
    }
}
```

使用反射和不使用反射可以根据性能和代码习惯取舍，比如依托基类不使用反射的情况，代码确实不美观；封装的代码来自于这个库[ViewBindingKTX](https://github.com/DylanCaiCoding/ViewBindingKTX)


```
https://juejin.cn/post/7064498827931156493

https://github.com/androidbroadcast/ViewBindingPropertyDelegate
```