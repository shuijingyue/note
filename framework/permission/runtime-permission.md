危险权限需要进行运行时权限处理

即不止要在Manifest文件中声明需要的权限，还要在运行时向用户索要权限

```java
/**
	* int checkSelfPermission(Context context, String permission) {}
  * checkSelfPerssion接收两个参数，第一个Context, 第二个是权限的名称
  * 权限名称在android.Manifest中
  * void requestPermissions(Activity activity, String[] permissions, int requestCode)
  * 接收三个参数，一个活动实例，所有需要的权限名称的数组，唯一的请求码
  */
if (ContextCompat.checkSelfPermission(MainActivity.this, Manifest.permission.CALL_PHONE) != PackageManager.PERMISSION_GRANTED) { 
  	ActivityCompat.requestPermissions(MainActivity.this, new String[]{Manifest.permission.CALL_PHONE}, 1);
} // 判断是否被授权，如果没有所需要的权限，则请求相应的权限
```
重写收到授权结果后执行的回调方法
```java
/**
  * 请求结果会传到onRequestPermissionResult回调中，如果想在获得授权结果之后执行相应的逻辑西需要重写该方法
  */
@Override
public void onRequestPermissionsResult(int requestCode, @NonNull String[] permissions, @NonNull int[] grantResults) {
  switch (requestCode) {
    case 1:
      if (grantResults.length > 0 && grantResults[0] == PackageManager.PERMISSION_GRANTED) {
        call();
      } else {
        Toast.makeText(this, "You denied the permission", Toast.LENGTH_SHORT).show();
      }
      break;
    default:
      break;
  }
}
```

## ContentProvider

通过Context类的getContentResolver获取ContentResolver实例

标准内容URI格式content://authority/tableName

CRUD

```java
Uri uri = Uri.parse("content://com.example.app/table1");
Cursor cursor = getContentResolver().query(uri, );
```

