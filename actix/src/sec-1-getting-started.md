# 开始

让我们试着创建并运行自己的第一个actix应用吧！我们将创建并运行一个依赖于actix的cargo项目！

## Ping actor

现在就来写第一个actix应用，我们用cargo新建一个二进制项目，并切换到这个文件夹。

```bash
cargo new actor-ping --bin
cd actor-ping
```

现在在cargo.toml里面添加依赖项

```toml
[dependencies]
actix = "0.8"
```

创建一个接受 `Ping` 「message」 并回复处理后的ping值的「actor」。

所谓「actor」就是是实现了 `Actor` trait的类型：

```rust
# extern crate actix;
use actix::prelude::*;

struct MyActor {
    count: usize,
}

impl Actor for MyActor {
    type Context = Context<Self>;
}

# fn main() {}
```

每个「actor」都有一个「execution context」（执行环境），我们将在`MyActor`上使用 `Context<A>`。下一节我们讲更多关于「contexts」的信息。

现在我们要定义「actor」要接收的 `Message` 。「message」 可以是任何实现了 `Message` trait的类型。

```rust
# extern crate actix;
use actix::prelude::*;

struct Ping(usize);

impl Message for Ping {
    // Message 的结果将是一个 usize 值
    type Result = usize;
}

# fn main() {}
```

实现`Message` trait 的主要目的就是去定义它的`type Result`。

`Ping` 「message」 的`type Result`定义为
`usize`，这样就意味着：任何一个接受 `Ping` 「message」的 「actor」 都需要返还一个 `usize` 值。

最后，再要给我们的「actor」 `MyActor` 实现 `Handler<Ping>` trait ,使其可以接收 `Ping` 并且处理（handle）它。

```rust
# extern crate actix;
# use actix::prelude::*;
#
# struct MyActor {
#    count: usize,
# }
# impl Actor for MyActor {
#     type Context = Context<Self>;
# }
#
# struct Ping(usize);
#
# impl Message for Ping {
#    type Result = usize;
# }

impl Handler<Ping> for MyActor {
    type Result = usize;

    fn handle(&mut self, msg: Ping, _ctx: &mut Context<Self>) -> Self::Result {
        self.count += msg.0;

        self.count
    }
}

# fn main() {}
```

就是这样!

现在我们只需要启动（start）我们的「actor」，再给它发送一条消息。

事实上，「actor」中「context」的具体实现决定了它的启动流程。在这个例子中，我们使用基于 tokio/future 的`Context<A>` 。

我们可以通过调用`Actor::start()`
或者`Actor::create()`来启动「actor」。其中,，`Actor::start()` 用于「actor」实例可以即刻创建的情景。而 `Actor::create()` 用于在创建实例之前，我们要访问「context」对象的情景。

在 `MyActor` 的例子中， 我们将使用 `start()`来启动。

所有「actor」之间的通讯都通过地址「address」来传递。你可以使用 `send()`给「actor」发送消息，有时候你不需要等待响应（response）的时候，你可以 `do_send()`。

`start()` 和 `create()` 都会返回一个「address」对象

下面我们将new一个 `MyActor`「actor」, 再发送一条消息「message」.

```rust
# extern crate actix;
# extern crate futures;
# use futures::Future;
# use std::io;
# use actix::prelude::*;
# struct MyActor {
#    count: usize,
# }
# impl Actor for MyActor {
#     type Context = Context<Self>;
# }
#
# struct Ping(usize);
#
# impl Message for Ping {
#    type Result = usize;
# }
# impl Handler<Ping> for MyActor {
#     type Result = usize;
#
#     fn handle(&mut self, msg: Ping, ctx: &mut Context<Self>) -> Self::Result {
#         self.count += msg.0;
#         self.count
#     }
# }
#
fn main() -> std::io::Result<()> {
    let system = System::new("test");

    // 启动一个actor
    let addr = MyActor{count: 10}.start();

    // 发送消息, 得到一个future作为结果
    let res = addr.send(Ping(10));

    Arbiter::spawn(
        res.map(|res| {
            # System::current().stop();
            println!("RESULT: {}", res == 20);
        })
        .map_err(|_| ()));

    system.run()
}
```

Ping这个例子可在此[示例目录](https://github.com/actix/actix/tree/master/examples/)查看。
