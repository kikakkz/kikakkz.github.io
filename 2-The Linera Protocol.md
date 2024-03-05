# [The Linera Protocol](https://linera.dev/core_concepts.html#the-linera-protocol)

We now describe the main concepts of the Linera protocol in more details.

# [Overview](https://linera.dev/core_concepts/overview.html#overview)

Linera is a decentralized infrastructure optimized for Web3 applications that require guaranteed performance for an unlimited number of active users.

The core idea of the Linera protocol is to run many lightweight blockchains, called **microchains**, in parallel in a single set of validators.

## [How does it work?](https://linera.dev/core_concepts/overview.html#how-does-it-work)

In Linera, user wallets operate their own microchains. The owner of a chain chooses when to add new blocks to the chain and what goes inside the blocks. Such chains with a single user are called **user chains**.

Users may add new blocks to their chains in order to process **incoming messages** from other chains or to execute secure **operations** on their accounts, for instance to transfer assets to another user.

Importantly, validators ensure that all new blocks are **valid**. For instance, transfer operations must originate from accounts with sufficient funds; and incoming messages must have been actually sent from another chain. Blocks are verified by validators in the same way for every chain.

A Linera **application** is a Wasm program that defines its own state and operations. Users can publish bytecode and initialize an application on one chain, and it will be automatically deployed to all chains where it is needed, with a separate state on each chain.

To ensure coordination across chains, an application may rely on asynchronous **cross-chain messages**. Message payloads are application-specific and opaque to the rest of the system.

```ignore
                               ┌───┐     ┌───┐     ┌───┐
                       Chain A │   ├────►│   ├────►│   │
                               └───┘     └───┘     └───┘
                                                     ▲
                                           ┌─────────┘
                                           │
                               ┌───┐     ┌─┴─┐     ┌───┐
                       Chain B │   ├────►│   ├────►│   │
                               └───┘     └─┬─┘     └───┘
                                           │         ▲
                                           │         │
                                           ▼         │
                               ┌───┐     ┌───┐     ┌─┴─┐
                       Chain C │   ├────►│   ├────►│   │
                               └───┘     └───┘     └───┘
```

The number of applications present on a single chain is not limited. On the same chain, applications are **composed** as usual using synchronous calls.

The current Linera SDK uses **Rust** as a source language to create Wasm applications. It relies on the normal Rust toolchains so that Rust programmers can work in their preferred environments.

## [How does Linera compare to existing multi-chain infrastructure?](https://linera.dev/core_concepts/overview.html#how-does-linera-compare-to-existing-multi-chain-infrastructure)

Linera is the first infrastructure designed to support many chains in parallel, and notably an arbitrary number of **user chains** meant to be operated by user wallets.

In traditional multi-chain infrastructures, each chain usually runs a full blockchain protocol in a separate set of validators. Creating a new chain or exchanging messages between chains is expensive. As a result, the total number of chains is generally limited. Some chains may be specialized to a given use case: these are called "app chains".

In contrast, Linera is optimized for a large number of user chains:

- Users only create blocks in their chain when needed;
- Creating a microchain does not require onboarding validators;
- All chains have the same level of security;
- Microchains communicate efficiently using the internal networks of validators;
- Validators are internally sharded (like a regular web service) and may adjust their capacity elastically by adding or removing internal workers.

> Besides user chains, the [Linera protocol](https://linera.io/whitepaper) is designed to support other types of microchains, called "permissioned" and "public" chains. Public chains are operated by validators. In this regard, they are similar to classical blockchains. Permissioned chains are meant to be used for temporary interactions between users, such as atomic swaps.

## [Why build on top of Linera?](https://linera.dev/core_concepts/overview.html#why-build-on-top-of-linera)

We believe that many high-value use cases are currently out of reach of existing Web3 infrastructures because of the challenges of serving **many active users** simultaneously without degrading user experience (unpredictable fees, latency, etc).

Examples of applications that require processing time-sensitive transactions created by many simultaneous users include:

- real-time micro-payments and micro-rewards,
- social data feeds,
- real-time auction systems,
- turn-based games,
- version control systems for software, data pipelines, or AI training pipelines.

Lightweight user chains are instrumental in providing elastic scalability but they have other benefits as well. Because user chains have fewer blocks than traditional blockchains, in Linera, the full-nodes of user chains will be embedded into the users' wallets, typically deployed as a browser extension.

This means that Web UIs connected to a wallet will be able to query the state of the user chain directly (no API provider, no light client) using familiar frameworks (React/GraphQL). Furthermore, wallets will be able to leverage the full node as well for security purposes, including to display meaningful confirmation messages to users.

## [What is the current state of the development of Linera?](https://linera.dev/core_concepts/overview.html#what-is-the-current-state-of-the-development-of-linera)

The [reference open-source implementation](https://github.com/linera-io/linera-protocol) of Linera is under active development. It already includes a Web3 SDK with the necessary features to prototype simple Web3 applications and test them locally on the same machine. Notably, Web UIs (possibly reactive) can already be built on top of Wasm-embedded GraphQL services, and tested locally in the browser.

The main limitations of our current Web3 SDK include:

- Web UIs need to query a local HTTP service acting as a wallet. This setup is meant to be temporary and for testing only: in the future, web UIs will securely connect to a Wallet installed as a browser extension, as usual.
- Only user chains are currently available for testing and documented in this manual. Support for other types of chain (called "public" and "permissioned") will be added later.

The main development workstreams of Linera, beyond its SDK, can be broken down as follows.

### [Core Protocol](https://linera.dev/core_concepts/overview.html#core-protocol)

-  User chains
-  Permissioned chain (core protocol only)
-  Cross-chain messages
-  Cross-chain pub/sub channels (initial version)
-  Bytecode publishing
-  Application creation
-  Reconfigurations of validators
-  Initial support for gas fees
-  Initial support for storage fees and storage limits
-  External services to help users create their first chain
-  Permissioned chains (adding operation access control, demo of atomic swaps, etc)
-  Public chains (adding leader election, inbox constraints, etc)
-  Support for easy onboarding of user chains into a new application (removing the need to accept requests)
-  Improved pub/sub channels (removing the need to accept subscriptions)
-  Blob storage for applications (generalizing bytecode storage)
-  Support for archiving chains
-  Wallet-friendly chain clients (compile to Wasm/JS, do not maintain execution states for other chains)
-  General tokenomics and incentives for all stakeholders
-  Governance on the admin chain (e.g. DPoS, onboarding of validators)
-  Auditing procedures

### [Wasm VM integration](https://linera.dev/core_concepts/overview.html#wasm-vm-integration)

-  Support for the Wasmer VM
-  Support for the Wasmtime VM (experimental)
-  Test gas metering and deterministic execution across VMs
-  Composing Wasm applications on the same chain (initial version)
-  Enhanced composability with "sessions"
-  Support for non-blocking (yet deterministic) calls to storage
-  Support for read-only GraphQL services in Wasm
-  Support for mocked system APIs (initial version)
-  More efficient cross-application calls
-  Improve host/guest stub generation to make mocks easier (currently wit-bindgen)
-  Compile user full node to Wasm/JS

### [Storage](https://linera.dev/core_concepts/overview.html#storage)

-  Object management library ("linera-views") on top of Key-Value store abstraction
-  Support for Rocksdb
-  Experimental support for DynamoDb
-  Initial derive macros for GraphQL
-  Initial support for ScyllaDb
-  Make library fully extensible by users (requires better GraphQL macros)
-  Performance benchmarks and improvements (including faster state hashing)
-  Production-grade support for the chosen main database
-  Support global object locks (needed for dynamic sharding)
-  Tooling for debugging
-  Make the storage library easy to use outside of Linera

### [Validator Infrastructure](https://linera.dev/core_concepts/overview.html#validator-infrastructure)

-  Simple TCP/UDP networking (used for benchmarks only)
-  GRPC networking
-  Basic frontend (aka. proxy) supporting fixed internal shards
-  Observability
-  Initial kubernetes support in CI
-  Initial deployment using a cloud provider
-  New frontend to support dynamic shard assignment
-  Cloud integration to demonstrate elastic scaling

### [Web3 SDK](https://linera.dev/core_concepts/overview.html#web3-sdk)

-  Initial traits for contract and service interfaces
-  Support for unit testing
-  Support for integration testing
-  Local GraphQL service to query and browse system state
-  Local GraphQL service to query and browse application states
-  Use GraphQL mutations to execute operations and create blocks
-  Initial support for unit tests
-  Support for integration tests
-  Initial ABIs for contract and service interfaces
-  Allowing message sender to pay for message execution fees
-  Bindings to use native cryptographic primitives from Wasm
-  Allowing applications to pay for user fees
-  Allowing applications to use permissioned chains and public chains
-  Wallet as a browser extension (no VM)
-  Wallet as a browser extension (with Wasm VM)

# [Microchains](https://linera.dev/core_concepts/microchains.html#microchains)

This section provides an introduction to microchains, the main building block of the Linera Protocol. For a more formal treatment refer to the [whitepaper](https://linera.io/whitepaper).

## [Background](https://linera.dev/core_concepts/microchains.html#background)

A **microchain** is a chain of blocks describing successive changes to a shared state. We will use the terms *chain* and *microchain* interchangeably. Linera microchains are similar to the familiar notion of blockchain, with the following important specificities:

- An arbitrary number of microchains can coexist in a Linera network, all sharing the same set of validators and the same level of security. Creating a new microchain only takes one transaction on an existing chain.
- The task of proposing new blocks in a microchain can be assumed either by validators or by end users (or rather their wallets) depending on the configuration of a chain. Specifically, microchains can be *single-owner*, *permissioned*, or *public*, depending on who is authorized to propose blocks.

## [Cross-Chain Messaging](https://linera.dev/core_concepts/microchains.html#cross-chain-messaging)

In traditional networks with a single blockchain, every transaction can access the entire execution state. This is not the case in Linera where the state of a microchain is only affected by its own blocks.

Cross-chain messaging is a way for different microchains to communicate with each other asynchronously. This method allows applications and data to be distributed across multiple chains for better scalability. When an application on one chain sends a message to another chain, a cross-chain request is created. These requests are implemented using remote procedure calls (RPCs) within the validators' internal network, ensuring that each request is executed only once.

Instead of immediately modifying the target chain, messages are placed first in the target chain's **inbox**. When an owner of the target chain creates its next block in the future, they may reference a selection of messages taken from the current inbox in the new block. This executes the selected messages and applies their messages to the chain state.

Below is an example set of chains sending asynchronous messages to each other over consecutive blocks.

```ignore
                               ┌───┐     ┌───┐     ┌───┐
                       Chain A │   ├────►│   ├────►│   │
                               └───┘     └───┘     └───┘
                                                     ▲
                                           ┌─────────┘
                                           │
                               ┌───┐     ┌─┴─┐     ┌───┐
                       Chain B │   ├────►│   ├────►│   │
                               └───┘     └─┬─┘     └───┘
                                           │         ▲
                                           │         │
                                           ▼         │
                               ┌───┐     ┌───┐     ┌─┴─┐
                       Chain C │   ├────►│   ├────►│   │
                               └───┘     └───┘     └───┘
```

The Linera protocol allows receivers to discard messages but not to change the ordering of selected messages inside the communication queue between two chains. If a selected message fails to execute, it is skipped during the execution of the receiver's block. The current implementation of the Linera client always selects as many messages as possible from inboxes, and never discards messages.

## [Chain Ownership Semantics](https://linera.dev/core_concepts/microchains.html#chain-ownership-semantics)

Only single-owner chains are currently supported in the Linera SDK. However, microchains can create new microchains for other users, and control of a chain can be transferred to another user by changing the owner id. A chain is permanently deactivated when its owner id is set to `None`.

For more detail and examples on how to open and close chains, see the wallet section on [chain management](https://linera.dev/core_concepts/wallets.html#opening-a-chain).

# [Wallets](https://linera.dev/core_concepts/wallets.html#wallets)

As in traditional blockchains, Linera wallets are in charge of holding user private keys. However, instead of signing transactions, Linera wallets are meant to sign blocks and propose them to extend the chains owned by their users.

In practice, wallets include a node which tracks a subset of Linera chains. We will see in the [next section](https://linera.dev/core_concepts/node_service.html) how a Linera wallet can run a GraphQL service to expose the state of its chains to web frontends.

> The command-line tool `linera` is the main way for developers to interact with a Linera network and manage the user wallets present locally on the system.

Note that this command-line tool is intended mainly for development purposes. Our goal is that end users eventually manage their wallets in a [browser extension](https://linera.dev/core_concepts/overview.html#web3-sdk).

## [Selecting a Wallet](https://linera.dev/core_concepts/wallets.html#selecting-a-wallet)

The private state of a wallet is conventionally stored in a file `wallet.json`, while the state of its node is stored in a file `linera.db`.

To switch between wallets, you may use the `--wallet` and `--storage` options of the `linera` tool, e.g. as in `linera --wallet wallet2.json --storage rocksdb:linera2.db`.

You may also define the environment variables `LINERA_STORAGE` and `LINERA_WALLET` to the same effect. E.g. `LINERA_STORAGE=$PWD/wallet2.json` and `LINERA_WALLET=$PWD/wallet2.json`.

Finally, if `LINERA_STORAGE_$I` and `LINERA_WALLET_$I` are defined for some number `I`, you may call `linera --with-wallet $I` (or `linera -w $I` for short).

## [Chain Management](https://linera.dev/core_concepts/wallets.html#chain-management)

### [Listing Chains](https://linera.dev/core_concepts/wallets.html#listing-chains)

To list the chains present in your wallet, you may use the command `show`:

```bash
linera wallet show
╭──────────────────────────────────────────────────────────────────┬──────────────────────────────────────────────────────────────────────────────────────╮
│ Chain Id                                                         ┆ Latest Block                                                                         │
╞══════════════════════════════════════════════════════════════════╪══════════════════════════════════════════════════════════════════════════════════════╡
│ 668774d6f49d0426f610ad0bfa22d2a06f5f5b7b5c045b84a26286ba6bce93b4 ┆ Public Key:         3812c2bf764e905a3b130a754e7709fe2fc725c0ee346cb15d6d261e4f30b8f1 │
│                                                                  ┆ Owner:              c9a538585667076981abfe99902bac9f4be93714854281b652d07bb6d444cb76 │
│                                                                  ┆ Block Hash:         -                                                                │
│                                                                  ┆ Timestamp:          2023-04-10 13:52:20.820840                                       │
│                                                                  ┆ Next Block Height:  0                                                                │
├╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┼╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌┤
│ 91c7b394ef500cd000e365807b770d5b76a6e8c9c2f2af8e58c205e521b5f646 ┆ Public Key:         29c19718a26cb0d5c1d28102a2836442f53e3184f33b619ff653447280ccba1a │
│                                                                  ┆ Owner:              efe0f66451f2f15c33a409dfecdf76941cf1e215c5482d632c84a2573a1474e8 │
│                                                                  ┆ Block Hash:         51605cad3f6a210183ac99f7f6ef507d0870d0c3a3858058034cfc0e3e541c13 │
│                                                                  ┆ Timestamp:          2023-04-10 13:52:21.885221                                       │
│                                                                  ┆ Next Block Height:  1                                                                │
╰──────────────────────────────────────────────────────────────────┴──────────────────────────────────────────────────────────────────────────────────────╯
```

Each row represents a chain present in the wallet. On the left is the unique identifier on the chain, and on the right is metadata for that chain associated with the latest block.

### [Default Chain](https://linera.dev/core_concepts/wallets.html#default-chain)

Each wallet has a default chain that all commands apply to unless you specify another `--chain` on the command line.

The default chain is set initially, when the first chain is added to the wallet. You can check the default chain for your wallet by running:

```bash
linera wallet show
```

The Chain Id which is in green text instead of white text is your default chain.

To change the default chain for your wallet, user the `set-default` command:

```bash
linera wallet set-default <chain-id>
```

### [Opening a Chain](https://linera.dev/core_concepts/wallets.html#opening-a-chain)

The Linera protocol defines semantics for how new chains are created, we call this "opening a chain". A chain cannot be opened in a vacuum, it needs to be created by an existing chain on the network.

#### [Open a Chain for Your Own Wallet](https://linera.dev/core_concepts/wallets.html#open-a-chain-for-your-own-wallet)

To open a chain for your own wallet, you can use the `open-chain` command:

```bash
linera open-chain
```

This will create a new chain (using the wallet's default chain) and add it to the wallet. Use the `wallet show` command to see your existing chains.

#### [Open a Chain for Another Wallet](https://linera.dev/core_concepts/wallets.html#open-a-chain-for-another-wallet)

Opening a chain for another `wallet` requires an extra two steps. Let's initialize a second wallet:

```bash
linera --wallet wallet2.json --storage rocksdb:linera2.db wallet init --genesis target/debug/genesis.json
```

First `wallet2` must create an unassigned keypair. The public part of that keypair is then sent to the `wallet` who is the chain creator.

```bash
linera --wallet wallet2.json keygen
6443634d872afbbfcc3059ac87992c4029fa88e8feb0fff0723ac6c914088888 # this is the public key for the unassigned keypair
```

Next, using the public key, `wallet` can open a chain for `wallet2`.

```bash
linera open-chain --to-public-key 6443634d872afbbfcc3059ac87992c4029fa88e8feb0fff0723ac6c914088888
e476187f6ddfeb9d588c7b45d3df334d5501d6499b3f9ad5595cae86cce16a65010000000000000000000000
fc9384defb0bcd8f6e206ffda32599e24ba715f45ec88d4ac81ec47eb84fa111
```

The first line is the message ID specifying the cross-chain message that creates the new chain. of the newly created chain. The second line is the new chain's ID.

Finally, to add the chain to `wallet2` for the given unassigned key we use the `assign` command:

```bash
 linera --wallet wallet2.json assign --key 6443634d872afbbfcc3059ac87992c4029fa88e8feb0fff0723ac6c914088888 --message-id e476187f6ddfeb9d588c7b45d3df334d5501d6499b3f9ad5595cae86cce16a65010000000000000000000000
```

## [Setting up Extra Wallets Automatically with `linera net up`](https://linera.dev/core_concepts/wallets.html#setting-up-extra-wallets-automatically-with-linera-net-up)

For testing, rather than using `linera open-chain` and `linera assign` as above, it is often more convenient to pass the option `--extra-wallets N` to `linera net up`.

This option will create create `N` additional user wallets and output Bash commands to define the environment variables `LINERA_{WALLET,STORAGE}_$I` where `I` ranges over `0..=N` (`I=0` being the wallet for the initial chains).

Once all the environment variables are defined, you may switch between wallets using `linera --with-wallet I` or `linera -w I` for short.

## [Automation in Bash](https://linera.dev/core_concepts/wallets.html#automation-in-bash)

To automate the process of setting the variables `LINERA_WALLET*` and `LINERA_STORAGE*` after creating a local test network in a shell, we provide a Bash helper function `linera_spawn_and_read_wallet_variables`.

To define the function `linera_spawn_and_read_wallet_variables` in your shell, run `source /dev/stdin <<<"$(linera net helper 2>/dev/null)"`. You may also add the output of `linera net helper` to your `~/.bash_profile` for future sessions.

Once the function is defined, call `linera_spawn_and_read_wallet_variables linera net up` instead of `linera net up`.

# [Node Service](https://linera.dev/core_concepts/node_service.html#node-service)

So far we've seen how to use the Linera client treating it as a binary in your terminal. However, the client also acts as a node which:

1. Executes blocks
2. Exposes an GraphQL API and IDE for dynamically interacting with applications and the system
3. Listens for notifications from validators and automatically updates local chains.

To interact with the node service, run `linera` in `service` mode:

```bash
linera service
```

This will run the node service on port 8080 by default (this can be overridden using the `--port` flag).

## [A Note on GraphQL](https://linera.dev/core_concepts/node_service.html#a-note-on-graphql)

Linera uses GraphQL as the query language for interfacing with different parts of the system. GraphQL enables clients to craft queries such that they receive exactly what they want and nothing more.

GraphQL is used extensively during application development, especially to query the state of an application from a front-end for example.

To learn more about GraphQL check out the [official docs](https://graphql.org/learn/).

## [GraphiQL IDE](https://linera.dev/core_concepts/node_service.html#graphiql-ide)

Conveniently, the node service exposes a GraphQL IDE called GraphiQL. To use GraphiQL start the node service and navigate to `localhost:8080/`.

Using the schema explorer on the left of the GraphiQL IDE you can dynamically explore the state of the system and your applications.

![graphiql.png](https://linera.dev/core_concepts/graphiql.png)

## [GraphQL System API](https://linera.dev/core_concepts/node_service.html#graphql-system-api)

The node service also exposes a GraphQL API which corresponds to the set of system operations. You can explore the full set of operations by clicking on `MutationRoot`.

## [GraphQL Application API](https://linera.dev/core_concepts/node_service.html#graphql-application-api)

To interact with an application, we run the Linera client in service mode. It exposes a GraphQL API for every application running on any owned chain at `localhost:8080/chains/<chain-id>/applications/<application-id>`.

Navigating there with your browser will open a GraphiQL interface which enables you to graphically explore the state of your application.

# [Applications](https://linera.dev/core_concepts/applications.html#applications)

The programming model of Linera is designed so that developers can take advantage of microchains to scale their applications.

Linera uses the WebAssembly Virtual Machine (Wasm) to execute user applications. Currently, the [Linera SDK](https://linera.dev/sdk.html) is focused on the [Rust](https://www.rust-lang.org/) programming language.

Linera applications are structured using the familiar notion of **Rust crate**: the external interfaces of an application (including initialization parameters, operations, messages, and cross-application calls) generally go into the library part of its crate, while the core of each application is compiled into binary files for the Wasm architecture.

## [The Application Deployment Lifecycle](https://linera.dev/core_concepts/applications.html#the-application-deployment-lifecycle)

Linera Applications are designed to be powerful yet re-usable. For this reason there is a distinction between the bytecode and an application instance on the network.

Applications undergo a lifecycle transition aimed at making development easy and flexible:

1. The bytecode is built from a Rust project with the `linera-sdk` dependency.
2. The bytecode is published to the network on a microchain, and assigned an identifier.
3. A user can create a new application instance, by providing the bytecode identifier and initialization arguments. This process returns an application identifier which can be used to reference and interact with the application.
4. The same bytecode identifier can be used as many times is needed by as many users are needed to create distinct applications.

Importantly, the application deployment lifecycle is abstracted from the user, and an application can be published with a single command:

```bash
linera publish-and-create <contract-path> <service-path> <init-args>
```

This will publish the bytecode as well as initialize the application for you.

## [Anatomy of an Application](https://linera.dev/core_concepts/applications.html#anatomy-of-an-application)

An **application** is broken into two major components, the *contract* and the *service*.

The **contract** is gas-metered, and is the part of the application which executes operations and messages, make cross-application calls and modifies the application's state. The details are covered in more depth in the [SDK docs](https://linera.dev/sdk.html).

The **service** is non-metered and read-only. It is used primarily to query the state of an application and populate the presentation layer (think front-end) with the data required for a user interface.

Finally, the application's state is shared by the contract and service in the form of a [View](https://linera.dev/advanced_topics/views.html), but more on that later.

## [Operations and Messages](https://linera.dev/core_concepts/applications.html#operations-and-messages)

> For this section we'll be using a simplified version of the example application called "fungible" where users can send tokens to each other.

At the system-level, interacting with an application can be done via operations and messages.

**Operations** are defined by an application developer and each application can have a completely different set of operations. Chain owners then actively create operations and put them in their block proposals to interact with an application.

Taking the "fungible token" application as an example, an operation for a user to transfer funds to another user would look like this:

```rust
#[derive(Debug, Deserialize, Serialize)]
pub enum Operation {
    /// A transfer from a (locally owned) account to a (possibly remote) account.
    Transfer {
        owner: AccountOwner,
        amount: Amount,
        target_account: Account,
    },
    // Meant to be extended here
}
```

**Messages** result from the execution of operations or other messages. Messages can be sent from one chain to another. Block proposers also actively include messages in their block proposal, but unlike with operations, they are only allowed to include them in the right order (possibly skipping some), and only if they were actually created by another chain (or the same chain, earlier).

In our "fungible token" application, a message to credit an account would look like this:

```rust
#[derive(Debug, Deserialize, Serialize)]
pub enum Message {
    Credit { owner: AccountOwner, amount: Amount },
    // Meant to be extended here
}
```

### [Authentication](https://linera.dev/core_concepts/applications.html#authentication)

Operations are always authenticated and messages may be authenticated. The signer of a block becomes the authenticator of all the operations in that block. As operations are executed by applications, messages can be created to be sent to other chains. When they are created, they can be configured to be authenticated. In that case, the message receives the same authentication as the operation that created it. If handling an incoming message creates new messages, those may also be configured to have the same authentication as the received message.

In other words, the block signer can have its authority propagated across chains through series of messages. This allows applications to safely store user state in chains that the user may not have the authority to produce blocks. The application may also allow only the authorized user to change that state, and not even the chain owner is able to override that.

The figure below shows four chains (A, B, C, D) and some blocks produced in them. In this example, each chain is owned by a single owner (aka. address). Owners are in charge of producing blocks and sign new blocks using their signing keys. Some blocks show the operations and incoming messages they accept, where the authentication is shown inside parenthesis. All operations produced are authenticated by the block proposer, and if these are all single user chains, the proposer is always the chain owner. Messages that have authentication use the one from the operation or message that created it.

One example in the figure is that chain A produced a block with Operation 1, which is authenticated by the owner of chain A (written `(a)`). That operations sent a message to chain B, and assuming the message was sent with the authentication forwarding enabled, it is received and executed in chain B with the authentication of `(a)`. Another example is that chain D produced a block with Operation 2, which is authenticated by the owner of chain D (written `(d)`). That operation sent a message to chain C, which is executed with authentication of `(d)` like the example before. Handling that message in chain C produced a new message, which was sent to chain B. That message, when received by chain B is executed with the authentication of `(d)`.

```ignore
                            ┌───┐     ┌─────────────────┐     ┌───┐
       Chain A owned by (a) │   ├────►│ Operation 1 (a) ├────►│   │
                            └───┘     └────────┬────────┘     └───┘
                                               │
                                               └────────────┐
                                                            ▼
                                                ┌──────────────────────────┐
                            ┌───┐     ┌───┐     │ Message from chain A (a) │
       Chain B owned by (b) │   ├────►│   ├────►│ Message from chain C (d) |
                            └───┘     └───┘     │ Operation 3 (b)          │
                                                └──────────────────────────┘
                                                            ▲
                                                   ┌────────┘
                                                   │
                            ┌───┐     ┌──────────────────────────┐     ┌───┐
       Chain C owned by (c) │   ├────►│ Message from chain D (d) ├────►│   │
                            └───┘     └──────────────────────────┘     └───┘
                                                 ▲
                                     ┌───────────┘
                                     │
                            ┌─────────────────┐     ┌───┐     ┌───┐
       Chain D owned by (d) │ Operation 2 (d) ├────►│   ├────►│   │
                            └─────────────────┘     └───┘     └───┘
```

An example where this is used is in the Fungible application, where a `Claim` operation allows retrieving money from a chain the user does not control (but the user still trusts will produce a block receiving their message). Without the `Claim` operation, users would only be able to store their tokens on their own chains, and multi-owner and public chains would have their tokens shared between anyone able to produce a block.

With the `Claim` operation, users can store their tokens on another chain where they're able to produce blocks or where they trust the owner will produce blocks receiving their messages. Only they are able to move their tokens, even on chains where ownership is shared or where they are not able to produce blocks.

## [Registering an Application across Chains](https://linera.dev/core_concepts/applications.html#registering-an-application-across-chains)

If Alice is using an application on her chain and starts interacting with Bob via the application, e.g. sends him some tokens using the `fungible` example, the application automatically gets registered on Bob's chain, too, as soon as he handles the incoming cross-chain messages. After that, he can execute the application's operations on his chain, too, and e.g. send tokens to someone.

But there are also cases where Bob may want to start using an application he doesn't have yet. E.g. maybe Alice regularly makes posts using the `social` example, and Bob wants to subscribe to her.

In that case, trying to execute an application-specific operation would fail, because the application is not registered on his chain. He needs to request it from Alice first:

```bash
linera request-application <application-id> --target-chain-id <alices-chain-id>
```

Once Alice processes his message (which happens automatically if she is running the client in service mode), he can start using the application.