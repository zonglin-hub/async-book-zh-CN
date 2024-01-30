# 最终项目：用异步 Rust 构建一个并发 Web 服务器

在这一章，我们会使用异步 Rust 来修改 Rust 书的 [一个单线程 web 服务器](https://doc.rust-lang.org/book/ch20-01-single-threaded.html) 来并发地服务请求

## 回顾

以下是我们那节课[^1]最后得到的代码：

`src/main.rs`:

```rust
use std::fs;
use std::io::prelude::*;
use std::net::TcpListener;
use std::net::TcpStream;

fn main() {
    // 在端口7878侦听传入链接
    let listener = TcpListener::bind("127.0.0.1:7878").unwrap();

    // 一直阻塞,处理到达这个IP地址的每一个请求
    for stream in listener.incoming() {
        let stream = stream.unwrap();

        handle_connection(stream);
    }
}

fn handle_connection(mut stream: TcpStream) {
    // 从流中读取前1024字节的数据
    let mut buffer = [0; 1024];
    stream.read(&mut buffer).unwrap();

    let get = b"GET / HTTP/1.1\r\n";

    // 根据请求的数据决定响应问候还是404.
    let (status_line, filename) = if buffer.starts_with(get) {
        ("HTTP/1.1 200 OK\r\n\r\n", "hello.html")
    } else {
        ("HTTP/1.1 404 NOT FOUND\r\n\r\n", "404.html")
    };
    let contents = fs::read_to_string(filename).unwrap();

    // 将响应写回流并刷新(flush)以确保响应被发送回客户端.
    let response = format!("{}{}", status_line, contents);
    stream.write(response.as_bytes()).unwrap();
    stream.flush().unwrap();
}
```

`hello.html`:

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="utf-8">
    <title>Hello!</title>
  </head>
  <body>
    <h1>Hello!</h1>
    <p>Hi from Rust</p>
  </body>
</html>
```

`404.html`:

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="utf-8">
    <title>Hello!</title>
  </head>
  <body>
    <h1>Oops!</h1>
    <p>Sorry, I don't know what you're asking for.</p>
  </body>
</html>
```

如果你使用 `cargo run` 运行这个服务器，然后在浏览器中访问 `127.0.0.1:7878`，你会受到 Ferris 友好欢迎！
