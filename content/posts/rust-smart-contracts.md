---
title: "Towards a Brighter Future for Smart Contracts"
date: 2018-05-14T14:17:23+02:00
draft: false
---

You might have heard recently that here at Parity, our Kovan testnet for Ethereum now supports [smart contracts written in WebAssembly][parity-110]. Deep in the bowels of the Ethereum Foundation's volcano lair, there begins a movement towards [the same ability being integrated into Ethereum proper][ethereum-proper]. I realise in hindsight that my phrasing would make that a "bowel movement", but I'm not changing it. Moving on.

[parity-110]: https://paritytech.io/parity-1-10-opportunity-released/
[ethereum-proper]: https://github.com/ewasm/design

Currently the only feasible way to write smart contracts for Ethereum is to use a language built for EVM. Solidity, Bamboo, Vyper. These are not general-purpose languages that were adapted for Ethereum, they were built from the ground up to be used on Ethereum. This is because the EVM's execution semantics are, to be diplomatic about it, nuts. It bears a lot of the scars of backwards compatibility, and a lot of peculiarities (like 256-bit arithmetic and a limited selection of types) are something that you can't help but expose to the programmer. Hiding it would be too complex and in an environment with the strict limitations of the Ethereum environment. You have to expose as much of the underlying system as possible in order to let the programmer optimise for gas price. Using a general-purpose language would only lead to mismatches between the target VM and the semantics that the language expects.

However, this has many downsides too - Solidity, the most popular and well-supported EVM language by far, still has tooling support that is leagues behind general-purpose languages that were created well after it, such as Go and Rust. There exist linters for Solidity, but the rules they have are relatively simple, and many of them are included as standard in the compilers of more popular languages (requiring no external tooling at all). More complex linting rules could be written, but that takes up valuable developer time and as much as we love to pretend that Ethereum is taking over the world, the truth is that the pool of developers willing and able to write complex tooling for EVM languages is small.

Not only that, but tooling is especially necessary for Solidity, since it is a language with many footguns and awkward features. It can be easy for smart contract developers both new and old to make [very small mistakes][parity-multisig] that cause catastrophic damage.

[parity-multisig]: https://paritytech.io/the-multi-sig-hack-a-postmortem/

The ideal situation would be to have the tooling support and professional language design of general-purpose languages for the blockchain, and that's what [WebAssembly][wasm] can help provide. WebAssembly is a VM target that (unlike most virtual machines) generally tries to match the semantics of a CPU architecture like ARM or x86 rather than trying to match the semantics of the language that runs on it. As a result, many languages can compile to it unchanged. Since most programming languages are built to expose the semantics of the CPU at some level, having the VM emulate a CPU is a good lowest common denominator. It also allows us to use world-class optimising compilers that were built for ISAs like x86 and ARM - very important when you're charging by the instruction. `solc` has an `--optimize` flag but as far as I can tell all that it does is [remove unused jump targets from the list of allowed targets in the assembly's metadata][jumpdest-remover], which is more of a security feature than an optimisation. Certainly, it's years behind even hobbyist C compilers and decades behind something like LLVM, which often [writes better assembly than a human hand-compiling the same code][better-assembly].

[wasm]: https://webassembly.org/
[jumpdest-remover]: https://github.com/ethereum/solidity/blob/develop/libevmasm/JumpdestRemover.cpp
[better-assembly]: http://troubles.md/posts/the-power-of-compilers/

One downside, however, is that these general-purpose languages were not built to write smart contracts in. That, however, can be solved with the right library. If you follow Parity at all you might have gathered that we're quite fond of Rust, since we [write almost all of our code in it][all-our-code]. We've had experiments in writing smart contracts in Rust for a long time, but the API is still very clunky. You have to manually convert any state that you want to store to and from a 256-bit number (since that's what the EVM stores things as), and since we generate code at compile-time a lot of tooling built for Rust won't work correctly, which negates a large part of the reason that we wanted to use a general-purpose language in the first place. With the current API, to write a simple contract that just does basic arithmetic on a stored number you need to write a lot of code:

[all-our-code]: https://github.com/paritytech?utf8=%E2%9C%93&q=&type=&language=rust

```rust
pub mod foo_contract {
  use pwasm_ethereum;
  use pwasm_std::*;
  use pwasm_std::hash::H256;
  use bigint::U256;

  use pwasm_abi_derive::eth_abi;

  static STATE_KEY: H256 = H256([
    2, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0,
    0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0
  ]);

  #[eth_abi(FooEndpoint)]
  pub trait FooInterface {
    /// This is called when the contract gets deployed on-chain
    fn constructor(&mut self, initial: U256);
    /// Add a number to the total
    fn add(&mut self, to_add: U256);
    /// Get the total
    fn get(&mut self) -> U256;
  }

  pub struct FooContract;

  impl FooInterface for FooContract {
    fn constructor(&mut self, initial: U256) {
      pwasm_ethereum::write(
        &STATE_KEY,
        &initial.into()
      );
    }
    
    fn add(&mut self, to_add: U256) {
      let current_state: U256 =
        pwasm_ethereum::read(&STATE_KEY).into();

      let new_state = current_state + to_add;
      
      pwasm_ethereum::write(
        &STATE_KEY, &new_state.into()
      );
    }

    fn get(&mut self) -> U256 {
      pwasm_ethereum::read(&STATE_KEY).into()
    }
  }
}

// We have to import a trait in order to use `dispatch`
// and `dispatch_ctor`.
use pwasm_abi::eth::EndpointInterface;

#[no_mangle]
pub fn call() {
  let mut endpoint = foo_contract::FooEndpoint::new(
    foo_contract::FooContract
  );

  pwasm_ethereum::ret(
    &endpoint.dispatch(&pwasm_ethereum::input())
  );
}

#[no_mangle]
pub fn deploy() {
  let mut endpoint = foo_contract::FooEndpoint::new(
    foo_contract::FooContract
  );

  endpoint.dispatch_ctor(&pwasm_ethereum::input());
}
```

This is, in a word, awful. The lines of boilerplate outnumber the lines of real logic by more than 10 to 1 (about 3 lines of real logic buried within over 50 lines of boilerplate). It has a bunch of fluff that has no use at all - the `self` parameter is never used because all state is read and written using `pwasm_ethereum::read` and `pwasm_ethereum::write`. You need to write `#[no_mangle]` over the `call` and `deploy` functions. There is no way that you'd figure out the `pwasm_ethereum::ret(&endpoint.dispatch(&pwasm_ethereum::input()))` incantation without a tutorial, it's totally undiscoverable. You can't get reliable code completion without compiling the entire binary since it uses a procedural macro (which can run arbitrary code). Not only that, but you need the procedural macro because without it you'd have no chance of correctly handling the function calls, except that the only thing that the procedural macro actually does for you is dispatch incoming methods to the right place and deserialize input/serialize output. Another frustrating thing is that although Rust has fantastic support for enforcing immutability (much better than Solidity's equivalent), we never use it here. We just take `&mut self` for everything and then never use it. This is a good first try, and it's exciting to have Rust code really running on our testnet, but it's got a long way to go before it is usable for real smart contract development by regular people.

Last week, however, I started sketching up an idea of what a better smart contract API could look like. I had a few goals:

* Smart contract code should be easy to read and easy to write.
* We should be able to catch obvious bugs at compile-time, and it should be easy to write a linter for anything that is impossible to catch at compile-time.
* It should compile to very compact code to ensure that it's cheap to deploy smart contracts.
* It should be easy to discover how to use the API even without a tutorial (just using [rustdoc][docs]).
* We should integrate as well with the rest of the ecosystem as possible, so that we can reuse their tools and avoid reimplementing code that's already implemented elsewhere.

[docs]: https://docs.rs/

With that in mind, here is what I would imagine the previous contract would look like with this new API (the crate name I chose is just `pwasm` but that's up for bikeshedding):

```rust
#[macro_use]
extern crate pwasm;
#[macro_use]
extern crate serde_derive;
extern crate serde;

use pwasm::{Contract, ContractDef};

#[derive(Serialize, Deserialize)]
pub struct State {
  current: U256,
}

contract!(foo_contract);

fn foo_contract() -> impl ContractDef<State> {
  messages! {
    Add(U256);
    Get() -> U256;
  }

  Contract::new()
    .constructor(|_txdata, initial| State {
      current: initial,
    })
    // These type annotations aren't necessary, they're just
    // to show that we don't receive a mutable reference to the
    // state in the `Get` method.
    .on_msg_mut::<Add>(|_env: &mut EthEnv, state: &mut State, to_add| {
      state.current += to_add;
    })
    .on_msg::<Get>(|_env: &EthEnv, state: &State, ()| state.current.clone())
}
```

Much nicer, I think. It may look weird at first since it no longer looks like defining functions on a type, but there's a method to my decision to get rid of, um, methods. For a start, using types to represent methods makes it easier to define idiomatic interfaces to external contracts (see the `const NAME` in the macro expansion below). If an external contract uses a different naming convention to Rust's (which if you're calling a Solidity contract, it probably does) you don't have to have the method names exposed to Rust directly match up with the external contract's. Doing it this way also enables us to better utilise Rust's tooling, we get documentation for `Contract` for free (in the form of `rustdoc`), whereas we'd have to manually maintain documentation on the syntax for our macro. With this API, new features can be done by libraries using the trait system, whereas prodedural macros don't compose by default. For example, you could implement a library for defining contracts as state machines without touching the code of `pwasm` itself.

One way to look at this is that it matches the semantics of a web server like `express`, you have immutable `GET` requests and mutable `POST` requests, all of which have endpoints on well-defined paths and have well-defined input types. The path that you access in a web server corresponds to the concept of a message in a contract. For example, in Rust you can use the `actix-web` crate to make a server that looks like the following:

```rust
server::new(|| {
  App::new()
    .route(
      "/add/{a}/{b}",
      http::Method::GET,
      |info: Path<(u32, u32)>| {
        format!("{}", info.0 + info.1)
      }
    )
  }).bind("127.0.0.1:8080")
    .unwrap()
    .run();
```

In `pwasm` that would look like this:

```rust
messages! {
  Add(u32, u32) -> u32;
}

Contract::new()
  .on_msg::<Add>(|_, _, (a, b)| a + b)
```

It uses `serde` (the standard serialisation and deserialisation framework for Rust) instead of hand-rolling our own serialisation. Hand-rolling serialisation is fine for our own types, but it means that if we want to allow users to serialise their own types we either need them to write a lot of boilerplate or we need to write an equivalent of `serde_derive` for our own serialisation method, duplicating work. It uses Rust's inbuilt mutability guarantees to enforce that we don't do any mutation in the functions that don't need it. It's immutable-by-default, you have to write `_mut` if you want a mutable method. This can prevent bugs caused by accidental mutation and can also allow us to avoid spending gas to save the state if we know that the method can't mutate it. The `contract` and `messages` macros are just `macro_rules!` macros, and you can quite simply write contracts without them. They just expand to something like the following:

```rust
// ...

#[no_mangle]
fn __pwasm_ethereum_call() {
  pwasm::ret(
    foo_contract().call(pwasm::read_state(), pwasm::input())
  );
}

#[no_mangle]
fn __pwasm_ethereum_deploy() {
  let state = foo_contract().construct(pwasm::input());
  pwasm::write_state(state);
}

fn foo_contract() -> impl ContractDef<State> {
  struct Add;
  struct Get;

  impl pwasm::Message for Add {
    type Input = U256;
    type Output = ();
    
    const NAME: &str = "Add";
  }

  impl pwasm::Message for Get {
    type Input = ();
    type Output = U256;
    
    const NAME: &str = "Get";
  }

  // ...
}
```

Additionally, the contract's methods take a `EthEnv` parameter which represents the blockchain environment itself, and can be used to (for example) get the current block header or send messages to another contract. We can use Rust's mutability guarantees again to enforce that an immutable method can't call a mutable method or suchlike. We also use a simple type-level state machine to make sure you write no more than one constructor (although it's not yet possible in Rust's type system to prevent you writing two handlers for the same message).

Another benefit of so much being regular Rust code is that you can go to the `Contract` page in the documentation and quite easily infer how to write a contract from that alone. Since you need to go through the `env` parameter to access the blockchain, it's easy to work out the complete set of everything a smart contract can do just by going to the documentation page for `EthEnv`. There are no top-level functions that manipulate global state.

Finally, it compiles to code without any unnecessary overhead because it extensively uses Rust's generics. Plus `constructor`, `on_msg` and `on_msg_mut` are all `const fn`s, and so the contract gets created at compile-time instead of runtime (just like with the macro). The API has no overhead compared to writing the verbose version that uses the existing API.

With this abstraction as a base it should be easy to implement other functionality on top. For example, if you need a key/value store and you don't want to have to serialise and deserialise the entire thing every time you access it (for example, if you're creating an ERC20 token and you want to store the mapping between addresses and balances) you could have a `Database` type that internally uses the raw calls to `pwasm_ethereum::read` and `pwasm_ethereum::write` but externally behaves more like Rust's stdlib `HashMap` type. This would also allow you to do things not possible in the current API, like having two databases that can have overlapping keys (by using a unique nonce per instance of the `Database` type). That could be implemented as a library on top of the API I've described here with relatively little code.

I should say that nothing that you see here is set in stone. I've just been playing with the idea of writing a new API for smart contracts and I want to get some feedback from real smart contract developers and from Rust programmers who are interested in writing some smart contracts. I'm not a smart contract developer myself, I'm from our Rust team, but I'm interested in making smart contracts safer and easier to write. I'd love to hear your ideas for anything that could be improved here, especially with regards to ergonomics. I'm not trying to make it look like Solidity code, I'm just trying to make it safe and readable. A simple working-but-extremely-incomplete implementation that doesn't yet run on the actual blockchain is on my GitHub account [here][pwasm-api] if you would like to see how this could be implemented.

[pwasm-api]: https://github.com/Vurich/wasm-contract-api
