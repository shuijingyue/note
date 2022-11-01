ZygoteInit.java

```java
public static void Main(String[] args) {
	pid = Zygote.forSystemServer(...);

	if (pid == 0) {
		// child process
		zygoteServer.closeServerSocket(); // 关闭socket连接
	}
}
```
