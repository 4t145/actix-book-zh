# Address

「actor」通过交换「message」来独立交流。发送方可以选择是否等待回复。「actor」不能被直接引用，只能通过地址「address」的方式相联系。

有几种方式可以得到「actor」的「address」，`Actor` 提供了两种快捷方式来启动一个「actor」。它们都返回actor的地址。

下面是一个 `Actor::start()` 使用例。此例中 `MyActor` 「actor」
是异步的，并在同一个调用线程中启动，这个我们会在[SyncArbiter]这一节提到。

```rust
# extern crate actix;
# use actix::prelude::*;
struct MyActor;
impl Actor for MyActor {
    type Context = Context<Self>;
}

# fn main() {
# System::new("test");
let addr = MyActor.start();
# }
```

异步的「actor」可以从它的`Context` 结构获得「address」。这个「context」需要实现了`AsyncContext` trait. `AsyncContext::address()` 提供了「actor」的地址

```rust
# extern crate actix;
# use actix::prelude::*;
struct MyActor;
impl Actor for MyActor {
    type Context = Context<Self>;

    fn started(&mut self, ctx: &mut Context<Self>) {
       let addr = ctx.address();
    }
}
# fn main() {}
```

[SyncArbiter]: ./sec-6-sync-arbiter.md

## 消息（Message）

为了能够处理特定的消息，「actor」提供了
[`Handler<M>`] impl，所有的「message」都被静态地标定了类型，并且可以以异步的方式处理。

「actor」可以派生其他的「actor」，或给「context」添加 streams 和 future.

`Actor` trait 提供了一些方法来控制其生命周期.

为了给「actor」发送消息，就需要使用`Addr`对象。 `Addr`提供了几种方法来发送「message」：

  * `Addr::do_send(M)` - 这个方法忽视发送消息「message」过程中，可能发生的任何错误：如果邮箱（mailbox）已经满了，「message」就会越界添加到队列中；如果「actor」的邮箱已经关闭，「message」就会被静默删除。这个方法并不会返回任何结果，所以，如果邮箱已经关闭，然后发生错误，你也不会得到任何提示。

  * `Addr::try_send(M)` - 这个方法会试着直接立刻马上发送「message」，如果邮箱满了或者关闭了（那个「actor」已经消失），这个方法会返回一个
  [`SendError`]。

  * `Addr::send(M)` - 这个方法会返回一个`Future`对象，其中会包含处理结果。如果返回的 `Future` 对象被删除，那么这个「message」也会被取消掉。

[`Handler<M>`]: https://actix.rs/actix/actix/trait.Handler.html
[`SendError`]: https://actix.rs/actix/actix/prelude/enum.SendError.html

## 受体（Recipient）

受体（Recipient）是一种特化的「address」，它只支持一种特定类型的消息。在要把消息「message」发给不同类型的「actor」的时候，可以使用。

一个「recipient」可以通过调用「address」的 `Addr::recipient()` 创建。


Recipient is a specialized version of an address that supports only one type of message.
It can be used in case the message needs to be sent to a different type of actor.
A recipient object can be created from an address with `Addr::recipient()`.

举个例子，「recipient」可以用在订阅系统中，在下例中，`ProcessSignals`「actor」发送了 `Signal`「message」给所有的订阅者，只要这些订阅者「actor」实现了`Handler<Signal>` trait，就可以接受它
```rust
# // This example is incomplete, so I don't think people can follow it and get value from what it's
# // trying to communicate or teach.

# #[macro_use] extern crate actix;
# use actix::prelude::*;
#[derive(Message)]
#[rtype(result = "()")]
struct Signal(usize);

/// Subscribe to process signals.
#[derive(Message)]
#[rtype(result = "()")]
struct Subscribe(pub Recipient<Signal>);

/// Actor that provides signal subscriptions
struct ProcessSignals {
    subscribers: Vec<Recipient<Signal>>,
}

impl Actor for ProcessSignals {
    type Context = Context<Self>;
}

impl ProcessSignals {

    /// Send signal to all subscribers
    fn send_signal(&mut self, sig: usize) {
        for subscr in &self.subscribers {
           subscr.do_send(Signal(sig));
        }
    }
}

/// Subscribe to signals
impl Handler<Subscribe> for ProcessSignals {
    type Result = ();

    fn handle(&mut self, msg: Subscribe, _: &mut Self::Context) {
        self.subscribers.push(msg.0);
    }
}
# fn main() {}
```
