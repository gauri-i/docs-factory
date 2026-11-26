---
title: Replicate a database into Iceberg tables with OLake Go
summary: Point OLake Go's Iceberg REST writer at Unity Catalog's Iceberg REST API, then replicate a database into managed Iceberg tables the catalog owns.
diataxis: how-to
project: unitycatalog
references:
  - unityCatalogServer
  - icebergRestSpec
  - icebergSpec
  - s3Spec
status: draft
---

[OLake Go](https://olake.io/) replicates an operational database into
[Apache Iceberg](model:icebergSpec) tables that
[Unity Catalog](model:unityCatalogServer) governs, starting with a full load
and continuing with change data capture. It writes through the
[Iceberg REST Catalog](model:icebergRestSpec) API, so it needs no Unity
Catalog-specific connector: its Iceberg writer treats the server as a generic
REST catalog. Its sources are PostgreSQL, MySQL, Oracle, SQL Server, IBM Db2,
MongoDB, Amazon S3, and Apache Kafka, and its
[connector documentation](https://olake.io/docs/connectors/overview/) gives the
sync modes each one supports.

:::prerequisites
- A catalog with managed storage on S3, called `retail` in these examples:
  [Set managed storage for catalogs and schemas on S3](../managed-storage-s3/index.md).
  OLake Go sends no table `location`, so the server owns every table it writes
  and needs somewhere to put it.
- Two schemas in that catalog, neither of which OLake Go creates: one to hold
  the replicated tables, called `sales` here, so they land in `retail.sales`,
  and one called `test_olake`, where OLake Go writes a probe table on every
  run: [Manage catalogs and schemas](../manage-catalogs-and-schemas/index.md).
- A source OLake Go can reach, with the replication prerequisites its connector
  documents.
- [OLake Go](https://olake.io/docs/install/docker-cli/), which runs as a
  container image per source.
:::

## Write the destination configuration

OLake Go takes its writer settings in a `destination.json`. Set
`catalog_type` to `rest` and point `rest_catalog_url` at the server's Iceberg
REST endpoint, `/api/2.1/unity-catalog/iceberg`:

```json
{
  "type": "ICEBERG",
  "writer": {
    "catalog_type": "rest",
    "catalog_name": "unity",
    "rest_catalog_url": "http://host.docker.internal:8080/api/2.1/unity-catalog/iceberg",
    "iceberg_s3_path": "retail",
    "s3_endpoint": "http://host.docker.internal:9000",
    "aws_region": "us-east-1",
    "aws_access_key": "<AWS_ACCESS_KEY>",
    "aws_secret_key": "<AWS_SECRET_KEY>",
    "arrow_writes": false
  }
}
```

OLake Go runs inside a container, so both addresses are the host's rather than
`localhost`. `host.docker.internal` resolves to the host under Docker Desktop;
on Linux, pass `--add-host=host.docker.internal:host-gateway` to the commands
below. A server elsewhere takes its own address.

What each field holds against this server:

| Field | What it holds here |
| --- | --- |
| `catalog_type` | `rest`, for the Iceberg REST Catalog API. |
| `catalog_name` | OLake Go's own name for the catalog. Defaults to `olake_iceberg` when empty. It does not have to match the Unity Catalog catalog. |
| `rest_catalog_url` | The server's Iceberg REST endpoint, including `/api/2.1/unity-catalog/iceberg`. |
| `iceberg_s3_path` | **The Unity Catalog catalog name**, such as `retail`, not a bucket path. OLake Go sends it as the Iceberg `warehouse`, which the server resolves to a catalog. |
| `s3_endpoint` | The S3 API that receives the data files. Omit it for Amazon S3; set it for S3-compatible storage, which OLake Go otherwise never reaches. |
| `aws_region` | The region of the bucket behind the catalog's managed storage. |
| `aws_access_key`, `aws_secret_key` | Keys OLake Go uses to write data files to [S3](model:s3Spec) itself. Leave them out to fall back to an IAM role or instance profile. |
| `arrow_writes` | Whether to use the Arrow-based writer. Defaults to `false`. |

`iceberg_s3_path` is the field most often filled in wrongly. The server reads
it as the `warehouse` query parameter and answers `404` to any value that
isn't one of its catalog names, an `s3://` URL included.

OLake Go writes the Iceberg data files with the keys above rather than the
short-lived credentials the server vends to readers, so those keys need write
access to the catalog's storage location directly. For how the server's own
vending works, and why other engines need no keys, see
[Credential vending](../../explanation/credential-vending/index.md).

## Authenticate to the server

The server recognizes one client credential: a Unity Catalog access token,
presented as a bearer token. A user obtains one by exchanging an identity
provider's token at `POST /api/1.0/unity-control/auth/tokens`, and the server
writes the first token, for `admin`, to `etc/conf/token.txt` the first time it
starts with authorization enabled. See
[Principals and users](../../explanation/uc-basics/index.md#principals-and-users).

Carry it in the writer as `token`:

```json
{
  "type": "ICEBERG",
  "writer": {
    "catalog_type": "rest",
    "catalog_name": "unity",
    "rest_catalog_url": "http://host.docker.internal:8080/api/2.1/unity-catalog/iceberg",
    "iceberg_s3_path": "retail",
    "s3_endpoint": "http://host.docker.internal:9000",
    "aws_region": "us-east-1",
    "aws_access_key": "<AWS_ACCESS_KEY>",
    "aws_secret_key": "<AWS_SECRET_KEY>",
    "token": "<UC_ACCESS_TOKEN>"
  }
}
```

A server with authorization disabled, the local one included, accepts every
request, so omit `token` there.

OLake Go's remaining authentication types belong to other catalogs. The flows
in [OLake Go's REST catalog reference](https://olake.io/docs/writers/iceberg/catalog/rest/?rest-catalog=unity)
are Databricks Unity Catalog's, built on personal access tokens that this
server doesn't issue.

## Select the streams to replicate

Put the source's connection settings in a `source.json`, as its connector page
describes, and the destination above in a `destination.json`, both in one
folder. OLake Go reads and writes that folder as `/mnt/config`, so mount it by
its full path. Then list what the source offers:

```bash
docker run --pull=always \
  -v "<PATH_TO_CONFIG_FOLDER>:/mnt/config" \
  olakego/source-postgres:latest \
  discover \
  --config /mnt/config/source.json
```

Replace `source-postgres` with the image for your source: `source-mysql`,
`source-oracle`, `source-mssql`, `source-db2`, `source-mongodb`, `source-s3`,
or `source-kafka`.

The command writes `available_streams.json`, `selected_streams.json`, and
`streams.json` into that folder. Reduce `selected_streams.json` to the tables
you want.

Every stream in `available_streams.json` carries a `destination_database`, the
schema its table goes to. `discover` fills it from the source's own name for
the data; set it to the schema you created, `sales`.

## Run the sync

```bash
docker run --pull=always \
  -v "<PATH_TO_CONFIG_FOLDER>:/mnt/config" \
  olakego/source-postgres:latest \
  sync \
  --config /mnt/config/source.json \
  --selected-streams /mnt/config/selected_streams.json \
  --available-streams /mnt/config/available_streams.json \
  --destination /mnt/config/destination.json
```

OLake Go creates one Iceberg table per selected stream, commits it through the
REST catalog, and records its position in a `state.json`, so a later run
resumes rather than reloading. The tables it writes read like any other Iceberg
table in the catalog, as in
[Use Iceberg REST clients with Unity Catalog](../iceberg-rest/index.md).

OLake Go also ships a web UI that holds the same source and destination
settings; its
[quickstart](https://olake.io/docs/getting-started/quickstart/) covers it.

## Next steps

- [Use Iceberg REST clients with Unity Catalog](../iceberg-rest/index.md) reads
  the tables OLake Go wrote.
- [External tables and catalog-managed Delta tables](../../explanation/external-and-managed-tables/index.md)
  covers what the server does with a table it owns.
- [Find an integration for your tool](../integrations/index.md) lists the other
  engines and tools that connect to the server.
