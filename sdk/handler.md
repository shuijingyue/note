```java
Message msg = me.mQueue.next(); // might block
if (msg == null) {
  // No message indicates that the message queue is quitting.
  return false;
}

final Observer observer = sObserver;
Object token = null;
if (observer != null) {
  token = observer.messageDispatchStarting();
}
long origWorkSource = ThreadLocalWorkSource.setUid(msg.workSourceUid);
try {
  msg.target.dispatchMessage(msg);
  if (observer != null) {
    observer.messageDispatched(token, msg);
  }
} catch (Exception exception) {
  if (observer != null) {
    observer.dispatchingThrewException(token, msg, exception);
  }
  throw exception;
} finally {
  ThreadLocalWorkSource.restore(origWorkSource)
}

// Make sure that during the course of dispatching the
// identity of the thread wasn't corrupted.
final long newIdent = Binder.clearCallingIdentity();
if (ident != newIdent) {
  Log.wtf(TAG, "Thread identity changed from 0x"
          + Long.toHexString(ident) + " to 0x"
          + Long.toHexString(newIdent) + " while dispatching to "
          + msg.target.getClass().getName() + " "
          + msg.callback + " what=" + msg.what);
}

msg.recycleUnchecked();

return true;
```

