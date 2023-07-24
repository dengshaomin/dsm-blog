### 概述
Java字节码以二进制的形式存储在.class文件中，每一个.class文件包含一个Java类或接口。Javaassist就是一个用来处理Java字节码的类库。它可以在一个已经编译好的类中添加新的方法，或者是修改已有的方法，并且不需要对字节码方面有深入的了解。同时也可以通过完全手动的方式生成一个新的类对象。

Maven依赖方式：
```
<dependency>
    <groupId>org.javassist</groupId>
    <artifactId>javassist</artifactId>
    <version>3.27.0-GA</version>
</dependency>
```
Gradle依赖方式：
```
implementation 'org.javassist:javassist:3.27.0-GA'
```

### ClassPool
ClassPool是CtClass对象的容器，它按需读取类文件来构造CtClass对象，并且保存CtClass对象以便以后使用。

从实现的角度来看，ClassPool 是一个存储 CtClass 的 Hash 表，类的名称作为 Hash 表的 key。ClassPool 的 get() 函数用于从 Hash 表中查找 key 对应的 CtClass 对象。如果没有找到，get() 函数会创建并返回一个新的 CtClass 对象，这个新对象会保存在 Hash 表中。

需要注意的是ClassPool会在内存中维护所有被它创建过的CtClass，当CtClass数量过多时，会占用大量的内存，API中给出的解决方案是重新创建ClassPool 或 有意识的调用CtClass的detach()方法以释放内存。

ClassPool需要关注的方法：

*   getDefault：返回默认的ClassPool，一般通过该方法创建我们的ClassPool；
*   appendClassPath, insertClassPath：将一个ClassPath加到类搜索路径的末尾位置 或 插入到起始位置。通常通过该方法写入额外的类搜索路径，以解决多个类加载器环境中找不到类的尴尬；
*   toClass：将修改后的CtClass加载至当前线程的上下文类加载器中，CtClass的toClass方法是通过调用本方法实现。需要注意的是一旦调用该方法，则无法继续修改已经被加载的class；
*   makeClass：根据类名创建新的CtClass对象；
*   get，getCtClass：根据类路径名获取该类的CtClass对象，用于后续的编辑。
可以使用toBytecode()函数来获取修改过的字节码：
```
byte[] b = cc.toBytecode();
```
也可以通过toClass()函数直接将CtClass转换成Class对象:
```
Class clazz = cc.toClass();
```

toClass()会请求当前线程的ClassLoader加载CtClass所代表的类文件,它返回此类文件的java.lang.Class对象。

### CtClass
CtClass类表示一个class文件，每个CtClass对象都必须从ClassPool中获取，CtClass需要关注的方法：

*   freeze：冻结一个类，使其不可修改；
*   isFrozen：判断一个类是否已被冻结；
*   defrost：解冻一个类，使其可以被修改；
*   prune：删除类不必要的属性，以减少内存占用。调用该方法后，许多方法无法将无法正常使用，慎用；
*   detach：将该class从ClassPool中删除；
*   writeFile：根据CtClass生成.class文件；
*   toClass：通过类加载器加载该CtClass；
*   addField，removeField：添加/移除一个CtField；
*   addMethod，removeMethod：添加/移除一个CtMethod；
*   addConstructor，removeConstructor：添加/移除一个CtConstructor。

如果一个 CtClass 对象通过 writeFile(), toClass(), toBytecode() 被转换成一个类文件，此 CtClass 对象会被冻结起来，不允许再修改，因为一个类只能被 JVM 加载一次。

但是，一个冷冻的 CtClass 也可以被解冻，例如：
```
CtClasss cc = ...;
cc.writeFile();
cc.defrost();
cc.setSuperclass(...); // 因为类已经被解冻，所以这里可以调用成功
```

调用 defrost() 之后，此 CtClass 对象又可以被修改了。

如果 ClassPool.doPruning 被设置为 true，Javassist 在冻结 CtClass 时，会修剪 CtClass 的数据结构。为了减少内存的消耗，修剪操作会丢弃 CtClass 对象中不必要的属性。例如，Code_attribute 结构会被丢弃。一个 CtClass 对象被修改之后，方法的字节码是不可访问的，但是方法名称、方法签名、注解信息可以被访问。修剪过的 CtClass 对象不能再次被解冻。ClassPool.doPruning 的默认值为 false。

stopPruning() 可以用来驳回修剪操作。
```
CtClasss cc = ...;
cc.stopPruning(true);
cc.writeFile(); // 转换成一个 class 文件
// cc is not pruned.
```

这个 CtClass 没有被修剪，所以在 writeFile() 之后，可以被解冻。

注意：调试的时候可能临时需要停止修剪和冻结，然后保存一个修改过的类文件到磁盘，debugWriteFile() 方法正是为此准备的。它停止修剪，然后写类文件，然后解冻并再次打开修剪（如果开始时修养是打开的）。

### CtMthod
CtMthod代表类中的某个方法，可以通过CtClass提供的API获取 或者 构造方法 或者 CtNewMethod.make()方法新建，通过CtMethod对象可以实现对方法的修改。

CtMethod中的一些重要方法：

*   insertBefore：在方法的起始位置插入代码；
*   insterAfter：在方法的所有 return 语句前插入代码以确保语句能够被执行，除非遇到exception；
*   insertAt：在指定的位置插入代码；
*   setBody：将方法的内容设置为要写入的代码，当方法被abstract修饰时，该修饰符被移除；
*   make：创建一个新的方法。

CtNewMethod是一个用来创建CtMethod实例的类，其一个make方法如下：
```
public static CtMethod make(int modifiers, CtClass returnType,
                            String mname, CtClass[] parameters,
                            CtClass[] exceptions,
                            String body, CtClass declaring)
    throws CannotCompileException
{
    try {
        CtMethod cm
            = new CtMethod(returnType, mname, parameters, declaring);
        cm.setModifiers(modifiers);
        cm.setExceptionTypes(exceptions);
        cm.setBody(body);
        return cm;
    }
    catch (NotFoundException e) {
        throw new CannotCompileException(e);
    }
}
```
也可以通过CtNewMethod.setter/getter方法为某个属性创建get/set方法：
```
CtClass cc = ...;
CtField param = ...;
cc.addMethod(CtNewMethod.setter("setName", param));
cc.addMethod(CtNewMethod.getter("getName", param));
```

### CtField
CtField代表类中的某个属性，可以直接通过其构造方法创建实例：
```
CtClass cc = pool.makeClass("com.hearing.demo.Person");
CtField param = new CtField(pool.get("java.lang.String"), "name", cc);
param.setModifiers(Modifier.PRIVATE);
cc.addField(param, CtField.Initializer.constant("hearing"));
```

### CtConstructor
CtConstructor代表类中的一个构造器，可以通过CtConstructor.make方法创建：
```
public static CtConstructor make(CtClass[] parameters,
                                    CtClass[] exceptions,
                                    String body, CtClass declaring)
    throws CannotCompileException
{
    try {
        CtConstructor cc = new CtConstructor(parameters, declaring);
        cc.setExceptionTypes(exceptions);
        cc.setBody(body);
        return cc;
    }
    catch (NotFoundException e) {
        throw new CannotCompileException(e);
    }
}
```
也可以通过构造方法直接创建：
```
CtConstructor cons = new CtConstructor(new CtClass[]{pool.get("java.lang.String")}, cc);
// $0=this / $1,$2,$3... 代表方法参数
cons.setBody("{$0.name = $1;}");
cc.addConstructor(cons);
```

### ClassPath
ClassPath是一个接口，代表类的搜索路径，含有具体的搜索实现。当通过其它途径无法获取要编辑的类时，可以尝试定制一个自己的ClassPath。API提供的实现中值得关注的有：

*   ByteArrayClassPath：将类以字节码的形式加入到该path中，ClassPool可以从该path中生成所需的CtClass。
*   ClassClassPath：通过某个class生成的path，通过该class的classloader来尝试加载指定的类文件。
*   LoaderClassPath：通过某个classloader生成path，并通过该classloader搜索加载指定的类文件。需要注意的是该类加载器以弱引用的方式存在于path中，当不存在强引用时，随时可能会被清理。

通过 ClassPool.getDefault() 获取的 ClassPool 使用 JVM 的类搜索路径。如果程序运行在 JBoss 或者 Tomcat 等 Web 服务器上，ClassPool 可能无法找到用户的类，因为 Web 服务器使用多个类加载器作为系统类加载器。在这种情况下，ClassPool 必须添加额外的类搜索路径。

下面的例子中，pool 代表一个 ClassPool 对象：
```
pool.insertClassPath(new ClassClassPath(this.getClass()));
```

上面的语句将 this 指向的类添加到 pool 的类加载路径中。你可以使用任意 Class 对象来代替 this.getClass()，从而将 Class 对象添加到类加载路径中。

也可以注册一个目录作为类搜索路径。下面的例子将 /usr/local/javalib 添加到类搜索路径中：
```
ClassPool pool = ClassPool.getDefault();
pool.insertClassPath("/usr/local/javalib");
```

类搜索路径不但可以是目录，还可以是 URL ：
```
ClassPool pool = ClassPool.getDefault();
ClassPath cp = new URLClassPath("www.javassist.org", 80, "/java/", "org.javassist.");
pool.insertClassPath(cp);
```

上述代码将 http://www.javassist.org:80/java/ 添加到类搜索路径。并且这个URL只能搜索 org.javassist 包里面的类。例如，为了加载 org.javassist.test.Main，它的类文件会从获取 http://www.javassist.org:80/java/org/javassist/test/Main.class 获取。

此外，也可以直接传递一个 byte 数组给 ClassPool 来构造一个 CtClass 对象，完成这项操作，需要使用 ByteArrayPath 类。示例：
```
ClassPool cp = ClassPool.getDefault();
byte[] b = a byte array;
String name = class name;
cp.insertClassPath(new ByteArrayClassPath(name, b));
CtClass cc = cp.get(name);
```

示例中的 CtClass 对象表示 b 代表的 class 文件。将对应的类名传递给 ClassPool 的 get() 方法，就可以从 ByteArrayClassPath 中读取到对应的类文件。

如果你不知道类的全名，可以使用 makeClass() 方法：
```
ClassPool cp = ClassPool.getDefault();
InputStream ins = an input stream for reading a class file;
CtClass cc = cp.makeClass(ins);
```

makeClass() 返回从给定输入流构造的 CtClass 对象。你可以使用 makeClass() 将类文件提供给 ClassPool 对象。如果搜索路径包含大的 jar 文件，这可能会提高性能。由于 ClassPool 对象按需读取类文件，它可能会重复搜索整个 jar 文件中的每个类文件。 makeClass() 可以用于优化此搜索。由 makeClass() 构造的 CtClass 保存在 ClassPool 对象中，从而使得类文件不会再被读取。

用户可以通过实现 ClassPath 接口来扩展类加载路径，然后调用 ClassPool 的 insertClassPath() 方法将路径添加进来。这种技术主要用于将非标准资源添加到类搜索路径中。

### ClassLoader
CtClass的toClass()方法请求当前线程的上下文类加载器，加载CtClass对象所表示的类：
```
public class Hello {
    public void say() {
        System.out.println("Hello");
    }
}

public class Test {
    public static void main(String[] args) throws Exception {
        ClassPool cp = ClassPool.getDefault();
        CtClass cc = cp.get("Hello");
        CtMethod m = cc.getDeclaredMethod("say");
        m.insertBefore("{ System.out.println(\"Hello.say():\"); }");
        Class c = cc.toClass();
        Hello h = (Hello) c.newInstance();
        h.say();
    }
}
```
注意：上面的程序要正常运行，Hello 类在调用 toClass() 之前不能被加载。如果 JVM 在 toClass() 调用之前加载了原始的 Hello 类，后续加载修改的 Hello 类将会失败（LinkageError 抛出）。例如，如果 Test 中的 main() 是这样的：
```
public static void main(String[] args) throws Exception {
    Hello orig = new Hello();
    ClassPool cp = ClassPool.getDefault();
    CtClass cc = cp.get("Hello");
}
```
那么，原始的 Hello 类在 main 的第一行被加载，toClass() 调用会抛出一个异常，因为类加载器不能同时加载两个不同版本的 Hello 类。

如果程序在某些应用程序服务器（如JBoss和Tomcat）上运行，toClass()使用的上下文类加载器可能是不合适的。在这种情况下，你会看到一个意想不到的 ClassCastException。为了避免这个异常，必须给 toClass() 指定一个合适的类加载器。例如：
```
CtClass cc = ...;
Class c = cc.toClass(bean.getClass().getClassLoader());
```

应该给toClass()传递加载了你的程序的类加载器（上例中，bean对象的类），toClass() 是为了简便而提供的方法，如果你需要更复杂的功能，你应该编写自己的类加载器。

Javassit 提供一个类加载器 javassist.Loader。它使用 javassist.ClassPool 对象来读取类文件。

例如，javassist.Loader 可以用于加载用 Javassist 修改过的类。
```
public class Main {
    public static void main(String[] args) throws Throwable {
        ClassPool pool = ClassPool.getDefault();
        Loader cl = new Loader(pool);

        CtClass ct = pool.get("test.Rectangle");
        ct.setSuperclass(pool.get("test.Point"));

        Class c = cl.loadClass("test.Rectangle");
        Object rect = c.newInstance();
    }
}
```
这个程序将 test.Rectangle 的超类设置为 test.Point。然后再加载修改的类，并创建新的 test.Rectangle 类的实例。

如果用户希望在加载时按需修改类，则可以向 javassist.Loader 添加事件监听器。当类加载器加载类时会通知监听器。事件监听器类必须实现以下接口：
```
public interface Translator {
    public void start(ClassPool pool)
        throws NotFoundException, CannotCompileException;
    public void onLoad(ClassPool pool, String classname)
        throws NotFoundException, CannotCompileException;
}
```

当事件监听器通过 addTranslator() 添加到 javassist.Loader 对象时，start() 方法会被调用。在 javassist.Loader 加载类之前，会调用 onLoad() 方法。可以在 onLoad() 方法中修改被加载的类的定义。

例如，下面的事件监听器在类加载之前，将所有类更改为 public 类。
```
public class MyTranslator implements Translator {
    void start(ClassPool pool) throws NotFoundException, CannotCompileException {}
    void onLoad(ClassPool pool, String classname) throws NotFoundException, CannotCompileException {
        CtClass cc = pool.get(classname);
        cc.setModifiers(Modifier.PUBLIC);
    }
}
```

注意，onLoad() 不必调用 toBytecode() 或 writeFile()，因为 javassist.Loader 会调用这些方法来获取类文件。

### 示例
####    创建Class文件
```
public class App {
    public static void main(String[] args) {
        try {
            createPerson();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    private static void createPerson() throws Exception {
        ClassPool pool = ClassPool.getDefault();

        // 1. 创建一个空类
        CtClass cc = pool.makeClass("com.hearing.demo.Person");

        // 2. 新增一个字段 private String name = "hearing";
        CtField param = new CtField(pool.get("java.lang.String"), "name", cc);
        param.setModifiers(Modifier.PRIVATE);
        cc.addField(param, CtField.Initializer.constant("hearing"));

        // 3. 生成 getter、setter 方法
        cc.addMethod(CtNewMethod.setter("setName", param));
        cc.addMethod(CtNewMethod.getter("getName", param));

        // 4. 添加无参的构造函数
        CtConstructor cons = new CtConstructor(new CtClass[]{}, cc);
        cons.setBody("{name = \"hearing\";}");
        cc.addConstructor(cons);

        // 5. 添加有参的构造函数
        cons = new CtConstructor(new CtClass[]{pool.get("java.lang.String")}, cc);
        // $0=this / $1,$2,$3... 代表方法参数
        cons.setBody("{$0.name = $1;}");
        cc.addConstructor(cons);

        // 6. 创建一个名为printName方法，无参数，无返回值，输出name值
        CtMethod ctMethod = new CtMethod(CtClass.voidType, "printName", new CtClass[]{}, cc);
        ctMethod.setModifiers(Modifier.PUBLIC);
        ctMethod.setBody("{System.out.println(name);}");
        cc.addMethod(ctMethod);

        //这里会将这个创建的类对象编译为.class文件
        cc.writeFile("/.../path/");
    }
}
```

创建的class文件如下：
```
//
// Source code recreated from a .class file by IntelliJ IDEA
// (powered by Fernflower decompiler)
//
package com.hearing.demo;

public class Person {
    private String name = "hearing";

    public void setName(String var1) {
        this.name = var1;
    }

    public String getName() {
        return this.name;
    }

    public Person() {
        this.name = "hearing";
    }

    public Person(String var1) {
        this.name = var1;
    }

    public void printName() {
        System.out.println(this.name);
    }
}
```
####    调用生成的类对象
#####   1.通过反射的方式调用：
```
Object person = cc.toClass().newInstance();
Method setName = person.getClass().getMethod("setName", String.class);
setName.invoke(person, "hearing1");
Method execute = person.getClass().getMethod("printName");
execute.invoke(person);
```

#####   2.通过读取class文件的方式调用：
```
ClassPool pool = ClassPool.getDefault();
// 设置类路径
pool.appendClassPath("/.../path/");
CtClass ctClass = pool.get("com.hearing.demo.Person");
Object person = ctClass.toClass().newInstance();
// 下面和通过反射的方式一样去使用
```

#####   3.通过接口的方式：

上面两种其实都是通过反射的方式去调用，问题在于我们的工程中其实并没有这个类对象，所以反射的方式比较麻烦，并且开销也很大。那么如果你的类对象可以抽象为一些方法的合集，就可以考虑为该类生成一个接口类。这样在newInstance()的时候我们就可以强转为接口，可以将反射的那一套省略掉了。

还拿上面的Person类来说，新建一个IPerson接口类：
```
public interface IPerson {
    void setName(String name);
    String getName();
    void printName();
}
```
实现部分的代码如下：
```
ClassPool pool = ClassPool.getDefault();
pool.appendClassPath("/.../path/");

// 获取接口
CtClass codeClassI = pool.get("com.hearing.demo.IPerson");
// 获取上面生成的类
CtClass ctClass = pool.get("com.hearing.demo.Person");
// 使代码生成的类，实现 IPerson 接口
ctClass.setInterfaces(new CtClass[]{codeClassI});

// 以下通过接口直接调用 强转
IPerson person = (IPerson)ctClass.toClass().newInstance();
System.out.println(person.getName());
person.setName("hearing1");
person.printName();
```

####    修改现有的类对象
有如下类对象：
```
public class Test {
    public void test1() {
        System.out.println("I am test1");
    }
}
```
然后进行修改：
```
public class App {
    public static void main(String[] args) {
        try {
            update();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    private static void update() throws Exception {
        ClassPool pool = ClassPool.getDefault();
        CtClass cc = pool.get("com.hearing.demo.Test");

        CtMethod personFly = cc.getDeclaredMethod("test1");
        personFly.insertBefore("System.out.println(\"...before...\");");
        personFly.insertAfter("System.out.println(\"...after...\");");

        //新增一个方法
        CtMethod ctMethod = new CtMethod(CtClass.voidType, "test2", new CtClass[]{}, cc);
        ctMethod.setModifiers(Modifier.PUBLIC);
        ctMethod.setBody("{System.out.println(\"I am test2\");}");
        cc.addMethod(ctMethod);

        Object test = cc.toClass().newInstance();
        Method personFlyMethod = test.getClass().getMethod("test1");
        personFlyMethod.invoke(test);
        Method execute = test.getClass().getMethod("test2");
        execute.invoke(test);
    }
}
```

需要注意的是：上面的insertBefore() 和 setBody()中的语句，如果是单行语句可以直接用双引号，但是有多行语句的情况下，需要将多行语句用{}括起来。javassist只接受单个语句或用大括号括起来的语句块。

***
https://ljd1996.github.io/2020/04/23/Javassist%E7%94%A8%E6%B3%95/
https://www.jianshu.com/p/dfc4681f8090
***