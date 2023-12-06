> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.jianshu.com](https://www.jianshu.com/p/d69a90bc44c0)

博文出处：[关于使用 AlarmManager 的注意事项](https://link.jianshu.com?t=http://yuqirong.me/2017/01/21/%E5%85%B3%E4%BA%8E%E4%BD%BF%E7%94%A8AlarmManager%E7%9A%84%E6%B3%A8%E6%84%8F%E4%BA%8B%E9%A1%B9/)，欢迎大家关注我的博客，谢谢！

快过年了，更新春节前的最后一篇博客。

最近在做一个需求：客户端按照规定的时间间隔向服务端发送定位。一看到这个需求就想到了使用 `AlarmManager` 来实现。 `AlarmManager` 经常被用来执行定时任务，比如设置闹铃、发送心跳包等。也许有人会有疑问：为什么不能使用相同具有定时效果的 `Timer` 和 `Handler` 呢？

其实答案非常简单，相对于 `Handler` 来说，使用 `sendEmptyMessageDelayed` 方法是依赖于 `Handler` 所在的线程的，如果线程结束，就起不到定时任务的效果；而 `AlarmManager` 依赖的是 Android 系统的服务，具备唤醒机制。比起 `Handler` 也就更合适了。

而至于 `Timer` 可以精确地做到定时操作，但是相比于 `AlarmManager` 而言还是差了一截。同理，如果手机关屏后长时间不使用， CPU 就会进入休眠模式。这个使用如果使用 `Timer` 来执行定时任务就会失败，因为 `Timer` 无法唤醒 CPU 。

所以，综上所述，`AlarmManager` 就成为了最佳选择。

SDK API < 19
============

一般情况下，使用 `AlarmManager` 来执行重复定时任务的代码如下所示：

```
alarmManager.setRepeating(AlarmManager.ELAPSED_REALTIME_WAKEUP, SystemClock.elapsedRealtime(), TIME_INTERVAL, pendingIntent);
```

或者

```
alarmManager.setRepeating(AlarmManager.RTC_WAKEUP, System.currentTimeMillis(), TIME_INTERVAL, pendingIntent);
```

`setRepeating` 该方法用于设置重复定时任务。

*   第一个参数表示闹钟类型：一般为 `AlarmManager.ELAPSED_REALTIME_WAKEUP` 或者 `AlarmManager.RTC_WAKEUP` 。它们之间的区别就是前者是从手机开机后的时间，包含了手机睡眠时间；而后者使用的就是手机系统设置中的时间。所以如果设置为 `AlarmManager.RTC_WAKEUP` ，那么可以通过修改手机系统的时间来提前触发定时事件。另外，对于相似的 `AlarmManager.ELAPSED_REALTIME` 和 `AlarmManager.RTC` 来说，它们不会唤醒 CPU 。所以使用的频率较少；
*   第二个参数表示任务首次执行时间：与第一个参数密切相关。第一个参数若为 `AlarmManager.ELAPSED_REALTIME_WAKEUP` ，那么当前时间就为 `SystemClock.elapsedRealtime()` ；若为 `AlarmManager.RTC_WAKEUP` ，那么当前时间就为 `System.currentTimeMillis()` ；
*   第三个参数表示两次执行的间隔时间：这个参数没什么好讲的，一般为常量；
*   第四个参数表示对应的响应动作：一般都是去发送广播，然后在广播接收 `onReceive(Context context, Intent intent)` 中做相关操作。

至此，一切顺利，畅通无阻。

SDK API >= 19 && SDK API < 23
=============================

当你写好代码、满心欢喜地将程序跑在手机上的时候，傻眼了！你会发现在 Android 4.4 及以上版本的定时任务不是按照规定时间间隔来执行的。比如你设置了每隔 3 分钟发出一个 HTTP 请求，结果你一看莫名其妙地变成了隔 5 分钟发一次。

What the fuck?

![](http://upload-images.jianshu.io/upload_images/1017209-9898adf9b761bc5e.png) what the fuck

然后你查阅 Android 官网中关于 [Android 4.4 API](https://link.jianshu.com?t=https://developer.android.google.cn/about/versions/android-4.4.html) 会看到如下几句话：

![](http://upload-images.jianshu.io/upload_images/1017209-9aeac689a801b0a7.png) Android 4.4 API

恍然大悟！原来是 Google 为了追求系统省电，所以 “偷偷加工” 了一下唤醒的时间间隔。但也正如上面官网中所说的那样，如果在 Android 4.4 及以上的设备还要追求精准的闹钟定时任务，要使用 `setExact()` 方法。

所以，相应的代码就变成了这样：

```
// pendingIntent 为发送广播
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.KITKAT) {
    alarmManager.setExact(AlarmManager.ELAPSED_REALTIME_WAKEUP, SystemClock.elapsedRealtime(), pendingIntent);
} else {
    alarmManager.setRepeating(AlarmManager.ELAPSED_REALTIME_WAKEUP, SystemClock.elapsedRealtime(), TIME_INTERVAL, pendingIntent);
}

private BroadcastReceiver alarmReceiver = new BroadcastReceiver() {
    @Override
    public void onReceive(Context context, Intent intent) {
        // 重复定时任务
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.KITKAT) {
            alarmManager.setExact(AlarmManager.ELAPSED_REALTIME_WAKEUP, SystemClock.elapsedRealtime() + TIME_INTERVAL, pendingIntent);
        }
        // to do something
        doSomething();
    }
};
```

当你写好了 “加强版” 的 `AlarmManager` 之后，内心肯定无比小激动。这下总应该行了吧？运行一下，果然没错！在 Android 4.4 上的确按照规定的时间间隔在执行任务。哈哈，这下大功告成了！！！

SDK API >= 23
=============

在 Android 4.4 上品尝到胜利的甜头后，你顺便在 Android 6.0 的设备上测试了一下。结果。。。。。。你又 TMD 傻眼了！

![](http://upload-images.jianshu.io/upload_images/1017209-37a0eaaecae714f5.png) What the fuck

发现在设备关屏静止一段时间后， `AlarmManager` 又又又不能正常工作了。相必此时你连日狗的心都有了吧！强忍着泪水，再次打开 Android 官网中关于 [Android 6.0 变更](https://link.jianshu.com?t=https://developer.android.google.cn/about/versions/marshmallow/android-6.0-changes.html) ，发现在 Android 6.0 中引入了低电耗模式和应用待机模式。然后接着往下看 [对低电耗模式和应用待机模式进行针对性优化](https://link.jianshu.com?t=https://developer.android.google.cn/training/monitoring-device-state/doze-standby.html) ，发现会有下面一段话：

![](http://upload-images.jianshu.io/upload_images/1017209-958b1bca16fe2651.png) Android 6.0 API

啊啊啊啊啊啊！之前在 Android 4.4 上能用的 `setExact()` 方法在 Android 6.0 上因为低电耗模式又不能正常使用了。但是，Google 又又又提供了新的方法 `setExactAndAllowWhileIdle()` 来解决在低电耗模式下的闹钟触发。

所以，Attention！相关的代码又被改写为这样：

```
// pendingIntent 为发送广播
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.M) {
    alarmManager.setExactAndAllowWhileIdle(AlarmManager.ELAPSED_REALTIME_WAKEUP, SystemClock.elapsedRealtime(), pendingIntent);
} else if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.KITKAT) {
    alarmManager.setExact(AlarmManager.ELAPSED_REALTIME_WAKEUP, SystemClock.elapsedRealtime(), pendingIntent);
} else {
    alarmManager.setRepeating(AlarmManager.ELAPSED_REALTIME_WAKEUP, SystemClock.elapsedRealtime(), TIME_INTERVAL, pendingIntent);
}

private BroadcastReceiver alarmReceiver = new BroadcastReceiver() {
    @Override
    public void onReceive(Context context, Intent intent) {
        // 重复定时任务
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.M) {
            alarmManager.setExactAndAllowWhileIdle(AlarmManager.ELAPSED_REALTIME_WAKEUP, SystemClock.elapsedRealtime() + TIME_INTERVAL, pendingIntent);
        } else if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.KITKAT) {
            alarmManager.setExact(AlarmManager.ELAPSED_REALTIME_WAKEUP, SystemClock.elapsedRealtime() + TIME_INTERVAL, pendingIntent);
        }
        // to do something
        doSomething();
    }
};
```

到了这里，总算是把因 Android 版本差异导致 `AlarmManager` 的 “坑” 填完了。这份代码已经可以满足日常的重复定时任务了。

好了，该讲的都讲完了，上床睡觉。仓促地结尾，预祝大家新年快乐！

Goodbye ！

References
==========

*   [AlarmManager](https://link.jianshu.com?t=https://developer.android.google.cn/reference/android/app/AlarmManager.html)
*   [Android 4.4 API](https://link.jianshu.com?t=https://developer.android.google.cn/about/versions/android-4.4.html)
*   [Android 6.0 变更](https://link.jianshu.com?t=https://developer.android.google.cn/about/versions/marshmallow/android-6.0-changes.html)
*   [对低电耗模式和应用待机模式进行针对性优化](https://link.jianshu.com?t=https://developer.android.google.cn/training/monitoring-device-state/doze-standby.html)