Bottom Sheet
====




## 实现原理

基本原理：  
1. 如果以behavior 绑定的view 为root 的view tree 存在支持嵌套滑动的子级控件，且触摸事件落在子级控件中，behavior 不会对事件进行拦截，会通过嵌套滑动的协调机制来对绑定的view 进行drag
2. 否则，通过ViewDragHelper 来对绑定的view 进行drag（触摸事件拦截 + ViewCompat#offsetTopAndBottom()）

源码解析：
```java
public class BottomSheetBehavior<V extends View> extends CoordinatorLayout.Behavior<V>
    implements MaterialBackHandler {

    int state = STATE_COLLAPSED;

    ViewDragHelper viewDragHelper;

    // 保存behavior 所绑定的view（CoordinatorLayout的子控件）
    WeakReference<V> viewRef;

    // 保存以behavior 所绑定的view为root 的view tree中，支持嵌套滑动的子控件
    WeakReference<View> nestedScrollingChildRef;

    private final ViewDragHelper.Callback dragCallback = new ViewDragHelper.Callback() {

        private long viewCapturedMillis;

        // 这个方法会影响ViewDragHelper#shouldInterceptTouchEvent() 的返回，从而决定是否要对触摸事件进行拦截
        @Override
        public boolean tryCaptureView(@NonNull View child, int pointerId) {
            if (state == STATE_DRAGGING) {
                return false;
            }
            // 存在支持嵌套滑动的子级控件，且触摸事件落在该子控件中，touchingScrollingChild 才会为true
            if (touchingScrollingChild) {
                return false;
            }
            // 存在支持嵌套滑动的子级控件，且触摸事件落在该子控件中，activePointerId 才不会为INVALID_POINTER_ID
            if (state == STATE_EXPANDED && activePointerId == pointerId) {
                View scroll = nestedScrollingChildRef != null ? nestedScrollingChildRef.get() : null;
                if (scroll != null && scroll.canScrollVertically(-1)) {
                    // Let the content scroll up
                    return false;
                }
            }
            viewCapturedMillis = System.currentTimeMillis();

            // 除了上面的情况，只要behavior 绑定了view，就要对事件进行拦截
            return viewRef != null && viewRef.get() == child;
        }

        @Override
        public void onViewPositionChanged(
                @NonNull View changedView, int left, int top, int dx, int dy) {
            dispatchOnSlide(top);
        }

        @Override
        public void onViewDragStateChanged(@State int state) {
            if (state == ViewDragHelper.STATE_DRAGGING && draggable) {
                setStateInternal(STATE_DRAGGING);
            }
        }

        ....
    };

    @Override
    public boolean onLayoutChild(
            @NonNull CoordinatorLayout parent, @NonNull final V child, int layoutDirection) {
        // 在这个方法中会绑定child 为bottom sheet -- 赋值viewRef
        // 还会调用findScrollingChild() 查找是否存在支持嵌套滑动的子级控件，如果存在，赋值nestedScrollingChildRef
        ....    

        if (viewDragHelper == null) {
            // ViewDragHelper 绑定的是CoordinatorLayout，要捕获并进行拖动的是child 参数（就是behavior 绑定的view）
            viewDragHelper = ViewDragHelper.create(parent, dragCallback);
        }
        ....
    }

    // 在以behavior 绑定的view 为root 的view tree 中，查找是否存在支持嵌套滑动的子级控件
    // 注意这里只要查找到第一个支持嵌套滑动的子级控件，就不再继续查找
    View findScrollingChild(View view) {
        if (view.getVisibility() != View.VISIBLE) {
            return null;
        }
        if (ViewCompat.isNestedScrollingEnabled(view)) {
            return view;
        }
        if (view instanceof ViewGroup) {
            ViewGroup group = (ViewGroup) view;
            for (int i = 0, count = group.getChildCount(); i < count; i++) {
                // 递归调用
                View scrollingChild = findScrollingChild(group.getChildAt(i));
                if (scrollingChild != null) {
                    return scrollingChild;
                }
            }
        }
        return null;
    }

    @Override
    public boolean onInterceptTouchEvent(
            @NonNull CoordinatorLayout parent, @NonNull V child, @NonNull MotionEvent event) {
        if (!child.isShown() || !draggable) {
            ignoreEvents = true;
            return false;
        }
        ....
        switch (action) {
            case MotionEvent.ACTION_UP:
            case MotionEvent.ACTION_CANCEL:
                touchingScrollingChild = false;
                activePointerId = MotionEvent.INVALID_POINTER_ID;
                // Reset the ignore flag
                if (ignoreEvents) {
                    ignoreEvents = false;
                    return false;
                }
                break;
            case MotionEvent.ACTION_DOWN:          // ACTION_DOWN 事件一般不会进行拦截
                int initialX = (int) event.getX();
                initialY = (int) event.getY();
                
                if (state != STATE_SETTLING) {
                    View scroll = nestedScrollingChildRef != null ? nestedScrollingChildRef.get() : null;
                    // 走事件拦截的处理逻辑，还是走嵌套滑动的逻辑，关键就在这里：
                    // 1. 存在支持嵌套滑动的子级控件
                    // 2. ACTION_DOWN 事件位于该子级控件的范围内
                    if (scroll != null && parent.isPointInChildBounds(scroll, initialX, initialY)) {
                        // 这两个变量，只有这里会进行赋值
                        activePointerId = event.getPointerId(event.getActionIndex());
                        touchingScrollingChild = true;
                    }
                }
                ignoreEvents = activePointerId == MotionEvent.INVALID_POINTER_ID
                        && !parent.isPointInChildBounds(child, initialX, initialY);
                break;
            default: // fall out
        }
        if (!ignoreEvents
                && viewDragHelper != null
                // 这个方法的返回值：
                // 1. 如果存在支持嵌套滑动的子级控件，且触摸事件在其范围内，这里会返回false；
                //   这一点由上面的 ViewDragHelper.Callback#tryCaptureView() 返回值决定
                // 2. 在条件1不满足的情况下，如果当前是展开状态，继续向上滑动，这里也会返回false；
                //   这一点是因为上面的 ViewDragHelper.Callback#getViewVerticalDragRange() 方法的
                //   实现对drag 的范围进行了限制
                // 3. 其他情况会返回true
                && viewDragHelper.shouldInterceptTouchEvent(event)) {
            return true;
        }

        // 即使ViewDragHelper 判断不需要对ACTION_MOVE 事件进行拦截，针对非支持嵌套滑动的场景，这里仍然会选择
        // 进行拦截，之后在onTouchEvent() 方法中，会尝试调用viewDragHelper.captureChildView() 来继续进行拖动
        View scroll = nestedScrollingChildRef != null ? nestedScrollingChildRef.get() : null;
        return action == MotionEvent.ACTION_MOVE
                && scroll != null
                && !ignoreEvents
                && state != STATE_DRAGGING
                && !parent.isPointInChildBounds(scroll, (int) event.getX(), (int) event.getY())
                && viewDragHelper != null
                && initialY != INVALID_POSITION
                && Math.abs(initialY - event.getY()) > viewDragHelper.getTouchSlop();
    }
}
```


嵌套滑动的处理  
```java
public void onNestedPreScroll(
        @NonNull CoordinatorLayout coordinatorLayout,
        @NonNull V child,   // child 是BottomSheetBehavior 绑定的view
        @NonNull View target,   // target 是触发嵌套滑动的view
        int dx,
        int dy,
        @NonNull int[] consumed,
        int type) {
    if (type == ViewCompat.TYPE_NON_TOUCH) {
      // Ignore fling here. The ViewDragHelper handles it.
      return;
    }
    View scrollingChild = nestedScrollingChildRef != null ? nestedScrollingChildRef.get() : null;
    // BottomSheetBehavior#isNestedScrollingCheckEnabled()必须返回true，才能响应嵌套滑动
    if (isNestedScrollingCheckEnabled() && target != scrollingChild) {
      return;
    }
    ...
}
```
当BottomSheetBehavior 中记录的可滑动嵌套控件，与当前触发嵌套滑动的view 不一致，会不响应嵌套滑动


## 遇到的问题

1. BottomSheetBehavior 绑定的view 中包含ViewPager2（内部没有可纵向滚动的控件），ViewPager2 可以左右滑动切换子页面，也可以拖动ViewPager2 内部的控件，展开bottom sheet，但是展开之后，无法拖动ViewPager2 内部的控件以收起bottom sheet    
通过调试代码发现，BottomSheetBehavior#onInterceptTouchEvent() 中如果判断view tree 中有ViewCompat.isNestedScrollingEnabled(view) 返回true，也就是说有支持嵌套滑动的子控件（onLayoutChild() -> findScrollingChild()），如果ACTION_MOVE 事件落在这个子控件中，就走嵌套滑动的机制来处理，不对事件进行拦截
所以这里问题原因在于layout 中包含了ViewPager2 这个支持嵌套滑动的子控件  

**如何解决？**  
设置ViewPager2 setNestedScrollingEnabled(false)  
问题在于ViewPager2 自身设置这个是无效的，ViewPager2 继承自ViewGroup，它本身的isNestedScrollingEnabled 默认就是false，它内部有一个RecyclerView，需要设置内部RecyclerView 的isNestedScrollingEnabled 为false：
```kotlin
viewPager.children.find { it is RecyclerView }?.run {
    isNestedScrollingEnabled = false
}
```

2. 接着上面的问题，如果ViewPager2 包含两个Fragment 页面，两个页面的内容都是RecyclerView，会发生什么？  
会出现第一个页面的RecyclerView 内容可以正常滑动，但是第二个页面在behavior 绑定的view 被展开后，RecyclerView 的内容无法被滑动，只能向下滑动，收起behavior绑定的view（表明事件被拦截了？）  

调试源码发现，在第一个页面，当RecyclerView 滑动到顶部的情况下，向下滑动，因为判断内部有支持嵌套滑动的控件，BottomSheetBehavior不会拦截事件，会走嵌套滑动的逻辑，收起bottom sheet（向上滑动，也是走嵌套滑动的逻辑，会滚动RecyclerView 的内容）  
但是在第二个页面，不管是向上，还是向下滑动，都会走事件拦截的逻辑。其根本原因在于，BottomSheetBehavior#findScrollingChild() 找到的是第一个页面的RecyclerView，这个方法是在onLayoutChild() 中调用的，之后ViewPager 切换第二个页面（第一个页面的RecyclerView 还保存在缓存中，behavior 中保存的软引用仍然有效），不会再次触发BottomSheetBehavior#onLayoutChild()，导致保存的嵌套滑动子控件不会更新  
另外，当第一个页面的RecyclerView 被滑出时，它在ViewPager2 中的位置也会改变，所以触摸事件是不可能位于第一个页面的Recyclerview 的范围内的  

```java
public boolean onInterceptTouchEvent(
        @NonNull CoordinatorLayout parent, @NonNull V child, @NonNull MotionEvent event) {
    ....

    if (!ignoreEvents
        && viewDragHelper != null
        // 1. ViewPager2 第二个页面，如果是向下滑动的ACTION_MOVE，这里会返回true
        //    因为事件不在behavior 中保存的第一个页面RecyclerView 的范围内，不能走嵌套滑动的处理逻辑，
        //    导致Callback#tryCaptureView() 会返回true
        // 2. 如果是向上滑动，这里会返回false，因为超出了ViewDragHelper.Callback 所限制的drag 范围
        && viewDragHelper.shouldInterceptTouchEvent(event)) {
      return true;
    }

    // 调试代码时，通过Evaluation express 可以拿到第二个页面的RecyclerView，发现与这里保存的scroll 
    // 不是同一个RecyclerView，证实behavior 中保存的支持嵌套滑动的控件仍然是第一个页面的RecyclerView
    View scroll = nestedScrollingChildRef != null ? nestedScrollingChildRef.get() : null;

    // 第二个页面，向上滑动，这里最终还是返回true，还是会拦截触摸事件
    return action == MotionEvent.ACTION_MOVE
        && scroll != null
        && !ignoreEvents
        && state != STATE_DRAGGING
        // 这里判断事件不在支持嵌套滑动的子控件（第一个页面的RecyclerView）的范围内
        && !parent.isPointInChildBounds(scroll, (int) event.getX(), (int) event.getY())
        && viewDragHelper != null
        && initialY != INVALID_POSITION
        && Math.abs(initialY - event.getY()) > viewDragHelper.getTouchSlop();
}
```

**解决思路**：在切换ViewPager2 页面时，更新BottomSheetBehavior 中记录的支持嵌套滑动子控件  
1. 切换ViewPager2 页面后，人为调用一次BottomSheetBehavior 的onLayoutChild() 方法  
ViewPager2.OnPageChangeCallback#onPageSelected() 调用时机过早，实际的页面切换还没有完成  
可以这样做：
```kotlin
viewPager.registerOnPageChangeCallback(object : ViewPager2.OnPageChangeCallback() {
    private var lastPage = currentItem

    override fun onPageScrollStateChanged(state: Int) {
        if (state == SCROLL_STATE_IDLE) {
            if (currentItem != lastPage) {
                Log.i(TAG, "onPageScrollStateChanged: $currentItem")
                lastPage = currentItem
                bottomSheetBehavior.onLayoutChild(coordinatorLayout, bottomSheet, View.LAYOUT_DIRECTION_LTR )
            }
        }
    }
})
```

2. 通过反射BottomSheetBehavior 进行更新



