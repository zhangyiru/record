
条件变量：
1、条件变量（std::condition_variable）是一个对象，该对象能够阻塞（wait）调用线程，直到被通知（notify）恢复。
2、当调用其等待函数（wait,wait_for.wait_until）之一时，使用unique_lock（通过互斥锁mutex）来锁定线程，该线程会保持阻塞状态，直到被同在condition_variable对象上调用通知功能的线程唤醒为止。
3、condition_variable类型的对象始终使用unique_lock<mutex>

程序示例：
https://segmentfault.com/a/1190000039195800


notify:
1、通知线程不需要在同个mutex上和等待线程持有相同的锁。
如果存在锁，那么因为无论什么等待线程变成可运行的，都将立即尝试获取通知线程持有的锁。并且在调用notify_one()之前释放锁可以显著提高性能

参考：在调用notify前是否需要加锁？
https://stackoverflow.com/questions/17101922/do-i-have-to-acquire-lock-before-calling-condition-variable-notify-one

2、被通知的线程会立刻进入阻塞，直到通知线程释放锁。
执行notify方法后，当前线程不会马上释放该对象锁，wait状态的其他线程也就不马上获取该对象锁，要等待执行notify方法的线程执行完毕

链接：wait/notify机制
https://blog.csdn.net/qq_26545305/article/details/79188912?utm_source=app&app_version=4.21.0&code=app_1562916241&uLinkId=usr1mkqgl919blen

