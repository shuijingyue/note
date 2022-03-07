```kotlin
class IndexActivity : AppCompatActivity() {

    @RequiresApi(Build.VERSION_CODES.S)
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_index)
        
        Glide.with(this).load(url).into(imageView)
    }
}
```

with

```java
  // Glide.java
  @NonNull
  public static RequestManager with(@NonNull FragmentActivity activity) {
    return getRetriever(activity).get(activity);
  }
```

```java
  // Glide.java
  @NonNull
  private static RequestManagerRetriever getRetriever(@Nullable Context context) {
    ...
    return Glide.get(context).getRequestManagerRetriever();
  }
```

```java
  // Glide.java
  @NonNull
  public RequestManagerRetriever getRequestManagerRetriever() {
    return requestManagerRetriever; // GlideBuilder.build初始化
  }
```
