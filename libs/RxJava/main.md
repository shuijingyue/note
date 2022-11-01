```java
Observable.fromArray(1, 2, 3) // ObservableFromArray
    .subscribeOn(Schedulers.io()) // ObservableSubscribeOn
    .observeOn(AndroidSchedulers.mainThread()) // ObservableObserveOn
    .subscribe(System.out::println)
```

### subscribeOn

```java
abstract class AbstractObservableWithUpstream<T, U> extends Observable<U> implements HasUpstreamObservableSource<T> {

    /** The source consumable Observable. */
    protected final ObservableSource<T> source;

    /**
     * Constructs the ObservableSource with the given consumable.
     * @param source the consumable Observable
     */
    AbstractObservableWithUpstream(ObservableSource<T> source) {
        this.source = source;
    }

    @Override
    public final ObservableSource<T> source() {
        return source;
    }
}

public final class ObservableSubscribeOn<T> extends AbstractObservableWithUpstream<T, T> {
    final Scheduler scheduler;

    // source ObservableFromArray
    // scheduler IoScheduler
    public ObservableSubscribeOn(ObservableSource<T> source, Scheduler scheduler) {
        super(source);
        this.scheduler = scheduler;
    }
}
```

### observeOn

```java
public final class ObservableObserveOn<T> extends AbstractObservableWithUpstream<T, T> {
    final Scheduler scheduler;
    final boolean delayError;
    final int bufferSize;

    // source ObservableSubscribeOn
    // scheduler HandlerScheduler
    // delayError false
    // bufferSize 128
    public ObservableObserveOn(ObservableSource<T> source, Scheduler scheduler, boolean delayError, int bufferSize) {
        super(source);
        this.scheduler = scheduler;
        this.delayError = delayError;
        this.bufferSize = bufferSize;
    }
}
```

### subscribe

即ObservableObserveOn.subscribeActual

```java
// observer LambdaObserver
// source ObservableSubscribeOn
@Override
protected void subscribeActual(Observer<? super T> observer) {
    if (scheduler instanceof TrampolineScheduler) {
        source.subscribe(observer);
    } else {
        Scheduler.Worker w = scheduler.createWorker();

        source.subscribe(new ObserveOnObserver<>(observer, w, delayError, bufferSize));
    }
}
```

```java
 static final class ObserveOnObserver<T> extends BasicIntQueueDisposable<T>
    implements Observer<T>, Runnable {

        private static final long serialVersionUID = 6576896619930983584L;
        final Observer<? super T> downstream; // LambdaObserver
        final Scheduler.Worker worker; // IoScheduler.EventLoopWorker
        final boolean delayError; // false
        final int bufferSize; // 128

        SimpleQueue<T> queue;

        Disposable upstream;

        Throwable error;
        volatile boolean done;

        volatile boolean disposed;

        int sourceMode;

        boolean outputFused;
    }
```

**ObservableSubscribeOn.subscribeActual**

```java
// observer ObserveOnObserver
// scheduler IoScheduler
@Override
public void subscribeActual(final Observer<? super T> observer) {
    final SubscribeOnObserver<T> parent = new SubscribeOnObserver<>(observer);

    observer.onSubscribe(parent); // 设置disposable

    parent.setDisposable(scheduler.scheduleDirect(new SubscribeTask(parent)));
}
```

```java
static final class SubscribeOnObserver<T> extends AtomicReference<Disposable> implements Observer<T>, Disposable {

    private static final long serialVersionUID = 8094547886072529208L;
    final Observer<? super T> downstream; // ObserveOnObserver

    final AtomicReference<Disposable> upstream; // AtomicReference<Disposable>
}
```


**ObserveOnObserver.onSubscribe**

```java
// d SubscribeOnObserver
@Override
public void onSubscribe(Disposable d) {
    // upstream(Disposable)校验this.upstream是不是null
    if (DisposableHelper.validate(this.upstream, d)) {
        this.upstream = d;
        // upstream SubscribeOnObserver
        if (d instanceof QueueDisposable) {
            @SuppressWarnings("unchecked")
            QueueDisposable<T> qd = (QueueDisposable<T>) d;

            int m = qd.requestFusion(QueueDisposable.ANY | QueueDisposable.BOUNDARY);

            if (m == QueueDisposable.SYNC) {
                sourceMode = m;
                queue = qd;
                done = true;
                downstream.onSubscribe(this);
                schedule();
                return;
            }
            if (m == QueueDisposable.ASYNC) {
                sourceMode = m;
                queue = qd;
                downstream.onSubscribe(this);
                return;
            }
        }

        queue = new SpscLinkedArrayQueue<>(bufferSize);

        downstream.onSubscribe(this);
    }
}
```
