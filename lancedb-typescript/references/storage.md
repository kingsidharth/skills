# Storage & Configuration

## Storage backends

| Backend | Latency | Scale | Cost |
|---|---|---|---|
| Local SSD/NVMe | <10ms p95 | Hard to scale | Highest |
| Block (EBS/PD) | <30ms | Not shareable | High |
| File (EFS/Filestore) | <100ms | High (IOPS-limited) | Medium |
| Object (S3/GCS/Azure) | 100ms+ | Unlimited | Lowest |

LanceDB is disk-first and works with all backends. Object store is the default for production at scale.

## Connection URIs

```typescript
// Local
await lancedb.connect("/path/to/db");

// S3
await lancedb.connect("s3://bucket/path");

// GCS
await lancedb.connect("gs://bucket/path");

// Azure
await lancedb.connect("az://container/path");

// Enterprise
await lancedb.connect("db://project-slug");
```

## Storage options

Pass via `storageOptions` on connect or per-table. Keys are case-insensitive.

### AWS S3

```typescript
await lancedb.connect("s3://bucket/path", {
  storageOptions: {
    region: "us-east-1",
    awsAccessKeyId: "...",
    awsSecretAccessKey: "...",
    // awsSessionToken: "...",   // for STS
  }
});
```

Or set env vars: `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_DEFAULT_REGION`.

Minimum permissions: `s3:PutObject`, `s3:GetObject`, `s3:DeleteObject`, `s3:ListBucket`, `s3:GetBucketLocation`.

### S3-compatible (MinIO, Tigris, etc.)

```typescript
await lancedb.connect("s3://bucket/path", {
  storageOptions: {
    region: "us-east-1",
    endpoint: "http://localhost:9000",
    allowHttp: "true",  // for non-TLS
  }
});
```

### DynamoDB commit store (concurrent writers on S3)

S3 lacks atomic writes. Use DynamoDB for safe concurrent writers:

```typescript
await lancedb.connect("s3+ddb://bucket/path?ddbTableName=my-lock-table", {
  storageOptions: { region: "us-east-1" }
});
```

DynamoDB table schema: hash key `base_uri` (string), range key `version` (number).

### Google Cloud Storage

```typescript
await lancedb.connect("gs://bucket/path", {
  storageOptions: {
    serviceAccount: "/path/to/service-account.json",
  }
});
```

Or env var: `GOOGLE_SERVICE_ACCOUNT`.

### Azure Blob Storage

```typescript
await lancedb.connect("az://container/path", {
  storageOptions: {
    azureStorageAccountName: "...",
    azureStorageAccountKey: "...",
  }
});
```

### General options

| Key | Description |
|---|---|
| `allowHttp` | Allow non-TLS connections |
| `connectTimeout` | Connect phase timeout |
| `timeout` | Full request timeout |
| `proxyUrl` | Route requests through proxy |
| `downloadRetryCount` | Retries for downloads |

### New table configuration

| Key | Default | Description |
|---|---|---|
| `newTableDataStorageVersion` | `stable` | Lance file format version (`legacy` for old clients) |
| `newTableEnableV2ManifestPaths` | `false` | V2 manifest naming (requires ≥0.10.0) |
| `newTableEnableStableRowIds` | `false` | Stable row IDs across compaction |

## Namespaces

Organize tables hierarchically. OSS uses directory-based namespaces (implicit single root). Enterprise uses REST-based catalog namespaces.

```typescript
// OSS — implicit root namespace
const db = await lancedb.connect("./local_lancedb");
// Tables stored in ./local_lancedb/data/

// Enterprise — explicit namespace
const table = await db.createTable("user", data, { namespace: "prod.search" });
```
