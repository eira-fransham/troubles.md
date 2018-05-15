---
title: "Towards a Brighter Future for Smart Contracts"
date: 2018-05-14T14:17:23+02:00
draft: false
---

You might have heard recently that here at Parity, our Kovan testnet for Ethereum now supports [smart contracts written in WebAssembly][parity-110]. Deep in the bowels of the Ethereum Foundation's volcano lair, there begins a movement towards [the same ability being integrated into Ethereum proper][ethereum-proper]. I realise in hindsight that my phrasing would make that a "bowel movement", but I'm not changing it. Moving on.

[parity-110]: https://paritytech.io/parity-1-10-opportunity-released/
[ethereum-proper]: https://github.com/ewasm/design

Currently the only feasible way to write smart contracts for Ethereum is to use a language built for EVM. Solidity, Bamboo, Vyper. These are not general-purpose languages that were adapted for Ethereum, they were built from the ground up to be used on Ethereum. This is because the EVM's execution semantics are, to be diplomatic about it, nuts. It bears a lot of the scars of backwards compatibility, and a lot of peculiarities (like 256-bit arithmetic and a limited selection of types) are something that you can't help but expose to the programmer. Hiding it would be too complex and in an environment with the strict limitations of the Ethereum environment. You have to expose as much of the underlying system as possible in order to let the programmer optimise for gas price. Using a general-purpose language would only lead to mismatches between the target VM and the semantics that the language expects.

However, this has many downsides too - Solidity, the most popular and well-supported EVM language by far, still has tooling support leagues behind general-purpose languages that were created well after it, such as Go and Rust. There exist linters for Solidity, but the rules they have are relatively simple, and many of them are included as standard in the compilers of more popular languages (requiring no external tooling at all). More complex linting rules could be written, but that takes up valuable developer time and as much as we love to pretend that Ethereum is taking over the world, the truth is that the pool of developers willing and able to write complex tooling for EVM languages is small.

Not only that, but tooling is especially necessary for Solidity, since it is a language with many footguns and awkward features. It can be easy for smart contract developers both new and old to make [very small mistakes][parity-multisig] that cause catastrophic damage.

[parity-multisig]: https://paritytech.io/the-multi-sig-hack-a-postmortem/

The ideal situation would be to have the tooling support and professional language design of general-purpose languages for the blockchain, and that's what [WebAssembly][wasm] can help provide. WebAssembly is a VM target that (unlike most virtual machines) generally tries to match the semantics of a CPU architecture like ARM or x86 rather than trying to match the semantics of the language that runs on it. As a result, many languages can compile to it unchanged - since most programming languages are built to expose the semantics of the CPU at some level, having the VM emulate a CPU is a good lowest-common-denominator. It also allows us to use world-class optimising compilers that were built for ISAs like x86 and ARM - very important when you're charging by the instruction. `solc` has an `--optimize` flag but as far as I can tell all that it does is [remove jump destinations from the assembly that are never referenced][jumpdest-remover], which might as well be no optimisation at all. In regards to optimisation, it's years behind even basic hobbyist C compilers and decades behind something like LLVM, which often [writes better assembly than a human hand-compiling the same code][better-assembly].

[wasm]: https://webassembly.org/
[jumpdest-remover]: https://github.com/ethereum/solidity/blob/develop/libevmasm/JumpdestRemover.cpp
[better-assembly]: http://troubles.md/posts/the-power-of-compilers/

One downside, however, is that these languages were not built to write smart contracts in. That, however, can be solved with the right library. Since at Parity we write almost all our code in Rust, you might have gathered that we're quite fond of the language. We've had experiments in writing smart contracts in Rust for a long time, but the API is still very clunky. You have to manually convert any state that you want to store to and from a 256-bit number (since that's what the EVM stores things as), and since we generate code at compile-time a lot of tools won't work correctly, which negates a large part of the reason that we wanted to use a general-purpose language in the first place! With the current API, to write a simple contract that just does basic arithmetic on a stored number you need to write a lot of code:

```rust
pub mod foo_contractfoo_contractfoo_contract
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
      let current_state = U256::from(pwasm_ethereum::read(&STATE_KEY));

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

// Declares the dispatch and dispatch_ctor methods
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

This is, in a word, awful. The lines of boilerplate outnumber the lines of real logic by more than 10 to 1 (about 3 lines of real logic buried within over 50 lines of boilerplate). It has a bunch of fluff that has no use at all - the `self` parameter is never used because all state is read and written using `pwasm_ethereum::read` and `pwasm_ethereum::write`. You need to write `#[no_mangle]` over the `call` and `deploy` functions. There is no way that you'd figure out the `pwasm_ethereum::ret(&endpoint.dispatch(&pwasm_ethereum::input()))` incantation without a tutorial, it's totally undiscoverable. You can't get reliable code completion without compiling the entire binary since it uses a procedural macro (which can run arbitrary code). Not only that, but you need the procedural macro because without it you'd have no chance of correctly handling the function calls, except that the only thing that the procedural macro actually does for you is dispatch incoming methods to the right place and deserialize input/serialize output. Another frustrating thing is that although Rust has fantastic support for enforcing immutability (much better than Solidity's equivalent), we never use it here. We just take `&mut self` for everything and then never use it. This is a great first start, and it's exciting to have Rust code really running on our testnet, but it's got a long way to go before it is usable for real smart contract development by regular people.

Last week, however, I started sketching up an idea of what a better smart contract API could look like. I had a few goals:

* Smart contract code should be easy to read and easy to write.
* We should be able to catch obvious bugs at compile-time, and it should be easy to write a linter for anything that is impossible to catch at compile-time.
* It should compile to very compact code to ensure that it's cheap to deploy smart contracts.
* It should be easy to discover how to use the API even without a tutorial (just using [rustdoc][docs]).
* We should integrate as well with the rest of the ecosystem as possible.

[docs]: https://docs.rs/

With that in mind, here is what I would imagine the previous contract would look like with this new API:

```rust
#[macro_use]
extern crate pwasm;
#[macro_use]
extern crate serde_derive;
extern crate serde;

use pwasm::Contract;

#[derive(Serialize, Deserialize)]
pub struct State {
  current: U256,
}

contract!(call);

fn call(state: State, message: pwasm::Request) -> pwasm::Response {
  messages! {
    Add(U256);
    Get() -> U256;
  }

  Contract::new()
    .constructor(|_txdata, initial| State {
      current: initial,
    })
    // These `&mut State` and `&State` annotations aren't
    // necessary, they're just to show that we don't receive
    // a mutable reference to the state in the `Get` method.
    .on_msg_mut::<Add>(|_env, state: &mut State, to_add| {
      state.current += to_add;
    })
    .on_msg::<Get>(|_env, state: &State, ()| state.current.clone())
    .call(state, message)
}
```

Much nicer, I think. It uses `serde` (the standard serialisation and deserialisation framework for Rust) instead of hand-rolling our own serialisation. Hand-rolling serialisation is fine for our own types, but it means that if we want to allow users to serialise their own types we either need them to write a lot of boilerplate or we need to write an equivalent of `serde_derive` for our own serialisation method, duplicating work. It uses Rust's inbuilt mutability guarantees to enforce that we don't do any mutation in the functions that don't need it. It's immutable-by-default, you have to write `_mut` if you want a mutable method. This can prevent bugs caused by accidental mutation and can also allow us to avoid writing the state if we know that the method can't mutate the state, which saves on the gas cost required to save the state. The `contract` and `messages` macros are just `macro_rules!` macros, and you can quite simply write contracts without them. They just expand to the following:

```rust
// ...

#[no_mangle]
fn __pwasm_ethereum_call() {
  pwasm::ret(call(pwasm::read_state(), pwasm::input()));
}

fn call(state: State, message: pwasm::Request) -> pwasm::Response {
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

Additionally, the contract's methods take a `env` parameter which represents the blockchain environment itself, and can be used to (for example) get the current block header or send messages to another contract. We can use Rust's mutability guarantees again to enforce that an immutable method can't call a mutable method or suchlike. We also use a simple type-level state machine to make sure you write precisely one constructor.

Since so much is done in the type system and with regular Rust code, you can easily do something like extracting the interface for your contract into a crate so that someone else writing a contract can get code completion and compile-time type safety for your methods, without us having to write any extra tooling ourselves. Another benefit of so much being regular Rust code is that in order to see everything you need to write a contract you just go to the `Contract` page in the documentation and you see all the methods available to you. Since you need to go through the `state` parameter to access the contract's state and the `env` parameter to access the blockchain, it's easy to work out the complete set of everything a smart contract can do - from the `Contract` page you look at the type signature of `on_msg`/`on_msg_mut`, see that the first parameter is the Ethereum environment and click through to its documentation to see everything that it can do to access the outside world. There are no top-level functions that manipulate global state and if you get anything wrong the compiler will give you a nice error message.

Finally, because it extensively uses Rust's generics it compiles to zero-cost code. The niceities that you see have no overhead compared to writing the verbose version that uses the existing API.

With this abstraction as a base it should be easy to implement other functionality on top. For example, if you need a database and you don't want to have to serialise and deserialise the entire thing every time you access it (for example, if you're creating an ERC20 token and you want to store the mapping between addresses and balances) you could have a `Database` type that internally uses the raw calls to `pwasm_ethereum::read` and `pwasm_ethereum::write` but externally behaves more like Rust's stdlib `HashMap` type. This would also allow you to do things not possible in the current API, like having two databases that can have overlapping keys (by using a unique nonce per instance of the `Database` type). That could be implemented as a library on top of the API I've described here with relatively little code.

I should say that nothing that you see here is set in stone. I've just been playing with the idea of writing a new API for smart contracts and I want to get some feedback from real smart contract developers (which I am not one of, I'm a Rust core developer) and from other members of the community. I'd love to hear your ideas for anything that could be improved here, especially with regards to ergonomics. I'm not trying to make it look like Solidity code, I'm just trying to make it safe and readable.
