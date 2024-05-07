# Staking Indexer

The staking indexer is a tool that extracts BTC staking relevant data from the
Bitcoin blockchain, ensures that it follows the pre-requisites for a valid
staking transaction, and determines whether the transaction should be active or
not. All valid staking transactions are transformed into a structured form,
stored in a database, and published as events in a RabbitMQ messaging queue for
consumption by consumers. The staking indexer is the enforcer of the Bitcoin
Staking protocol and serves as the ground truth for the Bitcoin Staking system.

## Features

* Polling BTC blocks data from a specified height in an ongoing manner. The 
  poller ensures that all the output blocks have at least `N` confirmations 
  where `N` is a configurable value, which should be large enough so that 
  the chance of the output blocks being forked is enormously low, e.g., 
  greater than or equal to `6` in Bitcoin mainnet.
* Extraction transaction data for staking, unbonding, and withdrawal. These
  transactions are verified and compared against the current system parameters
  to identify whether they are active, inactive due to staking cap overflow, or
  invalid. The details of the protocol for verifying and activating
  transactions can be found [here](./doc/staking.md).
* Storing the extracted transaction data in a database. The database schema 
  can be found [here](./doc/db_schema.md).
* Pushing staking, unbonding, and withdrawal events to the message queues.
  A reference implementation based on [rabbitmq](https://www.rabbitmq.com/) is 
  provided.
  The definition of each type of events can be found [here](./doc/events.md).
* Monitoring the status of the service through [Prometheus metrics](./doc/metrics.md).

## Usage

### Setup bitcoind node

The staking indexer relies on `bitcoind` as backend. Follow this [guide](./doc/bitcoind_setup.md)
to set up a `bitcoind` node.

### Install

Clone the repository to your local machine from Github:

```bash
git clone https://github.com/babylonchain/staking-indexer.git
```

Install the `sid` daemon binary by running:

```bash
cd staking-indexer # cd into the project directory
make install
```

### Configuration

To initiate the program with default config file, run:

```bash
sid init
```

This will create a `sid.conf` file in the default home directory. The 
default home directories for different operating systems are:

- **MacOS** `~/Users/<username>/Library/Application Support/Sid`
- **Linux** `~/.Sid`
- **Windows** `C:\Users\<username>\AppData\Local\Sid`

Use the `--home` flag to specify the home directory and use the `--force` to 
overwrite the existing config file.

### Run the Staking Indexer

To start the staking indexer from a specific height, run:

```bash
sid start --start-height <start-height>
```

If the `--start-height` is not specified, the indexer will retrieve the 
start height first from the database which saves the last processed height. 
If the database is empty, the start height will be retrieved from the 
`base-height` defined in the config file.
Note that `--start-height` should not be specified either lower than the last 
processed height from the database nor the `base-height` in config because 
this will be at risk of missing data.

To run the staking indexer, we need to prepare a `global-params.json` file 
which defines all the global params that are used across the BTC staking 
system. The indexer needs it to parse staking 
transaction data. An example of the global params can be found in
[test-params.json](./itest/test-params.json).

The program reads the file from the home directory by default. The user can 
specify the file path using the `--params-path` flag.

### Tests

Run unit tests:

```bash
make test
```

Run e2e tests:

```bash
make test-e2e
```

This will initiate docker containers for both a `bitcoind` node running in the 
signet mode and a `rabbitmq` instance.
