理解ActivityRecord https://juejin.cn/post/6856298463119409165

PhoneWindow

mDecor -> DecorView

mContext -> Activity

getApplicationContext ContextImpl = getBaseContext

```java
// PhoneWindow
protected DecorView generateDecor(int featureId) {
    Context context;
    // mUseDecorContext = true 在构造函数中设置为true
    if (mUseDecorContext) {
        // 获取的到context为在ActivityThread.performLaunchActivity生成的ContextImpl
        Context applicationContext = getContext().getApplicationContext();
        if (applicationContext == null) {
            context = getContext();
        } else {
            // DecorContext extends ContextThemeWrapper
            context = new DecorContext(applicationContext, this);
            if (mTheme != -1) {
                context.setTheme(mTheme);
            }
        }
    } else {
        context = getContext();
    }
    return new DecorView(context, featureId, this, getAttributes());
}
```
ViewRootImpl 如何与 DecorView建立起关联
handleResumeActivity

window.addView => root = new ViewRootImpl() => root.setView()

而setView中会建立事件分发的通道, InputChannel

事件在应用层首先通过ViewPostImeInputState分发给DecorView
```
mView.dispatchPointerEvent
```
decorView将事件分发给Activity

```java
// DecorView
// cb即为activity
final Window.Callback cb = mWindow.getCallback();
return cb != null && !mWindow.isDestroyed() && mFeatureId < 0
        ? cb.dispatchTouchEvent(ev) : super.dispatchTouchEvent(ev);
```

activity将事件分发给PhoneWindow

```java
if (ev.getAction() == MotionEvent.ACTION_DOWN) {
    onUserInteraction();
}
if (getWindow().superDispatchTouchEvent(ev)) {
    return true;
}
return onTouchEvent(ev);
```

PhoneWindow又将事件分发回DecorView

```java
return mDecor.superDispatchTouchEvent(event);
```

DecorView将事件分发给子view

```java
// 即调用ViewGroup的dispatchTouchEvent
return super.dispatchTouchEvent(ev)
```
