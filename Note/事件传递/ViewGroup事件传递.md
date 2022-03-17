ViewGroup事件传递

ViewGroup的dispatchTouchEvent是对View的dispatchTouchEvent函数的重写，两者具有很大的区别。如下表：

| View#dispatchTouchEvent                    | ViewGroup#dispatchTouchEvent                                 |
| :----------------------------------------- | :----------------------------------------------------------- |
| 1）将Event派发给自己的onTouchEvent函数处理 | 1）将Event派发给子View<br>2）只有在子View没有响应该事件或ViewGroup拦截了该事件的时候，才通过View的dispatchTouchEvent将事件派发给自身 |

ViewGroup的dispatchTouchEvent的主要任务：



1. 检查当前窗口是否被遮挡，如果被遮挡，返回false，不往下分发事件。
2. 窗口不被遮挡（当前View在顶层），事件向下分发：
   1. 获取事件类型。
   2. 如果是DOWN事件，清除所有TouchTarget，重置TouchState。
   3. 检查是否拦截此事件
   4. 检查事件是否被取消
   5. 在事件没有被拦截和取消的情况下，如果是DOWN事件，查找事件的 touchTarget
   6. 分发事件给目标View
3. 返回处理结果

```java
// 重写父类View的事件分发函数
/**
 * 将触摸屏事件向下传递到目标视图，或者如果当前视图是目标视图，则传递给自己。
 * @param event 要分派的事件。
 * @return 如果事件由视图处理，则为true，否则为false。
 */
public boolean dispatchTouchEvent(MotionEvent ev) {
    if (mInputEventConsistencyVerifier != null) {
        mInputEventConsistencyVerifier.onTouchEvent(ev, 1);
    }
 
    //[辅助功能] 事件将会第一个派发给开启了accessibility focused的view    
    if (ev.isTargetAccessibilityFocus() && isAccessibilityFocusedViewOrHost()) {
        ev.setTargetAccessibilityFocus(false);
    }
    
	// handled 是最终要返回的结果值
    boolean handled = false;
    // # 检查当前窗口是否被遮挡，如果被遮挡，事件不会下发
    if (onFilterTouchEventForSecurity(ev)) {
        final int action = ev.getAction();
        // 获取事件类型
        final int actionMasked = action & MotionEvent.ACTION_MASK;

        // # Down事件，清除上一个事件的所有状态
        // Down事件，是一次完整事件的开始
        if (actionMasked == MotionEvent.ACTION_DOWN) {
            // 开始新的触摸手势时丢弃所有以前的状态。
            // 由于应用程序切换、ANR 或某些其他状态更改，框架可能已删除前一个手势的 up 或 cancel 事件。
            cancelAndClearTouchTargets(ev); // mFirstTouchTarget置为null
            resetTouchState(); //状态位重置
        }

        // # 检查当前View是否拦截此事件
        final boolean intercepted;
        if (actionMasked == MotionEvent.ACTION_DOWN
            || mFirstTouchTarget != null) {	// Down事件或者可消费此事件的target不为null
            // 检查禁止拦截标志,FLAG_DISALLOW_INTERCEPT标志是requestDisallowInterceptTouchEvent设置的
            final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
            if (!disallowIntercept) {	// 子View没有设置禁止拦截的标志
                // 判断自己是否要拦截此事件
                intercepted = onInterceptTouchEvent(ev);
                ev.setAction(action); // 恢复操作以防它被更改
            } else {
                // 子View禁止拦截事件
                intercepted = false;
            }
        } else {
            // 没有触摸目标并且此操作不是down事件，因此该ViewGroup继续拦截触摸。
            intercepted = true;
        }

        // 如果ViewGroup自己要拦截，或者已经有一个正在处理手势的子View，则执行正常的事件调度过程。
        if (intercepted || mFirstTouchTarget != null) {
            ev.setTargetAccessibilityFocus(false);
        }

        // 检查事件是否被取消
        final boolean canceled = resetCancelNextUpFlag(this)
            || actionMasked == MotionEvent.ACTION_CANCEL;

        // 如果需要，更新指针向下的触摸目标列表。
        final boolean isMouseEvent = ev.getSource() == InputDevice.SOURCE_MOUSE;
        // 比如多个手指放到了屏幕上，是否要第二个手指的事件
        final boolean split = (mGroupFlags & FLAG_SPLIT_MOTION_EVENTS) != 0
            && !isMouseEvent;
        
        // # 查找目标View
        TouchTarget newTouchTarget = null;
        // 用于纪录事件是否传递给子View，或者说是否有子View成功处理了Touch事件，有则置为True.
        boolean alreadyDispatchedToNewTouchTarget = false; 
        if (!canceled && !intercepted) { // 没有被取消和被拦截
            // 判断事件是否有可达的焦点View，如果有的话，事件会先将其交由焦点View处理
            View childWithAccessibilityFocus = ev.isTargetAccessibilityFocus()
                ? findChildWithAccessibilityFocus() : null;
			
            // 只有DOWN事件才会查找目标View
            if (actionMasked == MotionEvent.ACTION_DOWN // 触摸事件down
                || (split && actionMasked == MotionEvent.ACTION_POINTER_DOWN) // 多点事件down
                || actionMasked == MotionEvent.ACTION_HOVER_MOVE) { // 鼠标指针move，但没有按下
                final int actionIndex = ev.getActionIndex(); // always 0 for down
                final int idBitsToAssign = split ? 1 << ev.getPointerId(actionIndex)
                    : TouchTarget.ALL_POINTER_IDS;

                // 清除此指针id对应的旧的数据
                removePointersFromTouchTargets(idBitsToAssign);

                final int childrenCount = mChildrenCount; // 有效的子View的数量
                // 开始查找本次事件的目标子View
                if (newTouchTarget == null && childrenCount != 0) {
                    // 获取事件的坐标
                    final float x =
                        isMouseEvent ? ev.getXCursorPosition() : ev.getX(actionIndex);
                    final float y =
                        isMouseEvent ? ev.getYCursorPosition() : ev.getY(actionIndex);
                    // 找到一个可以接收事件的子View。
                    // 后续遍历子View。
                    final ArrayList<View> preorderedList = buildTouchDispatchChildList();
                    final boolean customOrder = preorderedList == null
                        && isChildrenDrawingOrderEnabled();
                    final View[] children = mChildren;
                    for (int i = childrenCount - 1; i >= 0; i--) {
                        // ViewGroup在addView的时候后添加的会显示在上层，我们点击View的时候肯定是想先让浮于上层的View响应触摸事件,先添加的显示在底层，后添加的显示在上层，所以查找的时候，先从可见的上层查找起
                        final int childIndex = getAndVerifyPreorderedIndex(
                            childrenCount, i, customOrder);
                        // 获取childIndex位置处的view
                        final View child = getAndVerifyPreorderedView(
                            preorderedList, children, childIndex);
                        // 如果有一个view具有可达焦点，先把事件给这个view，
                        // 如果焦点view没有处理，执行正常的事件调度
                        if (childWithAccessibilityFocus != null) {
                            if (childWithAccessibilityFocus != child) {
                                continue;
                            }
                            childWithAccessibilityFocus = null;
                            i = childrenCount - 1;
                        }
                        // 判断事件坐标x，y是否在子View child内
                        if (!child.canReceivePointerEvents()
                            || !isTransformedTouchPointInView(x, y, child, null)) {
                            ev.setTargetAccessibilityFocus(false);
                            // 事件x，y坐标不在view之内，继续下一个view
                            continue;
                        }
                        
					  // 事件坐标x，y在子View child内，
                        // 判断child是否在TouchTarget链表中。如果在，返回此View，否则返回null
                        newTouchTarget = getTouchTarget(child);
                        if (newTouchTarget != null) {
                            // child已经在mFirstTouchTarget链表中。
                            // 更新 child的pointerIdBits
                            newTouchTarget.pointerIdBits |= idBitsToAssign;
                            // 退出循环
                            break;
                        }

                        // 执行到这里，事件落在child内，但child不在TouchTarget链表中。
                        
                        resetCancelNextUpFlag(child);
                        // 将事件传递给子View，在此处传递就不需要再后面遍历的地方再次传递了
                        if (dispatchTransformedTouchEvent(ev, false, child, idBitsToAssign)) {	// 子View消费了事件
                            // 记录本次事件的时间
                            mLastTouchDownTime = ev.getDownTime();
                            // 找到消费事件的view的index
                            if (preorderedList != null) {
                                // childIndex 指向预排序列表，找到原始索引
                                for (int j = 0; j < childrenCount; j++) {
                                    if (children[childIndex] == mChildren[j]) {
                                        mLastTouchDownIndex = j;
                                        break;
                                    }
                                }
                            } else {
                                mLastTouchDownIndex = childIndex;
                            }
                            // 记录本次事件的坐标
                            mLastTouchDownX = ev.getX();
                            mLastTouchDownY = ev.getY();
                            // 将子View加入到TouchTarget链表中
                            // mFirstTouchTarget和newTouchTarget 此时指向相同
                            newTouchTarget = addTouchTarget(child, idBitsToAssign);
                            alreadyDispatchedToNewTouchTarget = true;
                            // 子View消费了事件，退出循环
                            break;
                        }

                        // 有焦点的view没有处理事件，清除标记，将事件分发给所有子View
                        ev.setTargetAccessibilityFocus(false);
                    }
                    // 没有子View消费此事件，清除预排序列表
                    if (preorderedList != null) preorderedList.clear();
                }
				
                // 没有找到新的子View消费这个事件，
                if (newTouchTarget == null && mFirstTouchTarget != null) {
                    // 没有找到一个新的子View来接收本次事件。
                    // 将指针分配给最近添加的目标View（上一次事件的目标View）。
                    newTouchTarget = mFirstTouchTarget;
                    while (newTouchTarget.next != null) {
                        newTouchTarget = newTouchTarget.next;
                    }
                    newTouchTarget.pointerIdBits |= idBitsToAssign;
                }
            }
        }
        
        // 以上是对不拦截，未取消事件的处理和分发，如果找到了要消费事件的view，mFirstTouchTarget就不为空了

        // 调度触摸目标。
        if (mFirstTouchTarget == null) { // 一是可能事件都被拦截了，二是没有找到要消费事件的子View
            // 没有触摸目标，调用父类的dispatchTouchEvent()函数处理事件
            // 第三个参数传null，此ViewGroup自己消费事件。调用的是父类view的dispatchTouchEvent方法
            handled = dispatchTransformedTouchEvent(ev, canceled, null,
                                                    TouchTarget.ALL_POINTER_IDS);
        } else {
            // 分派事件到触摸目标。如有必要，取消触摸目标。
            TouchTarget predecessor = null;
            TouchTarget target = mFirstTouchTarget;
            // 遍历TouchTarget链表中的所有目标target
            while (target != null) {
                final TouchTarget next = target.next;
                if (alreadyDispatchedToNewTouchTarget && target == newTouchTarget) {
                    // 上面查找的时候已经分发给他了，不再重复分发
                    handled = true;
                } else {
                    // 如果事件被取消或者被拦截，给目标子View分发cancel事件
                    final boolean cancelChild = resetCancelNextUpFlag(target.child)
                        || intercepted;
                    // 分发事件给目标View
                    if (dispatchTransformedTouchEvent(ev, cancelChild,
                                                      target.child, target.pointerIdBits)) {
                        handled = true;
                    }
                    if (cancelChild) {
                        if (predecessor == null) {
                            mFirstTouchTarget = next;
                        } else {
                            predecessor.next = next;
                        }
                        // 回收目标View
                        target.recycle();
                        // 进入下一个循环
                        target = next;
                        continue;
                    }
                }
                // 进入下一个循环
                predecessor = target;
                target = next;
            }
        }

        // 如有必要，重置状态，清除TouchTarget链表
        if (canceled	// 事件被取消
            || actionMasked == MotionEvent.ACTION_UP	// up事件
            || actionMasked == MotionEvent.ACTION_HOVER_MOVE) { // 鼠标指针移动事件
            // 重置状态，清除TouchTarget链表
            resetTouchState();
        } else if (split && actionMasked == MotionEvent.ACTION_POINTER_UP) {
            // 多点触控的取消处理
            final int actionIndex = ev.getActionIndex();
            final int idBitsToRemove = 1 << ev.getPointerId(actionIndex);
            removePointersFromTouchTargets(idBitsToRemove);
        }
    }

    if (!handled && mInputEventConsistencyVerifier != null) {
        mInputEventConsistencyVerifier.onUnhandledEvent(ev, 1);
    }
    // 返回是否消费事件的结果
    return handled;
}
```

下面分步骤详细分析

### 首先看一下检查是否拦截事件

```java
        // # 检查当前View是否拦截此事件
        final boolean intercepted;
        if (actionMasked == MotionEvent.ACTION_DOWN
            || mFirstTouchTarget != null) {	// Down事件或者可消费此事件的target不为null
            // 检查禁止拦截标志,FLAG_DISALLOW_INTERCEPT标志是requestDisallowInterceptTouchEvent设置的
            final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
            if (!disallowIntercept) {	// 子View没有设置禁止拦截的标志
                // 判断自己是否要拦截此事件
                intercepted = onInterceptTouchEvent(ev);
                ev.setAction(action); // 恢复操作以防它被更改
            } else {
                // 子View禁止拦截事件，那么设置标志为false
                intercepted = false;
            }
        } else {
            // 没有触摸目标并且此操作不是down事件，因此该视图组继续拦截触摸。
            intercepted = true;
        }
```

首先是两个条件

- actionMasked == MotionEvent.ACTION_DOWN
- mFirstTouchTarget != null

如果满足了其中一个条件才会继续走下去，执行onInterceptTouchEvent方法等，否则就直接intercepted = true，表示拦截。 第一个条件很明显，就是表示当前事件为按下事件（ACTION_DOWN） 第二个条件是个字段，根据下面的代码可以得知，当后面有view消费掉事件的时候，这个`mFirstTouchTarget`字段就会赋值，否则就为空。

**所以什么意思呢**，当ACTION_DOWN事件时候，一定会执行到后面代码。当其他事件来的时候，要看当前viewGroup是否消费了事件，如果当前viewGroup已经消费了事件，没传到子view，那么`mFirstTouchTarget`字段就为空，所以就不会执行到后面的代码，就直接消费掉所有事件了。 这就符合了之前的所说的一种机制：

**某个view一旦开始拦截，那么后续事件就全部就给它处理了，也不会执行onInterceptTouchEvent方法了**

但是，两个条件满足了一个，就能执行到onInterceptTouchEvent了吗？不一定，这里看到还有一个判断条件：`disallowIntercept`。这个字段是由`requestDisallowInterceptTouchEvent方法`设置的，后面我们会讲到，主要用于滑动冲突，意思就是子view告诉你不想让你拦截，那么你就不拦截了，直接返回false。



### 查找目标View

只有DOWN事件才会查找目标View，找到之后更新其 pointerIdBits （用于再事件分发的时候区别是哪个事件，多点触控的时候，一个View可能对应多个事件，或者多个View对应多个事件） 。查找流程如下：

1. 判断事件是否是 CANCEL 事件
2. 如果事件不是CANCEL事件，同时没有被父View拦截，查找事件的目标View TouchTarget
   1. **如果事件不是 DOWN 事件，不查找**
   2. 如果事件是 DOWN 事件，开始查找目标View TouchTarget：
      1. 获取事件的坐标
      2. 倒序遍历VIewGroup中的所有子View，判断事件坐标是否在子View的区域内，
         1. 如果事件落在子View内，则从TouchTarget链表中查找当前子View对应的TouchTarget，
            1. 如果找到了，更新其 pointerIdBits 属性，退出循环
            2. 如果没有找到，说明是一个新的子View，将事件分发给他，如果子View消费了事件，将子View添加到TouchTarget链表中，并赋值给newTouchTarget，同时设置已处理标记（后面时间分发的时候不需要再发给它了），退出循环。
         2. 如果事件没有在当前子View内，遍历下一个
      3. 遍历完之后，newTouchTarget为null，说明没有找到可消费事件的子View，如果 mFirstTouchTarget 不为null，则将上次找到的目标View当作此次事件的目标View，更新它的 pointerIdBits 属性。

> 为什么会有 mFirstTouchTarget  不为null的情况？
> 因为只有在 `ACTION_DOWN` 的时候才会清除 `mFirstTouchTarget` ，而查找目标View的条件是 `ACTION_DOWN | ACTION_POINTER_DOWN | ACTION_HOVER_MOVE` 三个，包含多点触控，鼠标移动等事件，可能已经有DOWN的时候，另外的手指也按下了，或者鼠标移动了，都会查找目标View。



```java
        // 检查事件是否被取消
        final boolean canceled = resetCancelNextUpFlag(this)
            || actionMasked == MotionEvent.ACTION_CANCEL;

        // 如果需要，更新指针向下的触摸目标列表。
        final boolean isMouseEvent = ev.getSource() == InputDevice.SOURCE_MOUSE;
        // 比如多个手指放到了屏幕上，是否要第二个手指的事件
        final boolean split = (mGroupFlags & FLAG_SPLIT_MOTION_EVENTS) != 0
            && !isMouseEvent;
        
        // # 查找目标View
        TouchTarget newTouchTarget = null;
        // 用于纪录事件是否传递给子View，或者说是否有子View成功处理了Touch事件，有则置为True.
        boolean alreadyDispatchedToNewTouchTarget = false; 
        if (!canceled && !intercepted) { // 没有被取消和被拦截
            // 判断事件是否有可达的焦点View，如果有的话，事件会先将其交由焦点View处理
            View childWithAccessibilityFocus = ev.isTargetAccessibilityFocus()
                ? findChildWithAccessibilityFocus() : null;
			
            // 只有DOWN事件才会查找目标View
            if (actionMasked == MotionEvent.ACTION_DOWN // 触摸事件down
                || (split && actionMasked == MotionEvent.ACTION_POINTER_DOWN) // 多点事件down
                || actionMasked == MotionEvent.ACTION_HOVER_MOVE) { // 鼠标指针move，但没有按下
                final int actionIndex = ev.getActionIndex(); // always 0 for down
                final int idBitsToAssign = split ? 1 << ev.getPointerId(actionIndex)
                    : TouchTarget.ALL_POINTER_IDS;

                // 清除此指针id对应的旧的数据
                removePointersFromTouchTargets(idBitsToAssign);

                final int childrenCount = mChildrenCount; // 有效的子View的数量
                // 开始查找本次事件的目标子View
                if (newTouchTarget == null && childrenCount != 0) {
                    // 获取事件的坐标
                    final float x =
                        isMouseEvent ? ev.getXCursorPosition() : ev.getX(actionIndex);
                    final float y =
                        isMouseEvent ? ev.getYCursorPosition() : ev.getY(actionIndex);
                    // 找到一个可以接收事件的子View。
                    // 后续遍历子View。
                    final ArrayList<View> preorderedList = buildTouchDispatchChildList();
                    final boolean customOrder = preorderedList == null
                        && isChildrenDrawingOrderEnabled();
                    final View[] children = mChildren;
                    for (int i = childrenCount - 1; i >= 0; i--) {
                        // ViewGroup在addView的时候后添加的会显示在上层，我们点击View的时候肯定是想先让浮于上层的View响应触摸事件,先添加的显示在底层，后添加的显示在上层，所以查找的时候，先从可见的上层查找起
                        final int childIndex = getAndVerifyPreorderedIndex(
                            childrenCount, i, customOrder);
                        // 获取childIndex位置处的view
                        final View child = getAndVerifyPreorderedView(
                            preorderedList, children, childIndex);
                        // 如果有一个view具有可达焦点，先把事件给这个view，
                        // 如果焦点view没有处理，执行正常的事件调度
                        if (childWithAccessibilityFocus != null) {
                            if (childWithAccessibilityFocus != child) {
                                continue;
                            }
                            childWithAccessibilityFocus = null;
                            i = childrenCount - 1;
                        }
                        // 判断事件坐标x，y是否在子View child内
                        if (!child.canReceivePointerEvents()
                            || !isTransformedTouchPointInView(x, y, child, null)) {
                            ev.setTargetAccessibilityFocus(false);
                            // 事件x，y坐标不在view之内，继续下一个view
                            continue;
                        }
                        
					  // 事件坐标x，y在子View child内，
                        // 判断child是否在TouchTarget链表中。如果在，返回此View，否则返回null
                        newTouchTarget = getTouchTarget(child);
                        if (newTouchTarget != null) {
                            // child已经在mFirstTouchTarget链表中。
                            // 更新 child的pointerIdBits
                            newTouchTarget.pointerIdBits |= idBitsToAssign;
                            // 退出循环
                            break;
                        }

                        // 执行到这里，事件落在child内，但child不在TouchTarget链表中。
                        
                        resetCancelNextUpFlag(child);
                        // 将事件传递给子View，在此处传递就不需要再后面遍历的地方再次传递了
                        if (dispatchTransformedTouchEvent(ev, false, child, idBitsToAssign)) {	// 子View消费了事件
                            // 记录本次事件的时间
                            mLastTouchDownTime = ev.getDownTime();
                            // 找到消费事件的view的index
                            if (preorderedList != null) {
                                // childIndex 指向预排序列表，找到原始索引
                                for (int j = 0; j < childrenCount; j++) {
                                    if (children[childIndex] == mChildren[j]) {
                                        mLastTouchDownIndex = j;
                                        break;
                                    }
                                }
                            } else {
                                mLastTouchDownIndex = childIndex;
                            }
                            // 记录本次事件的坐标
                            mLastTouchDownX = ev.getX();
                            mLastTouchDownY = ev.getY();
                            // 将子View加入到TouchTarget链表中
                            // mFirstTouchTarget和newTouchTarget 此时指向相同
                            newTouchTarget = addTouchTarget(child, idBitsToAssign);
                            alreadyDispatchedToNewTouchTarget = true;
                            // 子View消费了事件，退出循环
                            break;
                        }

                        // 有焦点的view没有处理事件，清除标记，将事件分发给所有子View
                        ev.setTargetAccessibilityFocus(false);
                    }
                    // 没有子View消费此事件，清除预排序列表
                    if (preorderedList != null) preorderedList.clear();
                }
				
                // 没有找到新的子View消费这个事件，
                if (newTouchTarget == null && mFirstTouchTarget != null) {
                    // 没有找到一个新的子View来接收本次事件。
                    // 将指针分配给最近添加的目标View（上一次事件的目标View）。
                    newTouchTarget = mFirstTouchTarget;
                    while (newTouchTarget.next != null) {
                        newTouchTarget = newTouchTarget.next;
                    }
                    newTouchTarget.pointerIdBits |= idBitsToAssign;
                }
            }
        }
```



### TouchTarget链表的处理

```java
	// ViewGroup.java

    /** 清除所有 touch target */
    private void clearTouchTargets() {
        TouchTarget target = mFirstTouchTarget;
        if (target != null) {
            do {
                TouchTarget next = target.next;
                target.recycle();
                target = next;
            } while (target != null);
            mFirstTouchTarget = null;
        }
    }

    /** 给所有目标View分发cancel事件并清除所有目标View */
    private void cancelAndClearTouchTargets(MotionEvent event) {
        if (mFirstTouchTarget != null) {
            boolean syntheticEvent = false;
            if (event == null) {
                final long now = SystemClock.uptimeMillis();
                event = MotionEvent.obtain(now, now,
                        MotionEvent.ACTION_CANCEL, 0.0f, 0.0f, 0);
                event.setSource(InputDevice.SOURCE_TOUCHSCREEN);
                syntheticEvent = true;
            }

            for (TouchTarget target = mFirstTouchTarget; target != null; target = target.next) {
                resetCancelNextUpFlag(target.child);
                // 给所有目标view分发取消事件
                dispatchTransformedTouchEvent(event, true, target.child, target.pointerIdBits);
            }
            clearTouchTargets();

            if (syntheticEvent) {
                event.recycle();
            }
        }
    }

    /** 查找指定view对应的TouchTarget */
    private TouchTarget getTouchTarget(@NonNull View child) {
        for (TouchTarget target = mFirstTouchTarget; target != null; target = target.next) {
            if (target.child == child) {
                return target;
            }
        }
        return null;
    }

    /** 添加指定view对应的 TouchTarget 添加到链表，并返回 */
    private TouchTarget addTouchTarget(@NonNull View child, int pointerIdBits) {
        final TouchTarget target = TouchTarget.obtain(child, pointerIdBits);
        target.next = mFirstTouchTarget;
        mFirstTouchTarget = target;
        return target;
    }

	/** 取消指定view对应的TouchTarget，同时发送cancel事件 */
    private void cancelTouchTarget(View view) {
        TouchTarget predecessor = null;
        TouchTarget target = mFirstTouchTarget;
        while (target != null) {
            final TouchTarget next = target.next;
            if (target.child == view) {
                if (predecessor == null) {
                    mFirstTouchTarget = next;
                } else {
                    predecessor.next = next;
                }
                target.recycle();

                final long now = SystemClock.uptimeMillis();
                MotionEvent event = MotionEvent.obtain(now, now,
                        MotionEvent.ACTION_CANCEL, 0.0f, 0.0f, 0);
                event.setSource(InputDevice.SOURCE_TOUCHSCREEN);
                view.dispatchTouchEvent(event);
                event.recycle();
                return;
            }
            predecessor = target;
            target = next;
        }
    }
```



### 事件分发

事件可能分发给子View，也可能分发给自己处理。mFirstTouchTarget 为TouchTarget链表的头节点。

分发流程如下：

1. 如果 mFirstTouchTarget 为null，说明没有找到消费事件的子View，分发给自己。
2. 如果 mFirstTouchTarget 不为null，说明找到了子View消费事件，从链表的头节点开始，将事件依次分发给目标View。

```java
 		
		// 调度触摸目标。
        if (mFirstTouchTarget == null) { // 一是可能事件都被拦截了，二是没有找到要消费事件的子View
            // 没有触摸目标，调用父类的dispatchTouchEvent()函数处理事件
            // 第三个参数传null，此ViewGroup自己消费事件。调用的是父类view的dispatchTouchEvent方法
            handled = dispatchTransformedTouchEvent(ev, canceled, null,
                                                    TouchTarget.ALL_POINTER_IDS);
        } else {
            // 分派事件到触摸目标。如有必要，取消触摸目标。
            TouchTarget predecessor = null;
            TouchTarget target = mFirstTouchTarget;
            // 遍历TouchTarget链表中的所有目标target
            while (target != null) {
                final TouchTarget next = target.next;
                if (alreadyDispatchedToNewTouchTarget && target == newTouchTarget) {
                    // 上面查找的时候已经分发给他了，不再重复分发
                    handled = true;
                } else {
                    // 如果事件被取消或者被拦截，给目标子View分发cancel事件
                    final boolean cancelChild = resetCancelNextUpFlag(target.child)
                        || intercepted;
                    // 分发事件给目标View，如果cancelChild为true，分发CANCEL事件给子View
                    if (dispatchTransformedTouchEvent(ev, cancelChild,
                                                      target.child, target.pointerIdBits)) {
                        handled = true;
                    }
                    if (cancelChild) {
                        if (predecessor == null) {
                            mFirstTouchTarget = next;
                        } else {
                            predecessor.next = next;
                        }
                        // 回收目标View
                        target.recycle();
                        // 进入下一个循环
                        target = next;
                        continue;
                    }
                }
                // 进入下一个循环
                predecessor = target;
                target = next;
            }
        }
```



### dispatchTransformedTouchEvent 分发事件给目标子view

将事件转换为特定子视图的坐标空间，过滤掉不相关的指针 ID（*pointerIdBits，在查找目标子View的时候缓存了*），并在必要时覆盖其操作。

如果 child 为 null，则假定 MotionEvent 将被发送到此 ViewGroup。调用父类的dispatchTouchEvent()，ViewGroup没有重写这个方法。

返回true表示事件已被消费。

不过需要注意一点的就是，event被传递给child的时候将会做相应偏移，如下
final float offsetX = mScrollX - child.mLeft;
final float offsetY = mScrollY - child.mTop;
event.offsetLocation(offsetX, offsetY);
为什么要做偏移呢？ 因为event的getX得到的值是childView到parentView边境的距离，是一个相对值

```java
    private boolean dispatchTransformedTouchEvent(MotionEvent event, boolean cancel,
                                                  View child, int desiredPointerIdBits) {
        final boolean handled;

        // 取消事件是一种特殊情况。不需要执行任何转换或过滤。重要的部分是动作，而不是内容。
        final int oldAction = event.getAction();
        if (cancel || oldAction == MotionEvent.ACTION_CANCEL) {
            event.setAction(MotionEvent.ACTION_CANCEL);
            if (child == null) {
                // 此ViewGroup自己处理。调用父类的方法，也就是View#dispatchTouchEvent(), 进而调用 onTouchEvent()
                handled = super.dispatchTouchEvent(event);
            } else {
                // 调用子View的dispatchTouchEvent()将事件分发给子View
                handled = child.dispatchTouchEvent(event);
            }
            // 防止event事件被修改，因为还要分发给其他的目标view
            event.setAction(oldAction);
            // 分发完成，返回处理结果
            return handled;
        }

        // 计算要分发的指针数量。
        final int oldPointerIdBits = event.getPointerIdBits();
        final int newPointerIdBits = oldPointerIdBits & desiredPointerIdBits;

        // 和传入的参数不一致，可能出错了，返回
        if (newPointerIdBits == 0) {
            return false;
        }

        // 如果指针的数量相同并且我们不需要执行任何花哨的不可逆变换，
        // 那么只要我们小心地还原我们所做的任何更改，我们就可以为这次调度重用运动事件。
        // 否则我们需要复制一份。
        final MotionEvent transformedEvent;
        if (newPointerIdBits == oldPointerIdBits) {
            if (child == null || child.hasIdentityMatrix()) {
                if (child == null) {	// 此ViewGroup自己处理
                    // 调用父类的方法，也就是View#dispatchTouchEvent(), 进而调用 onTouchEvent()
                    handled = super.dispatchTouchEvent(event);
                } else {
                    final float offsetX = mScrollX - child.mLeft;
                    final float offsetY = mScrollY - child.mTop;
                    event.offsetLocation(offsetX, offsetY);
					// 给子View分发事件
                    handled = child.dispatchTouchEvent(event);
                    event.offsetLocation(-offsetX, -offsetY);
                }
                return handled;
            }
            transformedEvent = MotionEvent.obtain(event);
        } else {
            transformedEvent = event.split(newPointerIdBits);
        }

        // 执行必要的转换并分发事件。
        if (child == null) { // 此ViewGroup自己处理
            // 调用父类的方法，也就是View#dispatchTouchEvent(), 进而调用 onTouchEvent()
            handled = super.dispatchTouchEvent(transformedEvent);
        } else {
            final float offsetX = mScrollX - child.mLeft;
            final float offsetY = mScrollY - child.mTop;
            transformedEvent.offsetLocation(offsetX, offsetY);
            if (! child.hasIdentityMatrix()) {
                transformedEvent.transform(child.getInverseMatrix());
            }
            // 给子View分发转换后的事件
            handled = child.dispatchTouchEvent(transformedEvent);
        }

        // Done.
        transformedEvent.recycle();
        return handled;
    }
```

ViewGroup没有重写 onTouchEvent() 方法，在调用 super.dispatchTouchEvent(transformedEvent) 的时候，也就是View#dispatchTouchEvent(),  进而调用 onTouchEvent()。



### onInterceptTouchEvent 拦截事件

事件将按以下顺序接收：

1. 在此处接收down事件。

2. down 事件要么由此ViewGroup的子View处理，要么交给ViewGroup自己的 onTouchEvent() 方法来处理；

   如果ViewGroup的 onTouchEvent() 以返回 true，此手势后续的事件都由此ViewGroup处理。此外， onTouchEvent() 返回 true，将不会在 onInterceptTouchEvent() 中收到任何后续事件，并且所有触摸处理都必须像平常一样在 onTouchEvent() 中进行。

3. 只要从此函数返回 false，每个后续事件（包括最终事件）将首先传递到此处，然后传递到目标子View的 onTouchEvent()。

4. 如果此函数返回 true，将不会收到任何以下事件：目标子View将收到相同的手势事件，但是是MotionEvent#ACTION_CANCEL事件，并且此手势所有后续的事件将被传递到此 onTouchEvent() 方法，并且不再传递到此函数onInterceptTouchEvent()。

```java
    /**
     * 实现此方法以拦截所有触摸事件。这允许在事件发送给子View时监听事件，并随时掌握当前手势事件的所有权。
     * @param ev 沿层次结构向下调度的运动事件。
     * @return Return true to steal motion events from the children and have
     */
    public boolean onInterceptTouchEvent(MotionEvent ev) {
        //1、判断是否是鼠标设备操作
        //2、ACTION_DOWN事件
        //3、是否是首要按钮按下，如鼠标左键
        //4、是否是滑动条     
        //看意思以上是兼容安卓非手机设备用的判断，满足的话，就返回true，拦截事件分发        
        if (ev.isFromSource(InputDevice.SOURCE_MOUSE)
            && ev.getAction() == MotionEvent.ACTION_DOWN
            && ev.isButtonPressed(MotionEvent.BUTTON_PRIMARY)
            && isOnScrollbarThumb(ev.getX(), ev.getY())) {
            return true;
        }
        return false;
    }
```

