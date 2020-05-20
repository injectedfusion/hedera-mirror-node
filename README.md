[![CircleCI](https://circleci.com/gh/hashgraph/hedera-mirror-node/tree/master.svg?style=shield)](https://circleci.com/gh/hashgraph/hedera-mirror-node/tree/master)
[![codecov](https://img.shields.io/codecov/c/github/hashgraph/hedera-mirror-node/master)](https://codecov.io/gh/hashgraph/hedera-mirror-node)
[![GitHub](https://img.shields.io/github/license/hashgraph/hedera-mirror-node)](LICENSE)
[![Discord](https://img.shields.io/badge/discord-join%20chat-blue.svg)](https://hedera.com/discord)

# Hedera Mirror Node

Hedera Mirror Node exposes Hedera Hashgraph transactions, transaction records, account balances,
and events generated by the Hedera mainnet (or testnet, if so configured) via a REST API.

## Overview

Hedera mirror nodes receive the information from the mainnet nodes and they provide value-added services such as providing audit support, access to historical data, transaction analytics, visibility services, security threat modeling, data monetization services, etc. Mirror nodes can also run additional business logic to support applications built using Hedera mainnet.

While mirror nodes receive information from the mainnet nodes, they do not contribute to consensus on the mainnet, and their votes are not counted. Only the votes from the mainnet nodes are counted for determining consensus. The trust of Hedera mainnet is derived based on the the consensus reached by the mainnet nodes. That trust is transferred to the mirror nodes using cryptographic signatures on a chain of records (account balances, events, transactions, etc).

## Beta Mirror Node

Eventually, the mirror nodes can run the same code as the Hedera mainnet nodes so that they can see the transactions in real time. To make the initial deployments easier, the beta mirror node strives to take away the burden of running a full Hedera node through creation of periodic files that contain processed information (such as account balances or transaction records), and have the full trust of the Hedera mainnet nodes. The beta mirror node software reduces the processing burden by receiving pre-constructed files from the mainnet, validating those, populating a database and providing REST APIs.

### Advantages

-   Lower compute, bandwidth requirement
-   It allows users to only save what they care about, and discard what they don’t (lower storage requirement)
-   Easy searchable database so the users can add value quickly
-   Easy to consume REST APIs to make integrations faster

### Description

The beta mirror node works as follows:

-   When a transaction reaches consensus, Hedera nodes add the transaction and its associated record to a record file.
-   The file is closed on a regular cadence and a new file is created for the next batch of transactions and records. The interval is currently set to 5 seconds but may vary between networks.
-   Once the file is closed, nodes generate a signature file which contains the signature generated by the node for the record file.
-   Record files also contain the hash of the previous record file, thus creating an unbreakable validation chain.

-   The signature and record files are then uploaded from the nodes to Amazon S3 and Google File Storage.

-   This mirror node software downloads signature files from either S3 or Google File Storage.
-   The signature files are validated to ensure at least 1/3 of the nodes in the address book (stored in a `0.0.102` file) have the same signature.
-   For each valid signature file, the corresponding record file is then downloaded from the cloud.
-   Record files can then be processed and transactions and records processed for long term storage.

-   In addition, nodes regularly generate a balance file which contains the list of Hedera accounts and their corresponding balance which is also uploaded to S3 and Google File Storage.
-   The files are also signed by the nodes.
-   This mirror node software can download the balance files, validate at least 1/3 of nodes have signed and then process the balance files for long term storage.

## Getting Started

Ensure OpenJDK 11 and Docker Compose are installed, then run:

```bash
git clone git@github.com:hashgraph/hedera-mirror-node.git
cd hedera-mirror-node
./mvnw clean install -DskipTests
docker-compose up
```

> **_NOTE:_** This defaults to a bucket setup for demonstration purposes. The real bucket name is not currently publicly available.

## Documentation

-   [Installation](docs/installation.md)
-   [Configuration](docs/configuration.md)
-   [Database](docs/database.md)
-   [Operations](docs/operations.md)
-   [Testing](docs/testing.md)
-   [Troubleshooting](docs/troubleshooting.md)

## Releasing

To prepare for a new release:

```
./mvnw clean package -N -P=release -Drelease.version=x.y.z -Drelease.chartVersion=x.y.z
helm dependency update charts/hedera-mirror
```

## Contributing

Contributions are welcome. Please see the [contributing](CONTRIBUTING.md) guide to see how you can get
involved.

## Code of Conduct

This project is governed by the [Contributor Covenant Code of Conduct](CODE_OF_CONDUCT.md). By participating, you are
expected to uphold this code of conduct. Please report unacceptable behavior to [oss@hedera.com](mailto:oss@hedera.com)

## License

[Apache License 2.0](LICENSE)
