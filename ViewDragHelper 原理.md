ViewDragHelper 原理
====




## 使用流程

1. 父容器在onInterceptTouchEvent() 中调用ViewDragHelper#shouldInterceptTouchEvent()，该方法中会根据触摸事件的坐标来判断落在哪个子控件范围内，然后调用tryCaptureViewForDrag() 方法尝试捕获这个子控件，如果这个方法返回true，那么就表示要拖动这个子控件，意味着需要对事件进行拦截，此时ViewDragHelper#shouldInterceptTouchEvent() 就会返回true  
注：也可以不通过ViewDragHelper#shouldInterceptTouchEvent() 来找到要capture 的子控件，而是直接调用ViewDragHelper#captureChildView() 方法，让ViewDragHelper capture 指定的子控件
2. ViewDragHelper#tryCaptureViewForDrag() 会调用Callback#tryCaptureView() 方法，由父容器来反馈这个子控件是否可以被捕获（以进行拖动）
3. 由于父容器onInterceptTouchEvent() 返回true，之后会由父容器的onTouchEvent() 方法处理后续的触摸事件，此时在onTouchEvent() 中，需要调用ViewDragHelper#processTouchEvent() 来处理触摸事件，以对捕获的子控件修改定位，达到拖动的效果


shouldInterceptTouchEvent() 是否拦截触摸事件的基本逻辑：
1. ACTION_DOWN 一般不会拦截，除非事件落在上一次capture 的子控件范围内，且当前的滑动状态是STATE_SETTLING（一般是正在fling）
2. 触摸事件是否在ViewDragHelper.Callback 限定的范围，如果不在范围内，是不需要进行拦截的
3. Callback#tryCaptureView() 是否返回true（即是否能找到可以被capture 的子控件）  
   根据事件坐标来查找需要capture 的子控件时，是倒序查找。当坐标位置存在控件叠加的时候，最靠后（源码注释中用到的词topmost）的子控件会优先被capture，如果不能被capture（Callback#tryCaptureView(child) 返回false），会继续向前查找


## 实现原理


实现原理：
```java
// ViewDragHelper.java
public class ViewDragHelper {
    ....

    public void cancel() {
        mActivePointerId = INVALID_POINTER;
        clearMotionHistory();

        if (mVelocityTracker != null) {
            mVelocityTracker.recycle();
            mVelocityTracker = null;
        }
    }

    // 是否拦截触摸事件，取决于这个方法的返回
    boolean tryCaptureViewForDrag(View toCapture, int pointerId) {
        if (toCapture == mCapturedView && mActivePointerId == pointerId) {
            // Already done!
            return true;
        }
        // 实际取决于Callback#tryCaptureView() 方法
        if (toCapture != null && mCallback.tryCaptureView(toCapture, pointerId)) {
            mActivePointerId = pointerId;
            captureChildView(toCapture, pointerId);
            return true;
        }
        return false;
    }
    
    // 外部也可以调用这个方法对指定的子控件进行capture
    public void captureChildView(@NonNull View childView, int activePointerId) {
        if (childView.getParent() != mParentView) {
            throw new IllegalArgumentException("captureChildView: parameter must be a descendant "
                    + "of the ViewDragHelper's tracked parent view (" + mParentView + ")");
        }

        mCapturedView = childView;
        mActivePointerId = activePointerId;
        mCallback.onViewCaptured(childView, activePointerId);
        setDragState(STATE_DRAGGING);   // capture 子控件之后，将状态更新为STATE_DRAGGING
    }

    void setDragState(int state) {
        mParentView.removeCallbacks(mSetIdleRunnable);
        if (mDragState != state) {
            mDragState = state;
            mCallback.onViewDragStateChanged(state);
            if (mDragState == STATE_IDLE) {
                mCapturedView = null;
            }
        }
    }

    public boolean shouldInterceptTouchEvent(@NonNull MotionEvent ev) {
        final int action = ev.getActionMasked();
        final int actionIndex = ev.getActionIndex();

        if (action == MotionEvent.ACTION_DOWN) {
            cancel();
        }

        if (mVelocityTracker == null) {
            mVelocityTracker = VelocityTracker.obtain();
        }
        mVelocityTracker.addMovement(ev);

        switch (action) {
            case MotionEvent.ACTION_DOWN: {
                final float x = ev.getX();
                final float y = ev.getY();
                final int pointerId = ev.getPointerId(0);
                saveInitialMotion(x, y, pointerId);

                final View toCapture = findTopChildUnder((int) x, (int) y);

                // ACTION_DOWN 事件一般不会拦截
                if (toCapture == mCapturedView && mDragState == STATE_SETTLING) {
                    tryCaptureViewForDrag(toCapture, pointerId);
                }

                ....
                break;
            }

            case MotionEvent.ACTION_POINTER_DOWN: {
                ....
                break;
            }

            case MotionEvent.ACTION_MOVE: {
                if (mInitialMotionX == null || mInitialMotionY == null) break;

                // First to cross a touch slop over a draggable view wins. Also report edge drags.
                final int pointerCount = ev.getPointerCount();
                for (int i = 0; i < pointerCount; i++) {
                    final int pointerId = ev.getPointerId(i);

                    // If pointer is invalid then skip the ACTION_MOVE.
                    if (!isValidPointerForActionMove(pointerId)) continue;

                    final float x = ev.getX(i);
                    final float y = ev.getY(i);
                    final float dx = x - mInitialMotionX[pointerId];
                    final float dy = y - mInitialMotionY[pointerId];

                    final View toCapture = findTopChildUnder((int) x, (int) y);
                    final boolean pastSlop = toCapture != null && checkTouchSlop(toCapture, dx, dy);
                    if (pastSlop) {
                        // 根据触摸事件移动的距离，结合Callback#clampXxx() 方法对drag 的实际距离进行校正，
                        // 再根据Callback#getViewXxxRange() 方法判断校正后的距离是否在范围内，如果超出范围，
                        // 视为无效操作，不需要处理
                        final int oldLeft = toCapture.getLeft();
                        final int targetLeft = oldLeft + (int) dx;
                        final int newLeft = mCallback.clampViewPositionHorizontal(toCapture,
                                targetLeft, (int) dx);
                        final int oldTop = toCapture.getTop();
                        final int targetTop = oldTop + (int) dy;
                        final int newTop = mCallback.clampViewPositionVertical(toCapture, targetTop,
                                (int) dy);
                        final int hDragRange = mCallback.getViewHorizontalDragRange(toCapture);
                        final int vDragRange = mCallback.getViewVerticalDragRange(toCapture);
                        if ((hDragRange == 0 || (hDragRange > 0 && newLeft == oldLeft))
                                && (vDragRange == 0 || (vDragRange > 0 && newTop == oldTop))) {
                            // 超出范围，不需要拦截
                            break;
                        }
                    }
                    reportNewEdgeDrags(dx, dy, pointerId);
                    if (mDragState == STATE_DRAGGING) {
                        // Callback might have started an edge drag
                        break;
                    }

                    // 判断是否能够capture
                    if (pastSlop && tryCaptureViewForDrag(toCapture, pointerId)) {
                        break;
                    }
                }
                saveLastMotion(ev);
                break;
            }

            ....

            case MotionEvent.ACTION_UP:
            case MotionEvent.ACTION_CANCEL: {
                cancel();
                break;
            }
        }

        // 当tryCaptureViewForDrag() 返回true，在捕获的过程中，会将mDragState 切换为STATE_DRAGGING
        return mDragState == STATE_DRAGGING;
    }

    public void processTouchEvent(@NonNull MotionEvent ev) {
        ....

        switch (action) {
            case MotionEvent.ACTION_MOVE: {
                if (mDragState == STATE_DRAGGING) {
                    // If pointer is invalid then skip the ACTION_MOVE.
                    if (!isValidPointerForActionMove(mActivePointerId)) break;

                    final int index = ev.findPointerIndex(mActivePointerId);
                    final float x = ev.getX(index);
                    final float y = ev.getY(index);
                    final int idx = (int) (x - mLastMotionX[mActivePointerId]);
                    final int idy = (int) (y - mLastMotionY[mActivePointerId]);

                    // 通过 ViewCompat.offsetXxx() 方法修改子控件的位置
                    dragTo(mCapturedView.getLeft() + idx, mCapturedView.getTop() + idy, idx, idy);

                    saveLastMotion(ev);
                } else {
                    // 如果之前没有capture 子控件，这里会再次尝试capture，
                    // 和shouldInterceptTouchEvent() 中的逻辑基本一致
                    ....
                }
            }
        }
    }

    ....
}
```

