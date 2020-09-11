# Actor

Actix 是一个提供并发应用的开发框架的 rust 库，它基于 [Actor Model] ，让我们可以写出一组协同工作，却又独立运行的应用。

「actor」之间通过消息「message」交流。「actor」像是一种，运行在由actix提供的 *Actor System* 中，内含着状态（state）与行为（behavior）的小容器。

「actor」运行在一个具体的执行环境（execution context）[`Context<A>`]中.
「context」对象只有在执行期间才可以获得。每个「actor」都有一个分离的执行环境。「context」控制着 「actor」的生命周期。

「actor」通过交换「message」来独立交流。发送方可以选择是否等待回复。「actor」不能被直接引用，只能通过地址「address」的方式相联系。

只要实现了 [`Actor`] trait，rust中任意一种类型都可以作为「actor」。

「actor」提供了[`Handler<M>`] impl 来处理具体的某类消息，所有的消息都被静态地标定了类型。消息可以异步处理。

「actor」可以派生其他的「actor」，或给「context」添加 streams 和 future.

`Actor` trait 提供了一些方法来控制其生命周期.

[Actor Model]: https://en.wikipedia.org/wiki/Actor_model
[`Context<A>`]: ./sec-4-context.html
[`Actor`]: https://actix.rs/actix/actix/trait.Actor.html
[`Handler<M>`]: https://actix.rs/actix/actix/trait.Handler.html

## Actor 生命周期

### 启动（Started）

「actor」总是以 `Started` 状态开始。在这期间，「actor」 的 `started()`
方法被调用。 `Actor` trait 提供了此方法的的默认实现。

此时，「context」对于「actor」是可用的，「actor」在这一步可以做的事情有:
* 启动更多的「actor」
* 注册async streams
* 自我配置

### 运行（Running）

再 `started()` 方法调用之后，「actor」转换到 `Running` 状态。
「actor」可以在`Running`状态停留任意长的时间。

### 停机中（Stopping）

在以下情况下，「actor」会进入 `Stopping` 状态：

* 其调用了自身的 `Context::stop()` 方法
* 「actor」的所有「address」都已经被删除了，也就是说，这个时候没有别的「actor」可以联系上它。
* 「context」中不再有已注册的事件对象（event object）.

在`stopping()` 时，「actor」可以创造新的「address」或添加新的事件对象，并返回一个`Running::Continue`，从而使得自己重启，重返 `running` 状态。

如果一个「actor」经由调用 `Context::stop()` 进入 `stopping`，那么「context」就会立即停止处理收到的消息，并且调用 `Actor::stopping()` 。如果「actor」没有重启回到`running`状态，那么所有未处理的消息就被删除（drop）。默认情况下，这个方法会返回 `Running::Stop` 来确认停机的操作。

### 停机后（Stopped）
如果在停机的过程中，「actor」并没有调整「context」，那么就会进入 `Stopped` 状态。这个状态是一个actor生命的终点。在这个状态下，actor会被从系统中删除。


## 消息（Message）

两个「actor」之间通过发送「message」来交流。 actix中，所有「message」都是有类型的。
「message」可以是任何实现了
[`Message`] trait 的类型。 
「message」由 `Message::Result` 定义「message」返回的类型。

我们可以定义一个简单的 `Ping`「message」，某个「actor」会接受这个「message」并返回一个
`io::Result<bool>`.

```rust
# extern crate actix;
use std::io;
use actix::prelude::*;

struct Ping;

impl Message for Ping {
    type Result = Result<bool, io::Error>;
}

# fn main() {}
```

[`Message`]: https://actix.rs/actix/actix/trait.Message.html

## 派生（spawn）一个「actor」

如何去启动一个「actor」取决于其环境「context」。通过[`Actor`] trait 的 `start()` 与 `create()` 方法可以派生新的「actor」。

它提供了许多种派生方式：具体请查看文档。

## 完整的例子

```rust
# extern crate actix;
# extern crate futures;
use std::io;
use actix::prelude::*;
use futures::Future;

/// 定义消息
struct Ping;

impl Message for Ping {
    type Result = Result<bool, io::Error>;
}


// 定义actor
struct MyActor;

// 为我们的 actor 实现 Actor 接口
impl Actor for MyActor {
    type Context = Context<Self>;

    fn started(&mut self, ctx: &mut Context<Self>) {
       println!("Actor is alive");
    }

    fn stopped(&mut self, ctx: &mut Context<Self>) {
       println!("Actor is stopped");
    }
}

/// 定义 `Ping` 消息的handler
impl Handler<Ping> for MyActor {
    type Result = Result<bool, io::Error>;

    fn handle(&mut self, msg: Ping, ctx: &mut Context<Self>) -> Self::Result {
        println!("Ping received");

        Ok(true)
    }
}

fn main() {
    let sys = System::new("example");

    // 在当前线程中启动我们的 actor
    let addr = MyActor.start();

    // 发送一个 Ping 消息
    // send() 消息返回一个 Future 对象, 解析为 message result
    let result = addr.send(Ping);

    // 派生 future 为 reactor
    Arbiter::spawn(
        result.map(|res| {
            match res {
                Ok(result) => println!("Got result: {}", result),
                Err(err) => println!("Got error: {}", err),
            }
#           System::current().stop();
        })
        .map_err(|e| {
            println!("Actor is probably dead: {}", e);
        }));

    sys.run();
}
```

## 通过 MessageResponse 响应

让我们来研究上例中 `Result` 类型的 `impl Handler` 定义。我们是怎么返回  `Result<bool, io::Error>`呢? 我们之所以能够用这个类型作为收到「message」的响应，是因为这种类型实现了 `MessageResponse` trait. 

以下是这个trait的定义:
```rust
pub trait MessageResponse<A: Actor, M: Message> {
    fn handle<R: ResponseChannel<M>>(self, ctx: &mut A::Context, tx: Option<R>);
}
```
有时候，我们要对没有实现此 trait 的「message」做出响应，这时，我们可以自己实现这个trait

下面是一个例子:
- 通过`GotPing`来响应`Ping`「message」
- 通过`GotPong`来响应`Pong`「message」.


```rust
# extern crate actix;
# extern crate futures;
use actix::dev::{MessageResponse, ResponseChannel};
use actix::prelude::*;
use futures::Future;

// 消息
enum Messages {
    Ping,
    Pong,
}

// 响应
enum Responses {
    GotPing,
    GotPong,
}

// 手动为 Responses 实现 MessageResponse trait
impl<A, M> MessageResponse<A, M> for Responses
where
    A: Actor,
    M: Message<Result = Responses>,
{
    fn handle<R: ResponseChannel<M>>(self, _: &mut A::Context, tx: Option<R>) {
        if let Some(tx) = tx {
            tx.send(self);
        }
    }
}

// 消息的Result 为 Responses
impl Message for Messages {
    type Result = Responses;
}

// 定义 actor
struct MyActor;

// 为 MyActor 实现 Actor trait
impl Actor for MyActor {
    type Context = Context<Self>;

    fn started(&mut self, ctx: &mut Context<Self>) {
        println!("Actor is alive");
    }

    fn stopped(&mut self, ctx: &mut Context<Self>) {
        println!("Actor is stopped");
    }
}

/// 为 `Messages` enum 定义对应的 handler
impl Handler<Messages> for MyActor {
    type Result = Responses;

    fn handle(&mut self, msg: Messages, ctx: &mut Context<Self>) -> Self::Result {
        match msg {
            Messages::Ping => Responses::GotPing,
            Messages::Pong => Responses::GotPong,
        }
    }
}

fn main() {
    let sys = System::new("example");

    // Start MyActor in current thread
    let addr = MyActor.start();

    // Send Ping message.
    // send() message returns Future object, that resolves to message result
    let ping_future = addr.send(Messages::Ping);
    let pong_future = addr.send(Messages::Pong);

    // Spawn pong_future onto event loop
    Arbiter::spawn(
        pong_future
            .map(|res| {
                match res {
                    Responses::GotPing => println!("Ping received"),
                    Responses::GotPong => println!("Pong received"),
                }
#               System::current().stop();
            })
            .map_err(|e| {
                println!("Actor is probably dead: {}", e);
            }),
    );

    // Spawn ping_future onto event loop
    Arbiter::spawn(
        ping_future
            .map(|res| {
                match res {
                    Responses::GotPing => println!("Ping received"),
                    Responses::GotPong => println!("Pong received"),
                }
#               System::current().stop();
            })
            .map_err(|e| {
                println!("Actor is probably dead: {}", e);
            }),
    );

    sys.run();
}
```
