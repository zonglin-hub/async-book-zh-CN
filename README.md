# async-book

Asynchronous Programming in Rust

## Requirements

The async book is built with [`mdbook`], you can install it using cargo.

```sh
cargo install mdbook
cargo install mdbook-linkcheck
```

[`mdbook`]: https://github.com/rust-lang/mdBook

## Building

To create a finished book, run `mdbook build` to generate it under the `book/` directory.

```sh
mdbook build
```

## Development

While writing it can be handy to see your changes, `mdbook serve` will launch a local web
server to serve the book.

```sh
mdbook serve
```
