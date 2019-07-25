# Beta Mirror Node

This BetaMirrorNode implementation supports CryptoService, FileService and SmartContractService through a proxy.
It can also parse the RecordStream Files generated by Hedera nodes into `.log` and `.txt` files.

## Description

Beta mirror node is a temporary mirror node implementation until the Hedera Mirrornet and associated Mirror Nodes receiving gossip about gossip are available.

The Beta mirror node works as follows:

- When a transaction reaches consensus, Hedera nodes add the transaction and its associated record to a record file.
- The file is closed on a regular cadence and a new file is created for the next batch of transactions and records. The interval is currently set to 1 minute but may vary between networks.
- Once the file is closed, nodes generate a signature file which contains the signature generated by the node for the record file.
- Record files also contain the hash of the previous record file.

- On a regular cadence, the signature and record files are uploaded from the nodes to Amazon S3 and Google File Storage.

- This mirror node software downloads signature files from either S3 or Google File Storage.
- The signature files are validated to ensure more than 2/3 of the nodes in the address book (stored in a 0.0.102 file) have the same signature.
- For each valid signature file, the corresponding record file is then downloaded from the cloud.
- Record files can then be processed and transactions and records processed for long term storage.

- In addition, nodes regularly generate a balance file which contains the list of Hedera accounts and their corresponding balance is also uploaded to S3 or Google File Storage.
- The files are also signed by the nodes.
- This mirror node software can download the balance files, validate 2/3rd of nodes have signed and then process the balance files for long term storage.

## Prerequisites

This mirror node beta requires Java version 10.

## Compile from source code

Run `mvn install -DskipTests` from the `MirrorNode` directory.

This will compile a runnable mirror node jar file in the `target` directory and copy sample `nodesInfo.json.sample`, `config.json.sample` and `log4j2.xml` files into the same directory.

`cd target`

## Docker images

### Volumes

docker volume create MirrorNodePostgresData
docker volume create MirrorNodeData

### Database

```shell
cd docker
docker build -t hedera/mirrornode-db -f Postgres ../
docker run --name=hedera-mirrornode-db -d -p 127.0.0.1:5432:5432 --mount source=MirrorNodePostgresData,target=/var/lib/postgresql/data/pgdata hedera/mirrornode-db:latest
```

### Compile the mirror node software

```shell
mvn install -DskipTests
```

### Record files downloader

```shell
cd docker
docker build -t hedera/mirrornode-rcd-down -f RecordsDownloader ../
docker run --name=hedera-mirrornode-rcd-down -d --mount type=volume,source=MirrorNodeData,target=/mirrordata hedera/mirrornode-rcd-down:latest
```

### Record files parser and logger

```shell
cd docker
docker build -t hedera/mirrornode-rcd-parse -f RecordsParser ../
docker run --name=hedera-mirrornode-rcd-parse -d --mount source=MirrorNodeData,target=/mirrordata hedera/mirrornode-rcd-parse:latest
```

### Balance files downloader

```shell
cd docker
docker build -t hedera/mirrornode-bal-down -f BalanceDownloader ../
docker run --name=hedera-mirrornode-bal-down -d --mount source=MirrorNodeData,target=/mirrordata hedera/mirrornode-bal-down:latest
```

### Balance files parser and logger

```shell
cd docker
docker build -t hedera/mirrornode-bal-parse -f BalanceParser ../
docker run --name=hedera-mirrornode-bal-parse -d --mount source=MirrorNodeData,target=/mirrordata hedera/mirrornode-bal-parse:latest
```

### Proxy

### Volume inspector

Some operating systems won't enable you to look at the data mounted in volumes, this image will give you access to the data.

- the `MirrorNodePostgresData` volume is mounted to `/databasedata`
- the `MirrorNodeData` volume is mounted to `/mirrordata`

```shell
cd docker
docker build -t hedera/mirrornode-vol-inspect -f VolumeInspector ../
docker run --name=hedera-mirrornode-vol-inspect -d --mount source=MirrorNodeData,target=/mirrordata --mount source=MirrorNodePostgresData,target=/databasedata hedera/mirrornode-vol-inspect:latest
```

Assuming configuration files have been properly set up.

Run

```shell
mvn install -DskipTests
docker build -t hedera/mirrornode .
docker run --name=hedera-mirrornode -p 127.0.0.1:8080:8080 hedera/mirrornode:latest
```

## Changes since last release

Besides bug fixes, some features may have changed with this release which need your attention, these will be listed here.

### Database URL as environment parameter

The connection to the database can be specified as an environment variable `HEDERA_MIRROR_DB_URL` or added to the `.env.` file.

### Added optional persistence of crypto transfers, file contents, contract creation and call results and claim data

The `config.json` file now contains four addition parameters as follows:

- persistClaims : set to `true` or `false` to determine whether claim hashes will be persisted to the database. Default value is `false` if not supplied.
- persistFiles : set to `SYSTEM`, `ALL` or `NONE`. If set to `SYSTEM` only files with a file number below 1000 will be stored, if set to `ALL`, all files will be stored, if set to `NONE`, no file data will be stored. Default is `NONE`.
- persistContracts : set to `true` or `false` to determine whether contract call parameters (and constructor parameters) along with results will be persisted to the database. Default value is `false` if not supplied.
- persistCryptoTransferAmounts : set to `true` or `false` to determine whether individual `to` and `from` transfer amounts will be persisted to the database. Default value is `false` if not supplied.

### Addition of stopLoggingIfHashMismatch configuration item

When processing files after they have been downloaded, this flag will determine whether file processing should continue or stop in the event of a mismatch between the hash of the last file processed and the hash held for the previous file in the file being processed.

If set to `true` : Any mismatch in the sequence of file hashes will bring the processing to a stop, the missing file has to be downloaded in order for processing to be able to continue.
If set to `false`: Hash sequence mismatches will be logged but ignored and processing will continue until there are no files to process.

A file called `loggerStatus.json` will be created in the `./config` folder containing the hash of the last successfully processed file.

### node-log has been removes from log4.xml

All logging now goes to a single recordStream-log

### Addition of maxDownloadItems parameter in config.json file

For testing purposes, this value may be set to a number other than 0. This will limit the number of downloaded files accordingly. If set to 0, all files are downloaded from S3 or GCP.

### Addition of a postgresUpdate.sql script

This script contains the incremental changes to the database in the event you need to upgrade an older version of the database to the current version.

### The t_account_balance_history has a new `seconds` column

The `seconds` column contains the number of seconds since epoch in addition to the `snapshot_time` which is a `timestamp`, both contain the same value in different formats.

### Loggers now return true/false depending on success/failure to log

Before this change, failure to log wasn't detected by the class calling the logger which resulted in files being moved when they shouldn't be.

### Addition of cloud storage as an alternative to AWS

It is now possible to specify that files are downloaded from Google Compute Platform storage instead of Amazon S3.
The switch is enabled via the `config/config.json` file.

set `cloud-provider` to either `GCP` or `S3`, examples:

Google

```
{
  "cloud-provider": "GCP",
...
```

Amazon S3

```
{
  "cloud-provider": "S3",
...
```

### Removal of command line arguments

All configuration parameters are now sourced from `./config/config.json` by default, it no longer needs to be specified on the command line.

### Inclusion of logging to PostgreSQL database

The `recordFileLogger`, `balanceFileHistoryLogger` and `balanceFileLogger` write transactions, records and balances to a PostgreSQL database. The scripts for creating the database are provided in the `postreSQL` folder.

Note: this requires additional information to be stored in the `config.json`, `.env` or environment variables as follows:

config.json

```
  "dbUrl": "jdbc:mysql://127.0.0.1:3306/hederamirror?&useJDBCCompliantTimezoneShift=true&useLegacyDatetimeCode=false&serverTimezone=UTC",
  "dbUsername": "hederamirror",
  "dbPassword": "mysecretpassword"

```
.env file

```
HEDERA_MIRROR_DB_USER=hederamirror
HEDERA_MIRROR_DB_PASS=mysecretpassword
```

### Amazon Hedera S3 access keys

Access and secret keys to Hedera's Amazon S3 bucket may now be specified via environment variables or a `./.env` file, rather than in the `config.json` file. To make use of this feature, set the following two environment variables and remove them from the `config.json` file. The environment variables will always take precedence over those set in the `config.json` file if both are present.

```text
HEDERA_S3_ACCESS_KEY
HEDERA_S3_SECRET_KEY
```

Sample `./.env` file.

```text
HEDERA_S3_ACCESS_KEY=accessKey
HEDERA_S3_SECRET_KEY=secretKey
```

### Configuration files

All configuration files now reside in the `./config` folder.

### Config.json downloadPeriodSec

Since this version, if set to 0, the downloader will download available files and exit.
If set to another value, the downloader will download available files, wait `downloadPeriodSec` and start downloading new files again until it is stopped by the operator (`kill or ctrl-c`).

### Logging

Previous versions of the mirror node software embedded a `log4j2.xml` file in the jar, this is no longer the case, so by default, no logging will take place.
Should you wish to enable logging, copy the `log4j2.xml` file from `src/main/resources` or `target` to the same location as the `mirrorNode.jar` file, edit accordingly and include the following in your `java` command:

```shell
    -Dlog4j.configurationFile=./log4j2.xml
```

for example

```shell
    java -Dlog4j.configurationFile=./log4j2.xml -cp mirrorNode.jar com.hedera.recordFileParser.RecordFileParser
```

This will ensure that the `log4j2.xml` file is used to direct logging accordingly.
*Note: the sample `log4j2.xml` file is setup to log all `INFO` messages.*

If you do not specify a `log4j2.xml`, the following error will be output but will not prevent the software from operating.

```code
ERROR StatusLogger No Log4j 2 configuration file found. Using default configuration (logging only errors to the console), or user programmatically provided configurations. Set system property 'log4j2.debug' to show Log4j 2 internal initialization logging. See https://logging.apache.org/log4j/2.x/manual/configuration.html for instructions on how to configure Log4j 2
```

## Setup your environment

The build process has copied sample files to the `target/config` folder.

- `nodesInfo.json.sample` - rename this file to `nodesInfo.json` and edit so that the appropriate nodes are listed.
- `config.json.sample` - rename this file to `config.json` and edit so that the configuration parameters that are appropriate to your environment are setup.
- the files prefixed with '0.0.102' are the contents of a file hosted on Hedera with file ID `0.0.102`. This file contains the address book from the Hedera network which lists nodes and their public keys for signature verification purposes. Ensure the appropriate one for your network is identified in the `config.json` file (addressBookFile entry).

See the end of this document for steps to recreate this file from a different Hedera network.

## Installing the database

Ensure you have a postreSQL server running (versions 10 and 11 have been tested) with the mirror node software.

Log into the database as an administrator and run the `postgres/postgresInit.sql` script to create the database and necessary entities.

By default, this installation script creates a new database called `hederamirror`, with a user named `hederamirror` and a default password of `mysecretpassword`. Make the necessary changes to the script should you wish to use different values (and update the `config/config.json` or `.env` or environment variables accordingly).

## Note about error when running the software

The error below may appear on the console when running the `.jar` file, this is normal and nothing to be concerned about.

```code
WARNING: An illegal reflective access operation has occurred
WARNING: Illegal reflective access by com.google.protobuf.UnsafeUtil (file:/home/greg/mirrornode/lib/protobuf-java-3.5.1.jar) to field java.nio.Buffer.address
WARNING: Please consider reporting this to the maintainers of com.google.protobuf.UnsafeUtil
WARNING: Use --illegal-access=warn to enable warnings of further illegal reflective access operations
WARNING: All illegal access operations will be denied in a future release
```
## To Run BetaMirrorNode Proxy

Edit the `./config/nodesInfo.json` file to list all the nodes the proxy may interact with.

Edit the `./config/config.json` file to specify the port number of BetaMirrorNode, and nodeInfoFile path.

*Note: The beta mirror node proxy doesn't select nodes by itself, rather it inspects the transactions it receives, extracts the target node information from the transaction (`NodeAccountID` from `transactionBody`) and forwards the transaction to this node.*

Run the following command:

```shell
java -jar mirrorNode.jar
```

the config.json contains the port number of BetaMirrorNode, and nodeInfoFile path which contains host and port of Hedera Nodes.

## To Download RecordStream file(s)

Run the following command:

```shell
java -cp mirrorNode.jar com.hedera.downloader.RecordFileDownloader
```

`config.json` contains configurations for downloading:

  "clientRegion": clientRegion of the s3 bucket from which we download RecordStream files

  "bucketName": bucketName of the s3 bucket from which we download RecordStream files

  "accessKey": accessKey to connect to the s3 bucket

Note: this may be set via the `HEDERA_S3_ACCESS_KEY` environment variable instead or `./.env` file (see above).

  "secretKey": secretKey to connect to the s3 bucket

Note: this may be set via the `HEDERA_S3_SECRET_KEY` environment variable instead or `./.env` file (see above).

  "downloadPeriodSec": download period, in seconds, determines how long the downloader waits until attempting to download additional files. If set to 0, the downloader will make one attempt and exit.

  "downloadToDir": destination directory, files uploaded from the s3 bucket will be created in `downloadToDir/recordstreams/record0.0.x` for each of the nodes that publish the record files where 0.0.x is the account number related to the node that created the record files.

## To Download Balance file(s)

Setup your environment as per the section above, then run the following command:

```shell
java -cp mirrorNode.jar com.hedera.downloader.AccountBalancesDownloader
```

Balance files will be downloaded from S3 to `./accountBalances/balance/0.0.x` where `x` is the account number of the node the balance files were downloaded from.
A new balance file is created every few minutes (5 on testnet) containing the balances of every known account at the time the file is created.
Note: The timestamp for the balances is included in the file, do not use the file's name.

for example

```text
year,month,day,hour,minute,second
2019,JUNE,28,17,29,17
shard,realm,number,balance
0,0,1,0
0,0,2,4530999689861900540
```

## To Parse RecordStream file(s)

Run the following command:

``` shell
java -cp mirrorNode.jar com.hedera.recordFileParser.RecordFileParser
```

The `defaultParseDir` value of the `./config/config.json` file specifies where to find the record files to parse.

Parsed result will be shown in the Console as well as `output/recordStream.log` (depending on your `log4j2.xml` settings), a clear text version of the same will be output to `output/recordStream.txt`

The files which has been parsed will be moved to `./recordstreams/parsedRecordFiles/` directory

## To Parse Balance file(s)

This project provides two balance file parsing and logging options.

- Log only the latest balance - looks for the latest balance file and stores the balances in the database
- Log balances with timestamp history - loads every available balance file and stores account balances against the file's timestamp.

Note: You can run both, however it is imperative that 'latest' is run prior to 'history' since history moves files to a processed folder, thereby removing any files for 'latest' to process.

To parse and log the latest files, run the following command:

```
java -Dlog4j.configurationFile=./log4j2.xml -cp mirrorNode.jar com.hedera.balanceFileLogger.BalanceFileLogger ./config/config.json
```

To parse and log balances with history, run the following command:

```
java -Dlog4j.configurationFile=./log4j2.xml -cp mirrorNode.jar com.hedera.balanceFileLogger.BalanceFileHistoryLogger ./config/config.json
```

## To Send Transactions or Queries to the BetaMirrorNode Proxy

Using a client which is able to generate and send transactions to a Hedera node, update the configuration of the client application such that it sends its transactions to the proxy host and port instead of a Hedera node.

## Notes about the java project structure

The java project contains a number of packages and classes for its various modes of operation

- com.hedera.mirrorNodeProxy

Contains a `MirrorNodeProxy` class which is responsible for running the proxy.

- com.hedera.downloader

Contains a `RecordFileDownloader` class which connects to an s3 bucket and downloads record files from the bucket.

Contains a `AccountBalancesDownloader` class which connects to an s3 bucket and downloads account balance files from the bucket.

- com.hedera.recordFileParser

Contains a `RecordFileParser` class which given a number of record files will process them, it calls the static class below which is responsible for processing the output itself.

- com.hedera.recordFileLogger

Contains a `RecordFileLogger` class which is an example of how to receive transactions and records from `RecordFileParser` and output to a text file. It is recommended you modify this class for your purposes while leaving the other packages and classes untouched so that future updates do not impact your own development

`RecordFileLogger` contains the following public methods:

- public static void start()

Called whenever the `RecordFileParser` class starts running.

- public static void finish()

Called whenever the `RecordFileParser` class ends running.

- public static void initFile(String fileName)

Called whenever a new file starts processing

- public static void completeFile()

Called whenever a file completes processing

- public static void storeRecord(long counter, Instant consensusTimeStamp, Transaction transaction, TransactionRecord txRecord)

Called for each record found in the record files such that you can decide whether to store that transaction and its record in your own files or database.

- public static void storeSignature(String signature)

Called for each signature that is processed.

## Creating the address book file (0.0.102 file)

The java code below was used with the java SDK in order to create the file.

```java
package com.hedera.hashgraph.sdk.examples.advanced;

import com.google.protobuf.InvalidProtocolBufferException;
import com.hedera.hashgraph.sdk.HederaException;
import com.hedera.hashgraph.sdk.examples.ExampleHelper;
import com.hedera.hashgraph.sdk.file.FileId;
import com.hedera.hashgraph.sdk.file.FileContentsQuery;

import java.io.FileNotFoundException;
import java.io.FileOutputStream;
import java.io.IOException;

public final class GetHederaAddressBook {
    private GetHederaAddressBook() { }

    public static void main(String[] args) throws HederaException, InvalidProtocolBufferException {

        // Build the Hedera client using ExampleHelper class
        var client = ExampleHelper.createHederaClient();

        // Get file contents
        var contents = new FileContentsQuery(client)
            .setFileId(new FileId(0, 0, 102))
            .execute();
        try {
            FileOutputStream fos = new FileOutputStream("0.0.102");
            fos.write(contents.getFileContents().getContents().toByteArray());
            fos.close();
        } catch (FileNotFoundException e) {
            // TODO Auto-generated catch block
            e.printStackTrace();
        } catch (IOException e) {
            // TODO Auto-generated catch block
            e.printStackTrace();
        }
    }
}
```
