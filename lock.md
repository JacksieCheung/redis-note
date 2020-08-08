**分布式锁**

```redis
> setnx lock:codehole true
ok
...
> del lock:codehole
(integer) 1
```

&#8195;这里的冒号只是普通字符，没有特别含义

**超时问题**

**可重入性**
