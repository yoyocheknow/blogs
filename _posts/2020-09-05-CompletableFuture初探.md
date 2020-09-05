---
layout: post
title: 【翻译】CompletableFuture初探
date: 2020-09-05
author: yoyocheknow
tags: [Java, 译文]
comments: true
toc: true
pinned: true
---
翻译自https://www.baeldung.com/java-completablefuture

###1.介绍
这篇文章主要介绍一下Java 8 Concurrency API 改进的类-CompletableFuture的一些基本功能和使用。

###2.Java中的异步计算
异步计算是比较难以表述的，通常我们希望任何运算都是一步一步的，但是在异步运算的例子中，像callback回调这种行为通常会穿插在代码中，或者互相嵌套调用。当我们需要在其中的一个步骤处理可能会发生的异常时，事情会变得更加糟糕。

Future接口是Java5添加用来处理异步运算的，但是Future并没有提供将异步运算和处理异常相结合的方法。

在Java8，CompletableFuture类诞生了。它不仅实现了Future接口还实现了CompletionStage接口。CompletionStage接口定义了异步计算的步骤和其他步骤相结合的协议。

CompetableFuture同时提供了composing,combining,执行异步运算任务，处理异常等50多种方法。

虽然这样一个大的API是颠覆性的，但是这些大部分可以用几个清晰明了的case来演示。

###3.使用CompletableFuture作为一个简单的Future
首先，CompletanbleFuture类是Future接口的一个实现，所以你可以不需要额外的实现逻辑使将它作为一个普通的Future使用。

例如，你可以创建一个无参数构造方法的类实例来代表一些异步结果，通过消费者来调用处理，或者使用complete方法。消费者可以通过get方法来获取结果，但是在使用get方法时会被阻塞住直到异步的结果出来。

下面的例子中，我们创建来一个CompletableFuture实例，接着在另一个线程中获取它的计算然后再立马返回Future。当计算完成时，通过Future提供的complete方法可以结束这个运算。
{% highlight ruby linenos%}
public Future<String> calculateAsync() throws InterruptedException {
    CompletableFuture<String> completableFuture 
      = new CompletableFuture<>();
 
    Executors.newCachedThreadPool().submit(() -> {
        Thread.sleep(500);
        completableFuture.complete("Hello");
        return null;
    });
 
    return completableFuture;
}

 Future<String> completableFuture = calculateAsync();
 String result = completableFuture.get();
 System.out.println("result="+result);
{% endhighlight %}
结果：

result=Hello
为了实现异步运算，我们使用Executor API来实现。但是CompletableFuture的创建和结束方法可以被任何并发机制或者包含原生线程的API使用。

注意： calculateAsync( ）返回的是一个Future实例。

我们可以简单地调用Future实例的get方法来获取结果，但是会block住。同时需要注意的是get()方法会抛出一些异常，比如在运行期间发生的异常：ExecutionException，当一个线程执行一个方法时被中断的异常：InterruptedException。

如果你已经直到来计算的结果，你可以使用静态方法completedFuture()，带有一个参数代表它计算的一个结果。那么Future的get()方法将永远不会阻塞，相反会立马返回结果。
{% highlight ruby linenos%}
Future<String> completableFuture = CompletableFuture.completedFuture("Hello");
String result = completableFuture.get();
System.out.println("result="+result);
{% endhighlight %}
结果：

result=Hello
如果发生了另外一个场景，比如你想取消Future的执行。假设我们不想获取结果了，想要取消异步任务的执行，这个需求可以用Future的cancel()方法实现。这个方法会接受一个boolean参数mayInterruptIfRunning，但是在CompletableFuture中没有这个作用。因为对于CompletanleFuture来说不会使用中断来处理进程。

下面是修改过的异步方法：
{% highlight ruby linenos%}
public Future<String> calculateAsyncWithCancellation() throws InterruptedException {
    CompletableFuture<String> completableFuture = new CompletableFuture<>();
 
    Executors.newCachedThreadPool().submit(() -> {
        Thread.sleep(500);
        completableFuture.cancel(false);
        return null;
    });
 
    return completableFuture;
}
{% endhighlight %}
当我们使用Future的get()方法阻塞地去获取结果时，就会抛出CancellationException的异常/
{% highlight ruby linenos%}
Future<String> future = calculateAsyncWithCancellation();
future.get(); // CancellationException
{% endhighlight %}
结果：
{% highlight ruby linenos%}
Exception in thread "main" java.util.concurrent.CancellationException
	at java.base/java.util.concurrent.CompletableFuture.cancel(CompletableFuture.java:2475)
	at main.java.com.yoyocheknow.java8.CompletableFutureTest.lambda$calculateAsyncWithCancellation$6(CompletableFutureTest.java:126)
	at java.base/java.util.concurrent.FutureTask.run(FutureTask.java:264)
	at java.base/java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1128)
	at java.base/java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:628)
	at java.base/java.lang.Thread.run(Thread.java:830)
{% endhighlight %}
###4.CompletableFuture的封装方法
上面的代码我们都使用了并发的机制去执行的（比如使用线程池），但是如果我们想要跳过那些无用的模版方法仅仅是简单地异步执行一些code改如何做呢？

静态方法例如：runAsync()和supplyAsync()，都允许我们创建一个CompletableFuture 实例，相应地都是Runnable和Supplier实例。

由于Java8的新特性，Runnable和Suplier都允许通过lambda表达式传递他们的实例。

Runnable接口和之前在线程中使用的老接口一样，它不允许有返回值。

Supplier接口是一个没有参数的通用方法，它返回一个参数化的值。它允许我们通过传递一个lambda表达式作为一个Supplier实例，然后返回其结果。代码如下：
{% highlight ruby linenos%}
CompletableFuture<String> future= CompletableFuture.supplyAsync(() -> "Hello");
assertEquals("Hello", future.get());
{% endhighlight %}

###5.处理异步计算结果
处理计算结果的最通用的办法是传入一个方法。thenApply()方法就可以做到：接受一个Function实例，使用它处理结果并且返回一个包含结果的Future。
{% highlight ruby linenos%}
CompletableFuture<String> completableFuture
  = CompletableFuture.supplyAsync(() -> "Hello");
 
CompletableFuture<String> future = completableFuture
  .thenApply(s -> s + " World");
 
assertEquals("Hello World", future.get());
{% endhighlight %}

如果你不需要在Future链条下返回一个值，你可以使用Consumer方法接口的一个实例。它接受一个参数，但是不返回值。

对于不需要返回值的情况，有一种方法可以供你选择-thenAccept()方法，它接受一个Consumer然后传递给她计算的结果，最终调用future.get()方法会返回空值。

{% highlight ruby linenos%}
CompletableFuture<String> completableFuture
  = CompletableFuture.supplyAsync(() -> "Hello");
 
CompletableFuture<Void> future = completableFuture
  .thenAccept(s -> System.out.println("Computation returned: " + s));
 
 System.out.print("result="+future.get());
 {% endhighlight %}
 
输出：
{% highlight ruby linenos%}
Computation returned: Hello
result=null
{% endhighlight %}
最后，如果你既不需要计算返回的值也不想要在最后获取返回，那么那你可以传递一个Runnable lambda表达式给thenRun()方法。在下面这个离职中，future.get()方法调用后，我们仅仅打印在控制台。

{% highlight ruby linenos%}
CompletableFuture<String> completableFuture 
  = CompletableFuture.supplyAsync(() -> "Hello");
 
CompletableFuture<Void> future = completableFuture
  .thenRun(() -> System.out.println("Computation finished."));
 
System.out.print("result="+future.get());
{% endhighlight %}
输出：

Computation finished.
result=null
###6.合并特性
CompletableFuture API最好的功能就是在计算步骤的链条当中，合并CompletableFuture实例的能力。

CompletableFuture 链的结果就是允许成链和组合的能力，这种方式在编程语言当中是独一无二的。并且经常用来作为单一职责设计模式。

下面的例子种我们使用thenCompose()方法来依次合并两个Futures。注意，这个方法返回一个CompletableFuture实例。这个方法的参数是上一个计算步骤的结果。这样就允许我们在下一个CompletableFuture的lambda表达式中使用这个值。
{% highlight ruby linenos%}
CompletableFuture<String> completableFuture 
  = CompletableFuture.supplyAsync(() -> "Hello")
    .thenCompose(s -> CompletableFuture.supplyAsync(() -> s + " World"));
 
assertEquals("Hello World", completableFuture.get());
{% endhighlight %}

thenCompose()方法和thenApply()方法实现了构建单一模式的模版。他们特别像Java8中的Stream和Optional类中的map和flatMap方法。

上面两个方法都接受一个函数，然后将其应用于计算结果。但是thenCompose(flatMap)方法接受一个与之相同类型的函数。这个函数的结构体允许组合这些类的实例成一个模块。

如果你想执行两个独立的Futures，并且对他们的结果做一些操作，那么就可以使用thenCombine()方法，用它来接受一个Future，和一个带有两个参数的Funtion去处理两个Futures的结果。例如：
{% highlight ruby linenos%}
CompletableFuture<String> completableFuture 
  = CompletableFuture.supplyAsync(() -> "Hello")
    .thenCombine(
                CompletableFuture.supplyAsync(() -> " World"), (s1, s2) -> s1 + s2)
                 );
 
assertEquals("Hello World", completableFuture.get());
{% endhighlight %}
你想对两个Future的结果做一些操作，更为简单的例子是，不需要传递到Future chain中任何结果值。使用thenAcceptBoth()方法即可。
{% highlight ruby linenos%}
CompletableFuture future = CompletableFuture.supplyAsync(() -> "Hello")
  .thenAcceptBoth(CompletableFuture.supplyAsync(() -> " World"),
    (s1, s2) -> System.out.println(s1 + s2));
{% endhighlight %}

###7.thenApply()和thenCompose()之间的不同
在我们前面的部分当中，我们展示了thenApply()和thenCompose()的使用方法。两个API被用在CompletableFuture调用链当中，但是这两者之间使用是不同的。

####7.1 thenApply()
这个方法被用来处理前一个调用的结果。然而，一个关键的点就是要记住，所有调用的返回值将会被组合。所以这个方法在我们想要转化一个CompletableFuture结果时是有用的。
{% highlight ruby linenos%}
CompletableFuture<Integer> finalResult = compute().thenApply(s-> s + 1);
{% endhighlight %}

####7.2 thenCompose()
这个方法和thenApply()类似都会返回一个新的执行阶段。thenCompose()使用前面的阶段作为参数。它将会打平并且返回一个带有结果的Future，而不是像thenApply()那样嵌套的结果。

CompletableFuture<Integer> computeAnother(Integer i){
    return CompletableFuture.supplyAsync(() -> 10 + i);
}
CompletableFuture<Integer> finalResult = compute().thenCompose(this::computeAnother);
所以要想对CompletableFuture方法以链的形式呈现，最好使用thenCompose()方法。另外注意，这两个方法之间的不同是和map() 与flatMap()之间的不同是类似的。

###8.以并行的方式运行多个Futures
当我们需要以并行的方式执行多个Futures时，我们通常想要等待他们所有执行完然后处理他们组合的结果。

CompletableFuture的allOf静态方法允许等待所有Futures的完成。

CompletableFuture<String> future1  
  = CompletableFuture.supplyAsync(() -> "Hello");
CompletableFuture<String> future2  
  = CompletableFuture.supplyAsync(() -> "Beautiful");
CompletableFuture<String> future3  
  = CompletableFuture.supplyAsync(() -> "World");
 
CompletableFuture<Void> combinedFuture 
  = CompletableFuture.allOf(future1, future2, future3);
 
combinedFuture.get();
 
assertTrue(future1.isDone());
assertTrue(future2.isDone());
assertTrue(future3.isDone());
注意，CompletableFuture.allOf() 的返回值是一个CompletableFuture<Void> 。这个方法的限制就在于他不能发挥所有Futures组合的结果。相反你不得不手动从Futures里获取结果。幸运的是CompletableFeature.join()方法和Java 8 Streams API让他变得更加简单：

String combined = Stream.of(future1, future2, future3)
  .map(CompletableFuture::join)
  .collect(Collectors.joining(" "));
 
assertEquals("Hello Beautiful World", combined);
CompletableFuture.join()方法与get()方法类似。但是当Future没有正常完成时，他会抛出一个异常。在Stream.map()方法中使用join使之成为可能。

###9.处理错误
对于异步计算过程中的错误处理，throw/catch常用方法是一个流行的趋势。

相对于在语法块中捕获异常，CompletableFuture 允许你使用handle()方法去处理异常。这个方法接受两个参数：一个是计算结果（如果成功的话），一个是抛出的异常（如果计算过程中没有正常完成）。

在下面的例子中，当异步运算因为没有提供name而异常结束时，我们使用handle()方法来获取一个默认值。

String name = null;
 
CompletableFuture<String> completableFuture  
  =  CompletableFuture.supplyAsync(() -> {
      if (name == null) {
          throw new RuntimeException("Computation error!");
      }
      return "Hello, " + name;
  })}).handle((s, t) -> s != null ? s : "Hello, Stranger!");
 
assertEquals("Hello, Stranger!", completableFuture.get());
考虑另外一种场景，假设我们想手动结束一个带有返回值的Future，但是也有能力去处理异常，那么completeExceptionally()方法就很适合。下面的例子中completableFuture.get()方法抛出来一个RuntimeException 

CompletableFuture<String> completableFuture = new CompletableFuture<>();
  
completableFuture.completeExceptionally(
  new RuntimeException("Calculation failed!"));
 
completableFuture.get(); // ExecutionException
上面的例子中我们可以使用handle()方法处理异步异常，但是使用get()方法我们可以使用一个更为典型的异步异常处理方式。

###10.异步方法
CompletableFuture类的API中大部分都有两个以Async结尾的变种。这些方法通常是用于运行另一个线程的执行步骤。

带有Async后缀的方法通过调用一个线程来执行下一个执行阶段。Async方法中没有使用Executor线程池参数的 会使用公共的fork/join线程池框架比如ForkJoinPool.commonPool()来执行。带有Executor参数的Async方法通过Executor线程池运行下一步。

下面是一个使用Future实例处理计算结果过程的例子。这个唯一可见的不同之处是thenApplyAsync()方法。下面方法的一个参数应用是被修饰为ForkJoinTask实例。这样就允许你的计算可以并行执行，可以是你的系统资源更加有效。

CompletableFuture<String> completableFuture  
  = CompletableFuture.supplyAsync(() -> "Hello");
 
CompletableFuture<String> future = completableFuture
  .thenApplyAsync(s -> s + " World");
 
assertEquals("Hello World", future.get());
###11.总结
这篇文章我们总结了CompletableFuture类的一些方法和常用的用法。源码见：

https://github.com/eugenp/tutorials/tree/master/core-java-modules/core-java-concurrency-basic


