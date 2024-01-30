# 迭代与并发

与同步的 `Iterator` 类似，有很多不同的方法可以迭代处理 `Stream` 中的值。有很多组合子风格的方法，
如 `map`，`filter` 和 `fold`，以及它们的“遇错即断”版本 `try_map`，`try_filter` 和 `try_fold`。

不幸的是，`for` 循环不能用在 `Stream` 上，但是对于命令式编程风格（imperative style）的代码，
`while let` 以及 `next`/`try_next` 函数还可以使用：

```rust
async fn sum_with_next(mut stream: Pin<&mut dyn Stream<Item = i32>>) -> i32 {
    use futures::stream::StreamExt; // 对于 `next`
    let mut sum = 0;
    while let Some(item) = stream.next().await {
        sum += item;
    }
    sum
}

async fn sum_with_try_next(
    mut stream: Pin<&mut dyn Stream<Item = Result<i32, io::Error>>>,
) -> Result<i32, io::Error> {
    use futures::stream::TryStreamExt; // 对于 `try_next`
    let mut sum = 0;
    while let Some(item) = stream.try_next().await? {
        sum += item;
    }
    Ok(sum)
}
```

然而，如果我们每次只处理一个元素，我们就会失去并发的机会，而这又是我们编写异步代码的首要目的。
为了并发处理一个 `Stream` 的多个值，使用 `for_each_concurrent` 或 `try_for_each_concurrent` 方法：

```rust,no_run
async fn jump_around(
    mut stream: Pin<&mut dyn Stream<Item = Result<u8, io::Error>>>,
) -> Result<(), io::Error> {
    use futures::stream::TryStreamExt; // 对于 `try_for_each_concurrent`
    const MAX_CONCURRENT_JUMPERS: usize = 100;

    stream.try_for_each_concurrent(MAX_CONCURRENT_JUMPERS, |num| async move {
        jump_n_times(num).await?;
        report_n_jumps(num).await?;
        Ok(())
    }).await?;

    Ok(())
}
```
