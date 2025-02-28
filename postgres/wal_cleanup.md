
## Under what circumstances are wal files in the pg_wal directory deleted by Postgres？

In PostgreSQL, WAL (Write-Ahead Logging) files in the `pg_wal` directory are deleted under certain circumstances to manage disk space and ensure that the database operates efficiently. Here are the primary circumstances under which WAL files are deleted:

### 1. Checkpointing

- **Description**: During a checkpoint, PostgreSQL writes all dirty buffers (modified data pages) to disk and updates the control file to indicate the point up to which all data has been written to disk. WAL files that are no longer needed for crash recovery can be removed.
- **Mechanism**: Checkpoints occur periodically based on the `checkpoint_timeout` and `max_wal_size` configuration parameters.
- **Effect**: WAL files that are older than the checkpoint are deleted.

### 2. WAL Archiving

- **Description**: If WAL archiving is enabled (using the `archive_mode` and `archive_command` parameters), WAL files are archived to a specified location before they are deleted from the `pg_wal` directory.
- **Mechanism**: The `archive_command` is executed to copy the WAL files to the archive location.
- **Effect**: WAL files are deleted from the `pg_wal` directory only after they have been successfully archived.

### 3. Replication Slots

- **Description**: Replication slots ensure that WAL files are retained until they have been consumed by all connected replicas. This prevents WAL files from being deleted prematurely.
- **Mechanism**: Replication slots track the progress of each replica and ensure that WAL files are retained until they are no longer needed.
- **Effect**: WAL files are deleted only after all replication slots have confirmed that they are no longer needed.

### 4. WAL Retention Settings

- **Description**: PostgreSQL can be configured to retain a certain amount of WAL files using the `wal_keep_size` parameter. This parameter specifies the minimum size of WAL files to retain in the `pg_wal` directory.
- **Mechanism**: WAL files are retained until the total size of WAL files exceeds the `wal_keep_size` value.
- **Effect**: WAL files are deleted when the total size of WAL files exceeds the configured `wal_keep_size`.

### Example Configuration

Here is an example of how you might configure PostgreSQL to manage WAL files:

```sql
-- Enable WAL archiving
ALTER SYSTEM SET archive_mode = 'on';
ALTER SYSTEM SET archive_command = 'cp %p /path/to/archive/%f';

-- Set checkpoint timeout and max WAL size
ALTER SYSTEM SET checkpoint_timeout = '5min';
ALTER SYSTEM SET max_wal_size = '1GB';

-- Set WAL retention size
ALTER SYSTEM SET wal_keep_size = '128MB';

-- Reload the configuration
SELECT pg_reload_conf();
```

### Summary

WAL files in the `pg_wal` directory are deleted by PostgreSQL under the following circumstances:

1. **Checkpointing**: WAL files older than the checkpoint are deleted.
2. **WAL Archiving**: WAL files are deleted after they have been successfully archived.
3. **Replication Slots**: WAL files are deleted only after all replication slots have confirmed they are no longer needed.
4. **WAL Retention Settings**: WAL files are deleted when the total size of WAL files exceeds the configured `wal_keep_size`.

By understanding and configuring these mechanisms, you can effectively manage the retention and deletion of WAL files in PostgreSQL.


## What are the causes of WAL accumulation？

WAL (Write-Ahead Logging) accumulation in PostgreSQL can occur due to several reasons. Understanding these causes can help in diagnosing and resolving issues related to excessive WAL file growth. Here are some common causes of WAL accumulation:

### 1. Slow or Stalled WAL Archiving

- **Description**: If WAL archiving is enabled but the archiving process is slow or stalled, WAL files will accumulate in the `pg_wal` directory.
- **Causes**:
  - Inefficient `archive_command` that takes too long to execute.
  - Network issues or slow storage in the archive destination.
  - Insufficient resources (CPU, I/O) on the server performing the archiving.
- **Solution**: Optimize the `archive_command`, ensure the archive destination is performant, and allocate sufficient resources.

### 2. Replication Slots Retaining WAL

- **Description**: Replication slots ensure that WAL files are retained until they have been consumed by all connected replicas. If a replica is lagging or disconnected, WAL files will accumulate.
- **Causes**:
  - Slow or lagging replicas that cannot keep up with the WAL generation rate.
  - Disconnected replicas that are not consuming WAL files.
- **Solution**: Monitor and optimize replica performance, and remove unused or stale replication slots.

### 3. High WAL Generation Rate

- **Description**: A high rate of database activity (inserts, updates, deletes) can generate WAL files faster than they can be archived or replicated.
- **Causes**:
  - Bulk data loads or large transactions.
  - High-frequency updates or deletes.
  - Inefficient queries or application behavior causing excessive writes.
- **Solution**: Optimize application queries, batch large transactions, and consider increasing resources or scaling out.

### 4. Misconfigured WAL Retention Settings

- **Description**: Misconfigured settings related to WAL retention can cause unnecessary accumulation of WAL files.
- **Causes**:
  - `wal_keep_size` set too high, causing excessive retention of WAL files.
  - `max_wal_size` set too high, delaying checkpoints and causing more WAL files to be retained.
- **Solution**: Review and adjust WAL retention settings (`wal_keep_size`, `max_wal_size`) to appropriate values.

### 5. Checkpointing Issues

- **Description**: Inefficient checkpointing can lead to WAL accumulation as WAL files are retained until a checkpoint is completed.
- **Causes**:
  - Long-running checkpoints due to high I/O load or insufficient resources.
  - Misconfigured checkpoint settings (`checkpoint_timeout`, `checkpoint_completion_target`).
- **Solution**: Optimize checkpoint settings and ensure sufficient I/O capacity.

### 6. Backup and Restore Operations

- **Description**: During backup and restore operations, WAL files may be retained to ensure data consistency.
- **Causes**:
  - Long-running backup operations.
  - Delayed or stalled restore processes.
- **Solution**: Optimize backup and restore processes, and ensure they complete in a timely manner.

### Monitoring and Diagnostics

To diagnose and monitor WAL accumulation, you can use the following PostgreSQL views and functions:

- **pg_stat_archiver**: Provides information about the WAL archiver process.
- **pg_stat_replication**: Provides information about replication status and lag.
- **pg_stat_activity**: Provides information about currently running queries and their impact on the system.
- **pg_wal_lsn_diff**: Calculates the difference between WAL locations to measure lag.

### Example Queries

#### Check WAL Archiver Status

```sql
SELECT * FROM pg_stat_archiver;
```

#### Check Replication Status

```sql
SELECT
    client_addr,
    state,
    sent_lsn,
    write_lsn,
    flush_lsn,
    replay_lsn,
    pg_size_pretty(pg_wal_lsn_diff(sent_lsn, replay_lsn)) AS replay_lag
FROM
    pg_stat_replication;
```

#### Check WAL Retention Settings

```sql
SHOW wal_keep_size;
SHOW max_wal_size;
```

By understanding these causes and monitoring the relevant metrics, you can effectively manage and mitigate WAL accumulation in PostgreSQL.


## Under what circumstances are archive_command executed by Postgres？

In PostgreSQL, the `archive_command` is executed under specific circumstances to ensure that WAL (Write-Ahead Logging) files are archived properly. The `archive_command` is part of the WAL archiving process, which is used to create backups and ensure data durability. Here are the primary circumstances under which the `archive_command` is executed:

### 1. WAL File Completion

- **Description**: The `archive_command` is executed when a WAL file is completed. A WAL file is considered complete when it reaches its maximum size (typically 16 MB by default).
- **Mechanism**: When a WAL file is filled up, PostgreSQL closes the file and triggers the `archive_command` to archive the file.
- **Effect**: The completed WAL file is copied to the archive location specified by the `archive_command`.

### 2. Checkpoints

- **Description**: During a checkpoint, PostgreSQL ensures that all changes up to a certain point are written to disk. If there are any incomplete WAL files at the time of the checkpoint, they are also archived.
- **Mechanism**: Checkpoints occur periodically based on the `checkpoint_timeout` and `max_wal_size` configuration parameters.
- **Effect**: Any WAL files that are part of the checkpoint are archived.

### 3. Manual Archiving

- **Description**: The `archive_command` can be triggered manually by using the `pg_switch_wal()` function, which forces PostgreSQL to switch to a new WAL file and archive the current one.
- **Mechanism**: When `pg_switch_wal()` is called, PostgreSQL closes the current WAL file and triggers the `archive_command`.
- **Effect**: The current WAL file is archived immediately.

### Example Configuration

Here is an example of how you might configure the `archive_command` in PostgreSQL:

```sql
ALTER SYSTEM SET archive_mode = 'on';
ALTER SYSTEM SET archive_command = 'cp %p /path/to/archive/%f';
SELECT pg_reload_conf();
```

### Explanation

- **archive_mode**: Enables or disables WAL archiving. It must be set to `on` to enable archiving.
- **archive_command**: Specifies the shell command to be executed to archive a completed WAL file. The `%p` placeholder represents the path to the WAL file, and the `%f` placeholder represents the filename.
- **pg_reload_conf()**: Reloads the PostgreSQL configuration to apply the changes.

### Monitoring WAL Archiving

You can monitor the status of WAL archiving using the `pg_stat_archiver` view:

```sql
SELECT * FROM pg_stat_archiver;
```

### Example Output

- **archived_count**: The number of WAL files that have been successfully archived.
- **last_archived_wal**: The name of the last successfully archived WAL file.
- **last_archived_time**: The timestamp of the last successful archive.
- **failed_count**: The number of failed archive attempts.
- **last_failed_wal**: The name of the last WAL file that failed to archive.
- **last_failed_time**: The timestamp of the last failed archive attempt.

### Summary

The `archive_command` in PostgreSQL is executed under the following circumstances:

1. **WAL File Completion**: When a WAL file reaches its maximum size and is closed.
2. **Checkpoints**: During checkpoints, any incomplete WAL files are archived.
3. **Manual Archiving**: When the `pg_switch_wal()` function is called to force a WAL file switch and archive.

By configuring and monitoring the `archive_command`, you can ensure that WAL files are archived properly, which is essential for data durability and backup purposes.
