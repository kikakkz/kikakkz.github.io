# [Writing Linera Applications](https://linera.dev/sdk.html#writing-linera-applications)

In this section, we'll be exploring how to create Web3 applications using the Linera SDK.

We'll use a simple "counter" application as a running example.

We'll focus on the back end of the application, which consists of two main parts: a *smart contract* and its GraphQL service.

Both the contract and the service of an application are written in Rust using the crate [`linera-sdk`](https://crates.io/crates/linera-sdk), and compiled to Wasm bytecode.

This section should be seen as a guide versus a reference manual for the SDK. For the reference manual, refer to the [documentation of the crate](https://docs.rs/linera-sdk/latest/linera_sdk/).

## [Creating a Linera Project](https://linera.dev/sdk/creating_a_project.html#creating-a-linera-project)

To create your Linera project, use the `linera project new` command. The command should be executed outside the `linera-protocol` folder. It sets up the scaffolding and requisite files:

```bash
linera project new my-counter
```

`linera project new` bootstraps your project by creating the following key files:

- `Cargo.toml`: your project's manifest filled with the necessary dependencies to create an app;
- `src/lib.rs`: the application's ABI definition;
- `src/state.rs`: the application's state;
- `src/contract.rs`: the application's contract, and the binary target for the contract bytecode;
- `src/service.rs`: the application's service, and the binary target for the service bytecode.
- `.cargo/config.toml`: modifies the default target used by `cargo` to be `wasm32-unknown-unknown`

## [Creating the Application State](https://linera.dev/sdk/state.html#creating-the-application-state)

The `struct` which defines your application's state can be found in `src/state.rs`.

To represent our counter, we're going to need a single `u64`. To persist the counter we'll be using Linera's [view](https://linera.dev/advanced_topics/views.html) paradigm.

Views are a little like an [ORM](https://en.wikipedia.org/wiki/Object–relational_mapping), however instead of mapping data structures to a relational database like Postgres, they are instead mapped onto key-value stores like [RocksDB](https://rocksdb.org/).

In vanilla Rust, we might represent our Counter as so:

```rust
// do not use this
struct Counter {
  value: u64
}
```

However, to persist your data, you'll need to replace the existing `Application` state struct in `src/state.rs` with the following view:

```rust
/// The application state.
#[derive(RootView, async_graphql::SimpleObject)]
#[view(context = "ViewStorageContext")]
pub struct Counter {
    pub value: RegisterView<u64>,
}
```

and all other occurrences of `Application` in your app.

The `RegisterView<T>` supports modifying a single value of type `T`. There are different types of views for different use-cases, but the majority of common data structures have already been implemented:

- A `Vec` or `VecDeque` corresponds to a `LogView`
- A `BTreeMap` corresponds to a `MapView` if its values are primitive, or to `CollectionView` if its values are other views;
- A `Queue` corresponds to a `QueueView`

For an exhaustive list refer to the Views [documentation](https://linera.dev/advanced_topics/views.html).

Finally, run `cargo check` to ensure that your changes compile.

## [Defining the ABI](https://linera.dev/sdk/abi.html#defining-the-abi)

The Application Binary Interface (ABI) of a Linera application defines how to interact with this application from other parts of the system. It includes the data structures, data types, and functions exposed by on-chain contracts and services.

ABIs are usually defined in `src/lib.rs` and compiled across all architectures (Wasm and native).

For a reference guide, check out the [documentation of the crate](https://docs.rs/linera-base/latest/linera_base/abi/).

### [Defining a marker struct](https://linera.dev/sdk/abi.html#defining-a-marker-struct)

The library part of your application (generally in `src/lib.rs`) must define a public empty struct that implements the `Abi` trait.

```rust
struct CounterAbi;
```

The `Abi` trait combines the `ContractAbi` and `ServiceAbi` traits to include the types that your application exports.

```rust
/// A trait that includes all the types exported by a Linera application (both contract
/// and service).
pub trait Abi: ContractAbi + ServiceAbi {}
```

Next, we're going to implement each of the two traits.

### [Contract ABI](https://linera.dev/sdk/abi.html#contract-abi)

The `ContractAbi` trait defines the data types that your application uses in a contract. Each type represents a specific part of the contract's behavior:

```rust
/// A trait that includes all the types exported by a Linera application contract.
pub trait ContractAbi {
    /// Immutable parameters specific to this application (e.g. the name of a token).
    type Parameters: Serialize + DeserializeOwned + Send + Sync + Debug + 'static;

    /// Initialization argument passed to a new application on the chain that created it
    /// (e.g. an initial amount of tokens minted).
    ///
    /// To share configuration data on every chain, use [`ContractAbi::Parameters`]
    /// instead.
    type InitializationArgument: Serialize + DeserializeOwned + Send + Sync + Debug + 'static;

    /// The type of operation executed by the application.
    ///
    /// Operations are transactions directly added to a block by the creator (and signer)
    /// of the block. Users typically use operations to start interacting with an
    /// application on their own chain.
    type Operation: Serialize + DeserializeOwned + Send + Sync + Debug + 'static;

    /// The type of message executed by the application.
    ///
    /// Messages are executed when a message created by the same application is received
    /// from another chain and accepted in a block.
    type Message: Serialize + DeserializeOwned + Send + Sync + Debug + 'static;

    /// The argument type when this application is called from another application on the same chain.
    type ApplicationCall: Serialize + DeserializeOwned + Send + Sync + Debug + 'static;

    /// The argument type when a session of this application is called from another
    /// application on the same chain.
    ///
    /// Sessions are temporary objects that may be spawned by an application call. Once
    /// created, they must be consumed before the current transaction ends.
    type SessionCall: Serialize + DeserializeOwned + Send + Sync + Debug + 'static;

    /// The type for the state of a session.
    type SessionState: Serialize + DeserializeOwned + Send + Sync + Debug + 'static;

    /// The response type of an application call.
    type Response: Serialize + DeserializeOwned + Send + Sync + Debug + 'static;
}
```

All these types must implement the `Serialize`, `DeserializeOwned`, `Send`, `Sync`, `Debug` traits, and have a `'static` lifetime.

In our example, we would like to change our `InitializationArgument`, `Operation` to `u64`, like so:

```rust
impl ContractAbi for CounterAbi {
    type InitializationArgument = u64;
    type Parameters = ();
    type Operation = u64;
    type ApplicationCall = ();
    type Message = ();
    type SessionCall = ();
    type Response = ();
    type SessionState = ();
}
```

### [Service ABI](https://linera.dev/sdk/abi.html#service-abi)

The `ServiceAbi` is in principle very similar to the `ContractAbi`, just for the service component of your application.

The `ServiceAbi` trait defines the types used by the service part of your application:

```rust
/// A trait that includes all the types exported by a Linera application service.
pub trait ServiceAbi {
    /// Immutable parameters specific to this application (e.g. the name of a token).
    type Parameters: Serialize + DeserializeOwned + Send + Sync + Debug + 'static;

    /// The type of a query receivable by the application's service.
    type Query: Serialize + DeserializeOwned + Send + Sync + Debug + 'static;

    /// The response type of the application's service.
    type QueryResponse: Serialize + DeserializeOwned + Send + Sync + Debug + 'static;
}
```

For our Counter example, we'll be using GraphQL to query our application so our `ServiceAbi` should reflect that:

```rust
impl ServiceAbi for CounterAbi {
    type Query = async_graphql::Request;
    type QueryResponse = async_graphql::Response;
    type Parameters = ();
}
```

## [Writing the Contract Binary](https://linera.dev/sdk/contract.html#writing-the-contract-binary)

The contract binary is the first component of a Linera application. It can actually change the state of the application.

To create a contract, we need to implement the `Contract` trait, which is as follows:

```rust
#[async_trait]
pub trait Contract: WithContractAbi + ContractAbi + Send + Sized {
    /// The type used to report errors to the execution environment.
    type Error: Error + From<serde_json::Error> + From<bcs::Error> + 'static;

    /// The desired storage backend used to store the application's state.
    type Storage: ContractStateStorage<Self> + Send + 'static;

    /// Initializes the application on the chain that created it.
    async fn initialize(
        &mut self,
        context: &OperationContext,
        argument: Self::InitializationArgument,
    ) -> Result<ExecutionOutcome<Self::Message>, Self::Error>;

    /// Applies an operation from the current block.
    async fn execute_operation(
        &mut self,
        context: &OperationContext,
        operation: Self::Operation,
    ) -> Result<ExecutionOutcome<Self::Message>, Self::Error>;

    /// Applies a message originating from a cross-chain message.
    async fn execute_message(
        &mut self,
        context: &MessageContext,
        message: Self::Message,
    ) -> Result<ExecutionOutcome<Self::Message>, Self::Error>;

    /// Handles a call from another application.
    async fn handle_application_call(
        &mut self,
        context: &CalleeContext,
        argument: Self::ApplicationCall,
        forwarded_sessions: Vec<SessionId>,
    ) -> Result<ApplicationCallResult<Self::Message, Self::Response, Self::SessionState>, Self::Error>;

    /// Handles a call into a session created by this application.
    async fn handle_session_call(
        &mut self,
        context: &CalleeContext,
        session: Self::SessionState,
        argument: Self::SessionCall,
        forwarded_sessions: Vec<SessionId>,
    ) -> Result<SessionCallResult<Self::Message, Self::Response, Self::SessionState>, Self::Error>;

}
```

The full trait definition can be found [here](https://github.com/linera-io/linera-protocol/blob/main/linera-sdk/src/lib.rs).

There's quite a bit going on here, so let's break it down and take one method at a time.

For this application, we'll be using the `initialize` and `execute_operation` methods.

### [Initializing our Application](https://linera.dev/sdk/contract.html#initializing-our-application)

The first thing we need to do is initialize our application by using `Contract::initialize`.

`Contract::initialize` is only called once when the application is created and only on the microchain that created the application.

Deployment on other microchains will use the `Default` implementation of the application state if `SimpleStateStorage` is used, or the `Default` value of all sub-views in the state if the `ViewStateStorage` is used.

For our example application, we'll want to initialize the state of the application to an arbitrary number that can be specified on application creation using its initialization parameters:

```rust
    async fn initialize(
        &mut self,
        _context: &OperationContext,
        value: u64,
    ) -> Result<ExecutionOutcome<Self::Message>, Self::Error> {
        self.value.set(value);
        Ok(ExecutionOutcome::default())
    }
```

### [Implementing the Increment Operation](https://linera.dev/sdk/contract.html#implementing-the-increment-operation)

Now that we have our counter's state and a way to initialize it to any value we would like, a way to increment our counter's value. Changes made by block proposers to application states are broadly called 'operations'.

To create a new operation, we need to use the method `Contract::execute_operation`. In the counter's case, it will be receiving a `u64` which is used to increment the counter:

```rust
    async fn execute_operation(
        &mut self,
        _context: &OperationContext,
        operation: u64,
    ) -> Result<ExecutionOutcome<Self::Message>, Self::Error> {
        let current = self.value.get();
        self.value.set(current + operation);
        Ok(ExecutionOutcome::default())
    }
```

### [Declaring the ABI](https://linera.dev/sdk/contract.html#declaring-the-abi)

Finally, to link our `Contract` trait implementation with the ABI of the application, the following code is added:

```rust
impl WithContractAbi for Counter {
    type Abi = counter::CounterAbi;
}
```

### [Writing the Service Binary](https://linera.dev/sdk/service.html#writing-the-service-binary)

The service binary is the second component of a Linera application. It is compiled into a separate Bytecode from the contract and is run independently. It is not metered (meaning that querying an application's service does not consume gas), and can be thought of as a read-only view into your application.

Application states can be arbitrarily complex, and most of the time you don't want to expose this state in its entirety to those who would like to interact with your app. Instead, you might prefer to define a distinct set of queries that can be made against your application.

The `Service` trait is how you define the interface into your application. The `Service` trait is defined as follows:

```rust
/// The service interface of a Linera application.
#[async_trait]
pub trait Service: WithServiceAbi + ServiceAbi {
    /// Type used to report errors to the execution environment.
    type Error: Error + From<serde_json::Error>;

    /// The desired storage backend used to store the application's state.
    type Storage: ServiceStateStorage;

    /// Executes a read-only query on the state of this application.
    async fn handle_query(
        self: Arc<Self>,
        context: &QueryContext,
        argument: Self::Query,
    ) -> Result<Self::QueryResponse, Self::Error>;
}
```

The full service trait definition can be found [here](https://github.com/linera-io/linera-protocol/blob/main/linera-sdk/src/lib.rs).

Let's implement `Service` for our counter application.

First, we want to generate the necessary boilerplate for implementing the service [WIT interface](https://component-model.bytecodealliance.org/design/wit.html), export the necessary resource types and functions so that the host (the process running the bytecode) can call the service. Happily, there is a macro to perform this code generation, so just add the following to `service.rs`:

```rust
linera_sdk::service!(Counter);
```

Next, we need to implement the `Service` for `Counter`. To do this we need to define `Service`'s associated types and implement `handle_query`, as well as define the `Error` type:

```rust
#[async_trait]
impl Service for Counter {
    type Error = Error;
    type Storage = ViewStateStorage<Self>;

    async fn handle_query(
        self: Arc<Self>,
        _context: &QueryContext,
        request: Request,
    ) -> Result<Response, Self::Error> {
        let schema = Schema::build(
            // implemented in the next section
            QueryRoot { value: *self.value.get() },
            // implemented in the next section
            MutationRoot {},
            EmptySubscription,
        )
        .finish();
        Ok(schema.execute(request).await)
    }
}

/// An error that can occur during the contract execution.
#[derive(Debug, Error)]
pub enum Error {
    /// Invalid query argument; could not deserialize GraphQL request.
    #[error("Invalid query argument; could not deserialize GraphQL request")]
    InvalidQuery(#[from] serde_json::Error),
}
```

Finally, as before, the following code is needed to incorporate the ABI definitions into your `Service` implementation:

```rust
impl WithServiceAbi for Counter {
    type Abi = counter::CounterAbi;
}
```

### [Adding GraphQL compatibility](https://linera.dev/sdk/service.html#adding-graphql-compatibility)

Finally, we want our application to have GraphQL compatibility. To achieve this we need a `QueryRoot` for intercepting queries and a `MutationRoot` for introspection queries for mutations.

```rust
struct MutationRoot;

#[Object]
impl MutationRoot {
    async fn increment(&self, value: u64) -> Vec<u8> {
        bcs::to_bytes(&value).unwrap()
    }
}

struct QueryRoot {
    value: u64,
}

#[Object]
impl QueryRoot {
    async fn value(&self) -> &u64 {
        &self.value
    }
}
```

We haven't included the imports in the above code; they are left as an exercise to the reader (but remember to import `async_graphql::Object`). If you want the full source code and associated tests check out the [examples section](https://github.com/linera-io/linera-protocol/blob/main/examples/counter/src/service.rs) on GitHub.

## [Deploying the Application](https://linera.dev/sdk/deploy.html#deploying-the-application)

The first step to deploy your application is to configure a wallet. This will determine where the application will be deployed: either to a local net or to the devnet.

### [Local Net](https://linera.dev/sdk/deploy.html#local-net)

To configure the local network, follow the steps in the [Getting Started section](https://linera.dev/getting_started/hello_linera.html#using-the-initial-test-wallet).

Afterwards, the `LINERA_WALLET` and the `LINERA_STORAGE` environment variables should be set and can be used in the `publish-and-create` command to deploy the application while also specifying:

1. The location of the contract bytecode
2. The location of the service bytecode
3. The JSON encoded initialization arguments

```bash
linera publish-and-create \
  target/wasm32-unknown-unknown/release/my-counter_{contract,service}.wasm \
  --json-argument "42"
```

### [Devnet](https://linera.dev/sdk/deploy.html#devnet)

To configure the wallet for the devnet while creating a new microchain, the following command can be used:

```bash
linera wallet init --with-new-chain --faucet https://faucet.devnet.linera.net
```

The Faucet will provide the new chain with some tokens, which can then be used to deploy the application with the `publish-and-create` command. It requires specifying:

1. The location of the contract bytecode
2. The location of the service bytecode
3. The JSON encoded initialization arguments

```bash
linera publish-and-create \
  target/wasm32-unknown-unknown/release/my-counter_{contract,service}.wasm \
  --json-argument "42"
```

### [Interacting with the Application](https://linera.dev/sdk/deploy.html#interacting-with-the-application)

To interact with the deployed application, a [node service](https://linera.dev/core_concepts/node_service.html) must be used.

## [Cross-Chain Messages](https://linera.dev/sdk/messages.html#cross-chain-messages)

On Linera, applications are meant to be multi-chain: They are instantiated on every chain where they are used. An application has the same application ID and bytecode everywhere, but a separate state on every chain. To coordinate, the instances can send *cross-chain messages* to each other. A message sent by an application is always handled by the *same* application on the target chain: The handling code is guaranteed to be the same as the sending code, but the state may be different.

For your application, you can specify any serializable type as the `Message` type in your `ContractAbi` implementation. To send a message, return it among the [`ExecutionOutcome`](https://docs.rs/linera-sdk/latest/linera_sdk/struct.ExecutionOutcome.html)'s `messages`:

```rust
    pub messages: Vec<OutgoingMessage<Message>>,
```

`OutgoingMessage`'s `destination` field specifies either a single destination chain, or a channel, so that it gets sent to all subscribers.

If the `authenticated` field is `true`, the callee is allowed to perform actions that require authentication on behalf of the signer of the original block that caused this call.

The `message` field contains the message itself, of the type you specified in the `ContractAbi`.

You can also use [`ExecutionOutcome::with_message`](https://docs.rs/linera-sdk/latest/linera_sdk/struct.ExecutionOutcome.html#method.with_message) and [`with_authenticated_message`](https://docs.rs/linera-sdk/latest/linera_sdk/struct.ExecutionOutcome.html#method.with_authenticated_message) for convenience.

During block execution in the *sending* chain, messages are returned via `ExecutionOutcome`s. The returned message is then placed in the *target* chain inbox for processing. There is no guarantee that it will be handled: For this to happen, an owner of the target chain needs to include it in the `incoming_messages` in one of their blocks. When that happens, the contract's `execute_message` method gets called on their chain.

### [Example: Fungible Token](https://linera.dev/sdk/messages.html#example-fungible-token)

In the [`fungible` example application](https://github.com/linera-io/linera-protocol/tree/main/examples/fungible), such a message can be the transfer of tokens from one chain to another. If the sender includes a `Transfer` operation on their chain, it decreases their account balance and sends a `Credit` message to the recipient's chain:

```rust
async fn execute_operation(
    &mut self,
    context: &OperationContext,
    operation: Self::Operation,
) -> Result<ExecutionOutcome<Self::Message>, Self::Error> {
    match operation {
        Operation::Transfer {
            owner,
            amount,
            target_account,
        } => {
            // ...
            self.debit(owner, amount).await?;
            let message = Message::Credit {
                owner: target_account.owner,
                amount,
            };
            Ok(ExecutionOutcome::default().with_message(target_account.chain_id, message))
        }
        // ...
    }
}
```

On the recipient's chain, `execute_message` is called, which increases their account balance.

```rust
async fn execute_message(
    &mut self,
    context: &MessageContext,
    message: Message,
) -> Result<ExecutionOutcome<Self::Message>, Self::Error> {
    match message {
        Message::Credit { owner, amount } => {
            self.credit(owner, amount).await;
            Ok(ExecutionOutcome::default())
        }
        // ...
    }
}
```

## [Calling other Applications](https://linera.dev/sdk/composition.html#calling-other-applications)

We have seen that cross-chain messages sent by an application on one chain are always handled by the *same* application on the target chain.

This section is about calling other applications using *cross-application calls*.

Such calls happen on the same chain and typically use the `call_application` method implemented by default in the trait `Contract`:

```rust
async fn call_application<A: ContractAbi + Send>(
    &mut self,
    authenticated: bool,
    application: ApplicationId<A>,
    call: &A::ApplicationCall,
    forwarded_sessions: Vec<SessionId>,
) -> Result<(A::Response, Vec<SessionId>), Self::Error> { .. }
```

The `authenticated` argument specifies whether the callee is allowed to perform actions that require authentication on behalf of the signer of the original block that caused this call.

The `application` argument is the callee's application ID, and `A` is the callee's ABI.

`call` are the arguments of the application call, in a type defined by the callee.

`forwarded_sessions` are session data that need to be consumed within this transaction. Sessions will be explained in a separate section.

### [Example: Crowd-Funding](https://linera.dev/sdk/composition.html#example-crowd-funding)

The `crowd-funding` example application allows the application creator to launch a campaign with a funding target. That target can be an amount specified in any type of token based on the `fungible` application. Others can then pledge tokens of that type to the campaign, and if the target is not reached by the deadline, they are refunded.

If Alice used the `fungible` example to create a Pugecoin application (with an impressionable pug as its mascot), then Bob can create a `crowd-funding` application, use Pugecoin's application ID as `CrowdFundingAbi::Parameters`, and specify in `CrowdFundingAbi::InitializationArgument` that his campaign will run for one week and has a target of 1000 Pugecoins.

Now let's say Carol wants to pledge 10 Pugecoin tokens to Bob's campaign.

First she needs to make sure she has his crowd-funding application on her chain, e.g. using the `linera request-application` command. This will automatically also register Alice's application on her chain, because it is a dependency of Bob's.

Now she can make her pledge by running the `linera service` and making a query to Bob's application:

```json
mutation { pledgeWithTransfer(owner: "User:841…6c0", amount: "10") }
```

This will add a block to Carol's chain containing the pledge operation that gets handled by `CrowdFunding::execute_operation`, resulting in one cross-application call and two cross-chain messages:

First `CrowdFunding::execute_operation` calls the `fungible` application on Carol's chain to transfer 10 tokens to Carol's account on Bob's chain:

```rust
self.call_application(
    true,                 // The call is authenticated by Carol, who signed this block.
    Self::fungible_id()?, // The Pugecoin application ID.
    &fungible::ApplicationCall::Transfer {
        owner,            // Carol
        amount,           // 10 tokens
        destination,      // Bob's chain.
    },
    vec![],
).await?;
```

This causes `Fungible::handle_application_call` to be run, which will create a cross-chain message sending the amount 10 to the Pugecoin application instance on Bob's chain.

After the cross-application call returns, `CrowdFunding::execute_operation` continues to create another cross-chain message `crowd_funding::Message::PledgeWithAccount`, which informs the crowd-funding application on Bob's chain that the 10 tokens are meant for the campaign.

When Bob now adds a block to his chain that handles the two incoming messages, first `Fungible::execute_message` gets executed, and then `CrowdFunding::execute_message`. The latter makes another cross-application call to transfer the 10 tokens from Carol's account to the crowd-funding application's account (both on Bob's chain). That is successful because Carol does now have 10 tokens on this chain and she authenticated the transfer indirectly by signing her block. The crowd-funding application now makes a note in its application state on Bob's chain that Carol has pledged 10 Pugecoin tokens.

For the complete code please take a look at the `crowd-funding` and `fungible` applications in the `examples` folder in `linera-protocol`.

## [Printing Logs from an Application](https://linera.dev/sdk/logging.html#printing-logs-from-an-application)

Applications can use the [`log` crate](https://crates.io/crates/log) to print log messages with different levels of importance. Log messages are useful during development, but they may also be useful for end users. By default the `linera service` command will log the messages from an application if they are of the "info" importance level or higher (briefly, `log::info!`, `log::warn!` and `log::error!`).

During development it is often useful to log messages of lower importance (such as `log::debug!` and `log::trace!`). To enable them, the `RUST_LOG` environment variable must be set before running `linera service`. The example below enables trace level messages from applications and enables warning level messages from other parts of the `linera` binary:

```ignore
export RUST_LOG="warn,linera_execution::wasm=trace"
```

## [Writing Tests](https://linera.dev/sdk/testing.html#writing-tests)

Linera applications can be tested using unit tests or integration tests. Both are a bit different than usual Rust tests. Unit tests are executed inside a WebAssembly virtual machine in an environment that simulates a single microchain and a single application. System APIs are only available if they are mocked using helper functions from `linera_sdk::test`.

Integration tests run outside a WebAssembly virtual machine, and use a simulated validator for testing. This allows creating chains and adding blocks to them in order to test interactions between multiple microchains and multiple applications.

Applications should consider having both types of tests. Unit tests should be used to focus on the application's internals and core functionality. Integration tests should be used to test how the application behaves on a more complex environment that's closer to the real network.

> In most cases, the simplest way to run both unit tests and integration tests is to call `linera project test` from the project's directory.

### [Unit tests](https://linera.dev/sdk/testing.html#unit-tests)

Unit tests are written beside the application's source code (i.e., inside the `src` directory of the project). There are several differences to normal Rust unit tests:

- the target `wasm32-unknown-unknown` must be selected;
- the custom test runner `linera-wasm-test-runner` must be used;
- the [`#[webassembly_test\]`](https://docs.rs/webassembly-test/latest/webassembly_test/) attribute must be used instead of the usual `#[test]` attribute.

The first two items are done automatically by `linera project test`.

Alternatively, one may set up the environment and run `cargo test` directly as described [below](https://linera.dev/sdk/testing.html#manually-configuring-the-environment).

#### [Example](https://linera.dev/sdk/testing.html#example)

A simple unit test is shown below, which tests if the application's `do_something` method changes the application state.

```rust
#[cfg(test)]
mod tests {
    use crate::state::ApplicationState;
    use webassembly_test::webassembly_test;

    #[webassembly_test]
    fn test_do_something() {
        let mut application = ApplicationState {
            // Configure the application's initial state
            ..ApplicationState::default()
        };

        let result = application.do_something();

        assert!(result.is_ok());
        assert_eq!(application, ApplicationState {
            // Define the application's expected final state
            ..ApplicationState::default()
        });
    }
}
```

#### [Mocking System APIs](https://linera.dev/sdk/testing.html#mocking-system-apis)

Unit tests run in a constrained environment, so things like access to the key-value store, cross-chain messages and cross-application calls can't be executed. However, they can be simulated using mock APIs. The `linera-sdk::test` module provides some helper functions to mock the system APIs.

Here's an example mocking the key-value store.

```rust
#[cfg(test)]
mod tests {
    use crate::state::ApplicationState;
    use linera_sdk::test::mock_key_value_store;
    use webassembly_test::webassembly_test;

    #[webassembly_test]
    fn test_state_is_not_persisted() {
        let mut storage = mock_key_value_store();

        // Assuming the application uses views
        let mut application = ApplicationState::load(storage.clone())
            .now_or_never()
            .expect("Mock key-value store returns immediately")
            .expect("Failed to load view from mock key-value store");

        // Assuming `do_something` changes the view, but does not persist it
        let result = application.do_something();

        assert!(result.is_ok());

        // Check that the state in memory is different from the state in storage
        assert_ne!(application, ApplicatonState::load(storage));
    }
}
```

#### [Running Unit Tests with `cargo test`](https://linera.dev/sdk/testing.html#running-unit-tests-with-cargo-test)

Running `linera project test` is easier, but if there's a need to run `cargo test` explicitly to run the unit tests, Cargo must be configured to use the custom test runner `linera-wasm-test-runner`. This binary can be built from the repository or installed with `cargo install linera-sdk`.

```bash
cd linera-protocol
cargo build -p linera-sdk --bin linera-wasm-test-runner --release
```

The steps above build the `linera-wasm-test-runner` and places the resulting binary at `linera-protocol/target/release/linera-wasm-test-runner`.

With the binary available, the last step is to configure Cargo. There are a few ways to do this. A quick way is to set the `CARGO_TARGET_WASM32_UNKNOWN_UNKNOWN_RUNNER` environment variable to the path of the binary.

A more persistent way is to change one of [Cargo's configuration files](https://doc.rust-lang.org/cargo/reference/config.html#hierarchical-structure). As an example, the following file can be placed inside the project's directory at `PROJECT_DIR/.cargo/config.toml`:

```ignore
[target.wasm32-unknown-unknown]
runner = "PATH_TO/linera-wasm-test-runner"
```

After configuring the test runner, unit tests can be executed with

```bash
cargo test --target wasm32-unknown-unknown
```

Optionally, `wasm32-unknown-unknown` can be made the default build target with the following lines in `PROJECT_DIR/.cargo/config.toml`:

```ignore
[build]
target = "wasm32-unknown-unknown"
```

### [Integration Tests](https://linera.dev/sdk/testing.html#integration-tests)

Integration tests are usually written separately from the application's source code (i.e., inside a `tests` directory that's beside the `src` directory).

Integration tests are normal Rust integration tests, and they are compiled to the **host** target instead of the `wasm32-unknown-unknown` target used for unit tests. This is because unit tests run inside a WebAssembly virtual machine and integration tests run outside a virtual machine, starting isolated virtual machines to run each operation of each block added to each chain.

> Integration tests can be run with `linera project test` or simply `cargo test`.

If you wish to use `cargo test` and have overridden your default target to be in `wasm32-unknown-unknown` in `.cargo/config.toml`, you will have to pass a native target to `cargo`, for instance `cargo test --target aarch64-apple-darwin`.

Integration tests use the helper types from `linera_sdk::test` to set up a simulated Linera network, and publish blocks to microchains in order to execute the application.

#### [Example](https://linera.dev/sdk/testing.html#example-1)

A simple test that sends a message between application instances on different chains is shown below.

```rust
#[tokio::test]
async fn test_cross_chain_message() {
    let (validator, application_id) = TestValidator::with_current_application(vec![], vec![]).await;

    let mut sender_chain = validator.get_chain(application_id.creation.chain_id).await;
    let mut receiver_chain = validator.new_chain().await;

    sender_chain
        .add_block(|block| {
            block.with_operation(
                application_id,
                Operation::SendMessageTo(receiver_chain.id()),
            )
        })
        .await;

    receiver_chain.handle_received_messages().await;

    assert_eq!(
        receiver_chain
            .query::<ChainId>(application_id, Query::LastSender)
            .await,
        sender_chain.id(),
    );
}
```