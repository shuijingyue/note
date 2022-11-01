processDataSetCompletelyChanged
markKnownViewsInvalid

将所有的ViewHolder都标记为invalid

在dispatchLayoutStep2之后

```java
// 此时isPreLayout一定为false
if (mState.isPreLayout() && holder.isBound()) {
    ...
} else if (!holder.isBound() || holder.needsUpdate() || holder.isInvalid()) {
    // 如果 ViewHolder 需要更新或者无效了, 则重新为其绑定数据
    final int offsetPosition = mAdapterHelper.findPositionOffset(position);
    // 绑定数据
    bound = tryBindViewHolderByDeadline(holder, offsetPosition, position, deadlineNs);
}
```
