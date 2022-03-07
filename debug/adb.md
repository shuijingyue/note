touch事件传递情况
```bash
adb shell setprop persist.sys.perfdebug.monitor.catalog input
adb shell stop; adb shell start
```
抓systrace,在这个过程中的touch事件会被记录
