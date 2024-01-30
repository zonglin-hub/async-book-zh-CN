# `async` 块中的 `?` 运算符

和在 `async fn` 中一样, 在 `async` 块中使用 `?` 是很寻常的. 然而, `async` 块的返回类型不是被显式声明的. 这会导致编译器不能推断 `async` 块的错误类型.

如下面的代码:

```rust,edition2018
# struct MyError;
# async fn foo() -> Result<(), MyError> { Ok(()) }
# async fn bar() -> Result<(), MyError> { Ok(()) }
let fut = async {
    foo().await?;
    bar().await?;
    Ok(())
};
```

会触发这个错误:

```text
error[E0282]: type annotations needed
 --> src/main.rs:5:9
  |
4 |     let fut = async {
  |         --- consider giving `fut` a type
5 |         foo().await?;
  |         ^^^^^^^^^^^^ cannot infer type
```

不幸的是, 现在还不能"给 `fut` 一个类型", 也没有办法 显式地指定 `async` 块的返回类型. 要解决这个问题, 你可以使用"涡轮运算符"来为块 `async` 提供成功和错误时的类型:

```rust,edition2018
# struct MyError;
# async fn foo() -> Result<(), MyError> { Ok(()) }
# async fn bar() -> Result<(), MyError> { Ok(()) }
let fut = async {
    foo().await?;
    bar().await?;
    Ok::<(), MyError>(()) // <- note the explicit type annotation here
};
```
