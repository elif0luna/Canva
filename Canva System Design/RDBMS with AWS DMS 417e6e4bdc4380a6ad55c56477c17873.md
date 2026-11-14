# RDBMS with AWS DMS

1. **zero-downtime migration:** user generates a usage event → sent to the system through ***ingestion endpoint →*** event stored and data transformation is applied → store output in separate database table.

![image.png](RDBMS%20with%20AWS%20DMS/image.png)

As part of our migration strategy, we built a new service to shadow the existing system. This allowed us to verify that the new system worked correctly:

![image.png](RDBMS%20with%20AWS%20DMS/image%201.png)

migration process now: 

<aside>
🦩

1. Pause data processing in the new service while users continued to read data from the old service database.
2. Take a snapshot of the old service database for the new service.
3. Perform a point-in-time data migration from the old service to the new one.
4. Replay any queued messages and merge them with the migrated point-in-time data.
5. Cut over user traffic to the new service. Users shouldn’t notice any disruption!
</aside>

![image.png](RDBMS%20with%20AWS%20DMS/image%202.png)

- We use **AWS SQS (Simple Queue Service)** for our queues. When we say we're pausing the queue, we mean that our processing handlers (that is, the clients consuming SQS messages) stop consuming. SQS can hold messages for up to 14 days, which gives us enough time to do the point-in-time migration. SQS guarantees at-least-once delivery, so we can trust that no data is lost during this time. At-least-once delivery means that for each message handed to the SQS, potentially multiple attempts are made to deliver it, such that at least one succeeds. Although messages might be duplicated, they’re not lost.
- We also have background tasks (that is, the data processing runtimes) running periodically. These tasks do the heavy-lifting data transformations. We can pause these background tasks using feature flags.
- We consider usages as duplicated when the same user applies the template on the same design. The deduplication process guarantees that the same usage counts only once, so we can count precisely the usage numbers for our Creator templates. For our migration, this helps us remove any duplicate messages received during the time between when we pause processing (step 1) and take a snapshot of the old service (step 2).

**Point-in-time migration tool:**

1. SQL dump and batch inserts: For MySQL databases, we could use [**mysqldump**](https://dev.mysql.com/doc/refman/8.0/en/mysqldump.html) to dump the data into chunks using different primary key and ID ranges depending on our data volume. We could then insert the data into the target database in batches.
2. Trim Source DB: copy the original data → remove unnecessary tables → make changes to database schema and users to ensure compatibility with the new service → take snapshot of trimmed database and restore in new database followed by a switchover.
3. AWS Database Migration Service: It’s a cloud-based service enabling data migration between different types of databases, including RDS instances on AWS. Considering the following features:
    1. Source filters
    2. Truncate tables: The new system’s RDS had some test data and we found that DMS's **full load settings** could automate the safe truncation of tables.
    3. Transformation rules: These could help rename schemas when using different database names in the old and new services.
    4. Validation: For preventing data loss or mutation during migration.
    5. Parallel Loads

We checked some important metrics, like IOPS (input/output operations per second) and CPU usage and discovered that:

- The RDS instance we were migrating to was handling a lot of write operations compared to the one we were moving from.
- The CPU usage was high, which suggested that one of the ‘other things’ the RDS instance was busy with was the indexing process, which was using up a lot of resources.
- After dropping the indexes, the burst balance improved, and the replication tasks ran faster through to completion.
- We upgraded the storage volume of our target RDS to gp3, which allowed us to scale up the IOPS capacity independently of storage size.