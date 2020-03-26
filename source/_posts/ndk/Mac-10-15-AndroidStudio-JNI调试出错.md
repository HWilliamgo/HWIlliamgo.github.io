mac升级到10.15后，进行Jni代码调试的时候报错：
```
com.intellij.execution.ExecutionFinishedException: Execution finished
Process finished with exit code 0
```
clean、invalidate cache and restart都没用，最后将host重新配置一下，就能够进行jni调试了：
```
127.0.0.1 localhost
```

thanks ：[手机和AS中debug环境配置问题小记](http://pollux.cc/2019/12/01/2019-12-01%20%E6%89%8B%E6%9C%BA%E5%92%8CAS%E4%B8%ADdebug%E7%8E%AF%E5%A2%83%E9%85%8D%E7%BD%AE%E9%97%AE%E9%A2%98%E5%B0%8F%E8%AE%B0/#%E4%B8%80%E3%80%81%E5%9C%A8AS%E4%B8%AD%E5%B0%9D%E8%AF%95%E8%B0%83%E8%AF%95native%E4%BB%A3%E7%A0%81%EF%BC%8C%E6%8A%A5%E9%94%99)
