先说结论：

`Adapter.notifyItemChanged(int,Object)`方法会导致RecyclerView的onMeasure()和onLayout()方法调用。在onLayout()方法中会调用dispatchLayoutStep1()、dispatchLayoutStep2()和dispatchLayoutStep3()三个方法

- dispatchLayoutStep1()将更新信息存储到ViewHolder中，
- dispatchLayoutStep2()进行子View的布局，
- dispatchLayoutStep3()触发动画。

在dispatchLayoutStep2()中，会通过DefaultItemAnimator的canReuseUpdatedViewHolder()方法判断position处是否复用之前的ViewHolder，如果调用notifyItemChanged()时的payload不为空，则复用；否则，不复用。

在dispatchLayoutStep3()中，如果position处的ViewHolder与之前的ViewHolder相同，则执行DefaultItemAnimator的move动画；如果不同，则执行DefaultItemAnimator的change动画，旧View动画消失（alpha值从1到0），新View动画展现（alpha值从0到1），这样就出现了闪烁效果。

再说一点自己的认识：

3.0 前提

这部分以第二部分的例子入手，主要介绍下RecyclerView使用LinearLayoutManager和DefaultItemAnimator，**RecyclerView的`layout_width`和`layout_height`都声明为`match_parent`，RecyclerView未设置`hasFixedSize`且Adapter未设置`HasStableIds`时，`notifyItemChanged(int,Object)`的执行流程**。`notifyItemChanged()`调用前页面已经刷新完毕，调用后短时间内没有其他操作。

3.1 RecyclerView与Adapter建立观察者模式

从adapter.notifyItemChanged(int position, @Nullable Object payload)方法入手

```java
AdapterDataObservable mObservable
```
直接对mObservers列表进行遍历，并调用其onItemRangeChanged()方法。向mObservers中添加Observer的方法为registerObserver()方法。

RecyclerView中最终调用registerObserver的地方为setAdapterInternal()方法，这个方法又被setAdapter()和swapAdapter()调用，registerObserver()方法传入的参数为RecyclerView的mObserver变量。

总结而言，RecyclerView通过调用setAdapter(adapter)方法，与adapter建立观察者模式。当数据更改时，开发者通过adapter的notifyXXX()方法通知RecyclerView的mObserver变量进行UI更新。

3.2 onItemRangeChanged()

RecyclerView中的mObserver是RecyclerViewDataObserver类型的变量，看下它的onItemChanged()方法。

直接调用了mAdapterHelper的onItemRangeChanged()方法。

该方法将数据更新信息封装为UpdateOp对象，并将其添加到mPendingUpdates列表中，如果列表中数目为1，则返回true，否则false。在3.0的前提下，由于是在列表已经刷新完成的情况下调用notifyItemChanged()，所以应该返回true，会调用triggerUpdateProcessor方法。

再看下triggerUpdateProcessor()方法

由于未设置hasFixedSize，直接调用requestLayout()方法。requestLayout()方法会导致RecyclerView的onMeasure()和onLayout()方法得到调用。看下RecyclerView的onMeasure()方法。

mLayout是LinearLayoutManager类的对象，LinearLayoutManager的isAutoMeasureEnabled()为true；RecyclerView的宽高被设置为match_parent，因此widthMode和heightMode都为MeasureSpec.EXACTLY，进而measureSpecModeIsExactly为true。整个onMeasure()方法其实只调用了LayoutManager的onMeasure(Recycler recycler, State state, int widthSpec, int heightSpec)方法，最终调用的是RecyclerView的defaultOnMeasure()方法，在RecyclerView宽高被设置为match_parent时等价于：

因此图片闪烁问题的原因只能去onLayout()方法中查找。RecyclerView的onLayout()方法直接调用了dispatchLayout()方法，去除无关代码后如下：

在3.0的前提下，mState.mLayoutStep为State.STEP_START，因此会依序执行dispatchLayoutStep1()、dispatchLayoutStep2()、dispatchLayoutStep3()三个方法。

3.3 dispatchLayoutStep1()

先看下dispatchLayoutStep1()的简化代码，

对于第二部分的例子而言，该方法做了四件事，这四件事代码作者都写在方法的注释里了：

- process adapter updates 处理Adapter更新，将3.2中AdapterHelper里的mPendingUpdates中的内容存储到ViewHolder中
- decide which animation should run 确定会运行的动画
- save information about current views 记录当前Views的信息
- If necessary, run predictive layout and save its information 运行predictive layout，并保存它的信息

看下这四步都分别做了什么：

3.3.1 处理Adapter更新和确定会运行的动画

这两步都在processAdapterUpdatesAndSetAnimationFlags()方法中，

① 是判断是否允许predictiveItemAnimations，predictive animation介绍，对于3.0前提下的LinearLayoutManager而言为true。
② 将3.2中AdapterHelper里的mPendingUpdates中的内容存储到ViewHolder中。

重点看下mAdapterHelper的preProcess()方法

从3.2处的分析可知，`mPendingUpdates`列表中只包含一个`cmd`为`UpdateOp.UPDATE`的`UpdateOp`，因此只调用了一次`applyUpdate(op)`方法并将mPendingUpdates清空，看下AdapterHelper.applyUpdate()方法：

mCallback变量是在RecyclerView构造函数中初始化AdapterHelper变量时创建的，由于执行applyUpdate的View是显示的，所以vh值不为null。因此type为POSITION_TYPE_NEW_OR_LAID_OUT，进而调用了postponeAndUpdateViewHolders()方法。

postponeAndUpdateViewHolders()做了两件事，

① 将UpdateOp存入mPostponedList中，
② 调用mCallback的markViewHoldersUpdated方法，

markViewHoldersUpdated()方法如下：

做了两件事，
① 调用viewRangeUpdate()方法，
② 将mItemsChanged变量置为true。

viewRangeUpdate()方法完成了将mPendingUpdates列表里的更新存储到相应的ViewHolder中的功能：添加ViewHolder.FLAG_UPDATE的flag，表示该ViewHolder要发生改变，之后该viewHolder的isUpdated()和needsUpdate()方法将返回true；将payload保存到holder的mPayloads列表中。

回到processAdapterUpdatesAndSetAnimationFlags()中，由于mItemsChanged在markViewHoldersUpdated()方法中被置为true，所以animationTypeSupported变量为true，同时由于mFirstLayoutComplete为true，mItemAnimator默认为DefaultItemAnimator，不为null，mDataSetHasChangedAfterLayout为false，所以③ 处mState.mRunSimpleAnimations也为true。在④处，mState.mRunPredictiveAnimations也为true。

3.3.2 记录Views信息

回到dispatchLayoutStep1()中

由于processAdapterUpdatesAndSetAnimationFlags()方法中mState.mRunSimpleAnimations为true，且viewRangeUpdate()方法中将

mItemsChanged变量设置为true，所以mState.mTrackOldChangeHolders变量为true，mState.mInPreLayout也为true。由于mState.mRunSimpleAnimations为true，会对RecyclerView中的ChildView进行遍历，从view中取出holder，并记录所有holder到preLayout信息和要发生改变的holder的信息。

3.3.2.1 记录preLayout信息

看下ViewInfoStore的addPreLayout()方法，将holder和animationInfo存入到mLayoutHolderMap中。

3.3.2.2 记录要发生改变等holder等信息

对于3.3.1中viewRangeUpdate()方法中处理过的ViewHolder，holder.isUpdated()为true，holder.isRemoved()、holder.shouldIgnore()、holder.isInvalid()都为false，所以会存入到ViewInfoStore的mOldChangedHolders中，这个会在dispatchLayoutStep3()中用到。

3.3.3 运行predictive layout，并保存它的信息

此处的mLayout为LinearLayoutManager类型的对象，看下LinearLayoutManager的onLayoutChildren()方法，省略无关代码后如下。

onLayoutChildren()做了三件事：

①寻找锚点；
②将RecyclerView中的所有View detach并回收；
③重新attach view，并铺满屏幕。

重点关注下②和③。

看下②中detachAndScrapAttachedViews()方法。

遍历RecyclerView的所有child，并执行scrapOrRecycleView()方法。

由于使用notifyItemChanged()做局部刷新，所以viewHolder.isInvalid()为false，因此会执行①②③，重点关注下①和②。

看下detachViewAt(int index)方法及其内部调用到的方法。

可以看到detachViewAt(int index)方法，一共做了三件事，需要关注的是ViewHolder添加FLAG_TMP_DETACHED这个flag，另外调用了ViewGroup的detachViewFromParent()方法，暂时将View从ViewGroup的mChildren数组中清除，并将View的parent置为null。

回到scrapOrRecycleView()方法，看下② Recycler.scrapView()方法及其调用的方法。

从SimpleItemAnimator的canReuseUpdatedViewHolder()方法看起，由于mSupportsChangeAnimations默认值为true，除非调用setSupportsChangeAnimations将其设置为false；同时在notifyItemChanged()的情况下，viewHolder.isInvalid()为false。所以SimpleItemAnimator的canReuseUpdatedViewHolder方法返回false。那么对于DefaultItemAnimator的canReuseUpdatedViewHolder()方法，返回值就完全取决于`!payloads.isEmpty()`的值，如果payloads为空，则返回false；如果payloads不为空，则返回true。回到scrapView(View)方法中，对于要update的ViewHolder（ViewHolder.isUpdated()为true），如果payloads为空，ViewHolder会被添加到mChangedScrap列表中，并将ViewHolder的mInChangeScrap变量设置为true，否则会添加到mAttachedScrap列表中并将ViewHolder的mInChangeScrap变量设置为false。

回到LinearLayoutManager中的onLayoutChildren()方法中，看下fill()方法。fill方法中不停调用layoutChunk()方法添加View，直到屏幕占满。

注意看下layoutState的next()方法，这里有很多熟悉的方法，比如createViewHolder，bindViewHolder。

分析下layoutState的next()方法：从3.3.2可知，mState.isPreLayout()为true，所以会

①先从mChangedScrap列表中查找ViewHolder，如果未找到，则
②会从mAttachedScrap列表中查找。

回顾下scrapView()方法，RecyclerView中View所在的viewHolder不是存进mChangedScrap就是存进mAttachedScrap了。所以，对于notifyItemChanged()这种情况，在pre layout阶段，经过这①②这两步，holder的值就不为空了，由于mState.isPreLayout()为true，且holder.isBound()为true，所以不会执行bindViewHolder操作。

回到layoutChunk()方法，当next()获取到view后，会执行RecyclerView的addView()方法。

对于此种情况，执行了如下代码，跟scrapOrRecycleView()方法其实非常对称。

当所有RecyclerView添加完View，dispatchLayoutStep1()方法就执行完了，之后就开始执行dispatchLayoutStep2()方法。

3.4 dispatchLayoutStep2()

dispatchLayoutStep2()是真正进行布局的方法。3.0的前提下，①中什么都没做，就不分析了。重点是3.3.3中花了很长代码介绍的onLayoutChildren()方法，现在再次遇到就相对容易看了。dispatchLayoutStep2()中的onLayoutChildren与3.3.3中pre layout的不同之处，在于mState.mInPreLayout被设置为false了，造成的区别在tryGetViewHolderForPositionByDeadline()中。

由于mState.isPreLayout()为false，所以步骤①不会执行，也就是不会从mChangedScrap列表中查找，但使用notifyItemChanged(int position)方法进行刷新的ViewHolder，在scrapView()方法中被存入了mChangedScrap列表中，这就导致步骤②之后，holder 仍为null。之后的流程就是从缓存（cacheExtension、recyclerPool）中查找，如果查找到相同类型的ViewHolder，则从缓存中取出复用；如果没有相同类型的ViewHolder，就只能通过onCreateViewHolder来创建新的viewHolder。但如果对于notifyItemChanged(int position, Object payload)方式进行刷新的ViewHolder，在scrapView()方法中被存入了mAttachedScrap列表中，经过步骤②后，就取得了之前的holder。这两种情况的holder，在步骤⑦处，由于mState.isPreLayout()为false，且holder.needsUpdate() 为true，所以会调用onBindViewHolder(holder, position, holder.getUnmodifiedPayloads());来更新数据，回顾文章第二部分对onBindViewHolder(holder, position, payloads);的实现，可以看到此时喜欢的状态就更新了。

在3.0的前提下，dispatchLayoutStep2()中onLayoutChildren()其他地方都与pre layout流程中onLayoutChildren()无区别。

3.5 dispatchLayoutStep3()

dispatchLayoutStep3()对ChildHelper中view进行遍历，首先从view中获取到holder，再根据holder的key，从ViewStoreInfo获取3.3 dispatchLayoutStep1()步骤③中存入的oldChangeViewHolder，由于oldDisappearing为false且preInfo不为null，所以会调用animateChange(oldChangeViewHolder, holder, preInfo, postInfo, oldDisappearing, newDisappearing)方法。另外，需要注意的是，根据3.4的分析，如果是在notifyItemChanged(int position)调用导致的dispatchLayoutStep3()中，oldChangeViewHolder与holder不同；如果是在notifyItemChanged(int position, Object payload)调用导致的dispatchLayoutStep3()中，oldChangeViewHolder与holder相同。

接着看下animateChange()及其调用的相应方法。

可以看到如果oldHolder与newHolder相同，执行animateMove()方法，由于oldHolder与newHolder的位置相同，所以直接return了。而oldHolder与newHolder不同时，执行animateChange()方法，将newHolder 的view的alpha值设置为0。在动画真正运行，即执行animateChangeImpl()方法时，newHolder的view会动画显示（alpha从0到1)，oldHolder的view会动画消失（alpha从1到0），这样实现了CrossFade的效果，也就是出现了闪烁。'