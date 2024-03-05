# [Advanced Topics](https://linera.dev/advanced_topics.html#advanced-topics)

In this section, we present additional topics related to the Linera protocol.

## [Views](https://linera.dev/advanced_topics/views.html#views)

> Views are a specific functionality of the Linera system that allow to have data in memory and then seamlessly flush it to an underlying persistent datastore.

The [full documentation](https://docs.rs/linera-views/latest/linera_views/) is available on the crate documentation with all functions having examples.

Concretely, what is provided is the following:

- A trait `View` that provides `load`, `rollback`, `clear`, `flush`, `delete`. The idea is that we can do operation on the data and then flush it to the database storing them.
- Several other traits `HashableView`, `RootView`, `CryptoHashView`, `CryptoHashRootView` that are important for computing hash.
- A number of standard containers: `MapView`, `SetView`, `LogView`, `QueueView`, `RegisterView` that implement the `View` and `HashableView` traits.
- Two containers `CollectionView` and `ReentrantCollectionView` that are similar to `MapView` but whose values are views themselves.
- Derive macros that allow to implement the above mentioned traits on struct data types whose entries are views.

## [Persistent storage](https://linera.dev/advanced_topics/persistent_storage.html#persistent-storage)

Validators run the servers and the data is stored in persistent storage. As a consequence we need a tool for working with persistent storage and so we have added `linera-db` for that purpose.

### [Available persistent storage](https://linera.dev/advanced_topics/persistent_storage.html#available-persistent-storage)

The persistent storage that are available right now are `RocksDB`, `DynamoDB` and `ScyllaDB`. Each has its own strengths and weaknesses.

- [`RocksDB`](https://rocksdb.org/): Data is stored on disk and cannot be shared between shards but is very fast.
- [`DynamoDB`](https://aws.amazon.com/dynamodb/): Data is stored on a remote storage, that has to be on AWS. Data can be shared between shards.
- [`ScyllaDB`](https://www.scylladb.com/): Data is stored on a remote storage. Data can be shared between shards.

There is no fundamental obstacle to the addition of other persistent storage solutions.

In addition, the `DynamoDB` and `ScyllaDB` have the notion of a table which means that a given remote location can be used for several completely independent purposes.

### [The `linera-db` tool](https://linera.dev/advanced_topics/persistent_storage.html#the-linera-db-tool)

When operating on a persistent storage some global operations can be required. The command line tool `linera-db` helps in making them work.

The functionalities are the following:

- `list_tables`(`DynamoDB` and `ScyllaDB`): It lists all the tables that have been created on the persistent storage
- `initialize`(`RocksDB`, `DynamoDB` and `ScyllaDB`): It initializes a persistent storage.
- `check_existence`(`RocksDB`, `DynamoDB` and `ScyllaDB`): It tests the existence of a persistent storage. If the error code is 0 then the table exists, if the error code is 1 then the table is absent.
- `check_absence`(`RocksDB`, `DynamoDB` and `ScyllaDB`): It tests the absence of a persistent storage. If the error code is 0 then the table is absent, if the error code is 1 then the table does not exist.
- `delete_all`(`RocksDB`, `DynamoDB` and `ScyllaDB`): It deletes all the table of a persistent storage.
- `delete_single`(`DynamoDB` and `ScyllaDB`): It deletes a single table of a persistent storage.

If some error occurs during the operation, then the error code 2 is returned and 0 if everything went fine with the exception of `check_existence` and `check_absence` for which the value 1 can occur if the connection with the database was established correctly but the result is not what we expected.

## [Validators](https://linera.dev/advanced_topics/validators.html#validators)

Validators run the servers that allow users to download and create blocks. They validate, execute and cryptographically certify the blocks of all the chains.

> In Linera, every chain is backed by the same set of validators and has the same level of security.

The main function of validators is to guarantee the integrity of the infrastructure in the sense that:

- Each block is valid, i.e. it has the correct format, its operations are allowed, the received messages are in the correct order, and e.g. the balance was correctly computed.
- Every message received by one chain was actually sent by another chain.
- If one block on a particular height is certified, no other block on the same height is.

These properties are guaranteed to hold as long as two third of the validators (weighted by their stake) follow the protocol. In the future, deviating from the protocol may cause a validator to be considered malicious and to lose their *stake*.

Validators also play a role in the liveness of the system by making sure that the history of the chains stays available. However, since validators do not propose blocks on most chains (see [next section](https://linera.dev/advanced_topics/block_creation.html)), they do *not* guarantee that any particular operation or message will eventually be executed on a chain. Instead, chain owners decide whether and when to propose new blocks, and which operations and messages to include. The current implementation of the Linera client automatically includes all incoming messages in new blocks. The operations are the actions the chain owner explicitly adds, e.g. transfer.

### [Architecture of a validator](https://linera.dev/advanced_topics/validators.html#architecture-of-a-validator)

Since every chain uses the same validators, adding more chains does not require adding validators. Instead, it requires each individual validator to scale out by adding more computation units, also known as "workers" or "physical shards".

In the end, a Linera validator resembles a Web2 service made of

- a load balancer (aka. ingress/egress), currently implemented by the binary `linera-proxy`,
- a number of workers, currently implemented by the binary `linera-server`,
- a shared database, currently implemented by the abstract interface `linera-storage`.

```ignore
Example of Linera network

                    │                                             │
                    │                                             │
┌───────────────────┼───────────────────┐     ┌───────────────────┼───────────────────┐
│ validator 1       │                   │     │ validator N       │                   │
│             ┌─────┴─────┐             │     │             ┌─────┴─────┐             │
│             │   load    │             │     │             │   load    │             │
│       ┌─────┤  balancer ├────┐        │     │       ┌─────┤  balancer ├──────┐      │
│       │     └───────────┘    │        │     │       │     └─────┬─────┘      │      │
│       │                      │        │     │       │           │            │      │
│       │                      │        │     │       │           │            │      │
│  ┌────┴─────┐           ┌────┴─────┐  │     │  ┌────┴───┐  ┌────┴────┐  ┌────┴───┐  │
│  │  worker  ├───────────┤  worker  │  │ ... │  │ worker ├──┤  worker ├──┤ worker │  │
│  │    1     │           │    2     │  │     │  │    1   │  │    2    │  │    3   │  │
│  └────┬─────┘           └────┬─────┘  │     │  └────┬───┘  └────┬────┘  └────┬───┘  │
│       │                      │        │     │       │           │            │      │
│       │                      │        │     │       │           │            │      │
│       │     ┌───────────┐    │        │     │       │     ┌─────┴─────┐      │      │
│       └─────┤  shared   ├────┘        │     │       └─────┤  shared   ├──────┘      │
│             │ database  │             │     │             │ database  │             │
│             └───────────┘             │     │             └───────────┘             │
└───────────────────────────────────────┘     └───────────────────────────────────────┘
```

Inside a validator, components communicate using the internal network of the validator. Notably, workers use direct Remote Procedure Calls (RPCs) with each other to deliver cross-chain messages.

Note that the number of workers may vary for each validator. Both the load balancer and the shared database are represented as a single entity but are meant to scale out in production.

> For local testing during development, we currently use a single worker and RocksDB as a database.

## [Creating New Blocks](https://linera.dev/advanced_topics/block_creation.html#creating-new-blocks)

> In Linera, the responsibility of proposing blocks is separate from the task of validating blocks.

While all chains are validated in the same way, the Linera protocol defines several types of chains, depending on how new blocks are produced.

- The simplest and lowest-latency type of chain is called *single-owner* chain.
- Other types of Linera chains not currently supported in the SDK include *permissioned chains* and *public chains* (see the [whitepaper](https://linera.io/whitepaper) for more context).

> For most types of chains (all but *public chains*), Linera validators do not need to exchange messages with each other.

Instead, the wallets (aka. `linera` clients) of chain owners make the system progress by proposing blocks and actively providing any additional required data to the validators. For instance, client commands such as `transfer`, `publish-bytecode`, or `open-chain` perform multiple steps to append a block containing the token transfer, application publishing, or chain creation operation:

- The Linera client creates a new block containing the desired operation and new incoming messages, if there are any. It also contains the most recent block's hash to designate its parent. The client sends the new block to all validators.
- The validators validate the block, i.e. check that the block satisfies the conditions listed above, and send a cryptographic signature to the client, indicating that they vote to append the new block. But only if they have not voted for a different block on the same height earlier!
- The client ideally receives a vote from every validator, but only a quorum of votes (say, two thirds) are required: These constitute a "certificate", proving that the block was confirmed. The client sends the certificate to every validator.
- The validators "execute" the block: They update their own view of the most recent state of the chain by applying all messages and operations, and if it generated any cross-chain messages, they send these to the appropriate workers.

To guarantee that each incoming message in a block was actually sent by another chain, a validator will, in the second step, only *vote* for a block if it has already executed the block that sent it. However, when receiving a valid certificate for a block that receives a message it has not seen yet, it will accept and *execute* the block anyway. The certificate is proof that most other validators have seen the message, so it must be correct.

In the case of single-owner chains, clients must be carefully implemented so that they never propose multiple blocks at the same height. Otherwise, the chain may be stuck: once each of the two conflicting blocks has been signed by enough validators, it becomes impossible to collect a quorum of votes for either block.

In the future, we anticipate that most users will use *permissioned chains* even if they are the only owners of their chains. Permissioned chains have two confirmation steps instead of one, but it is not possible to accidentally make a chain unextendable. They also allow users to delegate certain admistrative tasks to third-parties, notably to help with epoch changes (i.e. when the validators change if reconfigured).