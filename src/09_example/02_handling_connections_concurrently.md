# 并发处理链接

现在我们代码的问题，是 `listener.incoming()` 是一个阻塞的迭代器。执行器不能在 `listener` 等待接入连接时运行其他 future，使得我们不能处理一个新连接，直到我们处理完前一个连接。

为了修复这个问题，我们要转化 `listener.incoming()`，从阻塞迭代器转换成非阻塞的流。流类似于迭代器，但是会异步地被消耗。更详细的请查看[关于流的章节](../05_streams/01_chapter.md).

我们来把阻塞的 `std::net::TcpListener` 替换成非阻塞的 `async_std::net::TcpListener`，并且将我们的连接处理器更新为接受 `async_std::net::TcpStream`：

```rust,ignore
use async_std::prelude::*;

async fn handle_connection(mut stream: TcpStream) {
    let mut buffer = [0; 1024];
    stream.read(&mut buffer).await.unwrap();

    //<-- snip -->
    stream.write(response.as_bytes()).await.unwrap();
    stream.flush().await.unwrap();
}
```

这异步版本的 `TcpListener` 为了能使用 `listener.incoming`，实现了`Stream` trait，这个改动有两个好处：首先，`listener.incoming()` 不再阻塞执行器了，执行器能够在没有其他接入的 TCP 连接需要处理时，让给其他还在等在的 future 对象继续执行。

第二个好处是，来自的流的元素能可选地被并发处理，通过流的 `for_each_concurrent` 方法。这里，我们会重复利用这个方法来并发处理每一个接入的请求。我们需要引入 `futures` 库的 `Stream` trait，所以我们的 Cargo.toml 现在看起来像这样：

```diff
+[dependencies]
+futures = "0.3"

 [dependencies.async-std]
 version = "1.6"
 features = ["attributes"]
```

现在，我们能并发地处理每一个连接了，只要我们把 `handle_connection` 传递给一个闭包函数。这个闭包函数获取了每一个 `TcpStream` 的所有权，然后一旦 `TcpStream` 可用就尽快执行。只要我们的 `handle_connection` 不阻塞，一个慢请求就不会组织其他请求完成了。

```rust,ignore
use async_std::net::TcpListener;
use async_std::net::TcpStream;
use futures::stream::StreamExt;

#[async_std::main]
async fn main() {
    let listener = TcpListener::bind("127.0.0.1:7878").await.unwrap();
    listener
        .incoming()
        .for_each_concurrent(/* limit */ None, |tcpstream| async move {
            let tcpstream = tcpstream.unwrap();
            handle_connection(tcpstream).await;
        })
        .await;
}
```

# 并行地服务请求

我们的例子现在能够提供极大的并发了（通过使用异步代码），作为并行（使用线程）的替代方案。然而，异步代码和线程不是二者只得其一。在我们 的例子中， `for_each_concurrent` 并发地处理每一个连接，但不是在同一个线程。 `async-std` 库也允许我们生成任务到一个分离开的线程。 因为 `handle_connection` 既是 `Send` 的 也是非阻塞的，所以使用 `async_std::task::spawn` 是安全的。现在代码看起来会像这样子：

```rust
use async_std::task::spawn;

#[async_std::main]
async fn main() {
    let listener = TcpListener::bind("127.0.0.1:7878").await.unwrap();
    listener
        .incoming()
        .for_each_concurrent(/* limit */ None, |stream| async move {
            let stream = stream.unwrap();
            spawn(handle_connection(stream));
        })
        .await;
}
```

现在我们同时使用了并发和并行，来同时处理多个请求！更详细的请查看 [关于多线程执行器的小节](../08_ecosystem/00_chapter.md#single-threading-vs-multithreading)。
