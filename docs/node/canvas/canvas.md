### GLES20RecordingCanvas 类
这个类是什么？为什么我从没用过？我们来看看它的代码：

```
class GLES20RecordingCanvas extends GLES20Canvas {
    ...
}

class GLES20Canvas extends HardwareCanvas {
    ...
}

public abstract class HardwareCanvas extends Canvas {
    ...
}
```

它是不暴露给开发者的，所以我们也使用不了它。
而由 extends Canvas 可见，它是 Canvas 的一个实现类，所以应当也提供和 Canvas 一样的功能。那么它在哪里被使用了呢？
它是在 Android framework 源码处的，我们可以自定义一个 view，在 debug 时，断点到
```
    @Override
    protected void onDraw(Canvas canvas) {
        super.onDraw(canvas);
    }
```
就可以看到这个 canvas 的实例是 GLES21RecordingCanvas 了。

也就是说，几乎 Android 的所有 View 控件以及我们的自定义 View ，onDraw(Canvas) 里的 Canvas 就是这个 Canvas。

好，现在我们来看它的其它特点：

```
    @Override
    public boolean isHardwareAccelerated() {
        return true;
    }
```

也就是说，这个 Canvas 是有硬件加速的。硬件加速的意思是会使用 GPU 进行加速，在 Android 上一般就是指会使用 OpenGL 进行绘制。

补充: 5.0之后用的是 DisplayListCanvas ，和 GLES20RecordingCanvas 类似，也是用硬件加速的。
可以在 View 里看到它的使用
```
    public RenderNode updateDisplayListIfDirty() {
            ...
            final DisplayListCanvas canvas = renderNode.start(width, height);
            ...
        return renderNode;
    }
```

### Canvas 类

好，我们现在回到 Canvas 类，分析一下它和 GLES20RecordingCanvas 有什么不同。

首先是硬件加速方面

```
    public boolean isHardwareAccelerated() {
        return false;
    }
```

很明显，这个 Canvas 是没有硬件加速的，也就是说，使用这个类做的绘制，全是使用 CPU 进行绘制的。

对于这个Canvas，我们平时应该会经常用到，例如在修改图片的时候

```
Canvas canvas = new Canvas(bitmap);
canvas.drawCircle(...);
```

所以使用这种方式修改图片的时候，我们要注意性能问题。

顺便说明一句，在这些 Canvas 类里可以看到很多 native 的方法。Canvas的底层是用 Skia 的库实现的，深入里面可以看到 Skia 有一部分是和 OpenGL 相关的。

### 使用建议

前文提到，几乎 Android 的所有 View 控件用的 Canvas 都是 GLES20RecordingCanvas，那么，例外的有哪些呢？

1.  没有开硬件加速的 View。所以关闭硬件加速的话要注意是不是真的需要关闭哦。
2.  SurfaceView 和 TextureView 里面 lockCanvas() 方法得到的 Canvas。

我们可以看看 SurfaceView 和 TextureView 里面的 Canvas 是怎么创建出来的。

####    SurfaceView

```
    ...
    private final Canvas mCanvas = new CompatibleCanvas();
    ...

    private final class CompatibleCanvas extends Canvas {
        // A temp matrix to remember what an application obtained via {@link getMatrix}
        private Matrix mOrigMatrix = null;

        @Override
        public void setMatrix(Matrix matrix) {
            if (mCompatibleMatrix == null || mOrigMatrix == null || mOrigMatrix.equals(matrix)) {
                // don't scale the matrix if it's not compatibility mode, or
                // the matrix was obtained from getMatrix.
                super.setMatrix(matrix);
            } else {
                Matrix m = new Matrix(mCompatibleMatrix);
                m.preConcat(matrix);
                super.setMatrix(m);
            }
        }

        @SuppressWarnings("deprecation")
        @Override
        public void getMatrix(Matrix m) {
            super.getMatrix(m);
            if (mOrigMatrix == null) {
                mOrigMatrix = new Matrix();
            }
            mOrigMatrix.set(m);
        }
    }
```

以上代码在 android.view.Surface 里，顺着 SurfaceView.getHolder().lockCanvas() 方法找就可以找到这段代码。可以看到，这个 canvas 是没有硬件加速的。
有趣的是，在 API 23 之后，android.view.Surface 里增加了新的方法 lockHardwareCanvas()，这个明显就是有硬件加速的，可惜在 API 23 以后才有。

####    TextureView

```
    public Canvas lockCanvas(Rect dirty) {
        if (!isAvailable()) return null;

        if (mCanvas == null) {
            mCanvas = new Canvas();
        }

        synchronized (mNativeWindowLock) {
            if (!nLockCanvas(mNativeWindow, mCanvas, dirty)) {
                return null;
            }
        }
        mSaveCount = mCanvas.save();

        return mCanvas;
    }
```

很明显，TextureView 里的 canvas 也是没有硬件加速的。

下面以B站很著名的开源库 DanmakuFlameMaster为例：
它的 feature 里有一条：
使用多种方式(View/SurfaceView/TextureView)实现高效绘制
能实现使用GPU绘制的只有View，因为它使用 SurfaceView/TextureView 实现时并没有使用到 OpenGL 的 API，canvas.drawText(...) 的 canvas 是使用 CPU 的，所以如果在CPU资源缺少的情况下效率并不高。

总结可得，使用 canvas 要注意是否有硬件加速，一般的 View 是有的，而 SurfaceView 和 TextureView 是没有的。

### 对比与验证
既然 GLES20RecordingCanvas 底层是使用 OpenGL 实现的，Android的 java 层上也提供了 OpenGL 相关的 API，那么我们能不能使用这些 API 实现类似的功能呢？

当然是可以的，我在Android-使用OpengGL实现的Canvas进行绘制(简单介绍)就提到了用 OpenGL 实现的 canvas 类。

那么我们就试试用它和 Android 自己的 canvas 做性能对比吧。

代码地址: [GitHub](https://github.com/ChillingVan/android-openGL-canvas)

[](0.webp)

图片中，左边的是 Android 有硬件加速的 canvas，中间是 SurfaceView 的 canvas，右边是自己实现的 OpenGL canvas。

上方的数字是产生的图片的数量，大致是数量越多说明绘制越快，否则就是因为有卡顿而造成掉帧没绘制。

不断点击按钮就可以不断绘制图片。

可以看到左边和右边是接近的，说明底层的确是用 OpenGL 绘制的，并且性能较好。中间数字较少，所以有掉帧的情况出现。

另外可以自己试试这个例子，当疯狂点击按钮的时候，可以看到中间有明显的卡顿。

### 总结
1.  要关闭硬件加速的话要注意是不是真的需要关闭。
2.  canvas 是分为使用 OpenGL 和非 OpenGL 实现的，没用 OpenGL 实现的会更依赖 cpu 的资源。

```
https://www.jianshu.com/p/5a0c61c286e6
```