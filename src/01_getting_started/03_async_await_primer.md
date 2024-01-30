# `async`/`.await`初步

`async`/`.await`是Rust内置语法，用于让异步函数编写得像同步代码。`async`将代码块转化成 实现了`Future` trait 的状态机。使用同步方法调用阻塞函数会阻塞整个线程，但阻塞`Future`只会 让出（yield）线程控制权，让其他`Future`继续执行。

我们来加些依赖到 `Cargo.toml` 文件：

```toml
[dependencies]
futures = "0.3"
```

你可以使用`async fn`语法创建异步函数：

```rust,edition2018
async fn do_something() { ... }
```

`async fn`函数返回实现了`Future`的类型。为了执行这个`Future`，我们需要执行器（executor）

```rust,edition2018
// `block_on` blocks the current thread until the provided future has run to
// completion. Other executors provide more complex behavior, like scheduling
// multiple futures onto the same thread.
use futures::executor::block_on;

async fn hello_world() {
    println!("hello, world!");
}

fn main() {
    let future = hello_world(); // Nothing is printed
    block_on(future); // `future` is run and "hello, world!" is printed
}
```

在`async fn`函数中， 你可以使用`.await`来等待其他实现了`Future` trait 的类型完成，例如 另外一个`async fn`的输出。和`block_on`不同，`.await`不会阻塞当前线程，而是异步地等待 future完成，在当前future无法进行下去时，允许其他任务运行。

举个例子，想想有以下三个`async fn`: `learn_song`, `sing_song`和`dance`：

```rust,ignore
async fn learn_song() -> Song { ... }
async fn sing_song(song: Song) { ... }
async fn dance() { ... }
```

一个“学，唱，跳舞”的方法，就是分别阻塞这些函数：

```rust,ignore
fn main() {
    let song = block_on(learn_song());
    block_on(sing_song(song));
    block_on(dance());
}
```

然而，这样性能并不是最优——我们一次只能干一件事！显然我们必须在唱歌之前学会它，但是学唱 同时也可以跳舞。为了做到这样，我们可以创建两个独立可并发执行的`async fn`：

```rust,ignore
async fn learn_and_sing() {
    // Wait until the song has been learned before singing it.
    // We use `.await` here rather than `block_on` to prevent blocking the
    // thread, which makes it possible to `dance` at the same time.
    let song = learn_song().await;
    sing_song(song).await;
}

async fn async_main() {
    let f1 = learn_and_sing();
    let f2 = dance();

    // `join!` is like `.await` but can wait for multiple futures concurrently.
    // If we're temporarily blocked in the `learn_and_sing` future, the `dance`
    // future will take over the current thread. If `dance` becomes blocked,
    // `learn_and_sing` can take back over. If both futures are blocked, then
    // `async_main` is blocked and will yield to the executor.
    futures::join!(f1, f2);
}

fn main() {
    block_on(async_main());
}
```

这个示例里，唱歌之前必须要学习唱这首歌，但是学习唱歌和唱歌都可以和跳舞同时发生。如果我们 用了`block_on(learning_song())`而不是`learn_and_sing`中的`learn_song().await`, 那么当`learn_song`在执行时线程将无法做别的事，也让同时跳舞变得不可能。但是通过`.await` 执行`learn_song`的future，我们就可以在`learn_song`阻塞时让其他任务来掌控当前线程。 这样就可以做到在单线程并发执行多个future到完成状态。
