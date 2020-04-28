# Cloud Spanner Emulator

Cloud Spanner Emulator provides application developers with a locally-running,
_emulated_ instance of Cloud Spanner to enable local development and testing.

> :warning: The emulator is currently in beta. See below for known issues and
            limitations.

The main focus of the emulator is correctness - an application that runs against
the emulator should be able to run against the Cloud Spanner service without any
changes. It is not a goal of the emulator to be a performant standalone
production database or to provide similar performance to Cloud Spanner. The
emulator is specifically intended for local unit testing of applications
targeting Cloud Spanner.

## Quickstart

There are multiple ways to invoke the emulator.

### Via gcloud

The emulator is included in the [Google Cloud SDK](https://cloud.google.com/sdk)
and can be invoked via the [gcloud emulators](
https://cloud.google.com/sdk/gcloud/reference/beta/emulators) command group:

```shell
gcloud beta emulators spanner start
```

### Via pre-built docker image

The emulator is also provided as a [pre-built docker image](
https://gcr.io/cloud-spanner-emulator/emulator). You can run the latest version
with:

```shell
docker pull gcr.io/cloud-spanner-emulator/emulator
docker run -p 9010:9010 -p 9020:9020 gcr.io/cloud-spanner-emulator/emulator
```

The first port is the gRPC port and the second port is the REST port. The docker
images are also tagged with version numbers, so you can run a specific version
with:

```shell
docker run -p 9010:9010 -p 9020:9020 gcr.io/cloud-spanner-emulator/emulator:0.7.3
```

### Via pre-built linux binaries

The emulator is also distributed as a standalone linux binary. Note that this
binary is not fully static, but has been tested on Ubuntu 16.04/18.04, CentOS 8,
RHEL 8, and Debian 9/10.

```shell
VERSION=0.7.3
wget https://storage.googleapis.com/cloud-spanner-emulator/releases/${VERSION}/cloud-spanner-emulator_linux_amd64-${VERSION}.tar.gz
tar zxvf cloud-spanner-emulator_linux_amd64-${VERSION}.tar.gz
chmod u+x gateway_main emulator_main
```

`emulator_main` contains the gRPC server. If you do not need REST functionality,
you can just use this binary. To override the default host/port at which the
emulator runs:

```shell
./emulator_main --host_port localhost:1234
```

`gateway_main` is the REST gateway which will also start the emulator gRPC
server as a subprocess. To override the default host/port for the gateway and
emulator:

```shell
./gateway_main --hostname localhost --grpc_port 1234 --http_port 1235
```

### Via bazel

Production releases of the emulator are built on Ubuntu 16.04 with bazel 2.0.0
and gcc 7.4. You may be able to compile on compatible systems with compatible
toolchains. From the emulator source directory, you can build and run the
emulator via bazel from the source root with:

```shell
bazel run binaries/gateway_main
```

### Via custom docker image

You can build the emulator docker image from the source root with:

```shell
docker build . -t emulator -f build/docker/Dockerfile.ubuntu
```

You can then invoke the emulator with:

```shell
docker run -p 9010:9010 -p 9020:9020 emulator
```

The first port is the gRPC port and the second port is the REST port.


## Technical Details

The Cloud Spanner Emulator is built using the [ZetaSQL](https://github.com/google/zetasql)
reference implementation and is divided into three layers (each in its own
directory):

- A REST gateway generated by [grpc-gateway](https://github.com/grpc-ecosystem/grpc-gateway)
- A gRPC frontend which implements Cloud Spanner's gRPC API
- A database backend which emulates Cloud Spanner's database features

The core emulator codebase is in C++, and the REST wrapper is written in Go.
SQL query execution, value/type classes, and SQL functions are provided by the
ZetaSQL reference implementation. The API surface, DDL, transactional semantics,
constraint enforcement, and in-memory storage are implemented in this codebase.

## Features and Limitations

> :warning: The emulator is currently in beta.

Notable supported features:

- DDL schema changes

- Full SQL/DML query execution (beta exceptions noted below)

- Non-SQL read and write methods

- Instance and Database admin APIs including long running operations

- Reads with stale timestamps

- Secondary indexes

- Commit timestamps

- Information schema

Limitations during the beta (will be removed at version 1.0.0):

- SQL functions TABLESAMPLE, JSON_VALUE, JSON_QUERY, CEILING, POWER,
  CHARACTER_LENGTH, and FORMAT are not supported.

- Queries using some SQL functionality present in ZetaSQL but not in the
  Cloud Spanner service will succeed instead of being rejected as invalid.

- Transactions are rolled-back on any error (in the Cloud Spanner service,
  certain statements with errors, e.g. queries with invalid table names, do not
  rollback the transaction).

- PartitionedRead, PartitionedQuery, PartitionedDML are not supported. Note that
  Cloud Spanner dataflow templates depend on the PartitionedQuery  API and hence
  are not supported.

- Untyped parameter bindings (used by client libraries written in languages with
  dynamic typing) are not supported.

- Foreign keys are not supported.

Notable limitations:

- The gRPC and REST endpoints run on separate ports and serve unencrypted
  traffic.

- gRPC request deadlines and cancellations are ignored by the emulator.

- IAM apis (SetIamPolicy, GetIamPolicy, SetIamPermissions) and Backup APIs
  are not supported.

- The emulator only allows one read-write transaction or schema change at a
  time. Any concurrent transaction will be aborted. Transactions should always
  be wrapped in a retry loop. This [recommendation](
  https://cloud.google.com/spanner/docs/transactions) applies to the Cloud
  Spanner service as well.

- The emulator does not support persistence - all data is kept in memory and
  discarded when the emulator terminates.

- Error messages may not be consistent between the emulator and the Cloud
  Spanner service. Error messages are not part of Cloud Spanner's API contract
  and application code should not depend on the text of the error message being
  consistent.

- If multiple constraint violations are found during a transaction commit, the
  violation reported by the emulator may be different from the one reported by
  the Cloud Spanner service.

- The SQL query modes EXPLAIN and PROFILE are disabled. The emulator does not
  guarantee the same query execution plan as the Cloud Spanner service, and
  hence query plans and statistics reporting are disabled on the emulator.

- Certain quotas and limits (such as admin api rate limits and mutation size
  limits) are not enforced.

- List APIs (ListSessions, ListInstances) do not support filtering by labels.

- Many tables related to runtime introspection in the SPANNER_SYS schema (e.g.,
  query stats tables) are not supported.

- Server-side monitoring and logging functionality such as audit logs,
  stackdriver logging, and stackdriver monitoring are not supported.

## Frequently Asked Questions (FAQ)

#### What is the recommended test setup?

Use a single emulator process and create a Cloud Spanner instance within it.
Since creating databases is cheap in the emulator, we recommend that each test
bring up and tear down its own database. This ensures hermetic testing and
allows the test suite to run tests in parallel if needed.

#### Why is the order of rows returned by the emulator different across runs?

The emulator intentionally randomizes query results with no ORDER BY clause.
You should not depend on ordering done by the Cloud Spanner service in the
absence of an ORDER BY clause.


## Contribute

During the beta, we are not accepting external code contributions to this
project.

## Issues

During the beta, we are not accepting issues for this project. Please feel free
to ask questions or file requests/bugs using the existing Cloud Spanner
[support channels](https://cloud.google.com/spanner/docs/getting-support).

## License

[Apache License 2.0](LICENSE)

