# trait 中的 `async`

目前，`async fn` 不能在 trait 中使用。原因一些复杂，但是有计划在未来移除这个限制。

不过，这个问题可以用[crates.io 的 `async_trait` 库](https://github.com/dtolnay/async-trait)来规避。

注意，这些 trait 方法会导致每个函数调用都需要分配堆内存。这可能对于大部分应用都不是特别严重的开销，但是在决定是否要把这个功能作为底层函数的公共API，尤其这个函数可能每秒调用上百万次时则需要多加考虑。
