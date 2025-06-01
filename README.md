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

2. Install wal2json plugin:
   From source (for any platform):

   ```bash
   git clone https://github.com/eulerto/wal2json.git
   cd wal2json
   make
   make install
   ```

   For Docker-based setup (preferred method):
   ```bash
   # Use a PostgreSQL image that includes wal2json
   docker run --name postgres-db -e POSTGRES_PASSWORD=password -e POSTGRES_USER=iceberg -e POSTGRES_DB=iceberg -p 5432:5432 -d debezium/postgres:13
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

2. Run the discover command

3. Run the Olake sync command

## Step 4: Querying Iceberg Tables with Spark SQL

1. Start Spark SQL with Iceberg support

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




### Challenge 1: wal2json Plugin Issues

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
   # The simplest solution is to use Debezium's PostgreSQL image which includes wal2json
   docker run --name postgres-db -e POSTGRES_PASSWORD=password -e POSTGRES_USER=iceberg -e POSTGRES_DB=iceberg -p 5432:5432 -d debezium/postgres:13
   ```

### Challenge 2: macOS-specific Issues

**Problem:** Difficulties installing or configuring PostgreSQL with wal2json on macOS.

**Cause:** macOS has specific paths and installation methods that differ from Linux environments.

**Solution:**
1. Use Docker-based PostgreSQL to avoid platform-specific installation issues:
   ```bash
   docker run --name postgres-db -e POSTGRES_PASSWORD=password -e POSTGRES_USER=iceberg -e POSTGRES_DB=iceberg -p 5432:5432 -d debezium/postgres:13
   ```

2. If using a local PostgreSQL installation:
   - Ensure PostgreSQL was installed via Homebrew: `brew install postgresql`
   - Set the proper PG_CONFIG path before compiling wal2json:
     ```bash
     export PG_CONFIG=/opt/homebrew/bin/pg_config  # For Apple Silicon Macs
     ```
   - After installation, ensure PostgreSQL loads the plugin by modifying the correct postgresql.conf:

### Challenge 3: Stream Discovery Issues

**Problem:** The `olake discover` command fails or doesn't show all expected tables.

**Cause:** Connection issues, permission problems, or replication slot configuration issues.

**Solution:**
1. Verify database connection works:
   ```bash
   psql -h localhost -U iceberg -d iceberg
   ```

2. Check permissions:
   ```sql
   -- Ensure the user has necessary permissions
   GRANT USAGE ON SCHEMA public TO iceberg;
   GRANT SELECT ON ALL TABLES IN SCHEMA public TO iceberg;
   ```

3. Verify replication slot status:
   ```sql
   SELECT * FROM pg_replication_slots;
   ```

4. Try running discovery with debug logs:
   ```bash
   olake discover --config-dir ./olake_config --log-level debug
   ```

### Challenge 4: Sync Process Errors

**Problem:** Sync process fails or gets stuck.

**Cause:** Misconfiguration, connection issues, or permission problems.

**Solution:**
1. Check logs in the `olake_config/logs/` directory
2. Verify PostgreSQL is accepting connections
3. Ensure MinIO/S3 storage is accessible
4. For permission issues, make sure the PostgreSQL user has the proper privileges
5. Try running with debug logging:
   ```bash
   olake sync --config-dir ./olake_config --log-level debug
   ```

## Conclusion

The PostgreSQL to Iceberg integration pipeline using Olake provides an effective way to sync data from a transactional database to an analytics-optimized storage format. When properly configured, it enables:

1. Real-time CDC from PostgreSQL using wal2json
2. Storage in the Apache Iceberg format on S3
3. SQL access to the data via Spark SQL

Key points for successful implementation:
- Ensure PostgreSQL is properly configured for logical replication
- Verify wal2json plugin is installed and properly configured
- Verify S3 storage is accessible with correct credentials
- Use the correct catalog and schema references in queries
- Check sync logs for any errors or warnings
- Confirm tables are actually synced before attempting to query them
