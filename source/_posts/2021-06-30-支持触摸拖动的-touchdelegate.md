---
title: 支持触摸拖动的 TouchDelegate
toc: true
date: 2021-06-30 00:45:50
categories:
- Android 应用开发
tags:
- Android
- Touch
---
# 需求

最近有一个小需求，就是在界面上有一个预览图片的区域，这个区域用户可以双击缩放图片、双指自由缩放图片、触摸图片进行移动，对图片的局部区域进行查看，像这样：

![](1.gif)

这个功能可以使用 github 中的开源库 PhotoView 实现，地址：[https://github.com/Baseflow/PhotoView](https://github.com/Baseflow/PhotoView)

同时另一个条件是当前这个图像预览区域较小，不像上图这样预览区域比较大，大概是 4 分之 1 左右，如果直接使用 PhotoView 设置图片，那么用户体验可能较差，因为图像展示区域较小，难以进行自由的图像移动预览操作，所以这个需求就是：扩大触摸区域，让用户在图像显示区域的外侧也可以自由的对图像进行移动预览操作。如下，白色区域是 PhotoView 的父控件的区域，用户触摸和双击这里时，可对图像进行预览操作：
<!-- more -->
![](0.gif)

先把 PhotoView 放置在布局中，然后设置一张图片，PhotoView 本身会支持图片的双击和双指缩放与移动预览，如果要实现上述的需求，当时想到了两种解决方案：

1. 将 PhotoView（图像预览控件）的父控件的触摸事件传递给 PhotoView，那么用户触摸在父控件上时，PhotoView 将会接收到触摸事件，从而控制内部图像的缩放和移动；
2. 先将 PhotoView 本身尺寸变大，填满白色区域，再想办法将图片显示区域限制到需要的大小，需要手动使用代码调整图片显示到限制区域

显然第二种方法是需要对 PhotoView 控件进行额外处理和调整，与 PhotoView 紧密相连，第一种方法只需要将触摸事件传递给 PhotoView 即可，那么第一种方法更灵活和通用，所以优先选择第一种方案进行处理

采取第一种方案，通常可以自定义一个 ViewGroup，将它作为 PhotoView 的父控件，并重写它的 `dispatchTouchEvent` 方法，将事件全部传递给 PhotoView 处理。但这种方法不够灵活，因为需要创建一个特定的 ViewGroup 类型。那么还有没有其他的方法？发现 Android 提供了一个 `TouchDelegate` 的类，使用它可以扩大一个 View 的触摸区域，听起来挺符合当前需求的场景，那么计划使用 TouchDelegate 来实现这个需求。

首先看一下 TouchDelegae 的基本用法

# TouchDelegate 用法

```java
public static void expandViewTouchBounds(final View view, final View parent) {
    Rect bounds = new Rect();
    bounds.left = 0;
    bounds.top = 0;
    bounds.right = parent.getWidth();
    bounds.bottom = parent.getHeight();

    TouchDelegate touchDelegate = new TouchDelegate(bounds, view);
    parent.setTouchDelegate(touchDelegate);
}
```

```java
// 扩展 photoView 的触摸区域到 parent 上
expandViewTouchBounds(photoView, parent);
```

使用方法很简单，首先创建一个 `TouchDelegate` 对象，然后设置给父控件，其中 `bounds` 为在父控件中扩展的触摸区域，坐标相对于父控件，示例中设置为整个父控件的区域。

# Bug

本以为这样就可以了，通过测试发现在 PhotoView 外部的区域用单个手指触摸移动，图像预览区域根本无法跟着移动，但是却可以响应双击和双指的缩放事件。难道是用法的不对。

通过查看 TouchDelegate 的源代码发现，其实 TouchDelegate 是无法支持移动的触摸事件的响应的，只能支持 click 事件。关键代码如下：

```java
/**
 * Forward touch events to the delegate view if the event is within the bounds
 * specified in the constructor.
 *
 * @param event The touch event to forward
 * @return True if the event was consumed by the delegate, false otherwise.
 */
public boolean onTouchEvent(@NonNull MotionEvent event) {
    int x = (int)event.getX();
    int y = (int)event.getY();
    boolean sendToDelegate = false;
    boolean hit = true;
    boolean handled = false;

    switch (event.getActionMasked()) {
        case MotionEvent.ACTION_DOWN:
            mDelegateTargeted = mBounds.contains(x, y);
            sendToDelegate = mDelegateTargeted;
            break;
        case MotionEvent.ACTION_POINTER_DOWN:
        case MotionEvent.ACTION_POINTER_UP:
        case MotionEvent.ACTION_UP:
        case MotionEvent.ACTION_MOVE:
            sendToDelegate = mDelegateTargeted;
            if (sendToDelegate) {
                Rect slopBounds = mSlopBounds;
                if (!slopBounds.contains(x, y)) {
                    hit = false;
                }
            }
            break;
        case MotionEvent.ACTION_CANCEL:
            sendToDelegate = mDelegateTargeted;
            mDelegateTargeted = false;
            break;
    }
    if (sendToDelegate) {
        if (hit) {
            // Offset event coordinates to be inside the target view
            event.setLocation(mDelegateView.getWidth() / 2, mDelegateView.getHeight() / 2);
        } else {
            // Offset event coordinates to be outside the target view (in case it does
            // something like tracking pressed state)
            int slop = mSlop;
            event.setLocation(-(slop * 2), -(slop * 2));
        }
        handled = mDelegateView.dispatchTouchEvent(event);
    }
    return handled;
}
```

核心逻辑并不复杂，其中 `mDelegateView` 为被代理处理事件的 View，也就是 PhotoView。 

主要关注下面 `if (sendToDelegate) { . . . }` 代码块中的逻辑，当触摸事件发生点的坐标落在 `bounds` 内，那么 `hit` 为 `true`，执行：

```java
event.setLocation(mDelegateView.getWidth() / 2, mDelegateView.getHeight() / 2);
```

`setLocation` 的含义是重新设置 `MotionEvent` 的 x，y 坐标，也就是 `getX()` 和 `getY()` 的返回值。

那么从这里看出来，无论触摸事件的 `action` 是什么，都会将触摸坐标设置为被代理 View 的中心点，自然用手指触摸 View 外部区域移动时图像并不会跟着移动了。不过双击事件还是没有问题的，可正常检测到。

按照需求，肯定需要支持触摸拖动预览图片，结合目前情况，只能自己修复 TouchDelegate 中的事件传递逻辑了。父控件比 PhotoView 的区域大，当用户触摸到父控件区域内时，应该怎样处理？通常一个简单的规则就是按比例映射，按照触摸点 x、y 坐标在父控件内所占父控件尺寸的比例，设置在相对于子 View 区域相同的宽高比例的坐标上。

# 改进 TouchDelegate

根据分析，那么需要改进 `TouchDelegate` 原始处理逻辑中将坐标固定的问题，代码如下：

```java
public boolean onTouchEvent(@NonNull MotionEvent event) {
  int x = (int)event.getX();
  int y = (int)event.getY();
  boolean sendToDelegate = false;
  boolean hit = true;
  boolean handled = false;

  switch (event.getActionMasked()) {
    case MotionEvent.ACTION_DOWN:
      mDelegateTargeted = mBounds.contains(x, y);
      sendToDelegate = mDelegateTargeted;
      break;
    case MotionEvent.ACTION_POINTER_DOWN:
    case MotionEvent.ACTION_POINTER_UP:
    case MotionEvent.ACTION_UP:
    case MotionEvent.ACTION_MOVE:
      sendToDelegate = mDelegateTargeted;
      if (sendToDelegate) {
        Rect slopBounds = mSlopBounds;
        if (!slopBounds.contains(x, y)) {
            hit = false;
        }
      }
      break;
    case MotionEvent.ACTION_CANCEL:
      sendToDelegate = mDelegateTargeted;
      mDelegateTargeted = false;
      break;
  }
  if (sendToDelegate) {
    if (hit) {
			// 按照父控件比例修正触摸点坐标
      float newX = x * 1F / mBounds.width() * mDelegateView.getWidth();
      float newY = y * 1F / mBounds.height() * mDelegateView.getHeight();
      event.setLocation(newX, newY);
    } else {
      int slop = mSlop;
      event.setLocation(-(slop * 2), -(slop * 2));
    }
    handled = mDelegateView.dispatchTouchEvent(event);
  }
  return handled;
}
```

# 新的 Bug

本以为没问题了，然而再次测试，发现单个手指触摸在 PhotoView 区域外，也就是父控件区域内时，图像也可以正常移动预览了。但是经过多次测试，发现存在有一个瑕疵，可以说就是一个 Bug。

问题描述：

首先在父控件区域按下一个手指，然后再按下第二个手指，此时在第二个手指不放开的情况下，抬起第一个手指时，图像会出现跳跃现象，而在 PhotoView 区域内进行这样的操作就不会出现跳跃的问题，那么说明上面的触摸区域扩大的逻辑处理还是存在问题的。

如下图，在第二个手指不放开的情况下，第一个手指抬起、放下、抬起、放下会触发图像的跳跃：

![](2.gif)

通过现象进行猜测，显然是两次触摸坐标之间的差值过大，导致 PhotoView 认为用户瞬间拖动了很长一段距离，图像跟着瞬间移动，就会出现跳跃现象

通过调试以及日志打印经过父控件的触摸事件的坐标，包括上面修复坐标的代码 `event.setLocation(newX, newY);` 发现坐标转换并没有问题，那么问题处在哪里？由于当时对于多点触控事件处理并不熟悉，而上面出现的 Bug 显然和多指触摸相关，结合上面观察到的现象：使用同样的流程在 PhotoView 内部就不会出现这种问题，而在父控件的区域操作就会出现，那么开始对 PhotoView 内部的接收到触摸事件的逻辑进行调试，通过对比在父控件和在 PhotoView 中的事件处理，最终发现了问题。

首先说一下像这种手指触摸控制图片移动时的场景在多指触摸时的典型处理方法。

# 图片移动预览典型处理方法

对于多点触摸的情况，`onTouchEvent` 方法接收到的触摸事件按照操作顺序是这样：

```java
1. 第一个手指按下，触发 `ACTION_DOWN` 事件

2. 第一个手指移动，触发 `ACTION_MOVE` 事件

3. 第二个手指按下，触发 `ACTION_POINTER_DOWN` 事件，此时处于两个手指都按下的状态，可用 `event.getActionIndex()` 方法获得抬起手指触摸点的索引
（继续按下手指，都将触发 `ACTION_POINTER_DOWN` 事件）

4. 任意一个手指移动，触发 `ACTION_MOVE` 事件

5. 任意一个手指抬起，触发 `ACTION_POINTER_UP` 事件，此时可用 `event.getActionIndex()` 方法获得抬起手指触摸点的索引
（只要屏幕上抬起的不是最后一个手指，那么就会触发 `ACTION_POINTER_UP` 事件）

6. 此时屏幕上只剩下一个手指，抬起触发将 `ACTION_UP` 事件
```

在处理手指触摸图片内容控制移动时，首先认为活动的手指（也就是负责移动图像内容的手指）只有一个，当第一个手指按下时那么第一个手指就是活动手指，可以使用一个变量进行记录它的 id：

```java
public boolean onTouchEvent(MotionEvent event) {
  switch (event.getActionMasked()) {
  ...
  case MotionEvent.ACTION_DOWN: {
    final int pointerIndex = event.getActionIndex();
    final float x = event.getX(pointerIndex);
    final float y = event.getY(pointerIndex);

    // 记录最后触摸坐标，用来计算图片移动偏移
    mLastTouchX = x;
    mLastTouchY = y;

		// 记录活动手指 id
    mActivePointerId = event.getPointerId(pointerIndex);
    break;
  }
  ...
  }
  return super.onTouchEvent(event);
}
```

当手指移动时，根据活动手指的坐标和最后触摸点的差计算出移动的偏移，然后对图片进行移动：

```java
public boolean onTouchEvent(MotionEvent event) {
  ...
  switch (event.getActionMasked()) {
  case MotionEvent.ACTION_MOVE: {
		// 根据活动手指 id 获取触摸点索引
    final int pointerIndex = event.findPointerIndex(mActivePointerId);
		
    final float x = event.getX(pointerIndex);
    final float y = event.getY(pointerIndex);

    // 计算图片拖动偏移
    final float dx = x - mLastTouchX;
    final float dy = y - mLastTouchY;

    // todo: 使用 dx 和 dy 对图片进行移动

    mLastTouchX = x;
    mLastTouchY = y;
    break;
  }
	...
  }
  return super.onTouchEvent(event);
}
```

此时虽然活动手指只有一个，但是触摸的手指可能是多个，不过我们只关心活动手指，需要关心一种特殊情况，就是当活动手指离开屏幕的时候，此时屏幕上还有多个手指，这种情况怎么处理？一种简单的情况就是把下一根手指作为活动手指，当屏幕上抬起的不是最后一个手指，那么就会触发 `ACTION_POINTER_UP` 事件，可以在这里进行判断处理，如果活动手指抬起，那么将活动手指的重任交给还在屏幕上的另一个手指：

```java
public boolean onTouchEvent(MotionEvent event) {
  ...
  switch (event.getActionMasked()) {
  case MotionEvent.ACTION_POINTER_UP: {
    final int pointerIndex = event.getActionIndex();
    final int pointerId = event.getPointerId(pointerIndex);

    // 如果抬起的手指是活动手指
    if (pointerId == mActivePointerId) {
        // 交接给下一个手指
        final int newPointerIndex = pointerIndex == 0 ? 1 : 0;
        // 更新活动手指 id 和坐标
        mLastTouchX = event.getX(newPointerIndex);
        mLastTouchY = event.getY(newPointerIndex);
        mActivePointerId = event.getPointerId(newPointerIndex);
    }
    break;
  }
  ...
  }
  return super.onTouchEvent(event);
}
```

# 分析 Bug 原因

了解了图片处理规则之后，可以对上面的 Bug 进行分析了。

其实 PhotoView 也是这样处理图片移动的，如果按照正常逻辑，当第一个手指按下时，它是活动手指，正常记录坐标，第二个手指按下时，忽略它的坐标，当第一个手指抬起时，这时触发 `ACTION_POINTER_UP` 事件，需要转换活动手指，将最后一次触摸的坐标设置为第二个还未抬起的手指最后坐标，活动手指也设置为第二个手指的 id，此时当第二个手指开始移动时，那么当前手指坐标与最后坐标做差获得图片移动的值，对图片进行移动。正常流程不会出现跳跃的问题，始终跟随手指移动。

而对于触摸在父控件出现 Bug 的情况，通过调试和日志发现在最后处理活动手指的转换时出现了坐标的转换失误，就在处理 `ACTION_POINTER_UP` 事件的逻辑中。当多个手指同时触摸在屏幕上时，`MotionEvent` 这个对象中包含所有手指的坐标信息，两个手指时，它包含 `x0`、`y0`、`x1`、`y1`，然而 `event.setLocation` 这个方法只能改变 `x0`、`x1` 的坐标，`x1` 和 `y1` 即第二个即将变为活动状态的手指坐标则没有经过处理，它还是原始父控件的坐标，那么 `mLastTouchX` 和 `mLastToachX` 被赋予了错误的坐标值。此时活动手指的工作交接已完成，屏幕上只剩下一个手指，就是之前的第二个手指，当它开始移动时，它的坐标将被存储在 `x0` 和 `y0` 中，此时能够获取正确的值，再与还未转换的 `mLastTouchX` 和 `mLastToachX` 做差值，计算出一个较大的错误的偏移，导致图片发生跳跃，这就是这个 Bug 的原因。

总结就是 `event.setLocation` 方法只能改变屏幕上第一个手指的坐标，而其它手指坐标无法改变。

# 完善 TouchDelegate

分析完了问题，那么就可以开始修复了，既然 `setLocation` 无法改变所有手指坐标，那么需要找到其他方法，在 `MotionEvent` 的 API 中浏览了一遍，貌似没有直接修改其他手指的 API，但是 `MotionEvent` 提供了 `obtain` 方法，它可以获得一个新的 `MotionEvent` 对象，从参数上看，可以生成多个手指的坐标信息，经过测试，完美解决了问题，关键代码如下：

```java
event = MotionEvent.obtain(
    System.currentTimeMillis(),
    System.currentTimeMillis(),
    event.getAction(),
    event.getPointerCount(),
    getPointerProperties(event),
    fixPointerCoords(event),
    event.getMetaState(),
    event.getButtonState(),
    event.getXPrecision(),
    event.getYPrecision(),
    event.getDeviceId(),
    event.getEdgeFlags(),
    event.getSource(),
    event.getFlags()
);

...

private MotionEvent.PointerProperties[] getPointerProperties(MotionEvent event) {
  int pointerCount = event.getPointerCount();
  MotionEvent.PointerProperties[] properties = new MotionEvent.PointerProperties[pointerCount];
  for (int i = 0; i < pointerCount; i++) {
    MotionEvent.PointerProperties pointerProperties = new MotionEvent.PointerProperties();
    event.getPointerProperties(i, pointerProperties);
    properties[i] = pointerProperties;
  }

  return properties;
}

private MotionEvent.PointerCoords[] fixPointerCoords(MotionEvent event) {
  int pointerCount = event.getPointerCount();
  MotionEvent.PointerCoords[] pointerCoords = new MotionEvent.PointerCoords[pointerCount];
  for (int i = 0; i < pointerCount; i++) {
    MotionEvent.PointerCoords coords = new MotionEvent.PointerCoords();
    event.getPointerCoords(i, coords);
    coords.x = coords.x * 1F / mBounds.width() * mDelegateView.getWidth();
    coords.y = coords.y * 1F / mBounds.width() * mDelegateView.getWidth();
    pointerCoords[i] = coords;
  }

  return pointerCoords;
}
```

# 完整源码

下面是修复过的 TouchDelegate 完整源码，可直接拿来用

```java
public class FixTouchDelegate extends TouchDelegate {
  private final View mDelegateView;
  private final Rect mBounds;
  private final Rect mSlopBounds;
  private boolean mDelegateTargeted;
  private final int mSlop;

  public FixTouchDelegate(Rect bounds, View delegateView) {
    super(bounds, delegateView);
    mBounds = bounds;

    mSlop = ViewConfiguration.get(delegateView.getContext()).getScaledTouchSlop();
    mSlopBounds = new Rect(bounds);
    mSlopBounds.inset(-mSlop, -mSlop);
    mDelegateView = delegateView;
  }

  @Override
  public boolean onTouchEvent(MotionEvent event) {
    int x = (int) event.getX();
    int y = (int) event.getY();
    boolean sendToDelegate = false;
    boolean hit = true;
    boolean handled = false;

    switch (event.getActionMasked()) {
      case MotionEvent.ACTION_DOWN:
        mDelegateTargeted = mBounds.contains(x, y);
        sendToDelegate = mDelegateTargeted;
        break;
      case MotionEvent.ACTION_POINTER_DOWN:
      case MotionEvent.ACTION_POINTER_UP:
      case MotionEvent.ACTION_UP:
      case MotionEvent.ACTION_MOVE:
        sendToDelegate = mDelegateTargeted;
        if (sendToDelegate) {
            if (!mSlopBounds.contains(x, y)) {
                hit = false;
            }
        }
        break;
      case MotionEvent.ACTION_CANCEL:
        sendToDelegate = mDelegateTargeted;
        mDelegateTargeted = false;
        break;
      }
      if (sendToDelegate) {
        MotionEvent obtain = null;
        if (hit) {
          // Offset event coordinates to be inside the target view
          // event.setLocation(mDelegateView.getWidth() / 2, mDelegateView.getHeight() / 2);
          obtain = MotionEvent.obtain(
              System.currentTimeMillis(),
              System.currentTimeMillis(),
              event.getAction(),
              event.getPointerCount(),
              getPointerProperties(event),
              fixPointerCoords(event),
              event.getMetaState(),
              event.getButtonState(),
              event.getXPrecision(),
              event.getYPrecision(),
              event.getDeviceId(),
              event.getEdgeFlags(),
              event.getSource(),
              event.getFlags()
          );
        } else {
          // Offset event coordinates to be outside the target view (in case it does
          // something like tracking pressed state)
          int slop = mSlop;
          event.setLocation(-(slop * 2), -(slop * 2));
        }

        if (obtain != null) {
          handled = mDelegateView.dispatchTouchEvent(obtain);
          obtain.recycle();
        } else {
          handled = mDelegateView.dispatchTouchEvent(event);
        }
      }
      return handled;
  }

  private MotionEvent.PointerProperties[] getPointerProperties(MotionEvent event) {
    int pointerCount = event.getPointerCount();
    MotionEvent.PointerProperties[] properties = new MotionEvent.PointerProperties[pointerCount];
    for (int i = 0; i < pointerCount; i++) {
      MotionEvent.PointerProperties pointerProperties = new MotionEvent.PointerProperties();
      event.getPointerProperties(i, pointerProperties);
      properties[i] = pointerProperties;
    }

    return properties;
  }

  private MotionEvent.PointerCoords[] fixPointerCoords(MotionEvent event) {
    int pointerCount = event.getPointerCount();
    MotionEvent.PointerCoords[] pointerCoords = new MotionEvent.PointerCoords[pointerCount];
    for (int i = 0; i < pointerCount; i++) {
      MotionEvent.PointerCoords coords = new MotionEvent.PointerCoords();
      event.getPointerCoords(i, coords);
      coords.x = coords.x * 1F / mBounds.width() * mDelegateView.getWidth();
      coords.y = coords.y * 1F / mBounds.width() * mDelegateView.getWidth();
      pointerCoords[i] = coords;
    }

    return pointerCoords;
  }
}
```

- 调用方法

```java
public static void expandViewTouchDelegate(final View view, final View parent) {
  view.post(() -> {
    Rect bounds = new Rect();
    view.setEnabled(true);
    ((ViewGroup) view.getParent()).getHitRect(bounds);

    bounds.left = 0;
    bounds.top = 0;
    bounds.right = parent.getWidth();
    bounds.bottom = parent.getHeight();

    TouchDelegate touchDelegate = new FixTouchDelegate(bounds, view);

    parent.setTouchDelegate(touchDelegate);
  });
}
```

# Example 地址

[https://github.com/l0neman/FixTouchDelegate](https://github.com/l0neman/FixTouchDelegate)

# 参考

- [https://developer.android.google.cn/training/gestures/scale?hl=zh-cn](https://developer.android.google.cn/training/gestures/scale?hl=zh-cn)