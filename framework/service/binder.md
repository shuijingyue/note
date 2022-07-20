`asInterface(IBinder binder)`

```java
if (binder == null)
    return null;
IInterface iin = binder.queryLocalInterface(DESCRIPTOR);
if (iin != null && iin instanceof BookManager)
    return (BookManager) iin;
return new Proxy(binder);
```