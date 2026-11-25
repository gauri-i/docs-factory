---
title: Replicate a database into Iceberg tables with OLake Go
summary: Point OLake Go's Iceberg REST writer at Unity Catalog's Iceberg REST API, then replicate a PostgreSQL, MySQL, MongoDB, or MSSQL database into managed Iceberg tables.
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
[Unity Catalog](model:unityCatalogServer) governs, with a full load followed by
change data capture. It writes through the
[Iceberg REST Catalog](model:icebergRestSpec) API, so it needs no Unity
Catalog-specific connector: its Iceberg writer treats the server as a generic
REST catalog. OLake Go supports PostgreSQL, MySQL, MongoDB, and MSSQL sources;
its [connector documentation](https://olake.io/docs/connectors/overview/) lists
the sync modes each one offers.

Because OLake Go creates tables rather than reading them, it needs a catalog
that can place new tables. See
[External tables and catalog-managed tables](../../explanation/external-and-managed-tables/index.md)
for what the server does with a table it owns.

:::prerequisites
- A catalog with managed storage on S3, such as `retail`:
  [Set managed storage for catalogs and schemas on S3](../managed-storage-s3/index.md).
  OLake Go sends no table `location`, so the server creates managed tables and
  needs somewhere to put them.
- A schema in that catalog for the replicated tables, such as `retail.sales`:
  [Manage catalogs and schemas](../manage-catalogs-and-schemas/index.md).
- A source database OLake Go can reach, with the replication prerequisites its
  connector documents.
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
    "rest_catalog_url": "http://localhost:8080/api/2.1/unity-catalog/iceberg",
    "iceberg_s3_path": "retail",
    "aws_region": "us-east-1",
    "aws_access_key": "<AWS_ACCESS_KEY>",
    "aws_secret_key": "<AWS_SECRET_KEY>",
    "arrow_writes": false
  }
}
```

Two of these fields behave differently against Unity Catalog than their names
suggest:

| Field | What it holds here |
| --- | --- |
| `catalog_type` | `rest`, for the Iceberg REST Catalog API. |
| `catalog_name` | OLake Go's own name for the catalog. Defaults to `olake_iceberg` when empty. It does not have to match the Unity Catalog catalog. |
| `rest_catalog_url` | The server's Iceberg REST endpoint, including `/api/2.1/unity-catalog/iceberg`. |
| `iceberg_s3_path` | **The Unity Catalog catalog name**, such as `retail`, not a bucket path. OLake Go sends it as the Iceberg `warehouse`, which the server resolves to a catalog. |
| `aws_region` | The region of the bucket behind the catalog's managed storage. |
| `aws_access_key`, `aws_secret_key` | Keys OLake Go uses to write data files to [S3](model:s3Spec) itself. Leave them out to fall back to an IAM role or instance profile. |
| `arrow_writes` | Whether to use the Arrow-based writer. Defaults to `false`. |

`iceberg_s3_path` is the field most often set wrongly. The server reads it as
the `warehouse` query parameter and answers `404` for a value that isn't one of
its catalog names, including an `s3://` URL.

OLake Go writes the Iceberg data files with the keys above, not with the
short-lived credentials the server vends to readers. Its keys therefore need
write access to the catalog's storage location directly. For how the server's
own vending works, and why other engines don't need keys, see
[Credential vending](../../explanation/credential-vending/index.md).

## Authenticate to the server

With authorization disabled, the server takes any request and the destination
needs no token.

With authorization enabled, exchange an identity provider's token for a Unity
Catalog access token at `POST /api/1.0/unity-control/auth/tokens`, then add it
to the writer as `token`:

```json
{
  "type": "ICEBERG",
  "writer": {
    "catalog_type": "rest",
    "catalog_name": "unity",
    "rest_catalog_url": "http://localhost:8080/api/2.1/unity-catalog/iceberg",
    "iceberg_s3_path": "retail",
    "aws_region": "us-east-1",
    "token": "<UC_ACCESS_TOKEN>"
  }
}
```

OLake Go sends `token` as a bearer token in the `Authorization` header. It has
to be a Unity Catalog access token: the Iceberg REST endpoint serves no
`/v1/oauth/tokens`, so OLake Go's `oauth2` authentication type has nothing to
exchange a client secret with.

The writer's remaining authentication options are in
[OLake Go's REST catalog reference](https://olake.io/docs/writers/iceberg/catalog/rest/).

## Select the streams to replicate

Put the source's `config.json` in a local `config/` directory, as its
connector page describes, then list what the database offers:

```bash
docker run --pull=always \
  -v "$(pwd)/config:/mnt/config" \
  olakego/source-postgres:latest \
  discover \
  --config /mnt/config/source.json
```

This writes `available_streams.json` and `selected_streams.json` beside the
configuration. Edit `selected_streams.json` down to the tables you want.

Replace `source-postgres` with the image for your database, such as
`source-mysql`, `source-mongodb`, or `source-mssql`.

## Run the sync

```bash
docker run --pull=always \
  -v "$(pwd)/config:/mnt/config" \
  olakego/source-postgres:latest \
  sync \
  --config /mnt/config/source.json \
  --destination /mnt/config/destination.json \
  --selected-streams /mnt/config/selected_streams.json \
  --available-streams /mnt/config/available_streams.json
```

OLake Go creates one Iceberg table per selected stream in the schema it
derives, commits them through the REST catalog, and records its position so a
later run resumes instead of reloading. The new tables then read like any
other Iceberg table in the catalog, as in
[Use Iceberg REST clients with Unity Catalog](../iceberg-rest/index.md).

OLake Go also ships a web UI that holds the same source and destination
settings, described in its
[quickstart](https://olake.io/docs/getting-started/quickstart/).

## Next steps

- [Use Iceberg REST clients with Unity Catalog](../iceberg-rest/index.md) reads
  the tables OLake Go wrote.
- [Find an integration for your tool](../integrations/index.md) lists the other
  engines and tools that connect to the server.
