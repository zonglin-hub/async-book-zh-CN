# `Future` trait

`Future` trait 是 Rust 异步编程中心内容。它是一种异步计算，可以产生值（尽管这个值可以为空， 如 `()`）。简化版 future trait看起来可能像这样：

```rust
trait SimpleFuture {
    type Output;
    fn poll(&mut self, wake: fn()) -> Poll<Self::Output>;
}

enum Poll<T> {
    Ready(T),
    Pending,
}
```

Future 能通过调用 `poll` 的方式推进，这会尽可能地推进 future 到完成状态。如果 future 完成了， 那就会返回 `poll::Ready(result)`。如果 future 尚未完成，则返回 `poll::Pending`，并且安排 `wake()` 函数在 `Future` 准备好进一步执行时调用（译者注：注册回调函数）。当 `wake()` 调用 时，驱动 `Future` 的执行器会再次 `poll` 使得 `Future` 有所进展。

没有 `wake()` 函数的话，执行器将无从获知一个 future 是否能有所进展，只能持续轮询（polling） 所有 future。但有了 `wake()` 函数，执行器就能知道哪些 future 已经准备好轮询了。

例如，考虑一下场景：我们准备读取一个套接字（socket），它可能还没有可以返回的数据。如果它有 数据了，我们可以读取数据并返回 `poll::Ready(data)`，但如果数据没有准备好，我们这个future 就会阻塞并且不能继续执行。当没有数据可用时，我们需要注册 `wake` 函数，以在有数据可用时告诉执行 器我们的 future 准备好进一步操作。一个简单的 `SocketRead`future 可能像这样:

```rust,ignore
pub struct SocketRead<'a> {
    socket: &'a Socket,
}

impl SimpleFuture for SocketRead<'_> {
    type Output = Vec<u8>;

    fn poll(&mut self, wake: fn()) -> Poll<Self::Output> {
        if self.socket.has_data_to_read() {
            // The socket has data -- read it into a buffer and return it.
            Poll::Ready(self.socket.read_buf())
        } else {
            // The socket does not yet have data.
            //
            // Arrange for `wake` to be called once data is available.
            // When data becomes available, `wake` will be called, and the
            // user of this `Future` will know to call `poll` again and
            // receive data.
            self.socket.set_readable_callback(wake);
            Poll::Pending
        }
    }
}
```

Futures的这种模型允许组合多个异步操作而无需立刻分配资源。同时运行多个future或者串行（chaining）future 能够通过零分配（allocation-free）状态机实现，像这种：

```rust,ignore
/// A SimpleFuture that runs two other futures to completion concurrently.
///
/// Concurrency is achieved via the fact that calls to `poll` each future
/// may be interleaved, allowing each future to advance itself at its own pace.
pub struct Join<FutureA, FutureB> {
    // Each field may contain a future that should be run to completion.
    // If the future has already completed, the field is set to `None`.
    // This prevents us from polling a future after it has completed, which
    // would violate the contract of the `Future` trait.
    a: Option<FutureA>,
    b: Option<FutureB>,
}

impl<FutureA, FutureB> SimpleFuture for Join<FutureA, FutureB>
where
    FutureA: SimpleFuture<Output = ()>,
    FutureB: SimpleFuture<Output = ()>,
{
    type Output = ();
    fn poll(&mut self, wake: fn()) -> Poll<Self::Output> {
        // Attempt to complete future `a`.
        if let Some(a) = &mut self.a {
            if let Poll::Ready(()) = a.poll(wake) {
                self.a.take();
            }
        }

        // Attempt to complete future `b`.
        if let Some(b) = &mut self.b {
            if let Poll::Ready(()) = b.poll(wake) {
                self.b.take();
            }
        }

        if self.a.is_none() && self.b.is_none() {
            // Both futures have completed -- we can return successfully
            Poll::Ready(())
        } else {
            // One or both futures returned `Poll::Pending` and still have
            // work to do. They will call `wake()` when progress can be made.
            Poll::Pending
        }
    }
}
```

上面代码展示了多个 future 如何同时执行而无需分别分配资源，这允许异步代码变得更高级。 类似，多个 future 可以一个接一个执行，像这样：

```rust,ignore
/// A SimpleFuture that runs two futures to completion, one after another.
//
// Note: for the purposes of this simple example, `AndThenFut` assumes both
// the first and second futures are available at creation-time. The real
// `AndThen` combinator allows creating the second future based on the output
// of the first future, like `get_breakfast.and_then(|food| eat(food))`.
pub struct AndThenFut<FutureA, FutureB> {
    first: Option<FutureA>,
    second: FutureB,
}

impl<FutureA, FutureB> SimpleFuture for AndThenFut<FutureA, FutureB>
where
    FutureA: SimpleFuture<Output = ()>,
    FutureB: SimpleFuture<Output = ()>,
{
    type Output = ();
    fn poll(&mut self, wake: fn()) -> Poll<Self::Output> {
        if let Some(first) = &mut self.first {
            match first.poll(wake) {
                // We've completed the first future -- remove it and start on
                // the second!
                Poll::Ready(()) => self.first.take(),
                // We couldn't yet complete the first future.
                Poll::Pending => return Poll::Pending,
            };
        }
        // Now that the first future is done, attempt to complete the second.
        self.second.poll(wake)
    }
}
```

这个例子展示 `future` trait 如何表达异步控制流而无需请求多个已分配对象或深嵌套回调， 有了基本控制流后，我们来讨论真正的 `Future` trait 以及它和示例有什么区别：

```rust,ignore
trait Future {
    type Output;
    fn poll(
        // Note the change from `&mut self` to `Pin<&mut Self>`:
        self: Pin<&mut Self>,
        // and the change from `wake: fn()` to `cx: &mut Context<'_>`:
        cx: &mut Context<'_>,
    ) -> Poll<Self::Output>;
}
```

我们首先注意到 `self` 参数类型不再是 `mut self` 而是 `Pin<&mut Self>,`。我们会在后面章节 更多地讨论固定（pinning）的问题，但现在我们只需要知道它能让我们创建不可移动的future类型。 不可移动对象能够储存指向另一字段（field）的指针，例如：`struct MyFut { a: i32, ptr_to_a: *const i32 }`。固定对于启动 async/await 是必需的。

然后 `wake: fn()` 变成了 `&mut Context<'_>`。在 `SimpleFuture` 里，我们调用函数指针（`fn()`） 来告诉执行器有future需要轮询。然而，因为 `fn()` 是仅仅是个函数指针，它不能储存任何信息说明哪个 `Future` 调用了 `wake`。

在现实场景中，像Web服务器这样复杂的应用可能有上千不同的连接，带有应该相互隔离来管理的 唤醒器（wakeups）。`Context` 类型通过提供对 `waker` 类型的访问来解决这个问题，这些 `waker` 会唤起持定任务。
