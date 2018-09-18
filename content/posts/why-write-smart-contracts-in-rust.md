---
title: "Why Write Smart Contracts in Rust?"
date: 2018-07-05T13:43:49+02:00
draft: false
---

<div class="spon">This post was sponsored by my employer, <a href="http://paritytech.io/">Parity Technologies</a></div>

As I alluded to in [my previous post on WebAssembly smart contracts][rust-smart-contracts], it's somewhat of an open secret that Solidity and the EVM ecosystem isn't up to par with other development environments like web or mobile development. Debugging it is difficult, the tooling support for essentials like autocompletion and go-to-definition might as well be non-existant, not to mention that developers well-versed in Solidity programming are extremely rare. Even some of the most-experienced Solidity developers in the world still shoot themselves in the feet with its wide array of novel and interesting footguns.

[rust-smart-contracts]: {{< ref "/posts/rust-smart-contracts.md" >}}

The future of smart contracts, in my eyes and the eyes of many others, lies with [WebAssembly][webassembly]. This is a virtual machine specification which essentially acts as a portable and simple RISC ISA - since it matches the runtime model of a CPU many existing languages can be compiled to it unchanged. Apart from special-case DSLs like Solidity most languages expose the runtime model of the CPU somehow. Not only that, but its similarity to the CPU allows it to be compiled to incredibly efficient machine code without complex optimisations that can affect correctness and increase code complexity.

[webassembly]: https://webassembly.org/

At Parity we're working on a library called Fleetwood, intended to make writing smart contracts in Rust as convenient as writing them in Solidity. Rust is still a young language, but the tooling support is already years ahead of Solidity. It has an integrated test and benchmark runner, a linter, a code formatter, syntax highlighting in every text editor I can think of and on GitHub, not to mention that the language itself has many features that makes writing Rust code both more ergonomic and easier to get right than the same in Solidity. Additionally, Rust data structures are very compact - actually more compact than C in many cases since the compiler reorders struct fields to make each type as small as possible. This is important in the space-constrained world of the blockchain.

A sample of what a simple ERC20 token contract written in this library looks like:

```rust
messages! {
    TotalSupply() -> U256;
    BalanceOf(Address) -> U256;
    Transfer(Address, U256) -> bool;
}

state! {
    struct State {
        balances: Database<Address, U256>,
        total: U256,
    }
}

Contract::new()
    .constructor(|_, total_supply: U256| {
        let mut database = Database::new();
        database.insert(&Address::default(), total_supply.clone());
        State {
            total: total_supply.into(),
            balances: database.into(),
        }
    })
    .on_msg::<BalanceOf>(|_env, state, address| state.balances.get(&address))
    .on_msg::<TotalSupply>(|_env, state, ()| *state.total)
    .on_msg_mut::<Transfer>(|env, state, (to, amount)| {
        if amount == U256::zero() {
            return false;
        }
        let from = env.original_caller();

        let existing_from = state.balances.get(&from);
        let existing_to = state.balances.get(&to);

        if existing_from >= amount {
            state.balances.insert(&from, existing_from - amount);
            state.balances.insert(&to, existing_to + amount);

            true
        } else {
            false
        }
    })
```

This is a reimplementation of the ERC20 contract given as an example in the tutorial for [our old API][pwasm-tutorial]. The "message" concept might seem a bit alien to you if you're used to these being methods on a contract but they work the exact same way. The "message" language betrays one of the magic powers that this library has - abstracting different WebAssembly smart contract platforms away from you. Currently the only WebAssembly smart contract platform supported by Parity is the Kovan testnet, but the idea is that you will be able to write contracts that work on either Kovan, Ethereum mainnet (if and when that supports WebAssembly), Polkadot and our yet-to-be-announced proof-of-stake smart contract platform built on [Substrate][substrate]. If you need to use specific features of any particular network you can be explicit about that, and maybe enable or disable certain features or smart contract methods depending on whether the chain supports what those features require. This can be done entirely within the Rust trait system, and so you can find out whether your contract will work on a particular network without even attempting to compile your contract for that network. Our intention is that a smart contract can start its life on Kovan and then as more networks come to maturity you can move to them with little-to-no code changes, and any necessary code changes being guided by the compiler.

[pwasm-tutorial]: https://github.com/paritytech/pwasm-tutorial
[substrate]: {{< ref "/posts/what-is-substrate.md" >}}

The "message" language is an abstraction of methods to include asynchronous methods - currently method calls in Ethereum and Kovan are synchronous but with a totally asynchronous smart contract platform you can have implicit parallelism and cheaper-and-better sharding without affecting correctness. Sharding Ethereum is artificially taxed with the requirement to maintain the existing synchronous semantics. The "message" concept supports both synchronous and actor-model asynchronous code, code written to accomodate asynchronous chains will work on Ethereum too but will be able to take advantage of future improvements in blockchain technology. We hope to reduce the difficulty of writing asynchronous code by allowing you to write synchronous-style code with the [`futures`][futures-rs] library. The `futures` library will be the backbone of Rust's `async`/`await` syntax, which will allow code that needs conceptual synchronicity to be simple and ergonomic without sacrificing the benefits of asynchronicity.

[futures-rs]: https://github.com/rust-lang-nursery/futures-rs

Having said that, the current version of the library is still synchronous. Asynchronicity is a future goal but is not yet implemented. When it is implemented, Rust's strong compile-time guarantees mean that you should just need to follow the compiler errors and add `await` where necessary. Even if we never take advantage of asynchronicity, the more modern smart contract platforms that our library is written to be compatible with will be significantly more efficient than Ethereum by virtue of a more-efficient consensus mechanism and a more-performant virtual machine.

For smart contracts that don't need Ethereum's more stable value and wide-reaching token supply (read: any blockchain project that isn't an ICO) you could imagine that the improved performance of the more modern blockchains could be an ["unfair advantage"][unfair] that other dApp developers can't take advantage of without rebuilding their product from scratch. In the blockchain world, performance is even more important than in other software. The more performant your software is, the lower the price of using it. You can start deploying now and let the platform improve underneath you.

[unfair]: https://hackernoon.com/startup-advice-build-for-your-unfair-advantage-f68efc61c599

If you're a developer and you care more about ergonomics and ease of use - since this library is just normal Rust code without any code generation or macro magic it works out-of-the-box with code completion using [RLS][rls] and (as long as you don't mind mocking out external contracts) immediate-feedback local testing with `cargo test`. Test-driven development - not impossible in Solidity but certainly difficult - can be done without even having a blockchain client installed at all, although if you want to interact with contracts not written in this library as part of your tests you will have to connect to a client. It's possible that in the future we will embed an instance of the Ethereum virtual machine in order to allow you to include Solidity contracts as part of your tests.

[rls]: https://github.com/rust-lang-nursery/rls

I hope this short article gave you an idea of some of the business and process benefits of writing smart contracts in Rust. For some more developer-focussed benefits to writing smart contract code in Rust, I highly recommend you check out the [first article in this series][rust-smart-contracts].
