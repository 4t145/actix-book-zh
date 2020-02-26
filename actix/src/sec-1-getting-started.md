# Getting Started

Let’s create and run our first actix application. We’ll create a new Cargo project
that depends on actix and then run the application.

In previous section we already installed required rust version. Now let's create new cargo projects.

## Ping actor

Let’s write our first actix application! Start by creating a new binary-based
Cargo project and changing into the new directory:

```bash
cargo new actor-ping --bin
cd actor-ping
```

Now, add actix as a dependency of your project by ensuring your Cargo.toml
contains the following:

```toml
[dependencies]
actix = "0.8"
```

创建一个接受 `Ping` message 并回复处理后的ping值的actor.

一个actor就是是实现了 `Actor` trait的类型:

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

每个actor都有一个execution context, 我们将在`MyActor`上使用 `Context<A>`. 下一节我们讲更多关于actor contexts的信息.

现在我们要定义actor要接收的 `Message` . message 可以是任何实现了 `Message` trait的类型.

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

 `Message` trait 的主要目的就是定义 result 类型. `Ping` 消息(message) 的 result 定义为
`usize`, 这样就暗示着任何一个接受 `Ping` 消息的 actor 都需要返回一个 `usize` 值.

最后, 我们要给我们我们的actor `MyActor` 实现 `Handler<Ping>` trait ,使其可以接收 `Ping` 并且处理(handle)它.

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

就是如此!
现在我们只需要启动(start)我们的actor再给它发送一条消息.

事实上, actor 的环境(context)实现决定了它的启动流程. 在这个例子中, 我们使用基于 tokio/future 的`Context<A>` . 我们可以通过 `Actor::start()`
或者 `Actor::create()` 来启动. 其中, `Actor::start()` 用于 actor 实例可以即刻创建的情形. 而 `Actor::start()` 用于在创建实例之前, 我们要访问 context 对象的情形.

在 `MyActor` 的例子中, 我们就使用 `start()`来启动.

所有 actors 之间的通讯都通过地址(address)来传递. 你可以使用 `send`给 actor 发送某条消息 , 当你不需要等待响应(response)的时候, 你可以 `do_send`.

`start()` 和 `create()` 都会返回一个地址对象

下面我们将new一个 `MyActor` actor , 再发送一条消息.

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

Ping这个例子可在此[示例目录](https://github.com/actix/actix/tree/master/examples/)查看.
