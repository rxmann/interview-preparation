# Database Notes: Operations and Security

> Part of the [Database Notes](index.md) series.

---

## 10. Operations and Security

### Connection pooling

Opening a database connection is expensive: TCP handshake, TLS handshake, authentication, and in Postgres a new OS process forked per connection consuming several MB of memory. Beyond memory, more connections mean more contention on internal database structures. Database throughput actually peaks at a small number of active connections (roughly two times the CPU core count) and then declines as contention grows — having more connections just means more waiting inside the database instead of queuing outside it.

A connection pool keeps a set of open connections and lends them to application requests. PgBouncer is the standard external pooler for Postgres — it accepts thousands of client connections while keeping only a small number of real server connections. In transaction mode, a server connection is held only for the duration of a transaction, not the whole session, allowing much higher multiplexing. RDS Proxy is the AWS-managed equivalent, with the added benefit of redirecting connections to the new primary during failover and integrating with IAM authentication.

### Network security

A database should never be directly reachable from the internet. Place it in a private subnet with no route to an internet gateway. Security groups should allow inbound connections only from the application tier's security group on the DB port — not from 0.0.0.0/0 ever. For human access (debugging, migrations), use a bastion host in the public subnet or AWS Systems Manager Session Manager, which requires no open inbound ports.

### Monitoring

The key signals to watch are: slow query log and pg_stat_statements for top queries by total time (calls × mean, not just the slowest individual query), EXPLAIN ANALYZE for diagnosing a specific plan, lock waits and blocking queries, replication lag, active connections vs max_connections, buffer cache hit ratio (should be above 99% for OLTP), dead tuple count and time since last autovacuum, and free disk space. On AWS, Performance Insights breaks down database load by wait event and query, which is the fastest way to find what the DB is spending time on.

### Backups and PITR

Backups protect against human error and ransomware — replication faithfully copies mistakes. Logical backups (pg_dump) are portable and selective but slow for large databases. Physical backups (pg_basebackup) are fast but restore the whole cluster at the same major version. Continuous archiving with WAL shipping allows point-in-time recovery: restore a base backup then replay WAL to any second within the retention window, letting you recover to just before an accidental DROP TABLE. Follow the 3-2-1 rule: three copies, two different media types, one offsite. Test restores on a schedule — a backup that has never been restored is a hope, not a backup.

### Zero-downtime migrations

Large table migrations can lock tables and cause outages. The expand-and-contract pattern avoids this: add the new structure, deploy code that writes to both old and new, backfill existing rows in small batches, switch reads to the new structure, then drop the old in a later release. Create indexes with CREATE INDEX CONCURRENTLY to avoid blocking writes. Always set lock_timeout on migration sessions — an ALTER TABLE that waits for a long-running query will queue all subsequent traffic behind it. Adding NOT NULL to an existing column requires adding a NOT VALID check constraint first (no table scan), then validating it separately (doesn't block writes).

### SQL injection

SQL injection happens when untrusted user input is concatenated into a SQL string, letting an attacker change the query's structure — bypassing authentication, exfiltrating other tables, or destroying data. The only real defense is parameterized queries or prepared statements, which send the SQL structure and the data separately so the database treats the data strictly as a value, never as code. ORMs parameterize automatically but their raw SQL escape hatches are still vulnerable if you concatenate input. Table and column names can't be parameterized — allow-list them explicitly instead.

### Encryption

Encryption at rest (storage-level, e.g., RDS with KMS) protects against stolen hardware but doesn't protect against someone who can query the database with valid credentials. Encryption in transit (TLS) protects data moving between the application and database from eavesdropping — use sslmode=verify-full to both encrypt and verify the server's identity. Application-level encryption (encrypting sensitive fields before storing) means even DBAs and backups only see ciphertext, but you lose the ability to index, sort, or query those fields efficiently.

### Least privilege

The application role should have only SELECT, INSERT, UPDATE, DELETE — never superuser or DDL permissions. Use separate roles for migrations, read-only services, and analytics. Store credentials in a secrets manager with automatic rotation rather than hardcoding them. IAM database authentication (RDS) issues short-lived tokens tied to an IAM role instead of long-lived passwords, reducing the risk of credential exposure.

---

### Gotchas

- PgBouncer transaction mode breaks session-level state like SET, temporary tables, and LISTEN/NOTIFY — always use SET LOCAL inside transactions so settings reset at transaction end.
- Postgres does not automatically index foreign key columns — a missing FK index means deleting a parent row forces a full scan of the child table.
- CDC replication slots that fall behind retain WAL indefinitely; a stopped consumer will eventually fill the disk — monitor and alert on slot lag.
- Never measure backup success by whether the backup ran — measure it by whether a restore succeeded on the last test.
- Input validation and WAFs are defense-in-depth, not substitutes for parameterized queries.
