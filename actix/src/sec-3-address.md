# Address

Actors 通过交换信息来独立交流. 发送方可以选择性等待回复. Actors 不能被直接引用, 而是通过地址(address)的方式相联系.

有几种方式可以得到Actor的地址, `Actor` 提供了两种快捷方式来启动一个actor. 它们都返回actor的地址.

下面展示一个 `Actor::start()` 方法的使用例. 此例中 `MyActor` actor
是异步的, 并在同一个调用线程中启动, 这个我们会在[SyncArbiter]这一节提到. 

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

An async actor can get its address from the `Context` struct. The context needs to
implement the `AsyncContext` trait. `AsyncContext::address()` provides the actor's address.

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

## Message

To be able to handle a specific message the actor has to provide a
[`Handler<M>`] implementation for this message.
All messages are statically typed. The message can be handled in an asynchronous
fashion. The actor can spawn other actors or add futures or
streams to the execution context. The actor trait provides several methods that allow
 controlling the actor's lifecycle.

To send a message to an actor, the `Addr` object needs to be used. `Addr` provides several
ways to send a message.

  * `Addr::do_send(M)` - this method ignores any errors in message sending. If the mailbox
  is full the message is still queued, bypassing the limit. If the actor's mailbox is closed,
  the message is silently dropped. This method does not return the result, so if the
  mailbox is closed and a failure occurs, you won't have an indication of this.

  * `Addr::try_send(M)` - this method tries to send the message immediately. If
  the mailbox is full or closed (actor is dead), this method returns a
  [`SendError`].

  * `Addr::send(M)` - This message returns a future object that resolves to a result
  of a message handling process. If the returned `Future` object is dropped, the
  message is cancelled.

[`Handler<M>`]: https://actix.rs/actix/actix/trait.Handler.html
[`SendError`]: https://actix.rs/actix/actix/prelude/enum.SendError.html

## Recipient

Recipient is a specialized version of an address that supports only one type of message.
It can be used in case the message needs to be sent to a different type of actor.
A recipient object can be created from an address with `Addr::recipient()`.

For example recipient can be used for a subscription system. In the following example
`ProcessSignals` actor sends a `Signal` message to all subscribers. A subscriber can
be any actor that implements the `Handler<Signal>` trait.

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
