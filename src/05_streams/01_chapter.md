# `Steam` trait

`Stream` trait 与 `Future` 类似，但能在完成前返还（yield）多个值，与标准库中的 `Iterator` 类似：

```rust,no_run
trait Stream {
    /// 由 `stream` 产生的值的类型.
    type Item;

    /// 尝试解析 `stream` 中的下一项.
    /// 如果已经准备好，就重新运行 `Poll::Pending`, 如果已经完成，就重新
    /// 运行`Poll::Ready(Some(x))`，如果已经完成，就重新运行 `Poll::Ready(None)`.
    fn poll_next(self: Pin<&mut Self>, cx: &mut Context<'_>)
        -> Poll<Option<Self::Item>>;
}
```

一个常见的使用 `Stream` 的例子是 `futures` 库中通道的 `Receiver`。每次 `Sender` 端发送一个值时，它就会返回一个 `Some(val)`，并且会在 `Sender` 关闭且所有消息都接收后返还 `None`:

```rust,no_run
async fn send_recv() {
    const BUFFER_SIZE: usize = 10;
    let (mut tx, mut rx) = mpsc::channel::<i32>(BUFFER_SIZE);

    tx.send(1).await.unwrap();
    tx.send(2).await.unwrap();
    drop(tx);

    // `StreamExt::next` 类似于 `Iterator::next`, 但会返回一个实现
    // 了 `Future<Output = Option<T>>` 的类型.
    assert_eq!(Some(1), rx.next().await);
    assert_eq!(Some(2), rx.next().await);
    assert_eq!(None, rx.next().await);
}
```
