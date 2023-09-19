
### 环境
```
buildscript {
    ext{
        navigation_version = "2.5.3"
    }
    
    dependencies {
        classpath "androidx.navigation:navigation-safe-args-gradle-plugin:$navigation_version"
    }
}
implementation "androidx.navigation:navigation-fragment-ktx:$navigation_version"
implementation "androidx.navigation:navigation-ui-ktx:$navigation_version"
```
[官方介绍](https://developer.android.com/guide/navigation?hl=zh-cn)


### 设计导航图
1. res下新建navigation目录
2. navigation目录下新建nav_graph.xml，内容如下：
```
<?xml version="1.0" encoding="utf-8"?>
<navigation xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    app:startDestination="@+id/fragment_a">

    <fragment
        android:id="@+id/fragment_a"
        android:name="com.balance.sample.fragment.navigation.FragmentA"
        tools:layout="@layout/fragment_base_navigation">
        <action
            android:id="@+id/action_fragment_a_to_b"
            app:destination="@id/fragment_b"
            app:enterAnim="@anim/slide_in_right"
            app:exitAnim="@anim/slide_out_left"
            app:popEnterAnim="@anim/slide_in_left"
            app:popExitAnim="@anim/slide_out_right">
            <argument
                android:name="from"
                app:argType="string"
                app:nullable="true" />
        </action>
    </fragment>
    <fragment
        android:id="@+id/fragment_b"
        android:name="com.balance.sample.fragment.navigation.FragmentB"
        tools:layout="@layout/fragment_base_navigation">
        <action
            android:id="@+id/action_fragment_b_to_c"
            app:destination="@id/fragment_c"
            app:enterAnim="@anim/slide_in_bottom"
            app:exitAnim="@anim/none"
            app:popEnterAnim="@anim/none"
            app:popExitAnim="@anim/slide_out_bottom">
            <argument
                android:name="from"
                app:argType="string" />
        </action>
    </fragment>
    <fragment
        android:id="@+id/fragment_c"
        android:name="com.balance.sample.fragment.navigation.FragmentC"
        tools:layout="@layout/fragment_base_navigation">
        <action
            android:id="@+id/action_c_to_a"
            app:destination="@id/fragment_a">
            <argument
                android:name="from"
                app:argType="string" />
        </action>
    </fragment>
</navigation>
```
* startDestination：默认导航视图
* action：当前fragment下定义的跳转动作
    * id: 跳转到的action id
    * destination：目标页面
    * enterAnim，exitAnim，popEnterAnim，popExitAnim视图切换动画
* argument：跳转参数

### 创建示例Activity
页面中定义了Fragment的承载FragmentContainerView，默认显示了导航视图中的FragmentA
```
class FragmentNavigationActivity : AppCompatActivity() {
    private lateinit var binding: ActivityFragmentNavigationBinding
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        binding = ActivityFragmentNavigationBinding.inflate(layoutInflater)
        setContentView(binding.root)
    }

    override fun onStart() {
        super.onStart()
        //只有在onstart之后才能获取到
        //Navigation.findNavController(binding.fragmentContainer)
         binding.fragmentContainer.findNavController().navigate(FragmentADirections.actionFragmentAToC("A"))
        
    }

    override fun onResume() {
        super.onResume()
        (supportFragmentManager.fragments[0] as? NavHostFragment)?.childFragmentManager?.fragments

    }

    override fun onBackPressed() {
        if (Navigation.findNavController(binding.fragmentContainer).currentDestination?.id == R.id.fragment_b) {
            finish()
        } else {
            super.onBackPressed()
        }
    }
}

<?xml version="1.0" encoding="utf-8"?>
<androidx.constraintlayout.widget.ConstraintLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context=".fragment.navigation.FragmentNavigationActivity">

    <androidx.fragment.app.FragmentContainerView
        android:id="@+id/fragment_container"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:name="androidx.navigation.fragment.NavHostFragment"
        app:defaultNavHost="true"
        app:navGraph="@navigation/nav_graph" />

</androidx.constraintlayout.widget.ConstraintLayout>
```
##### NavController
可以通过以下方式获取：
```
binding.fragmentContainer.findNavController()
Navigation.findNavController(binding.fragmentContainer)
```
!> NavController看源码只能在onStart之后才能获取到

##### 获取NavHostFragment
```
(supportFragmentManager.fragments[0] as? NavHostFragment)
```
##### 获取所有Fragment实例
```
(supportFragmentManager.fragments[0] as? NavHostFragment)?.childFragmentManager?.fragments
```
##### onBackPressed
通过获取当前导航视图中Fragment Id判断是否直接退出Activity而不返回上一个堆栈
```
override fun onBackPressed() {
        if (Navigation.findNavController(binding.fragmentContainer).currentDestination?.id == R.id.fragment_b) {
            finish()
        } else {
            super.onBackPressed()
        }
    }
```


### 创建基类BaseNavigationFragment
```
open class BaseNavigationFragment : Fragment() {
    protected lateinit var binding: FragmentBaseNavigationBinding
    var fromWhere: String? = null
    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        fromWhere = arguments?.get("from") as? String
        LogUtils.e(fromWhere)
    }

    override fun onCreateView(
        inflater: LayoutInflater,
        container: ViewGroup?,
        savedInstanceState: Bundle?
    ): View? {
        binding = FragmentBaseNavigationBinding.inflate(inflater, container, false)
        return binding.root
    }
}

<?xml version="1.0" encoding="utf-8"?>
<androidx.constraintlayout.widget.ConstraintLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <TextView
        android:id="@+id/tx_navigation"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:gravity="center"
        android:text="A"
        android:textSize="80sp"
        android:textStyle="bold" />
</androidx.constraintlayout.widget.ConstraintLayout>
```
基于BaseNavigationFragment新建3个子Fragment：FragmentA、FragmentB、FragmentC

#### FragmentA
演示通过 action_id跳转FragmentB
```
class FragmentA : BaseNavigationFragment() {
    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        binding.txNavigation.text = "A"
        binding.txNavigation.setOnClickListener {
            Navigation.findNavController(binding.root).navigate(
                R.id.action_fragment_a_to_b,
                bundleOf("from" to binding.txNavigation.text.toString())
            )
        }
    }
}
```

#### FragmentB
演示通过action跳转FragmentC
```
class FragmentB : BaseNavigationFragment() {
    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        binding.txNavigation.text = "B"
        binding.root.setBackgroundResource(R.color.purple_200)
        binding.txNavigation.setOnClickListener {
            Navigation.findNavController(binding.root)
                .navigate(FragmentBDirections.actionFragmentBToC(binding.txNavigation.text.toString()))
        }
    }
}
```
!> FragmentBDirections.actionFragmentBToC 由 androidx.navigation.safeargs.kotlin 根据action id插件自动生成

#### FragmentB
跳回FragmentA
```
class FragmentC: BaseNavigationFragment() {
    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        binding.txNavigation.text = "C"
        binding.root.setBackgroundResource(R.color.purple_500)
        binding.txNavigation.setOnClickListener {
            Navigation.findNavController(binding.root).navigate( FragmentCDirections.actionCToA(binding.txNavigation.text.toString()))
        }
    }
}
```

### 问题
通过： (supportFragmentManager.fragments[0] as? NavHostFragment)?.childFragmentManager?.fragments 获取所有Fragment实例时，永远只有一个Fragment，看源码因为NavController在Fragment切换时用的是replace方法，那上一个导航视图就无法保存，所有每次页面回退时前一个Fragment都会进行重建，这在部分业务场景下是不符合预期的：

解决方法一：
https://github.com/android/architecture-components-samples/issues/530#issuecomment-526455102

解决方法二：
通过viewmodel存储fragment数据，然后进行恢复，简单的页面处理比较简单，但是对较复杂的页面时恢复所需要的工足量比较大。
