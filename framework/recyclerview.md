缓存原理
优化

RecyclerView
Adapter
ViewHolder
Recycler
LayoutManager
ChildHelper
ViewFlinger
SmoothScroller
ViewInfoStore
ProcessCallback
ItemAnimator

dispatchLayoutStep1
dispatchLayoutStep2

dispatchLayoutStep3
    dispatchLayout

dispatchLayout
    consumePendingUpdateOperations
    stopInterceptRequestLayout
        removeAnimatingView // 动画结束时
        scrollStep
            scrollByInternal
                scrollBy
                nestedScrollByInternal
                    nestedScrollBy
                    onGenericMotionEvent
                MotionEvent.ACTION_MOVE
            ViewFlinger#run
            SmoothScroller#onAnimation
                ViewFlinger#run
        consumePendingUpdateOperations
        focusSearch
        onMeasure
        dispatchLayoutStep1
        dispatchLayoutStep2
        dispatchLayoutStep3
    onLayout

处理`adapter`更新
判断需要执行哪些动画
记录当前view的信息
判断是否需要执行预先布局并记录相关信息

## notifyXXX

RecyclerView.notifyXXX是异步的
```java
ViewCompat.postOnAnimation(RecyclerView.this, mUpdateChildViewsRunnable);
```

Adapter有一个mObservable(AdapterDataObservable)
拓展,单一职责

Adapter.registerObserver -> mObservable.registerObserver

notifyXXX之后实际操作的入口

RecyclerView.mObserver(RecyclerViewDataObserver)

## layout

```java
static final int STEP_START = 1;           // 001
static final int STEP_LAYOUT = 1 << 1;     // 010
static final int STEP_ANIMATIONS = 1 << 2; // 100
```

preLayout  dispatchLayoutStep1
layout     dispatchLayoutStep2
postLayout dispatchLayoutStep3

LinearLayoutManager#onLayoutChildren

计算锚点信息 AnchorInfo

```java
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
        if (measureSpecModeIsExactly || mAdapter == null) {
            return;
        }

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

```java
static class AnchorInfo {
    OrientationHelper mOrientationHelper;
    int mPosition;
    int mCoordinate;
    boolean mLayoutFromEnd;
    boolean mValid;
}
```

detachAndScrapAttachedViews


### detail

mPendingSavedState onRestoreInstanceState中赋值

ensureLayoutState mLayoutState = new LayoutState()

mAnchorInfo

## reuse and recycle

- mAttachedScrap和mChangedScrap
- mCachedViews
- ViewCacheExtension
- RecyclerViewPool

## init

`setLayoutManager` & `setAdapter` 都会触发requestLayout

setItemViewCacheSize -> mRecycler.setViewCacheSize(size) -> mRequestedCacheMax = viewCount
-> updateViewCacheSize() -> int extraCache = mLayout != null ? mLayout.mPrefetchMaxCountObserved : 0;
            mViewCacheMax = mRequestedCacheMax + extraCache;


## onTouch
RecyclerView.onTouch -> LayoutManager.scrollHorizontallyBy / scrollVerticallyBy
-> scrollBy -> fill

```java
case MotionEvent.ACTION_MOVE:
    if (mScrollState == SCROLL_STATE_DRAGGING) {
        ...
        if (scrollByInternal(
                canScrollHorizontally ? dx : 0,
                canScrollVertically ? dy : 0,
                e)) {
            getParent().requestDisallowInterceptTouchEvent(true);
        }
        ..
    }
```

```java
boolean scrollByInternal(int x, int y, MotionEvent ev) {
    ...
    if (mAdapter != null) {
        ...
        scrollStep(x, y, mReusableIntPair);
        ...
    }
    ...
}
```

```java
void scrollStep(int dx, int dy, @Nullable int[] consumed) {
    ...
    if (dx != 0) {
        consumedX = mLayout.scrollHorizontallyBy(dx, mRecycler, mState);
    }
    if (dy != 0) {
        consumedY = mLayout.scrollVerticallyBy(dy, mRecycler, mState);
    }
    ...
}
```

```java
public int scrollHorizontallyBy(int dx, RecyclerView.Recycler recycler,
        RecyclerView.State state) {
    if (mOrientation == VERTICAL) {
        return 0;
    }
    return scrollBy(dx, recycler, state);
}
```

```java
int scrollBy(int delta, RecyclerView.Recycler recycler, RecyclerView.State state) {
    ...
    final int consumed = mLayoutState.mScrollingOffset
            + fill(recycler, mLayoutState, state, false);
    ...
}
```

## 复用

```java
public final class Recycler {
    final ArrayList<ViewHolder> mAttachedScrap = new ArrayList<>();
    ArrayList<ViewHolder> mChangedScrap = null;
    final ArrayList<ViewHolder> mCachedViews = new ArrayList<ViewHolder>();
}

class ChildHelper {
    final Bucket mBucket;
    final List<View> mHiddenViews;
}
```

LinearLayoutManager

LinearLayoutManager.LayoutState#next
1. 先从mScrapList取
mScrapList唯一赋值的位置layoutForPredictiveAnimations
在onLayoutChildren中调用
2. RecyclerView.Recycler
   getViewForPosition(int) -> getViewForPosition(int,boolean) -> tryGetViewHolderForPositionByDeadline

```java
// tryGetViewHolderForPositionByDeadline
ViewHolder tryGetViewHolderForPositionByDeadline(int position,
                boolean dryRun, long deadlineNs) {
    ...
    ViewHolder holder = null;
    // 0) If there is a changed scrap, try to find from there
    if (mState.isPreLayout()) {
        holder = getChangedScrapViewForPosition(position);
        fromScrapOrHiddenOrCache = holder != null;
    }
    // 1) Find by position from scrap/hidden list/cache
    if (holder == null) {
        holder = getScrapOrHiddenOrCachedHolderForPosition(position, dryRun);
        if (holder != null) {
            if (!validateViewHolderForOffsetPosition(holder)) {
                ...
                holder = null;
            } else {
                fromScrapOrHiddenOrCache = true;
            }
        }
    }
    if (holder == null) {
        ...
        final int type = mAdapter.getItemViewType(offsetPosition);
        // 2) Find from scrap/cache via stable ids, if exists
        if (mAdapter.hasStableIds()) {
            holder = getScrapOrCachedViewForId(mAdapter.getItemId(offsetPosition),
                    type, dryRun);
            ...
        }
        if (holder == null && mViewCacheExtension != null) {
            // We are NOT sending the offsetPosition because LayoutManager does not
            // know it.
            final View view = mViewCacheExtension
                    .getViewForPositionAndType(this, position, type);
            if (view != null) {
                holder = getChildViewHolder(view);
                ...
            }
        }
        if (holder == null) { // fallback to pool
            ...
            holder = getRecycledViewPool().getRecycledView(type);
            ...
        }
        if (holder == null) {
            ...
            holder = mAdapter.createViewHolder(RecyclerView.this, type);
            ...
        }
    }
    ...
    return holder;
}
```

## 缓存

LinearLayoutManager.LayoutState.mScrapList

LinearLayoutManager#onLayoutChildren
LinearLayoutManager#layoutForPredictiveAnimations

### mChangedScrap & mAttachedScrap 

RecyclerView.Recycler#scrapView
两处入口：
1. RecyclerView.Recycler#getScrapOrHiddenOrCachedHolderForPosition

可以从mHiddenViews转到mAttachedScrap/mChangedScrap

2. LinearLayoutManager#onLayoutChildren
    -> RecyclerView.Recycler#detachAndScrapAttachedViews
    -> RecyclerView.Recycler#scrapOrRecycleView

### mHiddenViews

ChildHelper.mHiddenViews
    hide
    addView
    attachViewToParent

RecyclerView#addAnimatingView

1. mViewInfoProcessCallback.processDisappeared -> animateDisappearance
2. dispatchLayout -> dispatchLayoutStep3 -> animateChange
   1. consumePendingUpdateOperations
      1. ViewFlinger#run
      2. mUpdateChildViewsRunnable#run
         - RecyclerViewDataObserver#triggerUpdateProcessor
            - RecyclerViewDataObserver#onItemXXX
      3. RecyclerView#scrollByInternal
         1. RecyclerView#scrollBy
         2. onTouchEvent ACTION_MOVE
      4. RecyclerView#focusSearch
   2. onLayout 


### mCacheViews 

Recycler#recycleViewHolderInternal

RecyclerView#removeAnimatingView
    ItemAnimatorRestoreListener#onAnimationFinished
        ItemAnimator#dispatchAnimationFinished
            SimpleItemAnimator#dispatchRemoveFinished
            SimpleItemAnimator#dispatchMoveFinished
            SimpleItemAnimator#dispatchChangeFinished
            SimpleItemAnimator#dispatchAddFinished

animateMove
animateAdd
animateChange
animateRemove

animateChange
animateAppearance
animatePersistence
animateDisappearance


Recycler#tryGetViewHolderForPositionByDeadline
入口
LinearLayoutManager.LayoutState#next -> Recycler#next

Recycler#recycleView
1. LayoutManager#removeAndRecycleViewAt
   1. LayoutManager#removeAndRecycleAllViews
      1. LiearLayoutManager#onDetachedFromWindow
      2. LiearLayoutManager#onLayoutChildren
      3. RecyclerView#removeAndRecycleViews
         1. setAdapterInternal
      4. RecyclerView#setLayoutManager
   2. recycleChildren
      1. recycleViewsFromStart
         1. recycleByLayoutState
            1. fill
      2. recycleViewsFromEnd
         1. recycleByLayoutState
            1. fill
2. removeAndRecycleView mViewInfoProcessCallback.unused
   1. dispatchLayoutStep3 -> ViewInfoStore.process
3. Recycler#quickRecycleScrapView
   1. Recycler#getScrapOrCachedViewForId
      1. tryGetViewHolderForPositionByDeadline
   2. LayoutManager#removeAndRecycleScrapInt
      1. setLayoutManager
      2. RecyclerView#removeAndRecycleViews
         1. setAdapterInternal
      3. dispatchLayoutStep3


fill入口

onLayoutChildren
    step1
    step2
layoutForPredictiveAnimations
scrollBy
    scrollHorizontallyBy
        scrollStep
            scrollByInternal
    scrollVerticallyBy
        scrollStep
            scrollByInternal
    fixLayoutStartGap
        onLayoutChildren
    fixLayoutEndGap
        onLayoutChildren
onFocusSearchFailed
    focusSearch

### RecyclerViewPool

recycleViewHolderInternal

1. recycleCachedViewAt
   1. updateViewCacheSize
      1. setViewCacheSize
         1. setItemViewCacheSize
      2. setLayoutManager
      3. dispatchLayoutStep3
      4. setItemPrefetchEnabled
   2. recycleAndClearCachedViews
      1. clear
         1. removeAndRecycleViews
         2. setLayoutManager
         3. onAdapterChanged
            1. setAdapterInternal
         4. onDetachedFromWindow
      2. markKnownViewsInvalid
   3. recycleViewHolderInternal
      1. removeAnimatingView
      2. tryGetViewHolderForPositionByDeadline
      3. recycleView
         1. removeAndRecycleView
         2. removeAndRecycleViewAt
      4. quickRecycleScrapView
      5. scrapOrRecycleView
   4. getScrapOrCachedViewForId
      1. tryGetViewHolderForPositionByDeadline
   5. offsetPositionRecordsForRemove
      1. offsetPositionRecordsForRemove
         1. offsetPositionsForRemovingInvisible
            1. dispatchFirstPassAndUpdateViewHolders
               1. dispatchAndUpdateViewHolders
                  1. applyUpdate
                  2. applyRemove
                     1. preProcess
                        1. consumePendingUpdateOperations
                        2. processAdapterUpdatesAndSetAnimationFlags
                           1. onMeasure
                           2. dispatchLayoutStep1
            2. consumeUpdatesInOnePass
               1. onMeasure
               2. processAdapterUpdatesAndSetAnimationFlags
               3. dispatchLayoutStep2
         2. offsetPositionsForRemovingLaidOutOrNewView
            1. UpdateOp.REMOVE
   6. viewRangeUpdate
      1. markViewHoldersUpdated
         1. dispatchFirstPassAndUpdateViewHolders
         2. UpdateOp.UPDATE postponeAndUpdateViewHolders
         3. UpdateOp.UPDATE consumeUpdatesInOnePass


postponeAndUpdateViewHolders

1. applyAdd
2. applyRemove
3. applyMove
4. applyUpate

applyXXX唯一调用地方preProcess


```java
public static class RecycledViewPool {
    private static final int DEFAULT_MAX_SCRAP = 5;

    static class ScrapData {
        final ArrayList<ViewHolder> mScrapHeap = new ArrayList<>();
        int mMaxScrap = DEFAULT_MAX_SCRAP;
        long mCreateRunningAverageNs = 0;
        long mBindRunningAverageNs = 0;
    }

    // itemViewType作为键
    SparseArray<ScrapData> mScrap = new SparseArray<>();
}
```

## 动画

动画实际执行

dispatchLayoutStep3

mViewInfoStore.process(mViewInfoProcessCallback)