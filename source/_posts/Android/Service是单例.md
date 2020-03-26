stackoverflow上的3个问题解答了我的疑惑：
[Android: When to use Service vs Singleton?](https://stackoverflow.com/questions/3567667/android-when-to-use-service-vs-singleton)
[Android service isn't working as a singleton](https://stackoverflow.com/questions/16309494/android-service-isnt-working-as-a-singleton)
[Does startService() create a new Service instance or using the existing one?](https://stackoverflow.com/questions/2518238/does-startservice-create-a-new-service-instance-or-using-the-existing-one)

---

结论：
1. 当多次startService去启动Service，若Service对象存在，就只调用onStartCommand，若Service对象不存在，创建Service对象。
2. 服务时天然的单例模式，且可以被销毁。
