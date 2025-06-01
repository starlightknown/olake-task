# PostgreSQL to Apache Iceberg Integration with Olake

This document outlines the process of setting up a data pipeline to sync data from PostgreSQL to Apache Iceberg tables, along with troubleshooting steps for common issues.

## Prerequisites

- Docker and Docker Compose
- PostgreSQL database
- Apache Iceberg
- Olake for CDC (Change Data Capture)
- MinIO or lakehouse
- Spark SQL for querying
- wal2json logical decoding plugin for PostgreSQL

## Step 1: Setting Up PostgreSQL Source

1. Create a PostgreSQL database:

```bash
docker run --name postgres-db -e POSTGRES_PASSWORD=password -e POSTGRES_USER=iceberg -e POSTGRES_DB=iceberg -p 5432:5432 -d postgres:13
```

2. Install wal2json plugin (if not already included in your PostgreSQL image):

   a. For Debian/Ubuntu-based PostgreSQL:
   ```bash
   apt-get update
   apt-get install -y postgresql-13-wal2json
   ```

   b. For RHEL/CentOS-based PostgreSQL:
   ```bash
   yum install -y postgresql13-wal2json
   ```

   c. From source:
   ```bash
   git clone https://github.com/eulerto/wal2json.git
   cd wal2json
   make
   make install
   ```

3. Configure PostgreSQL for CDC:
   - Edit `postgresql.conf` to enable logical replication and wal2json:
   ```
   wal_level = logical
   max_wal_senders = 10
   max_replication_slots = 10
   shared_preload_libraries = 'wal2json'
   ```
   - Restart PostgreSQL to apply changes

4. Create a replication slot using wal2json:
```sql
SELECT pg_create_logical_replication_slot('postgres_slot', 'wal2json');
```

5. Verify the wal2json plugin is properly installed:
```sql
SELECT * FROM pg_replication_slots;
```

## Step 2: Setting Up the S3 Storage

1. Deploy MinIO for S3-compatible storage:

```bash
docker run -p 9000:9000 -p 9001:9001 -e "MINIO_ROOT_USER=admin" -e "MINIO_ROOT_PASSWORD=password" minio/minio server /data --console-address ":9001"
```

2. Create a bucket named "warehouse" for Iceberg data:
   - Access the MinIO console at http://localhost:9001
   - Log in with admin/password
   - Create a new bucket named "warehouse"

## Step 3: Configuring Olake for CDC

1. Create the necessary configuration files in `olake_config/`:

   - `source.json` - PostgreSQL connection details with wal2json:
   ```json
   {
       "host": "localhost",
       "port": 5432,
       "database": "iceberg",
       "username": "iceberg",
       "password": "password",
       "jdbc_url_params": {},
       "ssl": {
           "mode": "disable"
       },
       "update_method": {
           "replication_slot": "postgres_slot",
           "intial_wait_time": 10
       },
       "reader_batch_size": 100000,
       "default_mode": "cdc",
       "max_threads": 5
   }
   ```

   - `destination.json` - Iceberg connection details:
   ```json
   {
       "type": "ICEBERG",
       "writer": {
         "catalog_type": "jdbc",
         "jdbc_url": "jdbc:postgresql://localhost:5432/iceberg",
         "jdbc_username": "iceberg",
         "jdbc_password": "password",
         "normalization": false,
         "iceberg_s3_path": "s3a://warehouse",
         "s3_endpoint": "http://localhost:9000",
         "s3_use_ssl": false,
         "s3_path_style": true,
         "aws_access_key": "admin",
         "aws_secret_key": "password",
         "aws_region": "us-east-1",
         "iceberg_db": "iceberg"
       }
     }
   ```

   - `streams.json` - Configuration for the tables to sync

2. Run the Olake sync process:
```bash
olake sync --config-dir ./olake_config
```

## Step 4: Querying Iceberg Tables with Spark SQL

1. Start Spark SQL with Iceberg support:
```bash
spark-sql --packages org.apache.iceberg:iceberg-spark-runtime-3.3_2.12:1.3.0 \
--conf spark.sql.extensions=org.apache.iceberg.spark.extensions.IcebergSparkSessionExtensions \
--conf spark.sql.catalog.spark_catalog=org.apache.iceberg.spark.SparkSessionCatalog \
--conf spark.sql.catalog.spark_catalog.type=hive \
--conf spark.sql.catalog.olake_iceberg=org.apache.iceberg.spark.SparkCatalog \
--conf spark.sql.catalog.olake_iceberg.type=jdbc \
--conf spark.sql.catalog.olake_iceberg.uri=jdbc:postgresql://localhost:5432/iceberg \
--conf spark.sql.catalog.olake_iceberg.jdbc.user=iceberg \
--conf spark.sql.catalog.olake_iceberg.jdbc.password=password \
--conf spark.sql.catalog.olake_iceberg.warehouse=s3a://warehouse \
--conf spark.sql.catalog.olake_iceberg.s3.endpoint=http://localhost:9000 \
--conf spark.sql.catalog.olake_iceberg.s3.access-key=admin \
--conf spark.sql.catalog.olake_iceberg.s3.secret-key=password \
--conf spark.sql.catalog.olake_iceberg.s3.path-style-access=true
```

2. Check available catalogs:
```sql
SHOW CATALOGS;
```

3. Use the Olake Iceberg catalog:
```sql
USE olake_iceberg;
```

4. List available tables:
```sql
SHOW TABLES IN iceberg;
```

5. Query a table:
```sql
SELECT * FROM olake_iceberg.iceberg.endpoint_statistics LIMIT 10;
```

## Challenges and Solutions

### Challenge 1: Table Not Found Errors

**Problem:** Queries like `SELECT * FROM olake_iceberg.olake_iceberg.table_name;` fail with "TABLE_OR_VIEW_NOT_FOUND" errors.

**Cause:** Incorrect catalog and schema references or tables not synced properly.

**Solution:** 
1. Verify the actual catalog and schema structure:
   ```sql
   SHOW CATALOGS;
   SHOW SCHEMAS;
   ```

2. Check which tables are actually available:
   ```sql
   SHOW TABLES IN iceberg;
   ```

3. Use the correct fully-qualified name:
   ```sql
   SELECT * FROM olake_iceberg.iceberg.endpoint_statistics;
   ```

### Challenge 2: JDBC Catalog Configuration Issues

**Problem:** JDBC catalog doesn't properly recognize Iceberg tables.

**Cause:** Iceberg JDBC catalog needs specific configuration for view support.

**Solution:** Enable view support in the JDBC catalog configuration:
```
jdbc.schema-version=V1
```

This warning appears in logs but doesn't prevent table creation:
```
WARN org.apache.iceberg.jdbc.JdbcCatalog - JDBC catalog is initialized without view support. 
To auto-migrate the database's schema and enable view support, set jdbc.schema-version=V1
```

### Challenge 3: Sync Process Timing Out

**Problem:** Sync process terminates with "Idle timeout reached while waiting for new messages".

**Cause:** No new data changes detected in the source database.

**Solution:** This is normal behavior when there are no changes to sync. The process will wait for new changes and terminate after the configured idle timeout. To test, insert or update data in PostgreSQL and run the sync again.

### Challenge 4: wal2json Plugin Issues

**Problem:** CDC process fails with errors related to wal2json.

**Cause:** The wal2json plugin may not be properly installed or configured.

**Solution:**
1. Verify wal2json is properly installed:
   ```sql
   SELECT * FROM pg_available_extensions WHERE name = 'wal2json';
   ```

2. Check if the replication slot was created successfully:
   ```sql
   SELECT * FROM pg_replication_slots WHERE plugin = 'wal2json';
   ```

3. If using Docker, ensure the PostgreSQL image includes wal2json or mount a custom postgresql.conf:
   ```bash
   docker run -v /path/to/custom/postgresql.conf:/etc/postgresql/postgresql.conf -e POSTGRES_PASSWORD=password -e POSTGRES_USER=iceberg -e POSTGRES_DB=iceberg -p 5432:5432 -d postgres:13
   ```

## Conclusion

The PostgreSQL to Iceberg integration pipeline using Olake provides an effective way to sync data from a transactional database to an analytics-optimized storage format. When properly configured, it enables:

1. Real-time CDC from PostgreSQL using wal2json
2. Storage in the Apache Iceberg format on S3
3. SQL access to the data via Spark SQL

Key points for successful implementation:
- Ensure PostgreSQL is properly configured for logical replication
- Verify wal2json plugin is installed and properly configured
- Create appropriate publications for the tables you want to replicate
- Verify S3 storage is accessible with correct credentials
- Use the correct catalog and schema references in queries
- Check sync logs for any errors or warnings
- Confirm tables are actually synced before attempting to query them

For query troubleshooting, always start by verifying which tables are actually available and use the fully-qualified name pattern:
```sql
SELECT * FROM olake_iceberg.iceberg.[table_name];
```

The Olake catalog contains synced tables in the "iceberg" schema, not in the catalog name itself. 