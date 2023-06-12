### 一、开始

上一篇Android CoordinatorLayout之自定义Behavior中，我们简单介绍了CoordinatorLayout以及如何自定义Behavior。所以这次我们从源码的角度分析CoordinatorLayout的内部实现机制，以便它更好的服务我们！

本文内容主要围绕Behavior展开，还不了解Behavior的建议先看上一篇。

### 二、Behavior的初始化
通常使用Behavior是通过给View设置layout_behavior属性，属性值是Behavior的路径，很显然，这个属性值最终是绑定在了LayoutParams上。可以猜测，和FrameLayout等自带ViewGroup类似，CoordinatorLayout内部也有一个LayoutParams类！嗯，找到了如下代码段：
```
public static class LayoutParams extends ViewGroup.MarginLayoutParams {
     
        LayoutParams(Context context, AttributeSet attrs) {
            super(context, attrs);
            // 是否设置了layout_behavior属性
            mBehaviorResolved = a.hasValue(
                    R.styleable.CoordinatorLayout_Layout_layout_behavior);
            if (mBehaviorResolved) {
                mBehavior = parseBehavior(context, attrs, a.getString(
                        R.styleable.CoordinatorLayout_Layout_layout_behavior));
            }
            a.recycle();

            if (mBehavior != null) {
                // If we have a Behavior, dispatch that it has been attached
                mBehavior.onAttachedToLayoutParams(this);
            }
        }
```
首先判断是否给View设置了layout_behavior属性，如果设置了则先得到Behavior路径再去解析Behavior，我们重点看下parseBehavior()方法：
```
static Behavior parseBehavior(Context context, AttributeSet attrs, String name) {
        if (TextUtils.isEmpty(name)) {
            return null;
        }
        // behavior的全包名路径
        final String fullName;
        if (name.startsWith(".")) {
            // 如果设置behavior路径不包含包名，则需要拼接包名
            fullName = context.getPackageName() + name;
        } else if (name.indexOf('.') >= 0) {
            // 设置了behavior的全包名路径
            fullName = name;
        } else {
            // 系统内部实现，WIDGET_PACKAGE_NAME代表android.support.design.widget
            fullName = !TextUtils.isEmpty(WIDGET_PACKAGE_NAME)
                    ? (WIDGET_PACKAGE_NAME + '.' + name)
                    : name;
        }

        try {
            Map<String, Constructor<Behavior>> constructors = sConstructors.get();
            if (constructors == null) {
                constructors = new HashMap<>();
                sConstructors.set(constructors);
            }
            Constructor<Behavior> c = constructors.get(fullName);
            // 通过反射实例化behavior
            if (c == null) {
                final Class<Behavior> clazz = (Class<Behavior>) Class.forName(fullName, true,
                        context.getClassLoader());
                // 指定behavior的构造函数有两个参数
                // CONSTRUCTOR_PARAMS = new Class<?>[] {Context.class, AttributeSet.class}
                // 这也是我们自定义behavior需要两个参数的构造函数的原因
                c = clazz.getConstructor(CONSTRUCTOR_PARAMS);
                c.setAccessible(true);
                constructors.put(fullName, c);
            }
            // 返回behavior的实例
            return c.newInstance(context, attrs);
        } catch (Exception e) {
            throw new RuntimeException("Could not inflate Behavior subclass " + fullName, e);
        }
    }
```
parseBehavior()方法就是使用设置的Behavior的路径进一步通过反射得到Behavior实例。如果要使用该Behavior实例，可通过CoordinatorLayout.LayoutParams的getBehavior()方法得到：
```
public Behavior getBehavior() {
            return mBehavior;
        }
```
除此之外，还可以在代码中通过LayoutParams给View设置Behavior，例如修改上一篇demo-1的behavior设置方式：
```
TextView title = findViewById(R.id.title);
CoordinatorLayout.LayoutParams params = (CoordinatorLayout.LayoutParams) title.getLayoutParams();
params.setBehavior(new SampleTitleBehavior());
```

就是通过CoordinatorLayout.LayoutParams的setBehavior()方法完成的。

最后还有一种就是通过注解，系统的AppBarLayout就是用@CoordinatorLayout.DefaultBehavior()注解来设置behavioe的：

```
@CoordinatorLayout.DefaultBehavior(AppBarLayout.Behavior.class)
public class AppBarLayout extends LinearLayout {
}
```
如果某个View需要固定设置某个Behavior，注解是个不错的选择，至于注解的使用时机和原理后边会提到的。

到此，给View设置Behavior的方式和原理就基本结束了！

### 三、CoordinatorLayout的测量、布局
前边已经分析了Behavior的初始化过程，初始化好了，总要用吧，在哪里用呢？莫急，先看CoordinatorLayout在代码层面是什么：
```
public class CoordinatorLayout extends ViewGroup implements NestedScrollingParent2 {
}
```
嗯，一个自定义ViewGroup，同时实现了NestedScrollingParent2接口，既然这样，CoordinatorLayout必然遵循oMeasure()、onLayout()的执行流程。

所以我们从CoordinatorLayout的onMeasure()方法开始分析(只保留了核心代码)：
```
@Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        prepareChildren();
        ensurePreDrawListener();

        final int childCount = mDependencySortedChildren.size();
        // 遍历子View，并测量大小
        for (int i = 0; i < childCount; i++) {
            final View child = mDependencySortedChildren.get(i);
            if (child.getVisibility() == GONE) {
                // If the child is GONE, skip...
                continue;
            }

            final LayoutParams lp = (LayoutParams) child.getLayoutParams();
            // 得到给View设置的Behavior
            final Behavior b = lp.getBehavior();
            if (b == null || !b.onMeasureChild(this, child, childWidthMeasureSpec, keylineWidthUsed,
                    childHeightMeasureSpec, 0)) {
                onMeasureChild(child, childWidthMeasureSpec, keylineWidthUsed,
                        childHeightMeasureSpec, 0);
            }
        }
        setMeasuredDimension(width, height);
    }
```
先看prepareChildren()方法：
```
private void prepareChildren() {
        mDependencySortedChildren.clear();
        mChildDag.clear();

        for (int i = 0, count = getChildCount(); i < count; i++) {
            final View view = getChildAt(i);

            final LayoutParams lp = getResolvedLayoutParams(view);
            lp.findAnchorView(this, view);

            mChildDag.addNode(view);

            // 按照View之间的依赖关系，存储View
            for (int j = 0; j < count; j++) {
                if (j == i) {
                    continue;
                }
                final View other = getChildAt(j);
                if (lp.dependsOn(this, view, other)) {
                    if (!mChildDag.contains(other)) {
                        // Make sure that the other node is added
                        mChildDag.addNode(other);
                    }
                    // Now add the dependency to the graph
                    mChildDag.addEdge(other, view);
                }
            }
        }

        // mChildDag.getSortedList()会返回一个按照依赖关系排序后的View集合
        // 被依赖的View排在前边，没有被依赖的在后边
        mDependencySortedChildren.addAll(mChildDag.getSortedList());
        Collections.reverse(mDependencySortedChildren);
    }
```
很明显prepareChildren()就是完成CoordinatorLayout中子View按照依赖关系的排列，被依赖的View排在前面，并将结果保存在mDependencySortedChildren中，在每次测量前都会重新排序。

还记得我们前边遗留了一个问题吗？就是注解形式的Behavior初始化，答案就在getResolvedLayoutParams()中：
```
LayoutParams getResolvedLayoutParams(View child) {
        final LayoutParams result = (LayoutParams) child.getLayoutParams();
        if (!result.mBehaviorResolved) {
            Class<?> childClass = child.getClass();
            DefaultBehavior defaultBehavior = null;
            while (childClass != null &&
                    (defaultBehavior = childClass.getAnnotation(DefaultBehavior.class)) == null) {
                childClass = childClass.getSuperclass();
            }
            if (defaultBehavior != null) {
                try {
                    result.setBehavior(
                            defaultBehavior.value().getDeclaredConstructor().newInstance());
                } catch (Exception e) {
                    Log.e(TAG, "Default behavior class " + defaultBehavior.value().getName() +
                            " could not be instantiated. Did you forget a default constructor?", e);
                }
            }
            result.mBehaviorResolved = true;
        }
        return result;
    }
```
其实很简单，如果没有通过layout_behavior或者java代码给View设置Behavior，则LayoutParams的成员变量mBehaviorResolved为false，此时如果通注解设置了Behavior则会在此完成Behavior初始化操作。可见，通过注解设置的Behavior被处理的优先级最低。

先跳过ensurePreDrawListener()方法，继续看onMeasure()方法剩下的代码，即遍历CoordinatorLayout的子View，注意这里的子View从已排序的mDependencySortedChildren列表里得到，先拿到View的LayoutParams再从中取出Behavior，如果Behavior非空，并且重写了onMeasureChild()方法，则按照重写的规则测量该子View，否则执行系统的默认测量。可以发现，如果有需要我们可以重写Behavior的onMeasureChild()方法，拦截系统的默认onLayoutChild()方法。

继续分析onLayout()方法：
```
@Override
    protected void onLayout(boolean changed, int l, int t, int r, int b) {
        final int layoutDirection = ViewCompat.getLayoutDirection(this);
        final int childCount = mDependencySortedChildren.size();
        for (int i = 0; i < childCount; i++) {
            final View child = mDependencySortedChildren.get(i);
            if (child.getVisibility() == GONE) {
                // If the child is GONE, skip...
                continue;
            }

            final LayoutParams lp = (LayoutParams) child.getLayoutParams();
            final Behavior behavior = lp.getBehavior();

            if (behavior == null || !behavior.onLayoutChild(this, child, layoutDirection)) {
                onLayoutChild(child, layoutDirection);
            }
        }
    }
```
其实和onMeasure()方法类似，遍历子View，确定其位置。同样，如果有需要我们可以重写Behavior的onLayoutChild()方法，拦截系统的默认的onLayoutChild()方法。

初见端倪，我Behavior就是能为所欲为，想拦截就拦截。类似的Behavior拦截操作后边还会继续讲到！

### 四、CoordinatorLayout中的依赖、监听

上一篇我们讲到自定义Behavior的第一种情况是某个View要监听另一个View的位置、尺寸等状态的变化，需要重写layoutDependsOn()、onDependentViewChanged()两个方法，接下来剖析其中的原理。View状态的变化必然导致重绘操作，想必有一个监听状态变化的接口吧。上边我们将onMeasure()方法时有一个ensurePreDrawListener()没说，答案就在里边，现在来看：
```
void ensurePreDrawListener() {
        boolean hasDependencies = false;
        final int childCount = getChildCount();
        // 判断是否存在依赖关系
        for (int i = 0; i < childCount; i++) {
            final View child = getChildAt(i);
            if (hasDependencies(child)) {
                hasDependencies = true;
                break;
            }
        }
        // mNeedsPreDrawListener默认为false
        // 如果存在依赖关系
        if (hasDependencies != mNeedsPreDrawListener) {
            if (hasDependencies) {
                addPreDrawListener();
            } else {
                removePreDrawListener();
            }
        }
    }
```
大致的作用是判断CoordinatorLayout的子View之间是否存在依赖关系，如果存在则注册监听绘制的接口：
```
void addPreDrawListener() {
        if (mIsAttachedToWindow) {
            // Add the listener
            if (mOnPreDrawListener == null) {
                mOnPreDrawListener = new OnPreDrawListener();
            }
            final ViewTreeObserver vto = getViewTreeObserver();
            // 给ViewTreeObserver注册一个监听绘制的OnPreDrawListener接口
            vto.addOnPreDrawListener(mOnPreDrawListener);
        }

        // Record that we need the listener regardless of whether or not we're attached.
        // We'll add the real listener when we become attached.
        mNeedsPreDrawListener = true;
    }
```
所以第一次注册OnPreDrawListener接口是在onMeasure()里，并不是在onAttachedToWindow()里边。看一下OnPreDrawListener具体实现：
```
class OnPreDrawListener implements ViewTreeObserver.OnPreDrawListener {
        @Override
        public boolean onPreDraw() {
            onChildViewsChanged(EVENT_PRE_DRAW);
            return true;
        }
    }
```
显然核心就在onChildViewsChanged(type)里边了，这里type是EVENT_PRE_DRAW代表将要绘制，另外还有两个type：EVENT_NESTED_SCROLL代表嵌套滚动、EVENT_VIEW_REMOVED代表View被移除。我们来分析onChildViewsChanged()方法(只保留了核心代码)：
```
final void onChildViewsChanged(@DispatchChangeEvent final int type) {
        final int layoutDirection = ViewCompat.getLayoutDirection(this);
        final int childCount = mDependencySortedChildren.size();
        final Rect inset = acquireTempRect();
        final Rect drawRect = acquireTempRect();
        final Rect lastDrawRect = acquireTempRect();

        for (int i = 0; i < childCount; i++) {
            final View child = mDependencySortedChildren.get(i);
            final LayoutParams lp = (LayoutParams) child.getLayoutParams();
            if (type == EVENT_PRE_DRAW && child.getVisibility() == View.GONE) {
                // Do not try to update GONE child views in pre draw updates.
                continue;
            }

            if (type != EVENT_VIEW_REMOVED) {
                // 检查子View状态是否改变
                getLastChildRect(child, lastDrawRect);
                if (lastDrawRect.equals(drawRect)) {
                    continue;
                }
                // 记录最后一次View的状态
                recordLastChildRect(child, drawRect);
            }

            // 根据依赖关系更新View
            for (int j = i + 1; j < childCount; j++) {
                final View checkChild = mDependencySortedChildren.get(j);
                final LayoutParams checkLp = (LayoutParams) checkChild.getLayoutParams();
                final Behavior b = checkLp.getBehavior();
                // 如果Behavior不为空，并且checkChild依赖child，即重写了layoutDependsOn
                // 则继续执行if块，否则当前循环到此结束！
                if (b != null && b.layoutDependsOn(this, checkChild, child)) {
                    // 如果type是EVENT_PRE_DRAW，并且checkChild在嵌套滑动后已经更新
                    // 则重置标志，进行下一次循环
                    if (type == EVENT_PRE_DRAW && checkLp.getChangedAfterNestedScroll()) {
                        checkLp.resetChangedAfterNestedScroll();
                        continue;
                    }
                    
                    final boolean handled;
                    switch (type) {
                        case EVENT_VIEW_REMOVED:
                            // 如果type是EVENT_VIEW_REMOVED，即被依赖的view被移除
                            // 则需要执行Behavior的onDependentViewRemoved()
                            b.onDependentViewRemoved(this, checkChild, child);
                            handled = true;
                            break;
                        default:
                            // 如果type是EVENT_PRE_DRAW或者EVENT_NESTED_SCROLL
                            // 并且我们重写了Behavior的onDependentViewChanged则执行该方法
                            handled = b.onDependentViewChanged(this, checkChild, child);
                            break;
                    }

                    if (type == EVENT_NESTED_SCROLL) {
                        // 记录在嵌套滑动后是否已经更新了View
                        checkLp.setChangedAfterNestedScroll(handled);
                    }
                }
            }
        }
    }
```
onChildViewsChanged()方法中如果View之间有依赖关系，并重写了相应Behavior的layoutDependsOn()方法。则会执行Behavior的onDependentViewChanged()或onDependentViewRemoved()方法。这也就解释了上一篇中第一种情况的原理，可见这里起关键作用的还是Behavior。

### 五、NestedScrolling机制
可能你没听过这个概念，但你可能已经使用过它了，例如CoordinatorLayout嵌套RecyclerView的布局、NestedScrollView等，都有用到了这个原理。NestedScrolling提供了一套父View和子 View嵌套滑动的交互机制，前提条件是父View需要实现NestedScrollingParent接口，子View需要实现NestedScrollingChild接口。按照NestedScrolling[Parent|Child]接口的要求(可查看接口的注释)，实现该接口的View需要创建一个NestedScrolling[Parent|Child]Helper帮助类实例来辅助子View和父View的交互。

先后构造一个符合NestedScrolling机制的场景，前边我们已经提到了CoordinatorLayout实现了NestedScrollingParent2接口，NestedScrollingParent2又继承自NestedScrollingParent，所以CoordinatorLayout做为父View的条件是满足的；其实RecyclerView实现了NestedScrollingChild2接口，NestedScrollingChild2又继承自NestedScrollingChild接口也满足作为子View的条件。

NestedScrolling[Parent|Child]2接口可以看作是对NestedScrolling[Parent|Child]接口的扩展，本质和作用类似。

可以用上一篇自定义Behavior的第二种情况的例子来分析：在CoordinatorLayout里边嵌套一个RecyclerView和TextView，TextView跟随RecyclerView的滑动来移动。根据上边的分析，在CoordinatorLayout里应该有一个NestedScrollingParentHelper的实例，在RecyclerView里应该有一个NestedScrollingChildHelper的实例。

从哪里开始分析呢？因为例子的效果是从RecyclerView的滑动开始的，所以就从RecyclerView开始吧！滑动必然先进行事件的分发，所以先看它的onInterceptTouchEvent()方法(只保留核心代码)是否有我们想要的东西：
```
@Override
    public boolean onInterceptTouchEvent(MotionEvent e) {
        final int action = e.getActionMasked();
        final int actionIndex = e.getActionIndex();

        switch (action) {
            case MotionEvent.ACTION_DOWN:
                int nestedScrollAxis = ViewCompat.SCROLL_AXIS_NONE;
                if (canScrollHorizontally) {
                    nestedScrollAxis |= ViewCompat.SCROLL_AXIS_HORIZONTAL;
                }
                if (canScrollVertically) {
                    nestedScrollAxis |= ViewCompat.SCROLL_AXIS_VERTICAL;
                }
                startNestedScroll(nestedScrollAxis, TYPE_TOUCH);
                break;
        }
    }
```
发现了一个startNestedScroll()方法，和之前自定义Behavior重写的onStartNestedScroll()方法有点像哦！目测找对地方了，继续看startNestedScroll()方法：
```
@Override
    public boolean startNestedScroll(int axes, int type) {
        return getScrollingChildHelper().startNestedScroll(axes, type);
    }
```
原来是重写了NestedScrollingParent2接口的方法，getScrollingChildHelper()做了什么呢？
```
private NestedScrollingChildHelper getScrollingChildHelper() {
        if (mScrollingChildHelper == null) {
            mScrollingChildHelper = new NestedScrollingChildHelper(this);
        }
        return mScrollingChildHelper;
    }
```
就是实例化上边提到的NestedScrollingChildHelper，所以startNestedScroll(axes, type)就是该帮助类的方法：
```
public boolean startNestedScroll(@ScrollAxis int axes, @NestedScrollType int type) {
        if (hasNestedScrollingParent(type)) {
            // Already in progress
            return true;
        }
        if (isNestedScrollingEnabled()) {
            ViewParent p = mView.getParent();
            View child = mView;
            while (p != null) {
                if (ViewParentCompat.onStartNestedScroll(p, child, mView, axes, type)) {
                    setNestedScrollingParentForType(type, p);
                    ViewParentCompat.onNestedScrollAccepted(p, child, mView, axes, type);
                    return true;
                }
                if (p instanceof View) {
                    child = (View) p;
                }
                p = p.getParent();
            }
        }
        return false;
    }
```
这里插一段，isNestedScrollingEnabled()代表是否启用了嵌套滑动，只有启用了嵌套滑动，事件才能继续分发。这个是在哪里设置的呢？RecyclerView的构造函数中：
```
public RecyclerView(Context context, @Nullable AttributeSet attrs, int defStyle) {
        // Re-set whether nested scrolling is enabled so that it is set on all API levels
        setNestedScrollingEnabled(nestedScrollingEnabled);
    }
```
最终调用了NestedScrollingChildHelper的setNestedScrollingEnabled()方法：
```
@Override
    public void setNestedScrollingEnabled(boolean enabled) {
        getScrollingChildHelper().setNestedScrollingEnabled(enabled);
    }
```
所以我们自定义的实现NestedScrollingChild接口的View，也需要设置setNestedScrollingEnabled(true)，具体方法了RecyclerView中类似。好了插播结束，继续往下看。

mView是什么呢？还记得上边创建mScrollingChildHelper的构造函数吗？就是：
```
public NestedScrollingChildHelper(@NonNull View view) {
        mView = view;
    }
```
所以mView就是RecyclerView，则p就应该是CoordinatorLayout，看下if条件ViewParentCompat.onStartNestedScroll()里边的实现：
```
public static boolean onStartNestedScroll(ViewParent parent, View child, View target,
            int nestedScrollAxes, int type) {
        if (parent instanceof NestedScrollingParent2) {
            // First try the NestedScrollingParent2 API
            return ((NestedScrollingParent2) parent).onStartNestedScroll(child, target,
                    nestedScrollAxes, type);
        } else if (type == ViewCompat.TYPE_TOUCH) {
            // Else if the type is the default (touch), try the NestedScrollingParent API
            return IMPL.onStartNestedScroll(parent, child, target, nestedScrollAxes);
        }
        return false;
    }
```
看到了熟悉的NestedScrollingParent2，因为CoordinatorLayout实现了该接口，这也更加确定了parent就是CoordinatorLayout，所以((NestedScrollingParent2) parent).onStartNestedScroll(child, target, nestedScrollAxes, type)就回到了CoordinatorLayout去执行：
```
@Override
    public boolean onStartNestedScroll(View child, View target, int axes, int type) {
        boolean handled = false;

        final int childCount = getChildCount();
        // 遍历CoordinatorLayout的子View
        for (int i = 0; i < childCount; i++) {
            final View view = getChildAt(i);
            if (view.getVisibility() == View.GONE) {
                // If it's GONE, don't dispatch
                continue;
            }
            final LayoutParams lp = (LayoutParams) view.getLayoutParams();
            final Behavior viewBehavior = lp.getBehavior();
            // 如果当前View有Behavior，则调用其onStartNestedScroll()方法。
            if (viewBehavior != null) {
                final boolean accepted = viewBehavior.onStartNestedScroll(this, view, child,
                        target, axes, type);
                handled |= accepted;
                lp.setNestedScrollAccepted(type, accepted);
            } else {
                lp.setNestedScrollAccepted(type, false);
            }
        }
        return handled;
    }
```
该方法就是遍历CoordinatorLayout的子View，如果有Behavior则调用它的onStartNestedScroll方法，如果返回true，则Behavior就拦截了这次事件，进一步可以更新对应的View状态。

到这里一个NestedScrolling机制交互的流程就走完了，先简单总结一下，事件从RecyclerView开始到NestedScrollingChildHelper经过ViewParentCompat回到了CoordinatorLayout最后被Behavior处理掉了。

之前我们还重写了Behavior的onNestedPreScroll()方法，来处理RecyclerView的滑动事件，既然是滑动，那肯定逃不出onTouchEvent()的ACTION_MOVE事件(只保留核心代码)：
```
@Override
    public boolean onTouchEvent(MotionEvent e) {
        final int action = e.getActionMasked();
        switch (action) {
            case MotionEvent.ACTION_MOVE: {
                final int index = e.findPointerIndex(mScrollPointerId);
                final int x = (int) (e.getX(index) + 0.5f);
                final int y = (int) (e.getY(index) + 0.5f);
                int dx = mLastTouchX - x;
                int dy = mLastTouchY - y;

                if (dispatchNestedPreScroll(dx, dy, mScrollConsumed, mScrollOffset, TYPE_TOUCH)) {
                    dx -= mScrollConsumed[0];
                    dy -= mScrollConsumed[1];
                    vtev.offsetLocation(mScrollOffset[0], mScrollOffset[1]);
                    // Updated the nested offsets
                    mNestedOffsets[0] += mScrollOffset[0];
                    mNestedOffsets[1] += mScrollOffset[1];
                }
            } break;
        return true;
    }
```
有一个dispatchNestedPreScroll()方法，继续跟进，中间的流程和上的类似，可以自行打断点跟一遍，我们重点看一下回到CoordinatorLayout中的onNestedPreScroll()方法：
```
@Override
    public void onNestedPreScroll(View target, int dx, int dy, int[] consumed, int  type) {
        int xConsumed = 0;
        int yConsumed = 0;
        boolean accepted = false;

        final int childCount = getChildCount();
        for (int i = 0; i < childCount; i++) {
            final View view = getChildAt(i);
            if (view.getVisibility() == GONE) {
                // If the child is GONE, skip...
                continue;
            }

            final LayoutParams lp = (LayoutParams) view.getLayoutParams();
            if (!lp.isNestedScrollAccepted(type)) {
                continue;
            }

            final Behavior viewBehavior = lp.getBehavior();
            if (viewBehavior != null) {
                mTempIntPair[0] = mTempIntPair[1] = 0;
                viewBehavior.onNestedPreScroll(this, view, target, dx, dy, mTempIntPair, type);

                xConsumed = dx > 0 ? Math.max(xConsumed, mTempIntPair[0])
                        : Math.min(xConsumed, mTempIntPair[0]);
                yConsumed = dy > 0 ? Math.max(yConsumed, mTempIntPair[1])
                        : Math.min(yConsumed, mTempIntPair[1]);

                accepted = true;
            }
        }
        consumed[0] = xConsumed;
        consumed[1] = yConsumed;

        if (accepted) {
            onChildViewsChanged(EVENT_NESTED_SCROLL);
        }
    }
```
同样是遍历子View，如果可以则执行View对应Behavior的onNestedPreScroll()方法。当然这不是重点，这里有个mTempIntPair数组，对应Behavior的onNestedPreScroll()方法的consumed参数，所以之前我们写consumed[1] = dy，实际是给mTempIntPair复制，最终让父View即CoordinatorLayout消费掉事件，也就是你滑动的是RecyclerView但实际上CoordinatorLayout在整体移动！

所以在NestedScrolling机制中，当实现了NestedScrollingChild 接口的子View滑动时，现将自己滑动的dx、dy传递给实现了NestedScrollingParent接口的父View，让View先决定是否要消耗相应的事件，父View可以消费全部事件，如果父View消耗了部分，则剩下的再由子View处理。

### 六、CoordinatorLayout的TouchEvent
CoordinatorLayout作为一个自定义ViewGroup，必然会重写onInterceptTouchEvent()和onTouchEvent来进行事件的拦截和处理。结合之前多次的分析结果，可以猜测这两个方法最终也由相应子View的Behavior来处理！

看一下这两个方法：
```
@Override
    public boolean onInterceptTouchEvent(MotionEvent ev) {
        MotionEvent cancelEvent = null;
        final int action = ev.getActionMasked();
        // 重置Behavior的相关记录，为下次事件做准备
        if (action == MotionEvent.ACTION_DOWN) {
            resetTouchBehaviors();
        }
        // 是否拦截当前事件是由performIntercept()的返回值决定的
        final boolean intercepted = performIntercept(ev, TYPE_ON_INTERCEPT);
        if (cancelEvent != null) {
            cancelEvent.recycle();
        }
        // 同样是重置Behavior
        if (action == MotionEvent.ACTION_UP || action == MotionEvent.ACTION_CANCEL) {
            resetTouchBehaviors();
        }
        return intercepted;
    }
@Override
    public boolean onTouchEvent(MotionEvent ev) {
        boolean handled = false;
        boolean cancelSuper = false;
        MotionEvent cancelEvent = null;

        final int action = ev.getActionMasked();
        // mBehaviorTouchView不为空，表示某个View的Behavior正在处理当前事件
        if (mBehaviorTouchView != null || (cancelSuper = performIntercept(ev, TYPE_ON_TOUCH))) {
            // 继续由相应的Behavior处理当前事件
            final LayoutParams lp = (LayoutParams) mBehaviorTouchView.getLayoutParams();
            final Behavior b = lp.getBehavior();
            if (b != null) {
                handled = b.onTouchEvent(this, mBehaviorTouchView, ev);
            }
        }
        return handled;
    }
```
它们都调用了一个共同的方法performIntercept()：
```
private boolean performIntercept(MotionEvent ev, final int type) {
        boolean intercepted = false;
        boolean newBlock = false;
        MotionEvent cancelEvent = null;
        final int action = ev.getActionMasked();

        final List<View> topmostChildList = mTempList1;
        // 按照 z-order 排序，让最顶部的View先被处理
        getTopSortedChildren(topmostChildList);
        final int childCount = topmostChildList.size();
        for (int i = 0; i < childCount; i++) {
            final View child = topmostChildList.get(i);
            final LayoutParams lp = (LayoutParams) child.getLayoutParams();
            final Behavior b = lp.getBehavior();
            // 如果已经有Behavior拦截了事件或者是新的拦截，并且不是ACTION_DOWN
            if ((intercepted || newBlock) && action != MotionEvent.ACTION_DOWN) {
                if (b != null) {
                    if (cancelEvent == null) {
                        final long now = SystemClock.uptimeMillis();
                        cancelEvent = MotionEvent.obtain(now, now,
                                MotionEvent.ACTION_CANCEL, 0.0f, 0.0f, 0);
                    }
                     // 发送cancelEvent给拦截了事件之后的其它子View的Behavior
                    switch (type) {
                        case TYPE_ON_INTERCEPT:
                            b.onInterceptTouchEvent(this, child, cancelEvent);
                            break;
                        case TYPE_ON_TOUCH:
                            b.onTouchEvent(this, child, cancelEvent);
                            break;
                    }
                }
                continue;
            }
            // 如果当前事件还没被拦截，则先由当前遍历到的子View的Behavior处理
            if (!intercepted && b != null) {
                switch (type) {
                    case TYPE_ON_INTERCEPT:
                        intercepted = b.onInterceptTouchEvent(this, child, ev);
                        break;
                    case TYPE_ON_TOUCH:
                        intercepted = b.onTouchEvent(this, child, ev);
                        break;
                }
                if (intercepted) {
                    // 记录要处理当前事件的View
                    mBehaviorTouchView = child;
                }
            }

            // Don't keep going if we're not allowing interaction below this.
            // Setting newBlock will make sure we cancel the rest of the behaviors.
            final boolean wasBlocking = lp.didBlockInteraction();
            final boolean isBlocking = lp.isBlockingInteractionBelow(this, child);
            newBlock = isBlocking && !wasBlocking;
            if (isBlocking && !newBlock) {
                // Stop here since we don't have anything more to cancel - we already did
                // when the behavior first started blocking things below this point.
                break;
            }
        }

        topmostChildList.clear();

        return intercepted;
    }
```
相关的说明都在注释里了，所以CoordinatorLayout的事件处理，还是优先的交给子View的Behavior来完成。

### 七、小结

到此，CoordinatorLayout的几个重要的点就分析完了，其实核心还是CoordinatorLayout和Behavior之间的关系，理清了这个也就明白为什么Behavior可以实现拦截一切的效果！对自定义Behavior也是有很大帮助的。可以发现CoordinatorLayout最终都是将各种处理优先交给了Behavior来完成，所以CoordinatorLayout更像是Behavior的代理！

***
https://www.jianshu.com/p/7830b05b38bb
***