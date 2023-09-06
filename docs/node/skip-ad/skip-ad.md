通过无障碍功能跳过app首页的开屏广告。

### 问题

##### 1. app被关闭后会自动收回 无障碍权限

手机管家->应用启动管理中，将app改为自动管理可解决

![](0.png ':size=30%')


##### 2. 耗电问题
监听了：AccessibilityEvent.TYPE_WINDOW_STATE_CHANGED、AccessibilityEvent.TYPE_WINDOW_CONTENT_CHANGED事件，此事件当app切到后台再恢复后都会调用，导致进行重复扫描。


### 防止通过无障碍功能跳过广告
第一步：我们让“跳过广告”的View不支持辅助功能
```
adSkipView.setAccessibilityDelegate(object : AccessibilityDelegate() {
            override fun performAccessibilityAction(host: View?, action: Int, args: Bundle?): Boolean {
                //忽略AccessibilityService传过来的点击事件以达到防止模拟点击的目的
                return if (action == AccessibilityNodeInfo.ACTION_CLICK || action == AccessibilityNodeInfo.ACTION_LONG_CLICK) {
                    true
                } else super.performAccessibilityAction(host, action, args)
            }
        })
```
第二步: 增加版，因为无障碍服务可以设置坐标模拟点击，我们使用setOnTouchListener拦截

```
val accessibilityManager: AccessibilityManager = getSystemService(Context.ACCESSIBILITY_SERVICE) as AccessibilityManager
adSkipView.setOnTouchListener({ v, event ->
      if (event.getAction() == MotionEvent.ACTION_DOWN) {
          if (null != accessibilityManager && accessibilityManager.isEnabled()) {
              //来自无障碍服务的触摸，非人类的点击，禁用这个view
              adSkipView.setClickable(false)
            }
         }
         false
      })
```



### source code

<a href="node/skip-ad/DsmSkipAd.zip" download>壳工程</a>