```java
Observable.fromArray(1, 2, 3) // ObservableFromArray
    .subscribeOn(Schedulers.io()) // ObservableSubscribeOn
    .observeOn(AndroidSchedulers.mainThread()) // ObservableObserveOn
    .subscribe(System.out::println)
```

subscribeOn

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

observeOn

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

subscribe

即ObservableObserveOn.subscribeActual

```java
// observer LambdaObserver
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
