---
layout: post
title: 使用 Future 进行并发编程
excerpt: Future 的概念，Future 的使用，Future 的组合
categories: articles
author: ktn
date: 2016-08-27
modified: 2016-08-27
tags:
  - Asynchrony
  - Concurrency
  - Java
  - Scala
comments: true
share: true
---

# Future 的概念

在编程的时候，常常会遇到需要并行处理一些代码，最原始的做法就是创建不同的线程进行处理，但是线程之间的同步处理非常麻烦而且容易出错，如果要同时的到几个线程的结果并且通过这些结果进行进一步的计算，则需要共享变量或者进行线程间通信，无论如何都非常难以处理。Future 能够提供一个高层的抽象，使得这类处理更为方便。

Future 作为一个代理对象代表一个可能完成也可能未完成的值 [^wiki-future]，通过对 Future 进行操作，能够获取内部的计算是否已经完成，是否出现异常，计算结果是什么等信息。

[^wiki-future]: [Futures and promises - Wikipedia](https://en.wikipedia.org/wiki/Futures_and_promises)

# Java 中的 `Future`

Java 很早就提供了 `Future` 接口[^java-future]，使用起来大概是这样的：

[^java-future]: [Future - JavaSE 8 References](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/Future.html)

```java
ExecutorService executor = new ForkJoinPool();
Future<String> future = executor.submit(new Callable<String>() {
    public String call() {
        return someWork();
}});
// ...
String res = future.get();
// ...
```

这样，在 future 外的代码执行的同时，future 内的代码也能执行，最后再获取 future 内的结果并进行一些计算。注意如果 future 的结果尚未计算出来，那么 `Future` 的 `get` 方法将阻塞当前线程。Java 提供了一些对 future 的监控和操作手段：

```java
boolean cancel(boolean mayInterruptIfRunning)
V get()
V get(long timeout, TimeUnit unit)
boolean isCancelled()
boolean isDone()
```

从 API 可以看出，`Future` 里的值是只能获取不能设置的，因为 future 从概念上来说就是对某一值的一个只读的占位符，这个值可能暂时没有计算出来，也可能永远无法计算出来。对于这个值在计算过程中出现异常而无法获取的情况，在 Java 中使用 `get` 方法抛出的异常来表示，`get` 方法会抛出如下 3 个异常：

> `CancellationException` - if the computation was cancelled
>
> `InterruptedException` - if the current thread was interrupted while waiting
>
> `ExecutionException` - if the computation threw an exception

但是，这套 `Future` 的 API 存在一些问题，首先，要获取一个 future 的计算结果必须要同步获取，这就对灵活性产生了很多限制，另外，这套 API 没有提供 future 间的组合方式，复杂的组合变得困难。考虑如下情况：

```java
Future<String> fa = executor.submit(() -> workA());
Future<String> fb = executor.submit(() -> workB(fa.get()));
Future<String> fc = executor.submit(() -> workC(fa.get(), fb.get()));
String res = fc.get();
// ...
```

这里使用了 Java 8 的 Lambda 表达式，每一次的 `get` 操作都可能抛出异常，再加上嵌套的 future，使得调试变得困难，阻塞调用更是可能会对性能造成很大影响。

## 对 Java `Future` API 的改进

要改善 Java 的 `Future` API，首先要提供接口让用户从阻塞调用变为非阻塞调用，也就是使用回调函数（使用 Scala 表示）：

```scala 
trait Future[T] {
  def onComplete(callback: T => Unit): Unit
  def cancel(mayInterruptIfRunning: Boolean): Boolean
  def isCancelled: Boolean
  def isDone: Boolean
}
```

使用的时候大概是这样：

```scala
future.onComplete(res => consume(res));
```

使用回调函数之后，在 `onComplete` 处就不会阻塞线程，当 future 所代理的值被计算出来后，通过 `onComplete` 注册的回调函数就会被调用，从而执行所需的代码。但很快可以发现，由于整个过程是异步的，所以这样无法直接使用 `try-catch` 来捕获异常，如前所述，Java 的 `Future` 的 `get` 方法的完整声明其实是这样的：

```java 
V get() throws InterruptedException, ExecutionException
V get(long timeout, TimeUnit unit)
    throws InterruptedException, ExecutionException, TimeoutException
```

所以，`get` 的声明其实不是 `() => T` 而是 `() => Try[T]`，对应 `get` 改为异步后的 `onComplete` 不应该是 `T => Unit`，而应该是 `Try[T] => Unit`。

```scala 
trait Future[T] {
  def onComplete(callback: Try[T] => Unit): Unit
  def onSuccess(callback: T => Unit): Unit
  def onFailure(callback: Throwable => Unit): Unit
  // ...
}
```

但是，使用回调函数也是难以组合的，操作起来甚至比直接使用阻塞的 `get` 调用还要复杂，很容易就陷入 JavaScript 程序常常遇到的 **Callback Hell** [^callback-hell]：

[^callback-hell]: [Callback Hell - A guide to writing asynchronous JavaScript programs](http://callbackhell.com/)

```scala 
fa.onComplete {
  case Success(a) =>
    val fb = // create future b using a
    fb.onComplete {
      case Success(b) =>
        // do something
      case Failure(tb) => // handle failure
    }
  case Failure(ta) => // handle failure
}
```

`onComplete` 返回的是 `Unit`，所以整个回调过程完全是通过副作用的形式产生效果的。现在希望实现的是，尽管最后的回调是副作用过程，但在进行 future 间组合时不用关心这些副作用，也就是希望能将组合和最终的计算实现分离。一个常用的组合子就是 `map` 了：

```scala 
trait Future[T] {
  def map[R](f: T => R): Future[R]
  // ...
}
```

`map` 方法产生一个新的 future，如果原 future 成功计算出了结果，那么新的 future 的结果就是将 `f` 作用于原 future 所代理的值上所得出的结果，如果原 future 出现了异常导致失败，或者 `f` 的调用过程出现异常，那么新的 future 将会失败。

为了能处理某一 future 的构建依赖于前一个 future 的结果的情况，光有 `map` 这个组合子还不够，我们还需要 `flatMap`：

```scala 
trait Future[T] {
  def flatMap[R](f: T => Future[R]): Future[R]
  // ...
}
```

这样配合 Scala 的 for-comprehension，就可以这样写：

```scala 
val f = for {
  sa <- Future { getStringA }
  sb <- Future { getStringB(sa) }
  sc <- Future { getStringC(sa, sb) }
} yield sc

f.onComplete(/* do something */)
```

可见，虽然最后的回调是有副作用的，但是前面的组合根本不需要考虑这些副作用，可以将不同的 future 进行纯的组合，只有在最后才会碰到一次副作用调用。除了 `map` 和 `flatMap` 之外，Scala 的 `Future` 还提供了更多的组合子，例如用于从异常中恢复的 `recover`，用于筛选结果的 `filter` 等 [^scala-doc-future] [^scala-api-future]。

[^scala-doc-future]: [Futures and Promises - Scala Documentation](http://docs.scala-lang.org/overviews/core/futures.html)
[^scala-api-future]: [Future - Scala API References](http://www.scala-lang.org/files/archive/nightly/docs/library/scala/concurrent/Future.html)

虽然 Scala 的这一套 API 很优雅，但是受限于 Java 的语法，这个设计在 Java 上却无法直接照搬，例如上面那段代码中的 for-comprehension 部分将被翻译成：

```scala 
(Future { getStringA }).flatMap { sa =>
  (Future { getStringB(sa) }).flatMap { sb =>
    (Future { getStringC(sa, sb)}).map { sc => sc }
  }
}
```

这样的嵌套处理显然是很难看的，所以，Java 8 设计了另外一套 API，在 `CompletableFuture` 中 [^java-completable-future]，举例而言：

[^java-completable-future]: [CompletableFuture - JavaSE 8 References](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html)

```java
Class CompletableFuture<T> {
  
    public <U> CompletableFuture<U> thenApplyAsync
        (Function<? super T,? extends U> fn) { ... }
        
    public <U,V> CompletableFuture<V> thenCombineAsync
        (CompletionStage<? extends U> other, 
         BiFunction<? super T,? super U,? extends V> fn) { ... }
         
    public CompletableFuture<Void> thenAcceptAsync
        (Consumer<? super T> action) { ... }
    
    // ...
}
```

正如之前的在 [协变、逆变与不变](../covariant-and-contravariant) 一文中提到的一样，Java 的型变是在使用的地方进行限制的，所以这个方法签名非常难看，但使用的时候其实并不复杂，例如：

```java 
CompletableFuture<String> fa = // create completable future
CompletableFuture<String> fb = fa.thenApplyAsync((a) -> getStringB(a));
ComputerFuture<String> fc = 
    fa.thenCombineAsync(fb, (a, b) -> getStringC(a, b));
      .thenAcceptAsync((c) -> /* do something */);
```

虽然仍然没有 Scala 优雅，但是在 Java 的语法局限下，这个已经是一个比较好的处理了。

总之，在 Java 8 之后，应该使用新的 API 来操作 future，以便能更加简便地处理并发和异步代码。另外，对于 API 设计而言，要尽可能加强组件的可组合性，将无法组合的部分抽离，只有在最后才调用。

## 参考
