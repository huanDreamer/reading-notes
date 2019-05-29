---
title: Java8新的异步编程方式 CompletableFuture(二)
date: 2019-05-14 11:46:06:006
tags: [Java] 
published: true
feature: 
---
> 本文转载自 [https://juejin.im/post/59eae6e4f265da430e4e4cb5](https://juejin.im/post/59eae6e4f265da430e4e4cb5) 

[上一篇文章](https://juejin.im/post/59eae61b51882549fc512b34)，讲述了Future模式的机制、缺点，CompletableFuture产生的由来、静态工厂方法、complete()方法等等。

本文将继续整理CompletableFuture的特性。

## 3.3 转换

我们可以通过CompletableFuture来异步获取一组数据，并对数据进行一些转换，类似RxJava、Scala的map、flatMap操作。

### 3.3.1 map

| 方法名 | 描述 |
|:--:|:--: |
| thenApply(Function<? super T,? extends U> fn) | 接受一个Function<? super T,? extends U>参数用来转换CompletableFuture |
| thenApplyAsync(Function<? super T,? extends U> fn) | 接受一个Function<? super T,? extends U>参数用来转换CompletableFuture，使用ForkJoinPool |
| thenApplyAsync(Function<? super T,? extends U> fn, Executor executor) | 接受一个Function<? super T,? extends U>参数用来转换CompletableFuture，使用指定的线程池 |

thenApply的功能相当于将CompletableFuture<T>转换成CompletableFuture<U>。
`CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> );

        future = future.thenApply( Function<String, String>() {

            
            {

                 s + ;
            }
        }).thenApply( Function<String, String>() {
            
            {

                 s.toUpperCase();
            }
        });

         {
            System.out.println(future.get());
        }  (InterruptedException e) {
            e.printStackTrace();
        }  (ExecutionException e) {
            e.printStackTrace();
        }`

再用lambda表达式简化一下

`CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> )
                .thenApply(s -> s + ).thenApply(String::toUpperCase);

         {
            System.out.println(future.get());
        }  (InterruptedException e) {
            e.printStackTrace();
        }  (ExecutionException e) {
            e.printStackTrace();
        }`

执行结果：

`HELLO WORLD`

下面的例子，展示了数据流的类型经历了如下的转换：String -> Integer -> Double。

`CompletableFuture<Double> future = CompletableFuture.supplyAsync(() -> )
                .thenApply(Integer::parseInt)
                .thenApply(i->i*);

         {
            System.out.println(future.get());
        }  (InterruptedException e) {
            e.printStackTrace();
        }  (ExecutionException e) {
            e.printStackTrace();
        }`

执行结果：

``

### 3.3.2 flatMap

| 方法名 | 描述 |
|:--:|:--: |
| thenCompose(Function<? super T, ? extends CompletionStage<U>> fn) | 在异步操作完成的时候对异步操作的结果进行一些操作，并且仍然返回CompletableFuture类型。 |
| thenComposeAsync(Function<? super T, ? extends CompletionStage<U>> fn) | 在异步操作完成的时候对异步操作的结果进行一些操作，并且仍然返回CompletableFuture类型。使用ForkJoinPool。 |
| thenComposeAsync(Function<? super T, ? extends CompletionStage<U>> fn,Executor executor) | 在异步操作完成的时候对异步操作的结果进行一些操作，并且仍然返回CompletableFuture类型。使用指定的线程池。 |

thenCompose可以用于组合多个CompletableFuture，将前一个结果作为下一个计算的参数，它们之间存在着先后顺序。
`CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> )
                .thenCompose(s -> CompletableFuture.supplyAsync(() -> s + ));

         {
            System.out.println(future.get());
        }  (InterruptedException e) {
            e.printStackTrace();
        }  (ExecutionException e) {
            e.printStackTrace();
        }`

执行结果：

`Hello World`

下面的例子展示了多次调用thenCompose()

`CompletableFuture<Double> future = CompletableFuture.supplyAsync(() -> )
                .thenCompose(s -> CompletableFuture.supplyAsync(() -> s + ))
                .thenCompose(s -> CompletableFuture.supplyAsync(() -> Double.parseDouble(s)));

         {
            System.out.println(future.get());
        }  (InterruptedException e) {
            e.printStackTrace();
        }  (ExecutionException e) {
            e.printStackTrace();
        }`

执行结果：

``

## 3.4 组合

| 方法名 | 描述 |
|:--:|:--: |
| thenCombine(CompletionStage<? extends U> other, BiFunction<? super T,? super U,? extends V> fn) | 当两个CompletableFuture都正常完成后，执行提供的fn，用它来组合另外一个CompletableFuture的结果。 |
| thenCombineAsync(CompletionStage<? extends U> other, BiFunction<? super T,? super U,? extends V> fn) | 当两个CompletableFuture都正常完成后，执行提供的fn，用它来组合另外一个CompletableFuture的结果。使用ForkJoinPool。 |
| thenCombineAsync(CompletionStage<? extends U> other, BiFunction<? super T,? super U,? extends V> fn, Executor executor) | 当两个CompletableFuture都正常完成后，执行提供的fn，用它来组合另外一个CompletableFuture的结果。使用指定的线程池。 |

现在有CompletableFuture<T>、CompletableFuture<U>和一个函数(T,U)->V，thenCompose就是将CompletableFuture<T>和CompletableFuture<U>变为CompletableFuture<V>。
`CompletableFuture<String> future1 = CompletableFuture.supplyAsync(() -> );
        CompletableFuture<Integer> future2 = CompletableFuture.supplyAsync(() -> );

        CompletableFuture<Double> future = future1.thenCombine(future2, (s, i) -> Double.parseDouble(s + i));

         {
            System.out.println(future.get());
        }  (InterruptedException e) {
            e.printStackTrace();
        }  (ExecutionException e) {
            e.printStackTrace();
        }`

执行结果：

``
使用thenCombine()之后future1、future2之间是并行执行的，最后再将结果汇总。这一点跟thenCompose()不同。

thenAcceptBoth跟thenCombine类似，但是返回CompletableFuture类型。

| 方法名 | 描述 |
|:--:|:--: |
| thenAcceptBoth(CompletionStage<? extends U> other, BiConsumer<? super T,? super U> action) | 当两个CompletableFuture都正常完成后，执行提供的action，用它来组合另外一个CompletableFuture的结果。 |
| thenAcceptBothAsync(CompletionStage<? extends U> other, BiConsumer<? super T,? super U> action) | 当两个CompletableFuture都正常完成后，执行提供的action，用它来组合另外一个CompletableFuture的结果。使用ForkJoinPool。 |
| thenAcceptBothAsync(CompletionStage<? extends U> other, BiConsumer<? super T,? super U> action, Executor executor) | 当两个CompletableFuture都正常完成后，执行提供的action，用它来组合另外一个CompletableFuture的结果。使用指定的线程池。 |
`CompletableFuture<String> future1 = CompletableFuture.supplyAsync(() -> );
        CompletableFuture<Integer> future2 = CompletableFuture.supplyAsync(() -> );

        CompletableFuture<Void> future = future1.thenAcceptBoth(future2, (s, i) -> System.out.println(Double.parseDouble(s + i)));

         {
            future.get();
        }  (InterruptedException e) {
            e.printStackTrace();
        }  (ExecutionException e) {
            e.printStackTrace();
        }`

执行结果：

``

## 3.5 计算结果完成时的处理

当CompletableFuture完成计算结果后，我们可能需要对结果进行一些处理。

/#/#/#3.5.1 执行特定的Action

| 方法名 | 描述 |
|:--:|:--: |
| whenComplete(BiConsumer<? super T,? super Throwable> action) | 当CompletableFuture完成计算结果时对结果进行处理，或者当CompletableFuture产生异常的时候对异常进行处理。 |
| whenCompleteAsync(BiConsumer<? super T,? super Throwable> action) | 当CompletableFuture完成计算结果时对结果进行处理，或者当CompletableFuture产生异常的时候对异常进行处理。使用ForkJoinPool。 |
| whenCompleteAsync(BiConsumer<? super T,? super Throwable> action, Executor executor) | 当CompletableFuture完成计算结果时对结果进行处理，或者当CompletableFuture产生异常的时候对异常进行处理。使用指定的线程池。 |
`CompletableFuture.supplyAsync(() -> )
                .thenApply(s->s+)
                .thenApply(s->s+ )
                .thenApply(String::toLowerCase)
                .whenComplete((result, throwable) -> System.out.println(result));`

执行结果：

`hello world
 is completablefuture demo`

/#/#/#3.5.2 执行完Action可以做转换

| 方法名 | 描述 |
|:--:|:--: |
| handle(BiFunction<? super T, Throwable, ? extends U> fn) | 当CompletableFuture完成计算结果或者抛出异常的时候，执行提供的fn |
| handleAsync(BiFunction<? super T, Throwable, ? extends U> fn) | 当CompletableFuture完成计算结果或者抛出异常的时候，执行提供的fn，使用ForkJoinPool。 |
| handleAsync(BiFunction<? super T, Throwable, ? extends U> fn, Executor executor) | 当CompletableFuture完成计算结果或者抛出异常的时候，执行提供的fn，使用指定的线程池。 |
`CompletableFuture<Double> future = CompletableFuture.supplyAsync(() -> )
                .thenApply(s->s+)
                .handle((s, t) -> s !=  ? Double.parseDouble(s) : );

         {
            System.out.println(future.get());
        }  (InterruptedException e) {
            e.printStackTrace();
        }  (ExecutionException e) {
            e.printStackTrace();
        }`

执行结果：

``

在这里，handle()的参数是BiFunction，apply()方法返回R，相当于转换的操作。

`{

    
    ;

    
     <V> {
        Objects.requireNonNull(after);
         (T t, U u) -> after.apply(apply(t, u));
    }
}`

而whenComplete()的参数是BiConsumer，accept()方法返回void。

`{

    
    ;

    
    {
        Objects.requireNonNull(after);

         (l, r) -> {
            accept(l, r);
            after.accept(l, r);
        };
    }
}`
所以，handle()相当于whenComplete()+转换。

/#/#/#3.5.3 纯消费(执行Action)

| 方法名 | 描述 |
|:--:|:--: |
| thenAccept(Consumer<? super T> action) | 当CompletableFuture完成计算结果，只对结果执行Action，而不返回新的计算值 |
| thenAcceptAsync(Consumer<? super T> action) | 当CompletableFuture完成计算结果，只对结果执行Action，而不返回新的计算值，使用ForkJoinPool。 |
| thenAcceptAsync(Consumer<? super T> action, Executor executor) | 当CompletableFuture完成计算结果，只对结果执行Action，而不返回新的计算值 |

thenAccept()是只会对计算结果进行消费而不会返回任何结果的方法。
`CompletableFuture.supplyAsync(() -> )
                .thenApply(s->s+)
                .thenApply(s->s+ )
                .thenApply(String::toLowerCase)
                .thenAccept(System.out::print);`

执行结果：

`hello world
 is completablefuture demo`