# 递归

在内部，`async fn` 创建了一个包含了要 `.await` 的子 `Future` 的状态机。这样递归的 `async fn` 有点诡异，因为结果的状态机必须包含它自身：

```rust,edition2018
# async fn step_one() { /* ... */ }
# async fn step_two() { /* ... */ }
# struct StepOne;
# struct StepTwo;
// This function:
async fn foo() {
    step_one().await;
    step_two().await;
}
// 生成一个这样的类型:
enum Foo {
    First(StepOne),
    Second(StepTwo),
}

// 所以这个函数:
async fn recursive() {
    recursive().await;
    recursive().await;
}

// 生成一个这样的类型:
enum Recursive {
    First(Recursive),
    Second(Recursive),
}
```

这不会工作——我们创建了大小为无限大的类型！编译器会抱怨：

```text
error[E0733]: recursion in an `async fn` requires boxing
 --> src/lib.rs:1:22
  |
1 | async fn recursive() {
  |                      ^ an `async fn` cannot invoke itself directly
  |
  = note: a recursive `async fn` must be rewritten to return a boxed future.
```

为了允许这种做法，我们需要用 `Box` 来间接调用。而不幸的是，编译器限制意味着把 `recursive()` 的调用包裹在 `Box::pin` 并不够。为了让递归调用工作，我们必须把 `recursive` 转换成非 `async` 函数，然后返回一个 `.boxed()` 的异步块

```rust,edition2018
use futures::future::{BoxFuture, FutureExt};

fn recursive() -> BoxFuture<'static, ()> {
    async move {
        recursive().await;
        recursive().await;
    }.boxed()
}
```
