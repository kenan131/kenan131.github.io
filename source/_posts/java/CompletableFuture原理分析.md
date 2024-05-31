---
title: CompletableFuture原理分析
date: 2024-05-29 17:03:01
tags: JAVA
categories: JAVA
---

### 1、介绍

`CompletableFuture` 是 Java 8 引入的一个类，它提供了一个可异步计算的`Future`，并且可以以函数式编程的方式对其进行操作。`CompletableFuture` 允许开发者以声明式的方式处理异步逻辑，使得代码更加简洁和易于理解。

以下是一些关于 `CompletableFuture` 的关键点：

- **异步执行**: `CompletableFuture` 可以异步地执行任务，并且可以在任务完成时获取结果。
- **链式调用**: 它支持链式调用，可以对异步结果进行处理，例如通过 `.thenApply()`, `.thenAccept()`, `.thenRun()` 等方法。
- **异常处理**: `CompletableFuture` 允许开发者通过 `.exceptionally()` 方法来处理异步操作中发生的异常。
- **组合操作**: 可以使用 `CompletableFuture` 的 `allOf()`, `anyOf()`, `runAfterBoth()` 等方法来组合多个异步操作。
- **转换操作**: 可以通过 `.thenApply()`, `.thenCompose()` 等方法将一个 `CompletableFuture` 的结果转换为另一个 `CompletableFuture`。
- **取消操作**: `CompletableFuture` 支持取消操作，如果一个异步操作不再需要，可以通过 `.cancel(true)` 方法来取消它。
- **并行执行**: 可以并行执行多个 `CompletableFuture` 操作，并通过 `CompletableFuture.allOf()` 等待它们全部完成。
- **线程池管理**: `CompletableFuture` 可以与 `Executor` 一起使用，来控制异步操作在哪个线程池中执行。
- **响应式编程**: 它支持响应式编程模式，可以与 Java 的其他响应式编程库（如 RxJava）进行集成。

### 2、方法使用

#### 2.1、异步执行

通过CompletableFuture.runAsync调用runAsync静态方法，该方法内部参数为Runnable对象无返回值。

```java
CompletableFuture<Void> future = CompletableFuture.runAsync(new Runnable() {
    @Override
    public void run() {
		//执行逻辑
    }
});
// 异步获取结果
future.whenCompleteAsync((result,throwable)->{
    if (throwable != null) {
        System.out.println("Error: " + throwable.getMessage());
    }
    System.out.println(result);
});
// 同步获取结果
Integer result = future.get();
System.out.println(result);
```

通过CompletableFuture.supplyAsync调用supplyAsync静态方法，该方法内部参数为Supplier对象，可以返回执行结果。

```java
CompletableFuture<Integer> future = CompletableFuture.supplyAsync(new Supplier<Integer>() {
    @Override
    public Integer get() {
        return null;
    }
});
// 异步获取结果
future.whenCompleteAsync((result,throwable)->{
    if (throwable != null) {
        System.out.println("Error: " + throwable.getMessage());
    }
    System.out.println(result);
});
// 同步获取结果
Integer result = future.get();
System.out.println(result);
```

#### 2.2、一元依赖

一元依赖：即一个CompletableFuture对象需要在另一个CompletableFuture对象执行完之后再执行。

```java
CompletableFuture<Integer> future1 = CompletableFuture.supplyAsync(new Supplier<Integer>() {
    @Override
    public Integer get() {
        return 999;
    }
});
CompletableFuture future2 = future1.thenApply(result -> {
    System.out.println(result);
    return result + 1;
});
//结果为1000
System.out.println(future2.get());
```

代码中future2对象会在future1对象执行完毕后再执行自身的代码逻辑，并且能够获取到future1对象的返回结果，如果future1没有返回结果则为null对象。

#### 2.3、二元依赖

一元依赖：即一个CompletableFuture对象需要在两个CompletableFuture对象都执行完之后再执行。

```java
CompletableFuture<Integer> future1 = CompletableFuture.supplyAsync(new Supplier<Integer>() {
    @Override
    public Integer get() {
        return 2;
    }
});
CompletableFuture<Integer> future2 = CompletableFuture.supplyAsync(new Supplier<Integer>() {
    @Override
    public Integer get() {
        return 3;
    }
});
CompletableFuture<Integer> future3 = future1.thenCombine(future2, (r1, r2) -> {
    return r1 + r2 + 5;
});
//结果为10
System.out.println(future3.get());
```

代码中future3对象会在future1和future2对象都执行结束后再执行。并且能够获取到两个future的执行结果。

#### 2.4、多元依赖

CompletableFuture.allOf能够将多个future对象组合成一个future对象，组合的多个future对象能够并行执行。

```java
CompletableFuture<Integer> future1 = CompletableFuture.supplyAsync(new Supplier<Integer>() {
    @Override
    public Integer get() {
        return 2;
    }
});
CompletableFuture<Integer> future2 = CompletableFuture.supplyAsync(new Supplier<Integer>() {
    @Override
    public Integer get() {
        return 3;
    }
});
CompletableFuture<Integer> future3 = CompletableFuture.supplyAsync(new Supplier<Integer>() {
    @Override
    public Integer get() {
        return 5;
    }
});
CompletableFuture<Void> future4 = CompletableFuture.allOf(future1, future2, future3);
CompletableFuture<Integer> result = future4.thenApply(v -> {
    Integer r1 = future1.join();
    Integer r2 = future2.join();
    Integer r3 = future3.join();
    return r1 + r2 + r3;
});
//结果为10
System.out.println(result.get());
```

#### 2.5、其他方法

**join()**: 这个方法会返回 `CompletableFuture` 的结果，如果 `CompletableFuture` 尚未完成，则会阻塞直到完成。与 `get()` 不同，`join()` 不会抛出 `InterruptedException`，如果线程在等待时被中断，它将返回 `null`。

```j'a'v
String result = future.join(); // 阻塞等待结果，但不会抛出 InterruptedException
System.out.println(result);
```

**isDone()**: 这个方法用于检查 `CompletableFuture` 是否已经完成。它不会阻塞，但需要你手动检查状态。

**getNow()**: 这个方法会立即返回 `CompletableFuture` 的结果，如果 `CompletableFuture` 尚未完成，则返回 `null`（对于 `Future` 接口的实现）或调用 `getNow(U valueIfNotComplete)` 方法提供的默认值。

**get(timeout, unit)**: 这个方法类似于 `get()`，但它允许你指定一个超时时间。如果在超时时间内 `CompletableFuture` 没有完成，则会抛出 `TimeoutException`。

**handle()**: 这个方法接受一个 `BiConsumer`，它既可以处理结果也可以处理异常。如果 `CompletableFuture` 正常完成，`BiConsumer` 的第一个参数将被调用；如果 `CompletableFuture` 由于异常而完成，第二个参数将被调用。

### 3、原理分析

CompletableFuture中包含两个字段：**result**和**stack**。result用于存储当前CF的结果，stack（Completion）表示当前CF完成后需要触发的依赖动作（Dependency Actions），去触发依赖它的CF的计算，依赖动作可以有多个（表示有多个依赖它的CF），以栈的形式存储，stack表示栈顶元素。

CompletableFuture的回调时基于观察者模式实现，当有依赖当前对象的执行结果，则会被push到stack中，待当前对象的run方法执行结束后则会从栈中pop出来，依次执行。

```java
// 抽象类，stack中主要入栈的是该类的实现类。
abstract static class Completion extends ForkJoinTask<Void>
    implements Runnable, AsynchronousCompletionTask {
    volatile Completion next;      // Treiber stack link

    /**
     * Performs completion action if triggered, returning a
     * dependent that may need propagation, if one exists.
     *
     * @param mode SYNC, ASYNC, or NESTED
     */
    abstract CompletableFuture<?> tryFire(int mode);

    /** Returns true if possibly still triggerable. Used by cleanStack. */
    abstract boolean isLive();

    public final void run()                { tryFire(ASYNC); } //实现Runnable对象的run方法，调用抽象方法，子类实现
    public final boolean exec()            { tryFire(ASYNC); return true; }
    public final Void getRawResult()       { return null; }
    public final void setRawResult(Void v) {}
}
```

```java
// 方法执行完毕后，设置result值
final boolean completeValue(T t) {
    return UNSAFE.compareAndSwapObject(this, RESULT, null,
                                       (t == null) ? NIL : t);
}
```

#### 3.1、异步执行，同步获取结果

以CompletableFuture.supplyAsync方法为例

```java
CompletableFuture<Integer> future = CompletableFuture.supplyAsync(() -> {
    // 执行逻辑 并 返回结果
    return 1;
});
future.get(); // 此方法会同步阻塞
```

问题1：CompletableFuture.supplyAsync方法的执行逻辑怎么执行的，怎么封装的CompletableFuture。

问题2：get方法如何在没有结果的情况下阻塞线程，以及如何在执行完毕后对阻塞的线程进行唤醒。

下面将会针对上述两个问题进行讲解。

首先supplyAsync方法会调用asyncSupplyStage方法，并传入forkJoin默认的线程池作为参数。

而asyncSupplyStage方法，第一步新建CompletableFuture对象。第二步则是新建AsyncSupply对象传入刚刚新建的CompletableFuture对象和用户的执行逻辑。第三步则用线程池执行AsyncSupply对象并返回CompletableFuture对象，AsyncSupply对象实现了Runnable类，所以可以由线程池去执行。

```java
private static final Executor asyncPool = useCommonPool ?
    ForkJoinPool.commonPool() : new ThreadPerTaskExecutor();
// 1
public static <U> CompletableFuture<U> supplyAsync(Supplier<U> supplier) {
    return asyncSupplyStage(asyncPool, supplier); // 调用asyncSupplyStage方法，这里会传入forkJoin默认的线程池	
}
// 2
static <U> CompletableFuture<U> asyncSupplyStage(Executor e,
                                                 Supplier<U> f) {
    if (f == null) throw new NullPointerException();
    CompletableFuture<U> d = new CompletableFuture<U>();
    e.execute(new AsyncSupply<U>(d, f)); // 这里封装AsyncSupply对象，传入用户封装的supplier对象（执行逻辑）
    return d;//用线程池去执行，主线程返回封装的future对象。
}
```

```java
// 3
static final class AsyncSupply<T> extends ForkJoinTask<Void>
        implements Runnable, AsynchronousCompletionTask {
    CompletableFuture<T> dep; Supplier<T> fn;
    AsyncSupply(CompletableFuture<T> dep, Supplier<T> fn) {
        this.dep = dep; this.fn = fn;
    }

    public final Void getRawResult() { return null; }
    public final void setRawResult(Void v) {}
    public final boolean exec() { run(); return true; }
	// 核心逻辑，这里当被提交到线程池的时候，则会执行run方法。
    public void run() {
        CompletableFuture<T> d; Supplier<T> f;
        if ((d = dep) != null && (f = fn) != null) {
            dep = null; fn = null;
            if (d.result == null) {
                try {
                    // f.get是用户封装的执行方法，当get执行完毕后，执行completeValue方法
                    // completeValue 方法其实就是设置result的方法，上面有方法代码。
                    d.completeValue(f.get()); 
                } catch (Throwable ex) {
                    // 设置result为异常结果。
                    d.completeThrowable(ex);
                }
            }
            // 核心逻辑，这里会通知所有依赖当前future的对象，被观察对象执行完毕后，通知观察者对象执行逻辑
            d.postComplete();
        }
    }
}
```

postComplete方法为公共方法， 即所有的对象在执行结束后，都会调用postComplete，尝试从stack中pop值进行处理。

```java
// 4
final void postComplete() {
    /*
     * On each step, variable f holds current dependents to pop
     * and run.  It is extended along only one path at a time,
     * pushing others to avoid unbounded recursion.
     */
    CompletableFuture<?> f = this; Completion h;
    // 只要stack中还有对象则会一直循环执行
    while ((h = f.stack) != null ||
           (f != this && (h = (f = this).stack) != null)) {
        CompletableFuture<?> d; Completion t;
        // 获取stack头对象，并设置下一个对象为栈顶
        if (f.casStack(h, t = h.next)) {
            if (t != null) {
                if (f != this) {
                    pushStack(h);
                    continue;
                }
                h.next = null;    // detach
            }
            // 尝试调用tryFire逻辑，这里是上面Completion抽象类里面的抽象方法，由子类实现。
            // 这个tryFire类似于观察者自己内部实现的观察方法
            f = (d = h.tryFire(NESTED)) == null ? this : d;
        }
    }
}
```

**问题1总结：**当执行CompletableFuture.supplyAsync方法后，会将用户的执行代码封装到一个AsyncSupply对象中，并由线程池执行代码逻辑，该对象新建时会传入方法返回的CompletableFuture对象，后续代码执行完毕后则将该future对象栈中的依赖对象弹出来进行通知。

开始第二个问题讲解：

```java
// 1
public T get() throws InterruptedException, ExecutionException {
    Object r;
    // 主要是waitingGet方法，reportGet方法是在唤醒后封装返回结果。
    return reportGet((r = result) == null ? waitingGet(true) : r);
}

private static <T> T reportGet(Object r)
    throws InterruptedException, ExecutionException {
    if (r == null) // by convention below, null means interrupted
        throw new InterruptedException();
    if (r instanceof AltResult) {
        Throwable x, cause;
        if ((x = ((AltResult)r).ex) == null)
            return null;
        if (x instanceof CancellationException)
            throw (CancellationException)x;
        if ((x instanceof CompletionException) &&
            (cause = x.getCause()) != null)
            x = cause;
        throw new ExecutionException(x);
    }
    @SuppressWarnings("unchecked") T t = (T) r;
    return t;
}
```

```java
// 2 逻辑：自旋一段时间，如果还是没有结果则进行阻塞。
private Object waitingGet(boolean interruptible) {
    Signaller q = null;
    boolean queued = false;
    int spins = -1;
    Object r;
    while ((r = result) == null) {
        if (spins < 0)
            spins = SPINS;
        else if (spins > 0) { // 这里spins扣减小于等于0了才会走下面的流程
            if (ThreadLocalRandom.nextSecondarySeed() >= 0)
                --spins;// 扣减 
        }
        else if (q == null)
            // 这里新建Signaller对象，这个是Completion的实现类
            // 这个对象会保存当前线程。
            q = new Signaller(interruptible, 0L, 0L);
        else if (!queued)
            queued = tryPushStack(q); // 这里会入栈
        else if (interruptible && q.interruptControl < 0) {
            q.thread = null;
            cleanStack();
            return null;
        }
        else if (q.thread != null && result == null) {
            try {
                //这里会执行真正的阻塞逻辑，后续等结果执行完后，会从stack中pop出Signaller。将当前线程唤醒。
                ForkJoinPool.managedBlock(q);
            } catch (InterruptedException ie) {
                q.interruptControl = -1;
            }
        }
    }
    if (q != null) {
        q.thread = null;
        if (q.interruptControl < 0) {
            if (interruptible)
                r = null; // report interruption
            else
                Thread.currentThread().interrupt();
        }
    }
    postComplete();
    return r;
}

//3 Signaller对象实现的tryFire方法
final CompletableFuture<?> tryFire(int ignore) {
    Thread w; // no need to atomically claim
    if ((w = thread) != null) {
        thread = null;
        // 唤醒线程
        LockSupport.unpark(w);
    }
    return null;
}
```

**问题2总结：**在执行get方法后，首先会自旋获取结果，如果自旋失败，则封装Signaller对象，将该对象压future对象的栈中。等待future对象执行完后调用postComplete方法，该方法从stack中pop出Signaller对象，调用Signaller对象的tryFire方法唤醒线程。

#### 3.2、有依赖的future对象如何执行

在第二节讲了一元依赖，二元依赖和多元依赖是怎么使用的。本小节将讲解这些有依赖的future对象是怎么执行的，执行的原理是什么。

问题：有依赖的future对象也是和上面同步获取结果一样由被依赖对象进行通知吗？

以CompletableFuture.thenApply为例，样例代码如下，future1对象依赖于future对象。

```java
CompletableFuture<Integer> future = CompletableFuture.supplyAsync(() -> {
    // 执行逻辑 并 返回结果
    try {
        Thread.sleep(50000);
    } catch (InterruptedException e) {
        throw new RuntimeException(e);
    }
    return 1;
});
CompletableFuture<Integer> future1 = future.thenApply((result) -> {
    return result + 1;
});
Integer result = future1.get();
System.out.println(result);
```

thenApply方法主要就是调用了uniApplyStage方法，这方法的第一个参数是线程池参数，如果用户为传入则直接传入null值，用户传入了自定义线程池则将用户的线程池传入该方法中。

```java
// 1
public <U> CompletableFuture<U> thenApply(
    Function<? super T,? extends U> fn) {
    return uniApplyStage(null, fn); // 这里第一个参数是线程池，因用户未传入线程池所以传入null值。
}
```

uniApplyStage方法，新建一个CompletableFuture对象，这个对象就是方法的返回值。如果用户未传入线程池就会调用一次uniApply方法，这个方法后面介绍。基本第一次调用uniApply方法返回的都是false，然后进入if条件内部，封装UniApply对象并push进栈中。

```java
private <V> CompletableFuture<V> uniApplyStage(
    Executor e, Function<? super T,? extends V> f) {
    if (f == null) throw new NullPointerException();
    CompletableFuture<V> d =  new CompletableFuture<V>();
    if (e != null || !d.uniApply(this, f, null)) {
        UniApply<T,V> c = new UniApply<T,V>(e, d, this, f);
        push(c); // 入栈
        c.tryFire(SYNC); // 这方法还会尝试判断依赖对象是否执行完毕
    }
    return d;
}
```

uniApply方法，主要就是检测依赖的future对象是否执行完毕，如果执行完毕并且当前对象没有执行，则会使用当前线程去执行用户封装的执行逻辑，即f.apply(s)并传入依赖对象的执行结果。如果用户在构建的时间传入了线程池则使用线程池进行执行。执行完后设置result。

```java
final <S> boolean uniApply(CompletableFuture<S> a,
                           Function<? super S,? extends T> f,
                           UniApply<S,T> c) {
    Object r; Throwable x;
    if (a == null || (r = a.result) == null || f == null)
        return false;// 如果依赖对象没有执行完毕，则返回false
    tryComplete: if (result == null) {
        if (r instanceof AltResult) {
            if ((x = ((AltResult)r).ex) != null) {
                completeThrowable(x, r); // 如果依赖对象返回异常，则当前future对象封装异常结果。
                break tryComplete;
            }
            r = null;
        }
        try {
            // claim方法是判断当前UniApply对象是否有线程池，如果有则用线程池执行。
            if (c != null && !c.claim())
                return false;
            @SuppressWarnings("unchecked") S s = (S) r;//依赖对象的执行结果
            //f.apply即执行用户逻辑。completeValue将执行结果封装到当前CompletableFuture对象中。
            completeValue(f.apply(s));
        } catch (Throwable ex) {
            completeThrowable(ex);//封装异常。
        }
    }
    return true;
}
```

后续则调用future1对象的get方法去同步获取结果。当然也可以设置回调方法。

**总结：**有依赖的对象执行其实和上面讲异步执行，同步获取结果是一样的触发逻辑，即被依赖对象执行完后调用postComplete方法将栈中压入的对象pop出来进行处理，只不过入栈的对象是不同的，一元依赖的对象为UniApply，二元依赖的对象为BiApply，多元依赖为BiRelay。对象的执行则由被依赖对象的执行线程去执行，如果用户传入了自定义线程池则由线程池执行。被依赖对象通常是用的forkjoin线程池中的线程进行处理。