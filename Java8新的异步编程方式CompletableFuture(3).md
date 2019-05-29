---
title: Java8新的异步编程方式 CompletableFuture(三)
date: 2019-05-14 11:46:27:027
tags: [Java,RxJava] 
published: true
feature: 
---
> 本文转载自 [https://juejin.im/post/59eae7636fb9a045117044c6](https://juejin.im/post/59eae7636fb9a045117044c6) 

前面两篇文章已经整理了CompletableFuture大部分的特性，本文会整理完CompletableFuture余下的特性，以及将它跟RxJava进行比较。

## 3.6 Either

Either 表示的是两个CompletableFuture，当其中任意一个CompletableFuture计算完成的时候就会执行。

| 方法名 | 描述 |
|:--:|:--: |
| acceptEither(CompletionStage<? extends T> other, Consumer<? super T> action) | 当任意一个CompletableFuture完成的时候，action这个消费者就会被执行。 |
| acceptEitherAsync(CompletionStage<? extends T> other, Consumer<? super T> action) | 当任意一个CompletableFuture完成的时候，action这个消费者就会被执行。使用ForkJoinPool |
| acceptEitherAsync(CompletionStage<? extends T> other, Consumer<? super T> action, Executor executor) | 当任意一个CompletableFuture完成的时候，action这个消费者就会被执行。使用指定的线程池 |
`Random random =  Random();

        CompletableFuture<String> future1 = CompletableFuture.supplyAsync(()->{

             {
                Thread.sleep(random.nextInt());
            }  (InterruptedException e) {
                e.printStackTrace();
            }

             ;
        });

        CompletableFuture<String> future2 = CompletableFuture.supplyAsync(()->{

             {
                Thread.sleep(random.nextInt());
            }  (InterruptedException e) {
                e.printStackTrace();
            }

             ;
        });

        CompletableFuture<Void> future =  future1.acceptEither(future2,str->System.out.println(+str));

         {
            future.get();
        }  (InterruptedException e) {
            e.printStackTrace();
        }  (ExecutionException e) {
            e.printStackTrace();
        }`

执行结果：The future is from future1 或者 The future is from future2。
因为future1和future2，执行的顺序是随机的。

applyToEither 跟 acceptEither 类似。

| 方法名 | 描述 |
|:--:|:--: |
| applyToEither(CompletionStage<? extends T> other, Function<? super T,U> fn) | 当任意一个CompletableFuture完成的时候，fn会被执行，它的返回值会当作新的CompletableFuture<U>的计算结果。 |
| applyToEitherAsync(CompletionStage<? extends T> other, Function<? super T,U> fn) | 当任意一个CompletableFuture完成的时候，fn会被执行，它的返回值会当作新的CompletableFuture<U>的计算结果。使用ForkJoinPool |
| applyToEitherAsync(CompletionStage<? extends T> other, Function<? super T,U> fn, Executor executor) | 当任意一个CompletableFuture完成的时候，fn会被执行，它的返回值会当作新的CompletableFuture<U>的计算结果。使用指定的线程池 |
`Random random =  Random();

        CompletableFuture<String> future1 = CompletableFuture.supplyAsync(()->{

             {
                Thread.sleep(random.nextInt());
            }  (InterruptedException e) {
                e.printStackTrace();
            }

             ;
        });

        CompletableFuture<String> future2 = CompletableFuture.supplyAsync(()->{

             {
                Thread.sleep(random.nextInt());
            }  (InterruptedException e) {
                e.printStackTrace();
            }

             ;
        });

        CompletableFuture<String> future =  future1.applyToEither(future2,str->+str);

         {
            System.out.println(future.get());
        }  (InterruptedException e) {
            e.printStackTrace();
        }  (ExecutionException e) {
            e.printStackTrace();
        }`

执行结果也跟上面的程序类似。

## 3.7 其他方法

allOf、anyOf是CompletableFuture的静态方法。

### 3.7.1 allOf

| 方法名 | 描述 |
|:--:|:--: |
| allOf(CompletableFuture<?>... cfs) | 在所有Future对象完成后结束，并返回一个future。 |

allOf()方法所返回的CompletableFuture，并不能组合前面多个CompletableFuture的计算结果。于是我们借助Java 8的Stream来组合多个future的结果。
`CompletableFuture<String> future1 = CompletableFuture.supplyAsync(() -> );

        CompletableFuture<String> future2 = CompletableFuture.supplyAsync(() -> );

        CompletableFuture<String> future3 = CompletableFuture.supplyAsync(() -> );

        CompletableFuture.allOf(future1, future2, future3)
                .thenApply(v ->
                Stream.of(future1, future2, future3)
                        .map(CompletableFuture::join)
                        .collect(Collectors.joining()))
                .thenAccept(System.out::print);`

执行结果：

`tony cafei aaron`

### 3.7.2 anyOf

| 方法名 | 描述 |
|:--:|:--: |
| anyOf(CompletableFuture<?>... cfs) | 在任何一个Future对象结束后结束，并返回一个future。 |
`Random rand =  Random();
        CompletableFuture<String> future1 = CompletableFuture.supplyAsync(() -> {
             {
                Thread.sleep(rand.nextInt());
            }  (InterruptedException e) {
                e.printStackTrace();
            }
             ;
        });
        CompletableFuture<String> future2 = CompletableFuture.supplyAsync(() -> {
             {
                Thread.sleep(rand.nextInt());
            }  (InterruptedException e) {
                e.printStackTrace();
            }
             ;
        });
        CompletableFuture<String> future3 = CompletableFuture.supplyAsync(() -> {
             {
                Thread.sleep(rand.nextInt());
            }  (InterruptedException e) {
                e.printStackTrace();
            }
             ;
        });

        CompletableFuture<Object> future =  CompletableFuture.anyOf(future1,future2,future3);

         {
            System.out.println(future.get());
        }  (InterruptedException e) {
            e.printStackTrace();
        }  (ExecutionException e) {
            e.printStackTrace();
        }`

使用anyOf()时，只要某一个future完成，就结束了。所以执行结果可能是"from future1"、"from future2"、"from future3"中的任意一个。

anyOf 和 acceptEither、applyToEither的区别在于，后两者只能使用在两个future中，而anyOf可以使用在多个future中。

## 3.8 CompletableFuture异常处理

CompletableFuture在运行时如果遇到异常，可以使用get()并抛出异常进行处理，但这并不是一个最好的方法。CompletableFuture本身也提供了几种方式来处理异常。

### 3.8.1 exceptionally

| 方法名 | 描述 |
|:--:|:--: |
| exceptionally(Function fn) | 只有当CompletableFuture抛出异常的时候，才会触发这个exceptionally的计算，调用function计算值。 |
`CompletableFuture.supplyAsync(() -> )
                .thenApply(s -> {
                    s = ;
                     length = s.length();
                     length;
                }).thenAccept(i -> System.out.println(i))
                .exceptionally(t -> {
                    System.out.println( + t);
                     ;
                });`

执行结果：

`Unexpected error:java.util.concurrent.CompletionException: java.lang.NullPointerException`

对上面的代码稍微做了一下修改，修复了空指针的异常。

`CompletableFuture.supplyAsync(() -> )
                .thenApply(s -> {

                     length = s.length();
                     length;
                }).thenAccept(i -> System.out.println(i))
                .exceptionally(t -> {
                    System.out.println( + t);
                     ;
                });`

执行结果：

``

### 3.8.2 whenComplete

whenComplete 在上一篇文章其实已经介绍过了，在这里跟exceptionally的作用差不多，可以捕获任意阶段的异常。如果没有异常的话，就执行action。
`CompletableFuture.supplyAsync(() -> )
                .thenApply(s -> {
                    s = ;
                     length = s.length();
                     length;
                }).thenAccept(i -> System.out.println(i))
                .whenComplete((result, throwable) -> {

                     (throwable != ) {
                       System.out.println(+throwable);
                    }  {
                        System.out.println(result);
                    }

                });`

执行结果：

`Unexpected error:java.util.concurrent.CompletionException: java.lang.NullPointerException`

跟whenComplete相似的方法是handle，handle的用法在上一篇文章中也已经介绍过。

# 四. CompletableFuture VS Java8 Stream VS RxJava1 & RxJava2

CompletableFuture 有很多特性跟RxJava很像，所以将CompletableFuture、Java 8 Stream和RxJava做一个相互的比较。

|  | composable | lazy | resuable | async | cached | push | back pressure |
|:--:|:--:|:--:|:--:|:--:|:--:|:--:|:--: |
| CompletableFuture | 支持 | 不支持 | 支持 | 支持 | 支持 | 支持 | 不支持 |
| Stream | 支持 | 支持 | 不支持 | 不支持 | 不支持 | 不支持 | 不支持 |
| Observable(RxJava1) | 支持 | 支持 | 支持 | 支持 | 支持 | 支持 | 支持 |
| Observable(RxJava2) | 支持 | 支持 | 支持 | 支持 | 支持 | 支持 | 不支持 |
| Flowable(RxJava2) | 支持 | 支持 | 支持 | 支持 | 支持 | 支持 | 支持 |

# 五. 总结

Java 8提供了一种函数风格的异步和事件驱动编程模型CompletableFuture，它不会造成堵塞。CompletableFuture背后依靠的是fork/join框架来启动新的线程实现异步与并发。当然，我们也能通过指定线程池来做这些事情。

CompletableFuture特别是对微服务架构而言，会有很大的作为。举一个具体的场景，电商的商品页面可能会涉及到商品详情服务、商品评论服务、相关商品推荐服务等等。获取商品的信息时（/productdetails?productid=xxx），需要调用多个服务来处理这一个请求并返回结果。这里可能会涉及到并发编程，我们完全可以使用Java 8的CompletableFuture或者RxJava来实现。

先前的文章：
[Java8新的异步编程方式 CompletableFuture(一)](https://juejin.im/post/59eae61b51882549fc512b34)
[Java8新的异步编程方式 CompletableFuture(二)](https://juejin.im/post/59eae6e4f265da430e4e4cb5)
