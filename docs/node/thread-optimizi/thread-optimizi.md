
[抖音线程优化](https://juejin.cn/post/7212446354920407096#heading-10)

##### 减少线程占用的虚拟内存
CreateNativeThread 源码，该函数会执行 FixStackSize 方法将 stack_size 调整为 1M。那结合前面各种 hook 的案例，我们很容易就能想到，通过 hook FixStackSize 这个函数，是不是可以将 stack_size 的从 1M 减少到 512 KB 了呢？ 当时是可以的，但是这个时候我们没法通过 PLT Hook 的方案来实现了，而是要通过 Inline Hook 方案实现，因为 FixStackSize 是 so 库内部函数的调用，所以只有 FixStackSize 才能实现。
那如果我们想用 PLT Hook 方案来实现可以做到么？其实也可以。CreateNativeThread 是位于 libart.so 中的函数，但是 CreateNativeThread 实际是调用 pthread_create 来创建线程的，而 pthread_create 是位于 libc.so 库中的函数，如果在 CreateNativeThread 中调用 pthread_create ，同样需要通过走 plt 表和 got 表查询地址的方式，所以我们通过 bhook 工具 hook 住 libc.so 库中的 pthread_create 函数，将入参 &attr 中的 stack_size 直接设置成 512KB 即可，实现起来也非常简单，一行代码即可。
```
static int AdjustStackSize(pthread_attr_t const* attr) {
    pthread_attr_setstacksize(attr, 512 * 1024);
}
```
至于如何 hook 住 pthread_create 这个函数的方法也非常简单，通过 bhook 也是一行代码就能实现，前面的篇章已经讲过怎么使用了，所以这个方案剩下的部分就留给你自己去实践啦。
除了 Native Hook 方案，我们还能在 Java 层通过字节码操作的方式来实现该方案。stack_size 不就是通过 Java 层传递到 Native 层嘛，那我们直接在 Java 层调整 stack_size 的大小就可以了，但在这之前之前，要先看看在 FixStackSize 函数中是如何调整 stack_size 大小的。
```
static size_t FixStackSize(size_t stack_size) {

  if (stack_size == 0) {
    stack_size = Runtime::Current()->GetDefaultStackSize();
  }

  stack_size += 1 * MB;

  ……

  return stack_size;
}
```
FixStackSize 函数的源码实现很简单，就是通过 stack_size += 1 * MB 来设置 stack_size 的：如果我们传入的 stack_size 为 0 时，默认大小就是 1 M ；如果我们传入的 stack_size 为 -512KB 时，stack_size 就会变成 512KB（1M - 512KB）。那我们是不是只用带有 stackSize 入参的构造函数去创建线程，并且设置 stackSize 为 -512KB 就行了呢？
public Thread(ThreadGroup group, Runnable target, String name,
                  long stackSize) {
    this(group, target, name, stackSize, null, true);
}
复制代码
是的，但是因为应用中创建线程的地方太多很难一一修改，而且我们实际不需要这样去修改。前面我们已经将应用中的线程全部收敛到公共线程池中去创建了，所以只需要修改公共线程池中创建的线程方式就可以了，并且线程池刚好也可以让我们自己创建线程，那只需要传入自定义的 ThreadFactory 就能实现需求。








在我们自定义的 ThreadFactory 中，创建 stack_size 为 - 512 kb 的线程，这么一个简单的操作就能减少线程所占用的虚拟内存。

当我们将应用中线程栈的大小全改成 512 kb 后，可能会导致一些任务比较重的线程出现栈溢出，此时我们可以通过埋点收集会栈溢出的线程，不修改这部分线程的大小即可。总的来说，这是一个容易落地且投入产出比高的方案。
通过上面的方案介绍，我们也可以看到，减少一个线程所占用的虚拟内存的方案很多，可以通过 Native Hook，也可以通过 Java 代码直接修改。我们在做业务或者性能相关的工作时，往往都有多个实现方案，但是我们在敲定最终方案时，始终要选择最简单、最稳定且投入产出比最高的方案。


