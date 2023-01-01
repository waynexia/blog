---
title: The Hidden Control Flow
date: 2022-12-04 16:06:19
tags: ["Rust"]
---

# The Problem
This post is talking about a "weird" [problem](https://github.com/GreptimeTeam/greptimedb/issues/350) we encountered in [GreptimeDB](https://github.com/GreptimeTeam/greptimedb). And, a little spoiler, it's about the "async cancellation".

Let's first describe the (simplified) scenario. We observed metadata corruption in a long-run test. A series number is duplicated, but it should be increased monotonously. The update logic is very straightforward -- load value from an in-memory atomic counter, persist the new series number to file, and then update the in-memory counter. The entire procedure is serialized (`file` is a mutable reference):
``` rust
    async fn update_metadata(file: &mut File, counter: AtomicU64) -> Result<()> {
        let next_number = counter.load(Ordering::Relaxed) + 1;
        persist_number(file, number).await?;
        counter.fetch_add(1, Ordering::Relaxed);
    }
```

For some reason, we are not using `fetch_add` here, though it does work, and if we've done so, then the story won't happen 🤪 For example, we don't want to update the in-memory counter when this procedure fails halfway, like operation `persist_number()` cannot write to the file. Here the sweet sugar `?` stands for such a situation. We know clearly that this function call may fail, and if it fails the caller returns early to propagate the error. So we will handle it carefully with that in mind.

But things become tricky with `.await`, the hidden control flow comes due to async cancellation.

# Async Cancellation
## async task and runtime
If you have figured out the shape of the "criminal", you may want to skip this section. Instead, I'll start with some pseudocode to show what happens in the "await point", and how it interacts with the runtime. First is `poll_future`, it comes from the `Future`'s [`poll`](https://doc.rust-lang.org/std/future/trait.Future.html#tymethod.poll) function, as every `async fn`'s we write will be desugared to an anonymous `Future` implementation.
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

`async` block usually contains other async functions, like `update_metadata` and `persist_number`. Say `persist_number` is a sub-async-task of `update_metadata`. Each `.await` point will be expanded to something like `poll_future` -- `await`ing the subtask's output and make progress when the subtask is ready. Here we need to wait `persist_number`'s task returns `Ready` before we update the counter, otherwise we cannot do it.

And the second one is a (toy) runtime, which is in response to poll futures delivered to it. In GreptimeDB we use [`tokio`](https://docs.rs/tokio/latest/tokio/) as our runtime. An async runtime may have tons of features and logic, but the most basic one is to poll: as the name tells, keep running unfinished tasks until they finish (but consider the things I'm going to write later, the "until" might not be a proper word).
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

That is it, a very minimalist model of future and runtime. Combining these two functions, you will find that in some aspects, it is just a loop (again, I've omitted lots of details to keep tight on the topic; the real world is way more complex). I want to stress the thing that each `.await` imply one or more function calls (call to `poll()` or `poll_future()`). This is the "hidden control flow" in the title and the place cancellation takes effort.
``` rust
    fn run() {
        loop {
            if let Ready(result) = task.poll() {
                break result;
            }
        }
    }
```
Know it is not hard, but thinking about it is not easy (at least for me). I've stared at these lines for minutes after I narrow the scope down to as simple as the first code snippet. I know the problem is definitely in the `.await`. But don't know whether too many successful async calls have numbed me or my mental model hasn't linked these two points. The bulky garbage, me, spent a whole sleepless night doubting life and the world.

## cancellation
So far is the standard part. We will then talk about cancellation, which is runtime-dependent. Though many runtimes in rust have similar behavior, this is not a required feature, i.e., a runtime can not support cancellation at all like [this toy](https://github.com/waynexia/texn). And I'll take tokio as an example because the story happens there. Other runtimes may be similar.

In tokio, one can use [`JoinHandle::abort()`](https://docs.rs/tokio/latest/tokio/task/struct.JoinHandle.html#method.abort) to cancel a task. Tasks have a "cancel marker bit" tracks whether it's cancelled. And if the runtime finds a task is cancelled, it will kill that task (code from [here](https://github.com/tokio-rs/tokio/blob/00bf5ee8a855c28324fa4dff3abf11ba9f562a85/tokio/src/runtime/task/state.rs#L283-L291)):
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

The theory behind async cancellation is also elementary. It's just the runtime gives up to keep polling your task when it's not yet finished, just like `?` or even tougher because we cannot catch this cancellation like `Err`. But does it means that we need to take care of every single `.await`? It would be very annoying. Take this metadata updating as an example. If we have to consider this, we need to check if the file is consistent with the memory state and revert the persisted change if found inconsistency. Well... 🫠 In some aspects, yes. The runtime can literally do anything to your future. But the good thing is that most of them are disciplined.

# Runtime Behavior
This section will discuss what I would expect from a runtime and what we can get for now.

## marker trait
I want the runtime not to cancel my task unconditionally and turn to the type system for help. This is wondering if there is a marker trait like `CancelSafe`. For the word cancellation safety, tokio has said about it in its [documentation](https://docs.rs/tokio/latest/tokio/macro.select.html#cancellation-safety):

> To determine whether your own methods are cancellation safe, look for the location of uses of  `.await` . This is because when an asynchronous method is cancelled, that always happens at an  `.await` . If your function behaves correctly even if it is restarted while waiting at an  `.await` , then it is cancellation safe.  

That is, whether a task is safe to be cancelled. This is definitely an "attribute" of an async task. You can find that tokio has a long list of what is safe and what isn't in the library from the above link. And, in some ways, I think it's just like the [`UnwindSafe`](jhttps://doc.rust-lang.org/std/panic/trait.UnwindSafe.html) marker. Both are "*this sort of control flow is not always anticipated*" and "*has the possibility of causing subtle bugs*".

With such a `CancelSafe` trait, we can tell the runtime if our spawned future is ok to be cancelled, and we promise the "cancelling" control flow is carefully handled. And if without this, means we don't want the task to be cancelled. Simple and clear. This is also an approach for the runtimes to require their users (like you and me) to check if their tasks are able to be cancelled. Take `timeout()` as an example:
``` rust
    /// The marker trait
    trait CancelSafe {}
    
    /// Only cancellable task can be timeout-ed
    pub fn timeout<F>(duration: Duration, future: F) -> Timeout<F> where
        F: Future + CancelSafe
    {}
```

## volunteer cancel
Another approach is to cancel voluntarily. Like the [cooperative cancellation in Kotlin](https://kotlinlang.org/docs/cancellation-and-timeouts.html#cancellation-is-cooperative), it has an [`isActive`](https://kotlinlang.org/api/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/is-active.html) method for a task to check if it is cancelled. And this is only a tester method, to cancel or not is fully dependent on the task itself. I paste an example from Kotlin's document below, the "cooperative cancellation" happens in line 5. This way brings the "hidden control flow" on the table and makes it more natural to consider and handle the cancellation just like `Option` or `Result`.
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

And this is not hard to achieve in my opinion. Tokio already has the [`Cancelled` bit](https://github.com/tokio-rs/tokio/blob/00bf5ee8a855c28324fa4dff3abf11ba9f562a85/tokio/src/runtime/task/state.rs#L41) and [`CancellationToken`](https://docs.rs/tokio-util/latest/tokio_util/sync/struct.CancellationToken.html). But they look a bit different than what I describe. And after all of these, we need runtime to give the cancellation right back to our task. Or the situation might not have big difference.

## explicit detach
Can we force the runtime not to cancel our tasks at present? In tokio we can "detach" a task to the background by dropping the `JoinHandle`. A detached task means there is no foreground handle to the spawned task, and in some aspect, others cannot wrap a `timeout` or `select` over it, making it un-cancellable. And the problem in the very beginning is solved in this way.

> A  `JoinHandle`  *detaches* the associated task when it is dropped, which means that there is no longer any handle to the task, and no way to  `join` on it.  

Though there is the functionality, I would wonder if it's better to have an explicit `detach` method like [`glommio's`](https://docs.rs/glommio/0.7.0/glommio/struct.Task.html#method.detach), or even a `detach` method in the runtime like `spawn`, which doesn't return the `JoinHandle`. But these are trifles. A runtime usually won't cancel a task for no reason, and in most cases, it's required by the users. But sometimes you haven't noticed that, like those "unselect branches" in `select`, or the logic in `tonic`'s request handler. And if we are sure that a task is ready for cancellation, explicit detach may prevent it from tragedy sometime.

# Back To The Problem
So far everything is clear. Let's start to wipe out this bug! First is, why our future is cancelled? Through the function call graph we can easily find the entire process procedure is executed in-place in `tonic`'s request licensing runtime, and it's common for an internet request to have a timeout behavior. And the solution is also simple, just detaching the server processing logic into another runtime to prevent it from cancelled with the request. Only [a few lines](https://github.com/GreptimeTeam/greptimedb/pull/376/files#diff-9756dcef86f5ba1d60e01e41bf73c65f72039f9aaa057ffd03f3fc2f7dadfbd0R46-R54):
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

Now all fine.
