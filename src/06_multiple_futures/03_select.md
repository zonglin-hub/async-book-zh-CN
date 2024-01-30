# `select!`

`futures::select` 宏同时跑多个 future，允许用户在任意 future 完成时响应：

```rust,no_run
use futures::{
    future::FutureExt, // 为了 `.fuse()`
    pin_mut,
    select,
};

async fn task_one() { /* ... */ }
async fn task_two() { /* ... */ }

async fn race_tasks() {
    let t1 = task_one().fuse();
    let t2 = task_two().fuse();

    pin_mut!(t1, t2);

    select! {
        () = t1 => println!("task one completed first"),
        () = t2 => println!("task two completed first"),
    }
}
```

上面的函数会并发跑 `t1` 和 `t2`。当 `t1` 和 `t2` 结束时，对应的句柄（handler）会调用 `println!`，然后函数就会结束而不会完成剩下的任务。

`select` 的基本格式为 `<pattern> = <expression> => <code>,`，可以重复你想 `select` 的任意多future。

## `default => ...` 和 `complete => ...`

`select` 也支持 `default` 和 `complete` 分支。

`default` 会在被 `select` 的future都没有完成时执行，因此，带有 `default` 分支的 `select` 总是马上返回，因为 `default` 会在没有其它future准备好的时候返回。

`complete` 分支则用来处理所有被 `select` 的 future 都完成并且不需进一步处理的情况。这在循环 `select` 时很好用：

```rust,no_run
use futures::{future, select};

async fn count() {
    let mut a_fut = future::ready(4);
    let mut b_fut = future::ready(6);
    let mut total = 0;

    loop {
        select! {
            a = a_fut => total += a,
            b = b_fut => total += b,
            complete => break,
            default => unreachable!(), // 永远不会被执行(futures都准备好了,然后complete分支被执行)
        };
    }
    assert_eq!(total, 10);
}
```

## 和 `Unpin` 与 `FusedFuture` 交互

你会注意到，在上面第一个例子中，我们在两个 `async fn` 函数返回的future上调用了 `.fuse()`，然后用 `pin_mut` 来固定他们。这两个调用都是必需的，用在 `select` 中的 future 必须实现 `Unpin` 和 `FusedFuture`。

需要 `Unpin` 是因为 `select` 是用可变引用访问 future 的，不获取 future 的所有权。未完成的 future 因此可以在 `select` 调用后继续使用。

类似的，需要 `FusedFuture` 是因为 `select` 一定不能轮询已完成的 future。`FusedFuture` 用来追踪（track）future是否已完成。这种使得在循环中使用 `select` 成为可能，只轮询尚未完成的 future。这可以从上面的例子中看出，`a_fut` 或 `b_fut` 可能会在第二次循环的时候已经完成了。因为 `future::ready` 返回的 future 实现了 `FusedFuture`，所以 `select` 可以知道不必再次轮询它了。

注意，stream 也有对应的 `FusedStream` trait。实现了这个 trait 或者被 `.fuse()` 包装的 Stream 会从它们的 `.next`/`try_next()` 组合子中返还 `FusedFutre`。

```rust,no_run
use futures::{
    stream::{Stream, StreamExt, FusedStream},
    select,
};

async fn add_two_streams(
    mut s1: impl Stream<Item = u8> + FusedStream + Unpin,
    mut s2: impl Stream<Item = u8> + FusedStream + Unpin,
) -> u8 {
    let mut total = 0;

    loop {
        let item = select! {
            x = s1.next() => x,
            x = s2.next() => x,
            complete => break,
        };
        if let Some(next_num) = item {
            total += next_num;
        }
    }

    total
}
```

## 带有 `Fuse` 和 `FuturesUnordered` 的 `select` 循环中的并发任务

有个不太好找但是很趁手的函数叫 `Fuse::terminated()`。这个函数允许构造已经被终止的空 future，并且能够在之后填进需要运行的 future。

这个在一个任务需要 `select` 循环中运行但是它本身是在 `select` 循环中创建的场景中很好用。

注意下面 `.select_next_some()` 函数的用法。它可以用在 `select` 上，并且只运行从 stream 返回的 `Some(_)` 值而忽略 `None`。

```rust,no_run
use futures::{
    future::{Fuse, FusedFuture, FutureExt},
    stream::{FusedStream, Stream, StreamExt},
    pin_mut,
    select,
};

async fn get_new_num() -> u8 { /* ... */ 5 }

async fn run_on_new_num(_: u8) { /* ... */ }

async fn run_loop(
    mut interval_timer: impl Stream<Item = ()> + FusedStream + Unpin,
    starting_num: u8,
) {
    let run_on_new_num_fut = run_on_new_num(starting_num).fuse();
    let get_new_num_fut = Fuse::terminated();
    pin_mut!(run_on_new_num_fut, get_new_num_fut);
    loop {
        select! {
            () = interval_timer.select_next_some() => {
                // 计时器已经完成了.
                // 如果没有`get_new_num_fut`正在执行的话,就启动一个新的.
                if get_new_num_fut.is_terminated() {
                    get_new_num_fut.set(get_new_num().fuse());
                }
            },
            new_num = get_new_num_fut => {
                // 一个新的数字到达了
                // 启动一个新的`run_on_new_num_fut`并且扔掉旧的.
                run_on_new_num_fut.set(run_on_new_num(new_num).fuse());
            },
            // 执行`run_on_new_num_fut`
            () = run_on_new_num_fut => {},
            // 当所有都完成时panic,
            // 因为理论上`interval_timer`会不断地产生值.
            complete => panic!("`interval_timer` completed unexpectedly"),
        }
    }
}
```

当有很多份相同 future 的拷贝同时执行时，使用 `FutureUnordered` 类型。下面的例子和上面的例子很类似，但会运行 `run_on_new_num_fut` 的所有拷贝都到完成状态，而不是当一个新拷贝创建时就中断他们。它也会打印 `run_on_new_num_fut` 的返回值：

```rust,no_run
use futures::{
    future::{Fuse, FusedFuture, FutureExt},
    stream::{FusedStream, FuturesUnordered, Stream, StreamExt},
    pin_mut,
    select,
};

async fn get_new_num() -> u8 { /* ... */ 5 }

async fn run_on_new_num(_: u8) -> u8 { /* ... */ 5 }

// 用从`get_new_num`获取的最新的数字运行`run_on_new_num`.
//
// 每当定时器到期后,都会重新执行`get_new_num`,
// 并立即取消正在执行的`run_on_new_num`,随后用新返回值替换`run_on_new_num`.
async fn run_loop(
    mut interval_timer: impl Stream<Item = ()> + FusedStream + Unpin,
    starting_num: u8,
) {
    let mut run_on_new_num_futs = FuturesUnordered::new();
    run_on_new_num_futs.push(run_on_new_num(starting_num));
    let get_new_num_fut = Fuse::terminated();
    pin_mut!(get_new_num_fut);
    loop {
        select! {
            () = interval_timer.select_next_some() => {
                // 计时器已经完成了.
                // 如果没有`get_new_num_fut`正在执行的话,就启动一个新的.
                if get_new_num_fut.is_terminated() {
                    get_new_num_fut.set(get_new_num().fuse());
                }
            },
            new_num = get_new_num_fut => {
                // 一个新的数字到达了,启动一个新的`run_on_new_num_fut`.
                run_on_new_num_futs.push(run_on_new_num(new_num));
            },
            // 执行`run_on_new_num_futs`并检查有没有完成的.
            res = run_on_new_num_futs.select_next_some() => {
                println!("run_on_new_num_fut returned {:?}", res);
            },
            // 当所有都完成时panic,
            // 因为理论上`interval_timer`会不断地产生值.
            complete => panic!("`interval_timer` completed unexpectedly"),
        }
    }
}
```
