***
Gson

FastJson

data class bean

java class bean
***

### Class Bean

**Gson解析Kotlin data class bean**

```
package com.android.myapplication
import com.google.gson.annotations.SerializedName
data class Student(
    @SerializedName("age_x")
    val age: Int,
    val name: String = "xiao",
)  {

}
```
!> 使用Gson解析时，开启混淆，无需在proguard_rules.txt文件中keep class；使用@SerializedName重定向字段

**Gson解析java class bean**
```
package com.android.myapplication;

import com.google.gson.annotations.SerializedName;

import java.io.Serializable;

public class JavaStudent {
    public String name;
    @SerializedName("age_x")
    public String age;
}

```
!> 开启混淆，需要在proguard_rules.txt加入 -keep class com.android.myapplication.JavaStudent{*;}，否则无法正常解析；使用@SerializedName重定向字段

**fastjson解析kotlin data class bean**

```
package com.android.myapplication
import androidx.annotation.Keep
import com.alibaba.fastjson2.annotation.JSONCreator
import com.alibaba.fastjson2.annotation.JSONField

@Keep
data class FTStudent @JSONCreator constructor(
    @JSONField(name = "age_x")
    val age: Int,
    @JSONField(name = "name")
    val name: String
) {
}
```
!> 开始混淆后使用@Keep防止被混淆，无需在proguard_rules.txt中keep class bean；所有字段需要使用@JSONField注解；@JSONField可重定向字段

### Demo
```
val jsonStr = """
            {
                "name":"小明",
                "age":10,
                "age_x":11
            }
        """.trimIndent()
        val student = Gson().fromJson(jsonStr, Student::class.java)
        var str = Gson().toJson(student)
        Log.e("balance", str)
        val javaStudent = Gson().fromJson(jsonStr, JavaStudent::class.java)
        str = Gson().toJson(javaStudent)
        Log.e("balance", str)
        val fastStudent = JSON.parseObject(jsonStr, FTStudent::class.java)
        str = Gson().toJson(fastStudent)
        Log.e("balance", str)
```

**Result:**
```
2023-06-14 16:20:37.128 21624-21624 balance                 com.android.myapplication            E  {"age_x":11,"name":"小明"}
2023-06-14 16:20:37.129 21624-21624 balance                 com.android.myapplication            E  {"age_x":"11","name":"小明"}
2023-06-14 16:20:37.189 21624-21624 balance                 com.android.myapplication            E  {"age":11,"name":"小明"}
```
