+++
title = "Kafka 和 Rust入门 - 第一部分"
date = 2021-04-14
[taxonomies]
  tags = ["rust", "kafka"]
+++

这是一个两部分的系列，帮助你开始使用 Rust 和 Kafka。我们将使用 [rust-rdkafka](https://github.com/fede1024/rust-rdkafka/) crate，它本身就是基于 [librdkafka](https://github.com/edenhill/librdkafka)（C库）的。

在这篇文章中，我们将介绍 Kafka Producer API。

## 初始设置

确保你[安装了一个 Kafka broker](https://kafka.apache.org/downloads) - 本地设置应该足够了。当然，你也需要[安装Rust](https://www.rust-lang.org/tools/install) - 你需要[1.45或以上版本](https://github.com/fede1024/rust-rdkafka#minimum-supported-rust-version-msrv)。

在你开始之前，先克隆 GitHub repo。

```bash
git clone https://github.com/abhirockzz/rust-kafka-101
cd part1
```

检查 Cargo.toml 文件:

```
...
[dependencies]
rdkafka = { version = "0.25", features = ["cmake-build","ssl"] }
...
```

## 关于 cmake-build 功能的说明

`rust-rdkafka` 提供了几种解决 `librdkafka` 依赖关系的方法。我选择了 `static` 链接，其中 `librdkafka` 被编译。不过你也可以选择 `dynamic` 链接来引用本地安装的版本。

> 更多内容，请[参考以下链接](https://github.com/fede1024/rust-rdkafka/blob/master/rdkafka-sys/README.md#features)

好吧，我们先从基本的开始说起。

## 简单的生产者

这里是一个基于 [BaseProducer](https://docs.rs/rdkafka/0.26.0/rdkafka/producer/struct.BaseProducer.html) 的简单生产者。

```rust
    let producer: BaseProducer = ClientConfig::new()
        .set("bootstrap.servers", "localhost:9092")
        .set("security.protocol", "SASL_SSL")
        .set("sasl.mechanisms", "PLAIN")
        .set("sasl.username", "<update>")
        .set("sasl.password", "<update>")
        .create()
        .expect("invalid producer config");
```

`send` 方法开始产生消息 - 它是在紧缩 `loop` 中完成的，中间有一个 `thread::sleep`(不是在生产中会做的事情)，以使其更容易追踪/跟踪结果。键、值（有效载荷）和目标 Kafka 主题以 [BaseRecord](https://docs.rs/rdkafka/0.26.0/rdkafka/producer/struct.BaseRecord.html) 的形式表示。

```rust
    for i in 1..100 {
        println!("sending message");
        producer
            .send(
                BaseRecord::to("rust")
                    .key(&format!("key-{}", i))
                    .payload(&format!("value-{}", i)),
            )
            .expect("failed to send message");
        thread::sleep(Duration::from_secs(3));
    }
```

> 你可以在文件 `src/1_producer_simple.rs` 中查看整个代码。

## 要测试生产者是否在工作 ...

运行这段代码:

- 只需将文件 `src/1_producer_simple.rs` 重命名为 `main.rs`。
- 执行 `cargo run`

你应该看到这个输出:

```
sending message
sending message
sending message
...
```

到底发生了什么？要弄清楚 - 使用 Kafka CLI 消费者（或其他消费者客户端，如 kafkacat）连接到你的 Kafka 主题（我在上面的例子中使用 rust 作为 Kafka 主题的名称）。你应该看到消息流进来了。

例如

```bash
&KAFKA_HOME/bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic rust --from-beginning
```

## 生产者回调

我们现在是在瞎飞! 除非我们明确地创建一个消费者来查看我们的消息，否则我们不知道它们是否被发送到 Kafka。让我们通过实现 [ProducerContext](https://docs.rs/rdkafka/0.26.0/rdkafka/producer/trait.ProducerContext.html)(trait)来解决这个问题，以挂接到 produce 事件 - 它就像一个回调。

首先为 [ClientContext](https://docs.rs/rdkafka/0.26.0/rdkafka/client/trait.ClientContext.html) trait 创建一个结构体和一个空的实现（这是必须的）。

```rust
struct ProducerCallbackLogger;

impl ClientContext for ProducerCallbackLogger {}
```

现在到了主要部分，我们在 `ProducerContext` trait 中实现 `delivery` 函数。

```rust
impl ProducerContext for ProduceCallbackLogger {
    type DeliveryOpaque = ();

    fn delivery(
        &self,
        delivery_result: &rdkafka::producer::DeliveryResult<'_>,
        _delivery_opaque: Self::DeliveryOpaque,
    ) {
        let dr = delivery_result.as_ref();
        match dr {
            Ok(msg) => {
                let key: &str = msg.key_view().unwrap().unwrap();
                println!(
                    "produced message with key {} in offset {} of partition {}",
                    key,
                    msg.offset(),
                    msg.partition()
                )
            }
            Err(producer_err) => {
                let key: &str = producer_err.1.key_view().unwrap().unwrap();

                println!(
                    "failed to produce message with key {} - {}",
                    key, producer_err.0,
                )
            }
        }
    }
}
```

我们根据 [DeliveryResult](https://docs.rs/rdkafka/0.26.0/rdkafka/producer/type.DeliveryResult.html)(毕竟它是一个 `Result`)来匹配成功(`Ok`)和失败(`Err`)的情况。我们所做的只是简单地记录这两种情况下的消息，因为这只是一个例子。你可以在这里做任何你想做的事情（虽然不要太疯狂！）。

> 我们忽略了 `ProducerContext` trait 的关联类型 [DeliveryOpaque](https://docs.rs/rdkafka/0.26.0/rdkafka/producer/trait.ProducerContext.html#associatedtype.DeliveryOpaque)。

我们需要确保我们插入了 `ProducerContext` 的实现。我们通过使用 [create_with_context](https://docs.rs/rdkafka/0.26.0/rdkafka/config/struct.ClientConfig.html#method.create_with_context) 方法（而不是 [create](https://docs.rs/rdkafka/0.26.0/rdkafka/config/struct.ClientConfig.html#method.create)）来实现，并确保为 `BaseProducer` 提供正确的类型。

```rust
let producer: BaseProducer<ProduceCallbackLogger> = ClientConfig::new().set(....)
...
.create_with_context(ProduceCallbackLogger {})
...
```

## 如何调用 "回调"？

好了，我们有了实现，但我们需要一种方法来触发它! 其中一个方法就是在生产者上调用 [flush](https://docs.rs/rdkafka/0.26.0/rdkafka/producer/struct.BaseProducer.html#method.flush)。所以，我们可以把我们的 producer 写成这样。

- 添加 `producer.flush(Duration::from_secs(3));`, 并
- 注释掉 `sleep` (just for now)

```rust
        producer
            .send(
                BaseRecord::to("rust")
                    .key(&format!("key-{}", i))
                    .payload(&format!("value-{}", i)),
            )
            .expect("failed to send message");

        producer.flush(Duration::from_secs(3));
        println!("flushed message");
        //thread::sleep(Duration::from_secs(3));
```

## 等等，我们可以做得更好

`send` 方法是非阻塞的（默认），但通过在每次 `send` 后调用 `flush`，我们现在已经将其转换为同步调用 - 从性能角度来看，不推荐使用。

我们可以通过使用 [ThreadedProducer](https://docs.rs/rdkafka/0.26.0/rdkafka/producer/struct.ThreadedProducer.html) 来改善这种情况。它负责在后台线程中调用 [poll](https://docs.rs/rdkafka/0.26.0/rdkafka/producer/base_producer/struct.BaseProducer.html#method.poll) 方法，以确保发送回调通知的传递。这样做非常简单 - 只需将类型从 `BaseProducer` 改为 `ThreadedProducer` 即可!

```rust
# before: BaseProducer<ProduceCallbackLogger>
# after: ThreadedProducer<ProduceCallbackLogger>
```

而且，我们也不需要再调用 `flush` 了。

```rust
...
//producer.flush(Duration::from_secs(3));
//println!("flushed message");
thread::sleep(Duration::from_secs(3));
...
```

> 代码在 `src/2_threaded_producer.rs` 中可以找到。

## 再次运行该程序

- 将文件 `src/2_threaded_producer.rs` 重命名为 `main.rs`，并且
- 执行 `cargo run`

输出:

```
sending message
sending message
produced message with key key-1 in offset 6 of partition 2
produced message with key key-2 in offset 3 of partition 0
sending message
produced message with key key-3 in offset 7 of partition 2
```

正如预期的那样，你应该能够看到生产者事件回调，表示消息确实被发送到了 Kafka 主题。当然，你可以直接连接到主题，并再次检查，就像之前一样。

```bash
&KAFKA_HOME/bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic rust --from-beginning
```

> 要尝试失败的情况，请尝试使用一个不正确的主题名称，并注意 `delivery` 实现的 `Err` 变体是如何被调用的。

## 发送 JSON 消息

到目前为止，我们只是发送 `String` 作为 key 和 value。JSON 是一种常用的消息格式，让我们看看如何使用它。

假设我们要发送 `User` 信息，将使用这个结构体来表示。

```rust
struct User {
    id: i32,
    email: String,
}
```

然后我们可以使用 [serde_json](https://docs.serde.rs/serde_json/) 库将其序列化为 JSON。我们所需要的就是使用 [serde 中的自定义派生函数](https://serde.rs/derive.html) - `Deserialize` 和 `Serialize`。

```rust
use serde::{Deserialize, Serialize};

#[derive(Serialize, Deserialize, Debug)]
struct User {
    id: i32,
    email: String,
}
```

## 修改生产者循环。

- 创建一个 `User` 实例
- 使用 [to_string_pretty](https://docs.serde.rs/serde_json/fn.to_string_pretty.html) 将其序列化为 JSON 字符串。
- 在有效载荷中加入这一点

```rust
...

let user_json = serde_json::to_string_pretty(&user).expect("json serialization failed");

        producer
            .send(
                BaseRecord::to("rust")
                    .key(&format!("user-{}", i))
                    .payload(&user_json),
            )
            .expect("failed to send message");
...
```

> 你也可以使用 [to_vec](https://docs.serde.rs/serde_json/fn.to_vec.html)(而不是 `to_string()`)将其转换为一个字节的`Vec`(`Vec<u8>`)。

## 要运行该程序...

- 将文件 `src/3_JSON_payload.rs` 重命名为 `main.rs`，然后
- 执行 `cargo run`

从主题中消费:

```bash
&KAFKA_HOME/bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic rust --from-beginning
```

你应该看到带有 `String` 的键（如 `user-34`）和 JSON 值的消息。

```json
{
  "id": 34,
  "email": "user-34@foobar.com"
}
```

## 有更好的方法吗？

是的！如果你习惯了 Kafka Java 客户端中的声明式序列化/去序列化方法（可能其他客户端也一样），你可能不喜欢这种 "显式" 方法。只是为了让大家明白，这是你在 Java 中的做法。

```java
Properties props = new Properties();
props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG,
  "org.apache.kafka.common.serialization.StringSerializer");
props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG,
  "io.confluent.kafka.serializers.json.KafkaJsonSchemaSerializer");

....

ProducerRecord<String, User> record = new ProducerRecord<String, User>(topic, key, user);
producer.send(record);
```

> 请注意，您只需将 `Producer` 配置为使用 `KafkaJsonSchemaSerializer`，`User` 类就会被序列化为 JSON。

`rust-rdkafka` 用 [ToBytes](https://docs.rs/rdkafka/0.26.0/rdkafka/message/trait.ToBytes.html) trait 提供了类似的东西。下面是它的样子。

```rust
pub trait ToBytes {
    /// Converts the provided data to bytes.
    fn to_bytes(&self) -> &[u8];
}
```

不言而喻吧？`String`、`Vec<u8>` 等都有现有的实现。所以你可以使用这些类型作为键或值，而不需要任何额外的工作 - 这正是我们刚刚做的。但问题是我们的方法是 "显式" 的，即我们将 `User` 结构转换为 JSON 字符串，并将其传递出去。

## 如果我们可以为 `User` 实现 `ToBytes` 呢？

```rust
impl ToBytes for User {
    fn to_bytes(&self) -> &[u8] {
        let b = serde_json::to_vec_pretty(&self).expect("json serialization failed");
        b.as_slice()
    }
}
```

你会看到一个编译器错误。

```
cannot return value referencing local variable `b`
returns a value referencing data owned by the current function
```

> 更多的背景资料，请参考 [GitHub](https://github.com/fede1024/rust-rdkafka/issues/128) 的问题。我很乐意看到其他可以与 `ToBytes` 一起工作的例子 - 如果你有这方面的意见，请在留言中留下。

TL;DR是，最好坚持用 "显式" 的方式做事，除非你有一个 `ToBytes` 的实现，"不涉及分配，不能失败"。

## 总结

第一部分就到这里。第二部分将涉及围绕 Kafka 消费者的话题。

原文链接: https://dev.to/abhirockzz/getting-started-with-kafka-and-rust-part-1-4hkb
