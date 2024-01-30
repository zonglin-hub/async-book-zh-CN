# 应用：构建执行器

Rust 的 `Future` 是惰性的：它们不会干任何事，除非它们被驱动执行。一个驱动future类型的 方法是在 `async` 函数中使用 `.await` 调用，但这只是将问题抛到上一层：谁来跑在顶层 `async` 函数返回的 future 实例呢？为此，我们需要执行 `Future` 的执行器。

`Future` 执行器会拿一组顶层 `Future` 去跑 `poll` 方法，无论这些 `Future` 能否进展。通常， 执行器会 `poll` 一个 future 实例来启动。当 `Future` 通过调用 `wake()` 方法来指示他们准备好继续 进展，执行器就会把它们放入队列并再一次 `poll`，重复这一过程直到 `Future` 完成。

在这一小节，我们要写一个我们的简单执行器，能够并发地运行大量的顶层 future 实例。

这个例子中，我们依赖 `futures` 库的 `ArcWake` trait, 它提供了简便的构造 `Waker` 的方法。编辑 `Cargo.toml` 来引入新依赖：

```toml
[package]
name = "timer_future"
version = "0.1.0"
authors = ["XYZ Author"]
edition = "2018"

[dependencies]
futures = "0.3"
```

然后，我们在 `src/main.rs`中引入以下：

```rust,ignore
use {
    futures::{
        future::{BoxFuture, FutureExt},
        task::{waker_ref, ArcWake},
    },
    std::{
        future::Future,
        sync::mpsc::{sync_channel, Receiver, SyncSender},
        sync::{Arc, Mutex},
        task::{Context, Poll},
        time::Duration,
    },
    // The timer we wrote in the previous section:
    timer_future::TimerFuture,
};
```

我们的执行器通过给通道（channel）发送任务来工作。执行器会从通道中拉取事件并执行它们。当 一个任务准备好进一步工作（被唤醒了）时，它会被放到 channel 的末尾，来让自己再次被调度。

在设计时，执行器自身只需要任务通道的接收端。用户会拿到发送端，那样它们就可以开辟（spawn） 新的 future 实例。任务自身仅仅是能够重新调度自身的 future， 所以我们要把它们作为和发送端 配对的 future 存储。这个发送端能够让任务重新排队。

```rust,ignore
/// Task executor that receives tasks off of a channel and runs them.
struct Executor {
    ready_queue: Receiver<Arc<Task>>,
}

/// `Spawner` spawns new futures onto the task channel.
#[derive(Clone)]
struct Spawner {
    task_sender: SyncSender<Arc<Task>>,
}

/// A future that can reschedule itself to be polled by an `Executor`.
struct Task {
    /// In-progress future that should be pushed to completion.
    ///
    /// The `Mutex` is not necessary for correctness, since we only have
    /// one thread executing tasks at once. However, Rust isn't smart
    /// enough to know that `future` is only mutated from one thread,
    /// so we need to use the `Mutex` to prove thread-safety. A production
    /// executor would not need this, and could use `UnsafeCell` instead.
    future: Mutex<Option<BoxFuture<'static, ()>>>,

    /// Handle to place the task itself back onto the task queue.
    task_sender: SyncSender<Arc<Task>>,
}

fn new_executor_and_spawner() -> (Executor, Spawner) {
    // Maximum number of tasks to allow queueing in the channel at once.
    // This is just to make `sync_channel` happy, and wouldn't be present in
    // a real executor.
    const MAX_QUEUED_TASKS: usize = 10_000;
    let (task_sender, ready_queue) = sync_channel(MAX_QUEUED_TASKS);
    (Executor { ready_queue }, Spawner { task_sender })
}
```

我们来加一个方法，让开辟器（spawner）更容易开辟新 future 吧。这个方法会获取一个 future 类型， 把它装箱并把它变成一个 FutureObj 对象，然后把这对象放到新的 `Arc<Task>` 里面。这个 `Arc<Task>` 能够放到执行器的队列中。

```rust,ignore
impl Spawner {
    fn spawn(&self, future: impl Future<Output = ()> + 'static + Send) {
        let future = future.boxed();
        let task = Arc::new(Task {
            future: Mutex::new(Some(future)),
            task_sender: self.task_sender.clone(),
        });
        self.task_sender.send(task).expect("too many tasks queued");
    }
}
```

为了轮询 future，我们需要创建 `Waker`。正如在[任务唤醒小节]中讨论到，`Waker` 负责调度任务在 `wake` 函数调用时再次轮询。记住，`Waker` 告诉执行器具体哪个任务已经准备好了，这使得它们 可以只轮询已经准备好的 future。创建 `Waker` 的最简单方法是实现 `ArcWake` trait，然后使用  `waker_ref` 或者 `.into_waker()` 函数来把 `Arc<impl ArcWake>` 转变成 `Waker`。我们来给我们的任务实现 `ArcWake`，以便它们可以变成 `Waker` 并且被唤醒：

```rust,ignore
impl ArcWake for Task {
    fn wake_by_ref(arc_self: &Arc<Self>) {
        // Implement `wake` by sending this task back onto the task channel
        // so that it will be polled again by the executor.
        let cloned = arc_self.clone();
        arc_self
            .task_sender
            .send(cloned)
            .expect("too many tasks queued");
    }
}
```

当 `Waker` 从 `Arc<Task>` 创建了之后，调用 `wake()` 函数会拷贝一份 `Arc`，发送到任务的通道去。 我们的执行器就会拿到这个任务并轮询它。我们来实现这个吧：

```rust,ignore
impl Executor {
    fn run(&self) {
        while let Ok(task) = self.ready_queue.recv() {
            // Take the future, and if it has not yet completed (is still Some),
            // poll it in an attempt to complete it.
            let mut future_slot = task.future.lock().unwrap();
            if let Some(mut future) = future_slot.take() {
                // Create a `LocalWaker` from the task itself
                let waker = waker_ref(&task);
                let context = &mut Context::from_waker(&*waker);
                // `BoxFuture<T>` is a type alias for
                // `Pin<Box<dyn Future<Output = T> + Send + 'static>>`.
                // We can get a `Pin<&mut dyn Future + Send + 'static>`
                // from it by calling the `Pin::as_mut` method.
                if future.as_mut().poll(context).is_pending() {
                    // We're not done processing the future, so put it
                    // back in its task to be run again in the future.
                    *future_slot = Some(future);
                }
            }
        }
    }
}
```

恭喜！我们现在有一个能干活的 future 执行器了。我们甚至能用它来运行 `async/.await` 代码和定制的 future，例如我们前面写的 `TimeFuture`：

```rust,edition2018,ignore
fn main() {
    let (executor, spawner) = new_executor_and_spawner();

    // Spawn a task to print before and after waiting on a timer.
    spawner.spawn(async {
        println!("howdy!");
        // Wait for our timer future to complete after two seconds.
        TimerFuture::new(Duration::new(2, 0)).await;
        println!("done!");
    });

    // Drop the spawner so that our executor knows it is finished and won't
    // receive more incoming tasks to run.
    drop(spawner);

    // Run the executor until the task queue is empty.
    // This will print "howdy!", pause, and then print "done!".
    executor.run();
}
```

[任务唤醒小节]: ./03_wakeups.md
