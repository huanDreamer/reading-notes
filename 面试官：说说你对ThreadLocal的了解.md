---
title: 面试官：说说你对ThreadLocal的了解
date: 2019-05-10 10:49:48:048
tags: [Java] 
published: true
feature: https://user-gold-cdn.xitu.io/2019/5/8/16a97cde94bb4de0?imageView2/0/w/1280/h/960/ignore-error/1
---
> 本文转载自 [https://juejin.im/post/5cd2dcf4f265da03804386d0](https://juejin.im/post/5cd2dcf4f265da03804386d0) 

一般有多个孩子的家庭，买玩具都得买多个。如果就买一个，嘿嘿就比较刺激了。这就是**避免共享**，给孩子每人一个玩具对应到我们Java中也就是每个线程都有自己的本地变量，咱们自己玩自己的，避免争抢，和谐相处使得线程安全。

Java就是通过`ThreadLocal`来实现线程本地存储的。

这思路也很清晰，就是每个线程要有自己的本地变量呗，那就Thread里面搞一个私有属性呗`ThreadLocal.ThreadLocalMap threadLocals = null;` 就是如下图所示的这个关系

![](https://user-gold-cdn.xitu.io/2019/5/8/16a97cde94bb4de0?imageView2/0/w/1280/h/960/ignore-error/1)

## ThreadLocal

简单的应用如下
```
public class Demo {
		private static final ThreadLocal<Foo> fooLocal = new ThreadLocal<Foo>();
 
		public static Foo getFoo() {
			return fooLocal.get();
		}
 
		public static void setFoo(Foo foo) {
			fooLocal.set(foo);
		}
	}
```

再深入了解一下内部情况，`ThreadLocalMap`是`ThreadLocal`的内部静态类，它虽然叫`Map`但是和`java.util.Map`没有啥亲戚关系，只是它实现的功能像`Map`

```
static class ThreadLocalMap {
        static class Entry extends WeakReference<ThreadLocal<?>> {
            Object value;
            Entry(ThreadLocal<?> k, Object v) {
                super(k);
                value = v;
            }
        }
        private static final int INITIAL_CAPACITY = 16;
        private Entry[] table;
```

可以看到`ThreadLocalMap`里面有个`Entry`数组，只有数组没有像`HashMap`那样有链表，因此当hash冲突的之后，`ThreadLocalMap`是**采用线性探测的方式解决hash冲突。**

线性探测，就是先根据初始`key`的`hashcode`值确定元素在`table`数组中的位置，如果这个位置上已经有其他`key`值的元素被占用，则利用固定的算法寻找一定步长的下个位置，依次直至找到能够存放的位置。在`ThreadLocalMap`步长是1。

**用这种方式解决hash冲突的效率很低，因此要注意ThreadLocal的数量**。
```
/**
         * Increment i modulo len. 
         */
        private static int nextIndex(int i, int len) {
            return ((i + 1 < len) ? i + 1 : 0);
        }

        /**
         * Decrement i modulo len.
         */
        private static int prevIndex(int i, int len) {
            return ((i - 1 >= 0) ? i - 1 : len - 1);
        }
```

而且可以看到这个`Entry`把`ThreadLocal`的弱引用作为key。那为什么要搞成弱引用(只要发生了GC弱引用对象就会被回收)呢？

首先`ThreadLocal`内部没有存储任何的值，它的作用只是当我们的`ThreadLocalMap的key`，让线程可以拿到对应的`value`。当我们不需要用这个key的时候我们，我们把`fooLocal=null`这样强引用就没了。假设Entry里面也是强引用的话，那等于这个`ThreadLocal`实例还有个强引用在，那么我们想让GC回收`fooLocal`就回收不了了。那可能有人想，你弄成弱引用不是很危险啊，万一GC一下不是没了？别怕只要`fooLocal`这个强引用在这个`ThreadLocal`实例就不会回收的。(关于强软弱虚引用可以看我之前的文章[四种引用方式的区别](https://juejin.im/post/5cd386be51882511282b8746))

因此弄成弱引用，主要是让没用的`ThreadLocal`得以GC清除。

这里可能还有人问那key清除掉了，value咋办，这个Entry还在的呀。是的，当在使用线程池的情况下，由于线程的生命周期很长，某些大对象的key被移除了之后，value一直存在的就可能会导致内存泄漏。

不过java考虑到这点了。当调用`get()、set()`方法时会去找到那个key被干掉的entry然后干掉它。并且提供了`remove()`方法。虽然`get()、set()`会清理`key`为`null的Entry`,但是不是每次调用就会清理的，只有当`get`时候直接hash没中，或者`set`时候也是直接hash没中，开始线性探测时候，碰到key为null的才会清理。
```
//get 方法
        private Entry getEntry(ThreadLocal<?> key) {
            int i = key.threadLocalHashCode &amp; (table.length - 1);
            Entry e = table[i];
            if (e != null &amp;&amp; e.get() == key)
                return e;                           //命中就直接返回
            else
                return getEntryAfterMiss(key, i, e); //直接没命中
        }
        private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
            Entry[] tab = table;
            int len = tab.length;
            while (e != null) { //开始探测了
                ThreadLocal<?> k = e.get();
                if (k == key)  //命中了就返回
                    return e;
                if (k == null)  //探测到key是null的就清理
                    expungeStaleEntry(i);
                else
                    i = nextIndex(i, len); //否则继续
                e = tab[i];
            }
            return null;
        }
        //set 方法
        private void set(ThreadLocal<?> key, Object value) {
            Entry[] tab = table;
            int len = tab.length;
            int i = key.threadLocalHashCode &amp; (len-1);
            for (Entry e = tab[i];
                 e != null;
                 e = tab[i = nextIndex(i, len)]) {
                ThreadLocal<?> k = e.get();
                if (k == key) {
                    e.value = value;    //如果已经有就替换原有的value
                    return;
                }
                if (k == null) {
                    replaceStaleEntry(key, value, i);
                    return;
                }
            }
            tab[i] = new Entry(key, value);
            int sz = ++size;
            if (!cleanSomeSlots(i, sz) &amp;&amp; sz >= threshold)
                rehash();
        }
```

因此，当不需要`threadlocal`的时候还是显示调用`remove()`方法较好。

## 结语

线程本地存储本质就是**避免共享**，在使用中注意内存泄露问题和hash碰撞问题即可。使用还是很广泛的像spring中事务就用到`threadlocal`。

----

如有错误欢迎指正！

个人公众号:yes的练级攻略

有相关面试进阶(分布式、性能调优、经典书籍pdf)资料等待领取
