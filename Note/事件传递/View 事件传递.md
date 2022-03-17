

### View 事件分发和处理

*源码版本是28*

 **事件在View中的分发和处理过程：**

1. 判断事件是否被辅助功能处理
2. 获取事件的类型
3. 如果是 `MotionEvent.ACTION_DOWN` 事件，停止嵌套滚动
4.  检查当前窗口是否被遮挡，
   1. 如果View被遮挡(不在顶部)，事件不会下发，直接返回false
   2. View在顶部，分发事件
      1. 此View是 `ENABLED` 的，同时判断事件是否为滚轮(scroll bar)拖动事件，如果事件是在滚轮上拖动事件表示此View消费了此事件。
      2. 此View是 `ENABLED` 的，同时设置了OnTouchListener监听，不管第1步是否消费了事件都会调用OnTouchListener#onTouch()函数，如果onTouch()返回true，表示此View消费了此事件。
      3. 前面两次都没有消费，执行 `onTouchEvent(event)` , 如果 `onTouchEvent(event)` 返回true，设置消费事件的标志。如果第1步和第2步消费了事件，则不会执行onTouchEvent()，也不会执行onClick() 
5. 如果是 `MotionEvent.ACTION_UP` 或 `MotionEvent.ACTION_CANCEL`或者是 `MotionEvent.ACTION_DOWN 但不消费这个事件，停止嵌套滚动。
6. 返回是否处理此事件的结果

> View是Enable的，那么只要窗口不被遮挡，都会执行 `OnTouchListener#onTouch()` 。也就是说，不管View是不是clickable，只要是Enable都有可能消费事件。
>
> 事件传递给View时回调的处理顺序是：onTouch() -> onClick() / onLongClick()

```java
	/**
     * 将触摸屏事件向下传递到目标View，如果它是目标View，则传递给该View。
     *
     * @param event 需要传递的事件对象
     * @return 如果事件由此View处理，则返回true，否则返回false。
     */
public boolean dispatchTouchEvent(MotionEvent event) {
    // 辅助功能，可以不关注
    if (event.isTargetAccessibilityFocus()) {
        if (!isAccessibilityFocusedViewOrHost()) {
            return false;
        }
        // We have focus and got the event, then use normal event dispatch.
        event.setTargetAccessibilityFocus(false);
    }
    boolean result = false;
    
    if (mInputEventConsistencyVerifier != null) {
        mInputEventConsistencyVerifier.onTouchEvent(event, 0);
    }
    
	// 获取事件类型
    final int actionMasked = event.getActionMasked();
    if (actionMasked == MotionEvent.ACTION_DOWN) {
        // Down事件是一次完整事件的开始，停止事件的嵌套滚动
        stopNestedScroll();
    }

    // # 检查当前窗口是否被遮挡，如果被遮挡，事件不会下发，直接返回result (false)
    // 如果该View不位于顶部，并且有设置属性使该View不在顶部时不响应触摸事件(即使顶部覆盖的是透明view
    // 也不会处理事件)， 则不分发该触摸事件，即不会执行onTouch()与onTouchEvent()
    if (onFilterTouchEventForSecurity(event)) {
        // 1. 检查view是否是enable状态; 2. 同时是否是滚动条拖动
        // handleScrollBarDragging处理鼠标拖动滚动条事件，如果处理成功则消费事件
        if ((mViewFlags & ENABLED_MASK) == ENABLED && handleScrollBarDragging(event)) {
            result = true;
        }
        
        // # 判断此View上的mOnTouchListener监听是否消费了event
        // 获取设置到此View上的监听
        ListenerInfo li = mListenerInfo;
        if (li != null && li.mOnTouchListener != null //设置了touch监听(OnTouchListener)
            && (mViewFlags & ENABLED_MASK) == ENABLED //控件是enable的
            && li.mOnTouchListener.onTouch(this, event)) {  //OnTouchListener.onTouch()返回true
            // 此View是enable的，同时监听的onTouch()返回true，表示消费了此event
            // 因此result是否为true由OnTouchListener.onTouch()决定
            result = true;
        }
        
        // 将事件交给自身的onTouchEvent处理
		// # 判断此View是否消费event （如果onTouch() 返回true，不会执行onTouchEvent() ）
        if (!result && onTouchEvent(event)) {
            // onTouchEvent() 返回true，表示消费此事件
            result = true;
        }
    }

    if (!result && mInputEventConsistencyVerifier != null) {
        mInputEventConsistencyVerifier.onUnhandledEvent(event, 0);
    }
    
    // 停止嵌套滚动
    if (actionMasked == MotionEvent.ACTION_UP ||
        actionMasked == MotionEvent.ACTION_CANCEL ||
        (actionMasked == MotionEvent.ACTION_DOWN && !result)) {
        stopNestedScroll();
    }
	// 返回结果
    return result;
}

```

事件分发流程图

![事件传递-事件分发——View#dispatchTouchEvent](事件传递.assets/事件传递-事件分发——View#dispatchTouchEvent.jpg)



### View 事件消费

当我们对View进行一个点击，可以分为以下三个状态：

1. PrePress (姑且叫为预按)，这个状态是我们的view被加入到一个scrolling view中才会存在。具体的意义是，举个简单的例子，当我们将手放在listView的一item上的时候，由于当前还不知道用户是要点击item还是想滑动listview，所以先将view设为PrePress状态；如果View进入了PrePress，那么将会postDelay一个检测，也即CheckForTap()，默认时间是100ms，如果超过了100ms，将由PrePress进入到Press状态；
2. Press状态，用户将手放到view上面，如果不在可滚动的父容器内，直接设置为这个状态，触发页面重绘，显示按压效果。
3. LongPress状态，这个状态由Press状态过度过来，如果用户在一个view上停留超过一定的时间（默认为500ms），将进入该状态。

View事件消费流程：

1. 获取事件坐标

2. 判断此View是否是可点击的 clickable == (CLICKABLE | LONG_CLICKABLE | CONTEXT_CLICKABLE)

3. 如果View是 DISABLED，则直接返回 clickable 。也就是说可点击的disable视图仍然消费触摸事件，只是不响应事件。

4. 判断View是否设置了 TouchDelegate，如果设置了，返回TouchDelegate的处理结果。

5. 如果View是不可点击的 clickable != true，同时没有TOOLTIP标志，直接返回false，不消费事件。

6. 如果View是可点击的 clickable == true，或者设置了TOOLTIP标志，消费事件。 处理事件流程：

   1. Down事件：

   > View不可点击 clickable = false：检查是否可以执行长按操作，如果可以，postDelay处理长按的Runnable对象 mPendingCheckForLongPress。 到了延迟时间，runnable没有被移除的话就执行长按逻辑
   >
   > View可点击clickable = true：判断View是否在可滚动的容器内。1）在可滚动的容器内，postDelay处理点击的Runnable对象 mPendingCheckForTap。 2）不在可滚动的容器内，设置View的点击状态，检查是否可以执行长按操作，如果可以，postDelay处理长按的Runnable对象 mPendingCheckForLongPress。 

   2. Move事件：

   > 1. View可点击clickable = true，更新HotSpot坐标。
   >
   > 2. 如果事件在当前View的区域内，什么都不做
   > 3. 如果事件不在当前View的区域内，移除Runnable对象 mPendingCheckForTap 和 mPendingCheckForLongPress。如果已经设置了 Pressed 状态，清除 Pressed 状态。

   3. Up事件：

      > 如果有 TOOLTIP 标识，处理 tooltip
      >
      > 如果View是不可点击的，移除Runnable对象 mPendingCheckForTap 和 mPendingCheckForLongPress。返回。
      >
      > 如果View是可点击的，
      >
      > 1）如果View没设置  PFLAG_PRESSED 或者 PFLAG_PREPRESSED 标志，返回
      >
      > 2）如果View设置了  PFLAG_PRESSED 或者 PFLAG_PREPRESSED 标志，
      >
      > 1) 如果是prepressed状态，设置为pressed状态，显示按压效果
      > 2) 如果还没执行长按操作，移除长按回调，执行onClick()，也就是说，在500ms内没有up事件，就执行长按，不执行点击，反之亦然。
      > 3) 清除按压效果
      > 4) 移除点击回调(这个是延迟的点击操作 mPendingCheckForTap，如果在100ms内down->up事件，mPendingCheckForTap还在handler的消息队列里面，在第2步已经执行了onClick() 事件，这里就不重复执行了，需要移除)
   
   4. Cancel事件：
   
   > 如果View是可点击的，清除 Pressed 状态。
   >
   > 移除Runnable对象 mPendingCheckForTap 和 mPendingCheckForLongPress。
   >
   > 清除相关标志。



子类可以实现此方法来处理触摸屏运动事件。

```java
	/**
     * 如果使用该方法检测点击动作，建议通过实现和调用{performClick()}来执行动作。
     * 这将确保一致的系统行为，包括：
     * <li>遵守点击声音偏好
     * <li>调度 OnClickListener 的调用
     * <li>启用辅助功能时处理 {AccessibilityNodeInfo#ACTION_CLICK ACTION_CLICK}
     *
     * @param event 事件对象
     * @return 如果消费事件，返回true。否则，返回false
     */
public boolean onTouchEvent(MotionEvent event) {
    final float x = event.getX();
    final float y = event.getY();
    final int viewFlags = mViewFlags;
    final int action = event.getAction();
	// 判断此View是否可点击
    // CONTEXT_CLICKABLE 将会响应主触笔按下或鼠标右键单击这类的事件
    final boolean clickable = ((viewFlags & CLICKABLE) == CLICKABLE
                    || (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE)
        			|| (viewFlags & CONTEXT_CLICKABLE) == CONTEXT_CLICKABLE;

    if ((viewFlags & ENABLED_MASK) == DISABLED) {
        // View是DISABLED状态
        if (action == MotionEvent.ACTION_UP //up事件
            && (mPrivateFlags & PFLAG_PRESSED) != 0) {	// 已经设置了点击标志
            // 清理此View的 pressed状态
            setPressed(false);
        }
        mPrivateFlags3 &= ~PFLAG3_FINGER_DOWN;
        // 可点击的View，虽然是disabled仍然消耗触摸事件，它只是不响应它们。
        return clickable;
    }
    // 代理对象不为null(当View很小的时候可以使用代理对象扩大他的响应范围)
    // 如果一个view过小，不容易点击，通过扩大点击区域实现更好的交互。可以暂且叫为代理，代理处理该事件。
    if (mTouchDelegate != null) {
        if (mTouchDelegate.onTouchEvent(event)) {
            return true;
        }
    }

    // 此View是可点击的，或者设置了TOOLTIP标志，开始事件消费流程
    if (clickable || (viewFlags & TOOLTIP) == TOOLTIP) {
        switch (action) {
            case MotionEvent.ACTION_UP:	// Up事件
                mPrivateFlags3 &= ~PFLAG3_FINGER_DOWN;
                if ((viewFlags & TOOLTIP) == TOOLTIP) {
                    // 此视图可以在悬停或长按时显示工具提示
                    handleTooltipUp();	// 处理提示
                }
                if (!clickable) {	// 不可点击
                    // 移除点击检测计时器。handler.postDelay()实现的
                    removeTapCallback();
                    // 移除长按检测计时器。handler.postDelay()实现的
                    removeLongPressCallback();
                    // 清除其他标志位
                    mInContextButtonPress = false;
                    mHasPerformedLongPress = false;
                    mIgnoreNextUpEvent = false;
                    break;
                }
                
                // PFLAG_PREPRESSED：在可滑动的容器中，当收到DOWN事件时，会设置prepressed状态，
                // 此时真正的pressed需要延迟一小段时间，防止在滑动过程中显示点击状态
                boolean prepressed = (mPrivateFlags & PFLAG_PREPRESSED) != 0;
                if ((mPrivateFlags & PFLAG_PRESSED) != 0 || prepressed) {
                    // pressed或者prepressed状态下，Up事件获取焦点
                    boolean focusTaken = false;
                    if (isFocusable() && isFocusableInTouchMode() && !isFocused()) {
                        // View能获得焦点但还没有获取焦点，申请获取焦点
                        focusTaken = requestFocus();
                    }

                    //[lxb #7] prepressed状态时设置点击状态
                    //[lxb #7] 问：为什么要在UP的时候才设置点击态，UP不是应该取消点击态才对吗？
                    //[lxb #7] 这个的前提是用户ACTION_DOWN在100ms（TAP_TIME）内UP了，才会到达这里
                    //[lxb #7] 在CheckForTap中有取消该状态的操作
                    if (prepressed) {	// View已经是prepressed状态
                        // 此View还没有更新pressed状态，up事件的时候设置为pressed状态
                        // 让它现在显示按下状态（在安排点击之前）以确保用户看到它。
                        setPressed(true, x, y);
                    }
				
                    // 如果没有执行长按事件并且不忽略下次抬起事件
                    if (!mHasPerformedLongPress && !mIgnoreNextUpEvent) {
                        // 这次是点击事件，还没执行长按事件，移除长按回调
                        removeLongPressCallback();

                        // # 执行点击操作
                        // 已经获取到焦点，不需要临时获取焦点，仅当我们处于按下状态时才执行点击操作
                        if (!focusTaken) {
                            // 使用 Runnable 并发布它而不是直接调用 performClick。
                            // 这允许在单击操作开始之前更新视图的其他视觉状态。
                            if (mPerformClick == null) {
                                mPerformClick = new PerformClick();
                            }
                            if (!post(mPerformClick)) {
                                // post click的runnable失败了，直接执行onClick()操作
                                performClickInternal();
                            }
                        }
                    }

                    if (mUnsetPressedState == null) {
                        // 只调用了 setPressed(false)，取消点击状态
                        mUnsetPressedState = new UnsetPressedState();
                    }

                    if (prepressed) {
                        // 是prepressed状态，post一个延迟的runnable取消点击状态
                        // 延迟时间到了还没有执行点击，就取消点击状态
                        postDelayed(mUnsetPressedState,
                                    ViewConfiguration.getPressedStateDuration());
                    } else if (!post(mUnsetPressedState)) { // 直接post
                        // post失败了，直接执行
                        mUnsetPressedState.run();
                    }
					// 移除点击回调，如果还没来得及执行点击操作，就不会再执行点击了
                    removeTapCallback();
                }
                // 不再接收下一次的up事件
                mIgnoreNextUpEvent = false;
                break;

            case MotionEvent.ACTION_DOWN:
                if (event.getSource() == InputDevice.SOURCE_TOUCHSCREEN) {
                    mPrivateFlags3 |= PFLAG3_FINGER_DOWN;
                }
                // 清除已执行长按操作的标志
                mHasPerformedLongPress = false;

                if (!clickable) {
                    // 不可点击的时候，检查是否可以长按，如果可以长按，就post一个长按回调
                    checkForLongClick(0, x, y);
                    break;
                }

                if (performButtonActionOnTouchDown(event)) {
                    // 判断是否是鼠标点击事件
                    break;
                }

                // 沿着层次结构向上走以确定此View是否在正在滚动容器内。
                boolean isInScrollingContainer = isInScrollingContainer();

                // 对于正在滚动容器内的View，将按下的反馈延迟一小段时间。
                if (isInScrollingContainer) {
                    // 设置标志为 PFLAG_PREPRESSED 
                    mPrivateFlags |= PFLAG_PREPRESSED;
                    if (mPendingCheckForTap == null) {
                        // 创建一个Runnable CheckForTap 用于执行延迟后的操作
                        mPendingCheckForTap = new CheckForTap();
                    }
                    mPendingCheckForTap.x = event.getX();
                    mPendingCheckForTap.y = event.getY();
                    // 延迟一小段时间，时间到了如果这个runnable没有被移除
                    // 就执行ACTION_DOWN事件该执行的操作，设置为pressed状态
                    postDelayed(mPendingCheckForTap, ViewConfiguration.getTapTimeout());
                } else {		// 不在滚动容器内，立即显示反馈
                    setPressed(true, x, y);	//设置为pressed状态
                    // 创建一个runnable，检查长按操作
                    checkForLongClick(0, x, y);
                }
                break;

            case MotionEvent.ACTION_CANCEL:// 清除点击状态，移除回调计时，更新标记
                if (clickable) {
                    // 清除pressed状态，清除pressed的效果（比如点击阴影）
                    setPressed(false);
                }
                // 移除点击回调
                removeTapCallback();
                // 移除长按回调
                removeLongPressCallback();
                // 更新状态
                mInContextButtonPress = false;
                mHasPerformedLongPress = false;
                mIgnoreNextUpEvent = false;
                mPrivateFlags3 &= ~PFLAG3_FINGER_DOWN;
                break;

            case MotionEvent.ACTION_MOVE:
                if (clickable) {
                    // 更新视图热点
                    drawableHotspotChanged(x, y);
                }

                // # 是确定的手势类型，同时手指滑动超出了此View的范围
                if (!pointInView(x, y, mTouchSlop)) {
                    // 手指移除view
                    // 移除点击回调
                    removeTapCallback();
                    // 移除长按回调
                    removeLongPressCallback();
                    if ((mPrivateFlags & PFLAG_PRESSED) != 0) {
                        // 取消点击状态
                        setPressed(false);
                    }
                    mPrivateFlags3 &= ~PFLAG3_FINGER_DOWN;
                }
                break;
        }
		// 消费事件
        return true;
    }
	// 不消费事件
    return false;
}
```

事件消费流程图

![事件传递-事件分发——View事件消费](事件传递.assets/事件传递-事件分发——View事件消费-16421461896391.jpg)

Down事件

![](https://github.com/philuss/Notes/blob/dev/View%E4%BA%8B%E4%BB%B6%E6%B6%88%E8%B4%B9-Down%E4%BA%8B%E4%BB%B6-16421464033742.jpg?raw=true)



Move事件

![View事件消费-Move事件](事件传递.assets/View事件消费-Move事件-16421465778493.jpg)



Up事件

![View事件消费-Up事件](事件传递.assets/View事件消费-Up事件-16421465867934.jpg)



长按事件有关的函数

```java
    /**
     * 检查此View是否是LONG_CLICKABLE的，如果是则post一个延迟的runnable，
     * 如果延迟时间到了runnable还在则执行长按逻辑 
     */
	private void checkForLongClick(long delay, float x, float y) {
        if ((mViewFlags & LONG_CLICKABLE) == LONG_CLICKABLE || (mViewFlags & TOOLTIP) == TOOLTIP) {	
            // 此View可以被长按，或者此View可以在悬停或长按时显示工具提示。
            // 设置标志位
            mHasPerformedLongPress = false;

            if (mPendingCheckForLongPress == null) {
                // 创建一个runnable
                mPendingCheckForLongPress = new CheckForLongPress();
            }
            // 记录当前的坐标
            mPendingCheckForLongPress.setAnchor(x, y);
            mPendingCheckForLongPress.rememberWindowAttachCount();
            mPendingCheckForLongPress.rememberPressedState();
            // 通过handler post的方式将mPendingCheckForLongPress加入到主Looper中
            postDelayed(mPendingCheckForLongPress, 
                        ViewConfiguration.getLongPressTimeout() - delay);
        }
    }

    /** 删除长按回调 */
    private void removeLongPressCallback() {
        // mPendingCheckForLongPress 是 checkForLongClick()函数中创建并post的Runnable对象
        if (mPendingCheckForLongPress != null) {
            removeCallbacks(mPendingCheckForLongPress);
        }
    }

    /** 如果在主Looper中有长按回调，则返回 true。 否则，返回 false。 */
    private boolean hasPendingLongPressCallback() {
        if (mPendingCheckForLongPress == null) {
            return false;
        }
        final AttachInfo attachInfo = mAttachInfo;
        if (attachInfo == null) {
            return false;
        }
        return attachInfo.mHandler.hasCallbacks(mPendingCheckForLongPress);
    }

	// 处理长按事件的Runnable
	private final class CheckForLongPress implements Runnable {
        private int mOriginalWindowAttachCount;
        private float mX;	// x坐标
        private float mY;	// y坐标
        private boolean mOriginalPressedState;

        @UnsupportedAppUsage
        private CheckForLongPress() {
        }

        @Override
        public void run() {
            if ((mOriginalPressedState == isPressed()) && (mParent != null)
                    && mOriginalWindowAttachCount == mWindowAttachCount) {
                if (performLongClick(mX, mY)) {	// 执行长按操作
                    // 长按操作成功，设置标志位，设置了这个标志，在up事件中就不会执行onClick()操作
                    mHasPerformedLongPress = true;
                }
            }
        }

        public void setAnchor(float x, float y) {
            mX = x;
            mY = y;
        }

        public void rememberWindowAttachCount() {
            mOriginalWindowAttachCount = mWindowAttachCount;
        }

        public void rememberPressedState() {
            mOriginalPressedState = isPressed();
        }
    }

	/**
     * 调用此视图的 OnLongClickListener（如果已定义）。
     * 如果 OnLongClickListener 未消费事件，则调用上下文菜单，将其锚定到 (x,y) 坐标。
     *
     * @param x 锚定触摸事件的 x 坐标，或 {Float#NaN} 禁用锚定
     * @param y 锚定触摸事件的 y 坐标，或 {Float#NaN} 禁用锚定
     * @return 如果接收者消费了该事件，返回true，否则返回 false
     */
    public boolean performLongClick(float x, float y) {
        // 记录坐标
        mLongClickX = x;
        mLongClickY = y;
        final boolean handled = performLongClick();
        // 长按逻辑处理完之后，清除坐标
        mLongClickX = Float.NaN;
        mLongClickY = Float.NaN;
        return handled; // 返回是否消费事件
    }

    public boolean performLongClick() {
        return performLongClickInternal(mLongClickX, mLongClickY);
    }

    private boolean performLongClickInternal(float x, float y) {
        sendAccessibilityEvent(AccessibilityEvent.TYPE_VIEW_LONG_CLICKED);

        boolean handled = false;	// 是否消费事件
        final ListenerInfo li = mListenerInfo;
        if (li != null && li.mOnLongClickListener != null) {
            // 执行回调
            handled = li.mOnLongClickListener.onLongClick(View.this);
        }
        if (!handled) {	// 没有消费事件
            final boolean isAnchored = !Float.isNaN(x) && !Float.isNaN(y);
            // 显示上下文菜单
            handled = isAnchored ? showContextMenu(x, y) : showContextMenu();
        }
        if ((mViewFlags & TOOLTIP) == TOOLTIP) {
            // 有 TOOLTIP 标记
            if (!handled) {
                // 显示工具栏提示
                handled = showLongClickTooltip((int) x, (int) y);
            }
        }
        if (handled) {
            // 为此视图向用户提供触觉反馈。
            performHapticFeedback(HapticFeedbackConstants.LONG_PRESS);
        }
        return handled;
    }
```



点击事件相关函数

```java
    /**
     * {performClick()} 的入口点 - View 上的其他方法应该直接调用它而不是 {performClick()} 
     * 以确保在必要时通知自动填充管理器（因为子类可以扩展 {performClick()}不调用父方法）。
     */
    private boolean performClickInternal() {
        // 必须在执行单击操作之前通知自动填充管理器，
        // 以避免应用程序具有更改自动填充服务可能感兴趣的视图状态的单击侦听器的情况。
        notifyAutofillManagerOnClick();

        return performClick();
    }

    public boolean performClick() {
        notifyAutofillManagerOnClick();
        final boolean result;
        final ListenerInfo li = mListenerInfo;
        if (li != null && li.mOnClickListener != null) { // 设置了OnClickListener
            // 为此视图播放声音效果
            playSoundEffect(SoundEffectConstants.CLICK);
            li.mOnClickListener.onClick(this);	// 执行 onClick
            result = true;	// 有监听被执行
        } else {
            result = false;
        }
        sendAccessibilityEvent(AccessibilityEvent.TYPE_VIEW_CLICKED);
        notifyEnterOrExitForAutoFillIfNeeded(true);
        return result;
    }

	/** 移除点击回调 */
    private void removeTapCallback() {
        // mPendingCheckForTap 是onTouchEvent在处理DOWN事件的时候post的一个延迟runnable
        // 如果此View不在可滚动的容器内，mPendingCheckForTap为null
        if (mPendingCheckForTap != null) {
            mPrivateFlags &= ~PFLAG_PREPRESSED;	// 移除 prepressed标志
            removeCallbacks(mPendingCheckForTap);
        }
    }

    /** 设置此视图的按下状态并为动画提示提供触摸坐标。 */
    private void setPressed(boolean pressed, float x, float y) {
        if (pressed) {
            // 更新热点坐标
            drawableHotspotChanged(x, y);
        }
        setPressed(pressed);
    }


    /** 设置此视图的按下状态。 */
    public void setPressed(boolean pressed) {
        // 判断是否需要刷新：pressed的状态有变化
        final boolean needsRefresh = pressed != ((mPrivateFlags & PFLAG_PRESSED) == PFLAG_PRESSED);
		// 设置标记
        if (pressed) {
            mPrivateFlags |= PFLAG_PRESSED;
        } else {
            mPrivateFlags &= ~PFLAG_PRESSED;
        }

        if (needsRefresh) {
            // 刷新drawable状态 ==> 调用 invalidate() 刷新页面
            refreshDrawableState();
        }
        // 分发pressed状态给子类
        dispatchSetPressed(pressed);
    }


    /**
     * 调用它以强制视图更新其可绘制状态。
     * 这将导致在此视图上调用 drawableStateChanged。
     * 对新状态感兴趣的视图应该调用 getDrawableState。
     */
    public void refreshDrawableState() {
        mPrivateFlags |= PFLAG_DRAWABLE_STATE_DIRTY;
        // 会调用 invalidate()触发重绘
        drawableStateChanged();

        ViewParent parent = mParent;
        if (parent != null) {	// 递归调用，最终的节点parent为null
            // 通知父控件，绘制状态发生变化了
            parent.childDrawableStateChanged(this);
        }
    }

    /** 每当视图的状态发生变化从而影响正在显示的可绘制对象的状态时，都会调用此函数。 */
    protected void drawableStateChanged() {
        final int[] state = getDrawableState();
        boolean changed = false;

        final Drawable bg = mBackground;
        if (bg != null && bg.isStateful()) {
            changed |= bg.setState(state);
        }

        final Drawable hl = mDefaultFocusHighlight;
        if (hl != null && hl.isStateful()) {
            changed |= hl.setState(state);
        }

        final Drawable fg = mForegroundInfo != null ? mForegroundInfo.mDrawable : null;
        if (fg != null && fg.isStateful()) {
            changed |= fg.setState(state);
        }

        if (mScrollCache != null) {
            final Drawable scrollBar = mScrollCache.scrollBar;
            if (scrollBar != null && scrollBar.isStateful()) {
                changed |= scrollBar.setState(state)
                        && mScrollCache.state != ScrollabilityCache.OFF;
            }
        }

        if (mStateListAnimator != null) {
            mStateListAnimator.setState(state);
        }

        if (changed) {
            // 使整个视图无效 ==> 触发页面刷新
            invalidate();
        }
    }

    private final class PerformClick implements Runnable {
        @Override
        public void run() {
            performClickInternal();
        }
    }

	// 延迟点击，在down事件中，如果View在可滚动的父容器内，会postDelay这个Runnable，
	// 延迟100ms再开始执行点击逻辑
    private final class CheckForTap implements Runnable {
        public float x;
        public float y;

        @Override
        public void run() {
            mPrivateFlags &= ~PFLAG_PREPRESSED;	// 移除prepressed标志
            setPressed(true, x, y);
            checkForLongClick(ViewConfiguration.getTapTimeout(), x, y);
        }
    }
```





参考

[Android触摸事件（续）——点击长按事件 - 简书 (jianshu.com)](https://www.jianshu.com/p/f57bab1db012)