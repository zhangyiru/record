连接无线

```
gltest 12345678
adb tcpip
adb connect 10.91.8.179
```



抓取log
adb logcat 

登陆stationpro看信息：
adb shell 

打开原生设置：
adb shell am start com.android.settings

打印某个进程中所有线程调用栈

adb shell debuggerd -b pid



**查看user版本还是userdebug版本**

```
adb shell getprop ro.build.type
```
