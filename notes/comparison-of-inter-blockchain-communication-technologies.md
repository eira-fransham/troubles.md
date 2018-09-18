All 3 of Aion, Cosmos and Polkadot attempt to address the existing blockchain ecosystem's tooling problem in different ways

- Aion by having a bespoke language that is a strict subset of possibly the best-supported language in the world (tooling-wise);
- Cosmos by having a chain-creation toolkit that is disconnected from implementation language via a multiprocess client-server model;
- Polkadot by having a chain-creation toolkit that is disconnected from implementation language by using a C interface.
    - This is arguably less efficient/useful than a client-server model since it requires a Wasm compilation target (not available for JS, for example), although in practice this allows useful features like on-chain governance-driven updates. Cosmos does not seem to have any way to do governance for forks.

Aion:
- Appears to have a routing system instead of a hub-and-spoke mechanism, except it still has a bespoke blockchain that it uses to stake bridgers
- Why can't the bridgers be staked/rewarded on their own chain? Since it's unidirectional. There's no incentive for blockchain builders to build that into their chain, I guess? You could say that about all of these technologies, though.
- Extremely questionable use of LLVM for smart contracts
    - Compiler bombs
    - Nondeterminism due to undefined behaviour
    - Nondeterminism due to outright bugs
    - No ability to accurately price contract executions
- Recompiling Solidity contracts to use 128-bit integers? Why? 
- Weird "proof-of-intelligence" system - I like the idea, but why is it baked into an otherwise-unrelated technology?
    - It's not just a minor addition, either, 40% of consensus is based on this PoW algorithm.
- Uses a subset of Java for scripting and whole client is written in Java

Cosmos:
- Allows an arbitrary DAG in its communication protocol but the flagship product they're pushing is a hub like Polkadots but only transferring token value (i.e. no centralised arbitrary message bus)
    - This inverts Polkadot's order of priorities
    - In theory Cosmos's communication protocol or Polkadot's API could be reimplemented but in practice neither are likely. Probably Substrate chains will run on Polkadot and Cosmos chains will run using Cosmos Hub.
- Block reward system causes worries of centralisation, similar to DPoS.
- ABCI directly comparable to Substrate, both enforce a PoA system and handle external chains that don't conform to the interface by writing a bridge chain. Substrate also handles seamless updates to the runtime and since it doesn't require an external program it's less tied to the client being a piece of desktop software (it works more like a DSL). Substrate is also deterministic by default since it uses Wasm for code that affects consensus, but you can still get forks by having verification and block-creation code fall out of sync.
- Like Polkadot, Cosmos allows interop between private and public blockchains.
- The main difference is that Cosmos Hub doesn't seem to have any way to transfer arbitrary messages. You could, however, write a hub that does.
- IBC is totally glossed over in the whitepaper, which kinda invalidates the idea that Cosmos is the communication and Cosmos Hub is just one implementation.
- Does Cosmos allow shared security?
- Currently no way in Tendermint to do on-chain upgrades, developers are expected to organise versioning themselves. There is a proposal to allow version numbers in block headers but all that really is is a safety switch to prevent hard forks, you still have to distribute the runtime on a side channel.
- Tendermint has concept of "voting power", which more directly addresses proof-of-stake than Polkadot's validator set at the expense of greater complexity plus allows PoA networks to have variable voting weight (for example, based on some real-world staking)

Polkadot:
- Has no flagship decentralised exchange like Cosmos DEX - it's up to developers to build their own.
- Only hub-and-spoke, doesn't allow a DAG at this time (in practice this is the same as Cosmos with Cosmos Hub, however).

ICON:
- Appears to be very similar to Cosmos, with a hub-and-spoke model that also allows multi-layer hubs.
- Interesting concept of not having all dApps on-chain at once - you must download them individually if you want to verify their work or use them.
- It looks like you can't become a validator just by staking, you have to stay above a certain level of their proprietary AI score.
- Unlike Polkadot (and Cosmos?) it appears to disallow new blocks being created until the previous block has been finalised, making it fully serial instead of partially asynchronous. Why? This seems strictly worse than Substrate's implicit voting.
- The "leader" or chosen authority for any given block appears to be chosen predictably, which could allow forcing a node to appear offline.
- Block rewards are distributed via mysterious and opaque "I-score", which is nominally determined via deep learning but the whitepaper doesn't actually explain what that means. Specifically, it doesn't have any specification of what the network is optimising for.

Wanchain: 
- Attempts to supply loans, debt, deposit negotiation and other bank functions in a decentralised way (probably these are just dApps but it's encouraging that they're at least aiming towards this).
