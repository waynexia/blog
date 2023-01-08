---
title: 看不见的控制流
date: 2022-12-04 16:06:20
tags: ["Rust"]
---

# The Problem

这篇博客会讲一个在[GreptimeDB](https://github.com/GreptimeTeam/greptimedb)中遇到的“奇怪”的[问题](https://github.com/GreptimeTeam/greptimedb/issues/350)。先剧透一下是关于"async cancellation"的。

先描述一个简化的场景，我们在一个长时间运行的测试中发现了元信息损坏的问题，有一个应该单调递增的序列号重复了。它的更新逻辑非常简单：从一个原子变量中读取当前值，将新值写入文件里然后更新这个原子变量。整个流程都是串行化的（`file`是一个独占引用）。

``` rust
    async fn update_metadata(file: &mut File, counter: AtomicU64) -> Result<()> {
        let next_number = counter.load(Ordering::Relaxed) + 1;
        persist_number(file, number).await?;
        counter.fetch_add(1, Ordering::Relaxed);
    }
```

出于一些原因，我们没有在这里使用`fetch_add`（即使它可以用，并且如果用了就没这回儿事了🤪）。举例来说，当这个流程在中间出现错误的时候我们不希望更新内存中的计数器，比如`persist_number()`写入文件时失败就会从`?`提前返回。我们清楚地知道这里有些函数会失败，并且在失败的时候会提早结束执行来传播错误，所以编码的时候有注意这些问题。

但是到了`.await`这里事情就变得奇妙了起来，因为async cancellation带来了一个隐藏的控制流。

# Async Cancellation
## async task and runtime

如果这时候你已经猜到了是谁干的好事，可以跳过这一章节。或者让我从一些伪代码开始解释在"await point"那里到底发生了什么，以及runtime是如何参与其中的。首先是`poll_future`，来自定义在`Future`的[`poll`](https://doc.rust-lang.org/std/future/trait.Future.html#tymethod.poll)方法。我们写的异步方法都会被转化成类似这样子的一个匿名的`Future`实现。

``` rust
    fn poll_future() -> FutureOutput {
        match status_of_the_task {
            Ready(output) => {
                // the task is finished, and we have it output.
                // some logic
                return our_output;
            },
            Pending => {
                // it is not ready, we don't have the output.
                // thus we cannot make progress and need to wait
                return Pending;
            }
        }
    }
```

`async`块通常包含其他的异步方法，比如`update_metadata`和`persist_number`。这里把`persist_number`称为`update_metadata`的子异步任务。每个`.await`都会被展开成类似`poll_future`的东西，等待子任务的结果并继续执行。在这个例子中就是等待`persist_number`的结果返回`Ready`再更新计数器，否则不更新。

第二段伪代码是一个简化的runtime，它负责轮询(poll)异步任务直到它们完成（不过考虑到接下来要说的，“直到……完成”可能不是一个合适的表述）。在GreptimeDB中使用[`tokio`](https://docs.rs/tokio/latest/tokio/)作为runtime。现在的异步runtime可能有很多特性和功能，但是最基础的一个就是轮询这些任务。

``` rust
    fn runtime(&self) {
        loop {
            let future_tasks: Vec<Task> = self.get_tasks();
            for task in tasks {
                match task.poll_future(){
                    Ready(output) => {
                        // this task is finished. wake it with the result
                        task.wake(output);
                    },
                    Pending => {
                        // this task needs some time to run. poll it later
                        self.poll_later(task);
                    }
                }
            }
        }
    }
```

这就是一个非常简化的future和runtime的模型，就是把上面的两个方法结合起来。从某种方面来说，它们不过就是一个循环（真实的runtime非常复杂，这里为了内容集中省略了很多东西）。需要强调的是，每个`.await`都代表着一个或者多个函数调用(调用到`poll()`或者说是`poll_future()`的)。这就是标题所谓的“隐藏的控制流”，以及cancellation发生的地方。

``` rust
    fn run() {
        loop {
            if let Ready(result) = task.poll() {
                break result;
            }
        }
    }
```

弄明白原理并不复杂，但是（对我来说）能够想到并不自然。在把其他问题都排除掉之后我一直盯着和第一个示例里面差不多长的这几行，知道问题就发生在这里，在这个`.await`上。也不知道是太多次成功的异步函数调用麻痹了注意还是我的心智模型中没有把这两点联系起来，本垃圾花了一整晚来怀疑人生。

## cancellation

目前为止的内容是问题复盘的标准流程。我们接下来想展开讨论一下cancellation，它是与runtime的行为相关的。虽然rust中的很多runtime都有类似的行为，但是这不是一个必须的特性，比如我的这个[玩具runtime](https://github.com/waynexia/texn)就不支持cancellation。我会以tokio为例，因为这是这个问题发生的地方。其他的runtime可能也是类似的。

在tokio中，可以使用[`JoinHandle::abort()`](https://docs.rs/tokio/latest/tokio/task/struct.JoinHandle.html#method.abort)来取消一个task。task结构中有一个“cancel marker bit”来跟踪一个任务是否被取消了。如果它发现一个task被取消了，就会停止执行这个task。(代码在[这里](https://github.com/tokio-rs/tokio/blob/00bf5ee8a855c28324fa4dff3abf11ba9f562a85/tokio/src/runtime/task/state.rs#L283-L291)）

``` rust
                    // If the task is running, we mark it as cancelled. The thread
                    // running the task will notice the cancelled bit when it
                    // stops polling and it will kill the task.
                    //
                    // The set_notified() call is not strictly necessary but it will
                    // in some cases let a wake_by_ref call return without having
                    // to perform a compare_exchange.
                    snapshot.set_notified();
                    snapshot.set_cancelled();
```

async cancellation背后的逻辑也很简单，就是runtime放弃了继续轮询你的task，就和`?`差不多。某种程度上可能更棘手一点，因为我们不能像`Err`那样处理这个cancellation。不过这代表我们需要考虑每一个`.await`都有可能随时被cancel掉吗？这也太麻烦了。以本文的这个metadata更新的情况为例，如果把cancel纳入考虑范围，我们需要检查文件是否和内存中的状态一致，如果不一致就要回滚持久化的改动等等等等🫠 坏消息是，在某些方面答案是肯定的，runtime可以对你的future做任何事情。不过好在大多数情况下它们都还是很遵守规矩的。

# Runtime Behavior

这一小节打算讨论一下我所期望的runtime的行为，以及有哪些是我们现在已经可以用得上的。

## marker trait

首先自然希望runtime不要无条件地取消我的task，而是尝试通过类型系统来变得更友好，比如借助类似`CancelSafe`的marker trait 。对于cancellation safety这个词，tokio在它的[文档](https://docs.rs/tokio/latest/tokio/macro.select.html#cancellation-safety)中有提到：

> To determine whether your own methods are cancellation safe, look for the location of uses of  `.await` . This is because when an asynchronous method is cancelled, that always happens at an  `.await` . If your function behaves correctly even if it is restarted while waiting at an  `.await` , then it is cancellation safe.  

简单来说就是用来描述一个task是否可以安全地被取消掉。这肯定是一个async task的属性之一。在上面的链接中tokio维护了一个很长的列表，列出了哪些是安全的以及哪些是不安全的。看起来这和[`UnwindSafe`](jhttps://doc.rust-lang.org/std/panic/trait.UnwindSafe.html)这个marker trait很像。两者都是描述“这种控制流程并不总是被预料到的”，并且“有可能导致一些微妙的bug”的这样一种属性。

如果有这样一个`CancelSafe`的trait，我们有途径可以告诉runtime我们的异步任务是否可以安全地被取消掉，同时也是一种方式让用户承诺“cancelling”这个控制流程是被仔细处理过的。如果发现没有实现这个trait，那就意味着我们不希望这个task被取消掉，简单而清晰。以`timeout()`为例：

``` rust
    /// The marker trait
    trait CancelSafe {}
    
    /// Only cancellable task can be timeout-ed
    pub fn timeout<F>(duration: Duration, future: F) -> Timeout<F> where
        F: Future + CancelSafe
    {}
```

## volunteer cancel

另一个方式是让任务自愿地取消。就像Kotlin中的[cooperative cancellation](https://kotlinlang.org/docs/cancellation-and-timeouts.html#cancellation-is-cooperative)一样，它有一个[`isActive`](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/is-active.html)方法来检查一个task是否被取消掉。这只是一个检测方法，是否要取消完全取决于task本身。下面是Kotlin文档中的一个例子，cooperative cancellation发生在第5行。这种方式把“隐藏的控制流程”放在了桌面上，让我们能以一种更自然地来考虑和处理calcellation，就像`Option`或`Result`一样。

``` kotlin
    val startTime = System.currentTimeMillis()
    val job = launch(Dispatchers.Default) {
        var nextPrintTime = startTime
        var i = 0
        while (isActive) { // cancellable computation loop
            // print a message twice a second
            if (System.currentTimeMillis() >= nextPrintTime) {
                println("job: I'm sleeping ${i++} ...")
                nextPrintTime += 500L
            }
        }
    }
    delay(1300L) // delay a bit
    println("main: I'm tired of waiting!")
    job.cancelAndJoin() // cancels the job and waits for its completion
    println("main: Now I can quit.")
```

并且我认为这也不难实现，Tokio现在已经有了[`Cancelled` bit](https://github.com/tokio-rs/tokio/blob/00bf5ee8a855c28324fa4dff3abf11ba9f562a85/tokio/src/runtime/task/state.rs#L41)和[`CancellationToken`](https://docs.rs/tokio-util/latest/tokio_util/sync/struct.CancellationToken.html)。虽然看起来和期望的还有点不一样。最后还需要runtime把cancellation的权利交给task。否则情况可能没有什么大的不同。

## explicit detach

现在是否有手段能防止task被取消呢？在tokio中我们可以通过drop `JoinHandle`来“detach”一个任务到后台。一个detach task意味着没有前台的handle来控制这个任务，从某种意义上来说也就使得其他人不能在外面套一层`timeout`或`select`，从而间接地使它不会被取消执行。并且开头提到的问题就是通过这种方式解决的。

> A  `JoinHandle`  *detaches* the associated task when it is dropped, which means that there is no longer any handle to the task, and no way to  `join` on it.  

不过虽然有办法能够实现这个功能，我在想是否像[`glommio's`](https://docs.rs/glommio/0.7.0/glommio/struct.Task.html#method.detach)一样有一个显式的`detach`方法，类似一个不返回`JoinHandle`的`spawn`方法会更好。但这些都是琐碎的事情，一个runtime通常不会完全没有理由就取消一个task，并且在大多数情况下都是出于用户的要求，只不过有时候可能没有注意到，就像`select`中的那些“未选中的分支”或者`tonic`中请求处理的逻辑那样。所以如果我们确定一个task是不能被取消的话，显式地detach可能能预防某些悲剧的发生。

# Back To The Problem

目前为止所有问题都清晰了，让我们开始修复这个bug吧！首先，为什么我们的future会被取消呢？通过函数调用链路很容易就能发现整个处理过程都是在`tonic`的请求执行逻辑中就地执行的，而对于一个网络请求来说有一个超时行为是很常见的。解决方案也很简单，就是将服务器处理逻辑detach到另一个runtime中，从而防止它被取消。只需要[几行代码](https://github.com/GreptimeTeam/greptimedb/pull/376/files#diff-9756dcef86f5ba1d60e01e41bf73c65f72039f9aaa057ffd03f3fc2f7dadfbd0R46-R54)就能完成。

``` diff
    @@ -30,12 +40,24 @@ impl BatchHandler {
             }
             batch_resp.admins.push(admin_resp);
    
    -        for db_req in batch_req.databases {
    -            for obj_expr in db_req.exprs {
    -                let object_resp = self.query_handler.do_query(obj_expr).await?;
    -                db_resp.results.push(object_resp);
    +        let (tx, rx) = oneshot::channel();
    +        let query_handler = self.query_handler.clone();
    +        let _ = self.runtime.spawn(async move {
    +            // execute request in another runtime to prevent the execution from being cancelled unexpected by tonic runtime.
    +            let mut result = vec![];
    +            for db_req in batch_req.databases {
    +                for obj_expr in db_req.exprs {
    +                    let object_resp = query_handler.do_query(obj_expr).await;
    +
    +                    result.push(object_resp);
    +                }
                 }
```

现在一切正常了。