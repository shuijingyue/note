dispatchLayoutStep1
dispatchLayoutStep2

```java
// RecyclerView
protected void onMeasure(int widthSpec, int heightSpec) {
    if (mLayout == null) {
        defaultOnMeasure(widthSpec, heightSpec);
        return;
    }
    if (mLayout.isAutoMeasureEnabled()) {
        final int widthMode = MeasureSpec.getMode(widthSpec);
        final int heightMode = MeasureSpec.getMode(heightSpec);

        /**
         * 此处应该用defaultMesure代替,但是如果真的改成了defaultMesure会影响现存的三方应用
         * 并且文档也指出当isAutoMeasureEnabled返回true时,不要重写LayoutManager的onMeasure方法
         *    
         * 即此处也调用了defaultMeasure
         */
        mLayout.onMeasure(mRecycler, mState, widthSpec, heightSpec);

        final boolean measureSpecModeIsExactly =
                widthMode == MeasureSpec.EXACTLY && heightMode == MeasureSpec.EXACTLY;
        // setLayoutManager和setAdapter都会requestLayout
        // 如果setLayoutManager之前没有setAdapter,onMeasure至此结束    
        if (measureSpecModeIsExactly || mAdapter == null) {
            return;
        }

        // 以下逻辑只有adapter和LayoutManager均已设置后才会执行

        // State在RecyclerView创建时初始化
        // State.mLayoutStep 初始值State.STEP_START
        // State.mLayoutStep 只会在step123和prefetch时更改
        if (mState.mLayoutStep == State.STEP_START) {
            dispatchLayoutStep1();
            // mState.mLayoutStep = State.STEP_LAYOUT;
        }
        // set dimensions in 2nd step. Pre-layout should happen with old dimensions for
        // consistency
        mLayout.setMeasureSpecs(widthSpec, heightSpec);
        mState.mIsMeasuring = true;
        // mState.assertLayoutStep(State.STEP_LAYOUT | State.STEP_ANIMATIONS);
        dispatchLayoutStep2();
        // mLayout.onLayoutChildren
        // mState.mLayoutStep = State.STEP_ANIMATIONS;

        // now we can get the width and height from the children.
        mLayout.setMeasuredDimensionFromChildren(widthSpec, heightSpec);

        // if RecyclerView has non-exact width and height and if there is at least one child
        // which also has non-exact width & height, we have to re-measure.
        if (mLayout.shouldMeasureTwice()) {
            mLayout.setMeasureSpecs(
                    MeasureSpec.makeMeasureSpec(getMeasuredWidth(), MeasureSpec.EXACTLY),
                    MeasureSpec.makeMeasureSpec(getMeasuredHeight(), MeasureSpec.EXACTLY));
            mState.mIsMeasuring = true;
            dispatchLayoutStep2();
            // now we can get the width and height from the children.
            mLayout.setMeasuredDimensionFromChildren(widthSpec, heightSpec);
        }
    } else {
        if (mHasFixedSize) {
            mLayout.onMeasure(mRecycler, mState, widthSpec, heightSpec);
            return;
        }
        // custom onMeasure
        if (mAdapterUpdateDuringMeasure) {
            startInterceptRequestLayout();
            onEnterLayoutOrScroll();
            processAdapterUpdatesAndSetAnimationFlags();
            onExitLayoutOrScroll();

            if (mState.mRunPredictiveAnimations) {
                mState.mInPreLayout = true;
            } else {
                // consume remaining updates to provide a consistent state with the layout pass.
                mAdapterHelper.consumeUpdatesInOnePass();
                mState.mInPreLayout = false;
            }
            mAdapterUpdateDuringMeasure = false;
            stopInterceptRequestLayout(false);
        } else if (mState.mRunPredictiveAnimations) {
            // If mAdapterUpdateDuringMeasure is false and mRunPredictiveAnimations is true:
            // this means there is already an onMeasure() call performed to handle the pending
            // adapter change, two onMeasure() calls can happen if RV is a child of LinearLayout
            // with layout_width=MATCH_PARENT. RV cannot call LM.onMeasure() second time
            // because getViewForPosition() will crash when LM uses a child to measure.
            setMeasuredDimension(getMeasuredWidth(), getMeasuredHeight());
            return;
        }

        if (mAdapter != null) {
            mState.mItemCount = mAdapter.getItemCount();
        } else {
            mState.mItemCount = 0;
        }
        startInterceptRequestLayout();
        mLayout.onMeasure(mRecycler, mState, widthSpec, heightSpec);
        stopInterceptRequestLayout(false);
        mState.mInPreLayout = false; // clear
    }
}
```

默认的measure过程

```java
// RecyclerView
void defaultOnMeasure(int widthSpec, int heightSpec) {
    // calling LayoutManager here is not pretty but that API is already public and it is better
    // than creating another method since this is internal.
    //
    // ViewCompat.getMinimumWidth 
    // >= 16 View.getMinimumWidth
    // else 通过反射取mMinWidth                           
    final int width = LayoutManager.chooseSize(widthSpec,
            /* desired */ getPaddingLeft() + getPaddingRight(),
            /* min */ ViewCompat.getMinimumWidth(this));
    final int height = LayoutManager.chooseSize(heightSpec,
            getPaddingTop() + getPaddingBottom(),
            ViewCompat.getMinimumHeight(this));

    setMeasuredDimension(width, height);
}

// RecyclerView.LayoutManager
public static int chooseSize(int spec, int desired, int min) {
    final int mode = View.MeasureSpec.getMode(spec);
    final int size = View.MeasureSpec.getSize(spec);
    switch (mode) {
        case View.MeasureSpec.EXACTLY:
            return size;
        case View.MeasureSpec.AT_MOST:
            return Math.min(size, Math.max(desired, min));
        case View.MeasureSpec.UNSPECIFIED:
        default:
            return Math.max(desired, min);
    }
}

// RecyclerView.LayoutManager
public void onMeasure(@NonNull Recycler recycler, @NonNull State state, int widthSpec,
        int heightSpec) {
    mRecyclerView.defaultOnMeasure(widthSpec, heightSpec);
}
```

如果`isAutoMeasureEnabled`方法返回true,官方文档是不建议重写`LayoutManager#onMeasure`

```java

public static class ItemHolderInfo {
    public int left;
    public int top;
    public int right;
    public int bottom;
}

/**
 * The first step of a layout where we;
 * - 处理adapter的更新
 * - 决定执行哪个动画
 * - 保存当前view的信息
 * - 如果需要,执行predictive layout并保存相关的信息
 */
private void dispatchLayoutStep1() {
    mState.assertLayoutStep(State.STEP_START);
    fillRemainingScrollValues(mState);
    mState.mIsMeasuring = false;
    startInterceptRequestLayout();
    // mViewInfoStore保存动画相关信息
    mViewInfoStore.clear();
    onEnterLayoutOrScroll();
    // 消费add/update/remove/move op, 仅更新了viewHolder的状态,并没有实际更新view
    // 设置mState.mRunSimpleAnimations和mState.mRunPredictiveAnimations
    processAdapterUpdatesAndSetAnimationFlags(); 
    saveFocusInfo();
    mState.mTrackOldChangeHolders = mState.mRunSimpleAnimations && mItemsChanged;
    mItemsAddedOrRemoved = mItemsChanged = false;
    mState.mInPreLayout = mState.mRunPredictiveAnimations;
    mState.mItemCount = mAdapter.getItemCount();
    findMinMaxChildLayoutPositions(mMinMaxLayoutPositions);
    // 记录信息
    if (mState.mRunSimpleAnimations) {
        // Step 0: Find out where all non-removed items are, pre-layout
        int count = mChildHelper.getChildCount(); // ViewGroup.getChildCount - mHiddenViews.size
        for (int i = 0; i < count; ++i) {
            // mChildHelper.getChildAt ViewGroup.getChildAt
            // ((LayoutParams) child.getLayoutParams()).mViewHolder;
            final ViewHolder holder = getChildViewHolderInt(mChildHelper.getChildAt(i));

            // 通过LayoutManager#ignoreView给vh添加ignore标记
            // rv框架中没有调用的位置,是留给开发者的接口
            if (holder.shouldIgnore() || (holder.isInvalid() && !mAdapter.hasStableIds())) {
                continue;
            }
            // 记录itemView的left top right bottom
            // step1 recordPreLayoutInformation
            // step3 recordPostLayoutInformation
            final ItemHolderInfo animationInfo = mItemAnimator
                    .recordPreLayoutInformation(mState, holder,
                            ItemAnimator.buildAdapterChangeFlagsForAnimations(holder),
                            holder.getUnmodifiedPayloads());
            // 将animationInfo记录到mViewInfoStore.mLayoutHolderMap   
            mViewInfoStore.addToPreLayout(holder, animationInfo);
            if (mState.mTrackOldChangeHolders && holder.isUpdated() && !holder.isRemoved()
                    && !holder.shouldIgnore() && !holder.isInvalid()) {
                long key = getChangedHolderKey(holder);
                // This is NOT the only place where a ViewHolder is added to old change holders
                // list. There is another case where:
                //    * A VH is currently hidden but not deleted
                //    * The hidden item is changed in the adapter
                //    * Layout manager decides to layout the item in the pre-Layout pass (step1)
                // When this case is detected, RV will un-hide that view and add to the old
                // change holders list.
                //
                // 加入 mViewInfoStore.mOldChangedHolders
                mViewInfoStore.addToOldChangeHolders(key, holder);
            }
        }
    }
    if (mState.mRunPredictiveAnimations) {
        // Step 1: run prelayout: This will use the old positions of items. The layout manager
        // is expected to layout everything, even removed items (though not to add removed
        // items back to the container). This gives the pre-layout position of APPEARING views
        // which come into existence as part of the real layout.

        // Save old positions so that LayoutManager can run its mapping logic.
        // 所有的view,包括mHiddenViews holder.mOldPosition = mPosition;
        saveOldPositions();
        final boolean didStructureChange = mState.mStructureChanged;
        mState.mStructureChanged = false;
        // temporarily disable flag because we are asking for previous layout
        // 关键调用, step1被称为pre layout的原因所在
        mLayout.onLayoutChildren(mRecycler, mState); 
        mState.mStructureChanged = didStructureChange;

        for (int i = 0; i < mChildHelper.getChildCount(); ++i) {
            final View child = mChildHelper.getChildAt(i);
            final ViewHolder viewHolder = getChildViewHolderInt(child);
            if (viewHolder.shouldIgnore()) {
                continue;
            }
      
            if (!mViewInfoStore.isInPreLayout(viewHolder)) {
                int flags = ItemAnimator.buildAdapterChangeFlagsForAnimations(viewHolder);
                boolean wasHidden = viewHolder
                        .hasAnyOfTheFlags(ViewHolder.FLAG_BOUNCED_FROM_HIDDEN_LIST);
                if (!wasHidden) {
                    flags |= ItemAnimator.FLAG_APPEARED_IN_PRE_LAYOUT;
                }
                final ItemHolderInfo animationInfo = mItemAnimator.recordPreLayoutInformation(
                        mState, viewHolder, flags, viewHolder.getUnmodifiedPayloads());
                if (wasHidden) {
                    recordAnimationInfoIfBouncedHiddenView(viewHolder, animationInfo);
                } else {
                    mViewInfoStore.addToAppearedInPreLayoutHolders(viewHolder, animationInfo);
                }
            }
        }
        // we don't process disappearing list because they may re-appear in post layout pass.
        clearOldPositions();
    } else {
        clearOldPositions();
    }
    onExitLayoutOrScroll();
    stopInterceptRequestLayout(false);
    mState.mLayoutStep = State.STEP_LAYOUT;
}
```

```java
private void processAdapterUpdatesAndSetAnimationFlags() {
    // setAdapter或者notifyDataSetChanged
    // 会将mDataSetHasChangedAfterLayout设置为true
    if (mDataSetHasChangedAfterLayout) {
        // Processing these items have no value since data set changed unexpectedly.
        // Instead, we just reset it.
        // mPendingUpdates
        // mPostponedList
        mAdapterHelper.reset();
        // setAdapter false
        // notifyDataSetChanged true
        if (mDispatchItemsChangedEvent) {
            mLayout.onItemsChanged(this);
        }
    }
    // simple animations are a subset of advanced animations (which will cause a
    // pre-layout step)
    // If layout supports predictive animations, pre-process to decide if we want to run them
    // mPendingUpdates
    // mPostponedList
    if (predictiveItemAnimationsEnabled()) {
        mAdapterHelper.preProcess(); // 消费mPendingUpdates,将op加入mPostponedList
    } else {
        mAdapterHelper.consumeUpdatesInOnePass(); // 消费mPostponedList和mPendingUpdates
    }
    // add/remove/move会将mItemsAddedOrRemoved设置为true 
    // change会将mItemsChanged设置为true
    boolean animationTypeSupported = mItemsAddedOrRemoved || mItemsChanged;
    // mLayout.mRequestedSimpleAnimations 通过 requestSimpleAnimationsInNextLayout
    // 设置为true
    // 当adapter的hasStableIds返回true时,调用notifyDataSetChanged时
    // 会将mState.mRunSimpleAnimations设置为true,此时mState.mRunPredictiveAnimations == false
    //
    // notifyDataSetChanged会导致mDataSetHasChangedAfterLayout = true
    // 而mDataSetHasChangedAfterLayout为true后,在执行dispatchLayoutStep1时,将pendingUpdates清空
    // 此时mItemsAddedOrRemoved和mItemsChanged都为false
    // animationTypeSupported为false
    mState.mRunSimpleAnimations = mFirstLayoutComplete
            && mItemAnimator != null
            && (mDataSetHasChangedAfterLayout
            || animationTypeSupported
            || mLayout.mRequestedSimpleAnimations)
            && (!mDataSetHasChangedAfterLayout
            || mAdapter.hasStableIds());

    // mRunPredictiveAnimations为true的前提是mRunSimpleAnimations为true
    mState.mRunPredictiveAnimations = mState.mRunSimpleAnimations
            && animationTypeSupported
            && !mDataSetHasChangedAfterLayout
            && predictiveItemAnimationsEnabled();
}
```

```java
// AdapterHelper.java
 void preProcess() {
     // 将所有的UpdateOp.MOVE放到最后处理
    mOpReorderer.reorderOps(mPendingUpdates);
    final int count = mPendingUpdates.size();
    for (int i = 0; i < count; i++) {
        UpdateOp op = mPendingUpdates.get(i);
        switch (op.cmd) {
            // insert
            case UpdateOp.ADD:
                applyAdd(op);
                break;
            // remove    
            case UpdateOp.REMOVE:
                applyRemove(op);
                break;
            // change    
            case UpdateOp.UPDATE:
                applyUpdate(op);
                break;
            // move    
            case UpdateOp.MOVE:
                applyMove(op);
                break;
        }
        if (mOnItemProcessedCallback != null) {
            mOnItemProcessedCallback.run();
        }
    }
    mPendingUpdates.clear();
}

private void postponeAndUpdateViewHolders(UpdateOp op) {
    if (DEBUG) {
        Log.d(TAG, "postponing " + op);
    }
    mPostponedList.add(op);
    switch (op.cmd) {
        case UpdateOp.ADD:
            mCallback.offsetPositionsForAdd(op.positionStart, op.itemCount);
            break;
        case UpdateOp.MOVE:
            mCallback.offsetPositionsForMove(op.positionStart, op.itemCount);
            break;
        case UpdateOp.REMOVE:
            mCallback.offsetPositionsForRemovingLaidOutOrNewView(op.positionStart,
                    op.itemCount);
            break;
        case UpdateOp.UPDATE:
            mCallback.markViewHoldersUpdated(op.positionStart, op.itemCount, op.payload);
            break;
        default:
            throw new IllegalArgumentException("Unknown update op type for " + op);
    }
}

 mAdapterHelper = new AdapterHelper(new AdapterHelper.Callback() {
   ...

    @Override
    public void offsetPositionsForRemovingInvisible(int start, int count) {
        offsetPositionRecordsForRemove(start, count, true);
        mItemsAddedOrRemoved = true;
        mState.mDeletedInvisibleItemCountSincePreviousLayout += count;
    }

    @Override
    public void offsetPositionsForRemovingLaidOutOrNewView(
            int positionStart, int itemCount) {
        offsetPositionRecordsForRemove(positionStart, itemCount, false);
        mItemsAddedOrRemoved = true;
    }


    @Override
    public void markViewHoldersUpdated(int positionStart, int itemCount, Object payload) {
        viewRangeUpdate(positionStart, itemCount, payload);
        mItemsChanged = true;
    }

    // ...

    @Override
    public void offsetPositionsForAdd(int positionStart, int itemCount) {
        offsetPositionRecordsForInsert(positionStart, itemCount);
        mItemsAddedOrRemoved = true;
    }

    @Override
    public void offsetPositionsForMove(int from, int to) {
        offsetPositionRecordsForMove(from, to);
        // should we create mItemsMoved ?
        mItemsAddedOrRemoved = true;
    }
});
```

```java
// step2做的事情相对简单
// 1. 消费mPendingUpdates和mPostponedList内存储的所有的op
// 2. 调用mLayout.onLayoutChildren
private void dispatchLayoutStep2() {
    startInterceptRequestLayout();
    onEnterLayoutOrScroll();
    mState.assertLayoutStep(State.STEP_LAYOUT | State.STEP_ANIMATIONS);
    // 消费mPendingUpdates和mPostponedList内存储的所有的op
    mAdapterHelper.consumeUpdatesInOnePass();
    mState.mItemCount = mAdapter.getItemCount();
    mState.mDeletedInvisibleItemCountSincePreviousLayout = 0;

    // Step 2: Run layout
    mState.mInPreLayout = false;
    mLayout.onLayoutChildren(mRecycler, mState);

    mState.mStructureChanged = false;
    mPendingSavedState = null;

    // onLayoutChildren may have caused client code to disable item animations; re-check
    mState.mRunSimpleAnimations = mState.mRunSimpleAnimations && mItemAnimator != null;
    mState.mLayoutStep = State.STEP_ANIMATIONS;
    onExitLayoutOrScroll();
    stopInterceptRequestLayout(false);
}