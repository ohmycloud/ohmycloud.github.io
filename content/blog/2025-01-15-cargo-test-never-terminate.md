+++
title = "cargo test 线程阻塞"
date = 2025-01-03
[taxonomies]
  tags = ["rust", "async", "tokio"]
+++

在使用 Axum 框架改造《Zero To Production》这本书中的例子时, 遇到了一个问题:

```
test health_check_works has been running for over 60 seconds
```

这个测试就是使用 reqwest 请求一个 API 接口, 测试跑了一分钟还没有结果, 应该是被阻塞了。
代码很简单, 但是这个问题值得记录一下：

`src/lib.rs` 中的内容:

```rs
use axum::{
    Router,
    extract::Path,
    http::Response,
    response::IntoResponse,
    routing::{IntoMakeService, get},
    serve::Serve,
};
use tokio::net::TcpListener;

pub async fn health_check() -> impl IntoResponse {
    let response = Response::new("hello world");
    response.status()
}

async fn index() -> impl IntoResponse {
    "Rust Rocks!"
}

async fn greet(Path(name): Path<String>) -> impl IntoResponse {
    format!("Hello, {}", name)
}

pub fn run(
    listener: std::net::TcpListener,
) -> Result<Serve<TcpListener, IntoMakeService<Router>, Router>, std::io::Error> {
    let app = Router::new()
        .route("/", get(index))
        .route("/{name}", get(greet));

    let listener = TcpListener::from_std(listener)?;
    println!("Listening on {:?}", listener.local_addr());

    let server = axum::serve(listener, app.into_make_service());
    Ok(server)
}
```

`src/main.rs` 的内容:

```rs
use std::net::TcpListener;

use zero2prod::run;

#[tokio::main]
async fn main() -> Result<(), std::io::Error> {
    let listener = TcpListener::bind("127.0.0.1:0").expect("Failed to bind random port");
    run(listener)?.await
}
```

`tests/health_check.rs` 中的内容:

```rs
use std::net::TcpListener;

// Launch our application in the background
fn spawn_app() -> String {
    let listener = TcpListener::bind("127.0.0.1:0").expect("Failed to bind random port");
    let port = listener.local_addr().unwrap().port();
    let server = zero2prod::run(listener)
        .expect("Failed to bind address")
        .into_future();
    let _ = tokio::spawn(server);
    format!("http://127.0.0.1:{}", port)
}

#[tokio::test]
async fn health_check_works() {
    // Arrange
    let address = spawn_app();
    let client = reqwest::Client::new();

    // Act
    let response = client
        .get(&format!("{}/health_check", &address))
        .send()
        .await
        .expect("Falied to execute reqwest.");

    // Assert
    assert!(response.status().is_success());
    assert_eq!(Some(0), response.content_length());
}
```

执行 `cargo run`, 在浏览器中请求 http://127.0.0.1:3333/health_check 没有问题，使用 `curl -v http://127.0.0.1:3333/health_check` 请求 health_check 路由时也能正常返回。

问题是执行 `cargo test` 的时候, 会卡住测试程序。目前还没有找出来问题的原因, 这个问题虽然小, 但是值得研究。
