# `join!`

`futures::join` 宏等待并发执行的多个不同 future 完成。

## join

当进行多个异步操作时，可以简单地用 `.await` 串行执行：

```rust,no_run
async fn get_book_and_music() -> (Book, Music) {
    let book = get_book().await;
    let music = get_music().await;
    (book, music)
}
```

然而，这实际上比必要的慢，因为我们不必在 `get_book` 完成后再 `get_music`。在其它编程语言 中，future 是运行至完成的，所以两个操作可以通过先调起 `async fn` 来启动 future，然后再分别 await 他们来并发操作：

```rust,no_run
// 这是错误示例,不要模仿
async fn get_book_and_music() -> (Book, Music) {
    let book_future = get_book();
    let music_future = get_music();
    (book_future.await, music_future.await)
}
```

然而，Rust future 不会干任何事情，除非它们已经 `.await` 了。这意味着上面这两段代码都会串行执行 `book_future` 和 `music_future` 而非并发执行。为了正确地并发这两个future，使用 `futures::join!`：

```rust,no_run
use futures::join;

async fn get_book_and_music() -> (Book, Music) {
    let book_fut = get_book();
    let music_fut = get_music();
    join!(book_fut, music_fut)
}
```

`join!` 返回值是包含每个传入 future 的输出的元组。

## `try_join!`

对于那些返回 `Result` 的 future，考虑使用 `try_join!` 而非 `join`。因为 `join` 只会在所有子 future 都完成后才会完成，它甚至会在子 future 返回 `Err` 之后继续处理。

与 `join!` 不同，`try_join!` 会在其中的子future返回错误后立即完成。

```rust,no_run
use futures::try_join;

async fn get_book() -> Result<Book, String> { /* ... */ Ok(Book) }
async fn get_music() -> Result<Music, String> { /* ... */ Ok(Music) }

async fn get_book_and_music() -> Result<(Book, Music), String> {
    let book_fut = get_book();
    let music_fut = get_music();
    try_join!(book_fut, music_fut)
}
```

注意，传进 `try_join!` 的 future 必须要用相同的错误类型。考虑使用 `futures::future::TryFutureExt` 库的 `.map_err(|e| ...)` 或 `err_into()` 函数来统一错误类型：

```rust,no_run
use futures::{
    future::TryFutureExt,
    try_join,
};

async fn get_book() -> Result<Book, ()> { /* ... */ Ok(Book) }
async fn get_music() -> Result<Music, String> { /* ... */ Ok(Music) }

async fn get_book_and_music() -> Result<(Book, Music), String> {
    let book_fut = get_book().map_err(|()| "Unable to get book".to_string());
    let music_fut = get_music();
    try_join!(book_fut, music_fut)
}
```
