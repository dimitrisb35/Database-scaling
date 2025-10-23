# Database Scaling: A Comprehensive Guide to Horizontal, Vertical Scaling, Sharding, Partitioning, Backup & Replication

## Introduction

As applications grow, databases become critical bottlenecks that require strategic scaling. This guide explores the fundamental approaches to database scaling, data distribution techniques, backup strategies, and replication patterns. Whether you're running a startup or managing enterprise infrastructure, understanding these concepts is essential for building resilient, high-performance systems.

## Table of Contents

- [Scaling Strategies](#scaling-strategies)
- [Sharding](#sharding)
- [Partitioning](#partitioning)
- [Backup Strategies](#backup-strategies)
- [Replication](#replication)
- [Hybrid Deployments](#hybrid-deployments)
- [Cross-Region Architecture](#cross-region-architecture)
- [Best Practices](#best-practices)

## Scaling Strategies

### Vertical Scaling (Scaling Up)

Vertical scaling involves adding more resources to a single server: more CPU, RAM, storage, or faster disks.

#### Advantages

- **Simple Implementation**: No application changes required
- **Strong Consistency**: Single database maintains ACID properties
- **No Data Distribution**: No complex sharding logic needed
- **Lower Latency**: All data is local, no network hops between nodes

#### Disadvantages

- **Hardware Limits**: Physical constraints on how much you can upgrade
- **Single Point of Failure**: One server failure means complete downtime
- **Expensive**: High-end hardware costs increase exponentially
- **Downtime Required**: Upgrades typically require database restart

#### When to Use

- Applications with strong consistency requirements
- Early-stage applications with growing but manageable data
- Systems where operational complexity must be minimized
- Workloads that don't fit distributed database patterns

#### Example Progression

```
Stage 1: 4 CPU cores, 16GB RAM, 500GB SSD
         ↓
Stage 2: 8 CPU cores, 32GB RAM, 1TB SSD
         ↓
Stage 3: 16 CPU cores, 64GB RAM, 2TB NVMe
         ↓
Stage 4: 32 CPU cores, 128GB RAM, 4TB NVMe
         ↓
Stage 5: Consider horizontal scaling
```

### Horizontal Scaling (Scaling Out)

Horizontal scaling involves adding more servers to distribute the load across multiple machines.

#### Advantages

- **Unlimited Scalability**: Add servers as needed
- **Fault Tolerance**: System continues operating if nodes fail
- **Cost-Effective**: Use commodity hardware instead of expensive servers
- **Geographic Distribution**: Deploy nodes closer to users

#### Disadvantages

- **Complex Implementation**: Requires application changes
- **Eventual Consistency**: CAP theorem trade-offs apply
- **Data Distribution**: Sharding logic adds complexity
- **Network Overhead**: Cross-node communication latency

#### When to Use

- Massive scale requirements (millions of requests/second)
- Geographic distribution needs
- High availability requirements with no single point of failure
- Write-heavy workloads that exceed single-server capacity

#### Architecture Example

```
                    ┌─────────────────┐
                    │  Load Balancer  │
                    └────────┬────────┘
                             │
        ┌────────────────────┼────────────────────┐
        │                    │                    │
        ▼                    ▼                    ▼
   ┌─────────┐         ┌─────────┐         ┌─────────┐
   │  Node 1 │         │  Node 2 │         │  Node 3 │
   │  Shard  │         │  Shard  │         │  Shard  │
   │  (A-H)  │         │  (I-P)  │         │  (Q-Z)  │
   └─────────┘         └─────────┘         └─────────┘
```

### Hybrid Approach

Most production systems use both vertical and horizontal scaling:

1. **Vertical first**: Scale individual nodes to handle more load
2. **Horizontal second**: Add nodes when vertical limits are reached
3. **Optimize continuously**: Tune queries, add caching, use read replicas

## Sharding

Sharding is the process of splitting data across multiple database instances (shards), where each shard contains a subset of the total data.

### Sharding Strategies

#### 1. Range-Based Sharding

Data is distributed based on ranges of a shard key.

```
User IDs 1-1000000    → Shard 1
User IDs 1000001-2000000 → Shard 2
User IDs 2000001-3000000 → Shard 3
```

**Pros:**
- Simple to implement and understand
- Range queries are efficient
- Easy to add new ranges

**Cons:**
- Can lead to uneven distribution (hotspots)
- Older data may be accessed less frequently

#### 2. Hash-Based Sharding

Data is distributed using a hash function on the shard key.

```python
shard_number = hash(user_id) % number_of_shards
```

**Pros:**
- Even distribution of data
- Prevents hotspots
- Simple algorithm

**Cons:**
- Range queries require checking all shards
- Rebalancing requires rehashing all data
- Adding shards is complex

#### 3. Geographic Sharding

Data is distributed based on geographic location.

```
Users in North America → US-East Shard
Users in Europe       → EU-West Shard
Users in Asia         → Asia-Pacific Shard
```

**Pros:**
- Reduced latency for users
- Compliance with data residency requirements
- Natural data isolation

**Cons:**
- Uneven distribution based on user geography
- Complex cross-region queries
- Requires geographic metadata

#### 4. Directory-Based Sharding

A lookup table maintains mapping between data and shards.

```
┌─────────────────────────────────┐
│      Shard Directory/Lookup     │
├──────────────┬──────────────────┤
│  Customer A  │  Shard 1         │
│  Customer B  │  Shard 2         │
│  Customer C  │  Shard 1         │
│  Customer D  │  Shard 3         │
└──────────────┴──────────────────┘
```

**Pros:**
- Flexible data distribution
- Easy to rebalance
- Can implement custom logic

**Cons:**
- Additional lookup overhead
- Directory becomes single point of failure
- Extra complexity

### Implementing Sharding

#### Application-Level Sharding

```python
class ShardedDatabase:
    def __init__(self, shards):
        self.shards = shards
    
    def get_shard(self, user_id):
        shard_index = hash(user_id) % len(self.shards)
        return self.shards[shard_index]
    
    def get_user(self, user_id):
        shard = self.get_shard(user_id)
        return shard.query(f"SELECT * FROM users WHERE id = {user_id}")
    
    def create_user(self, user_data):
        shard = self.get_shard(user_data['id'])
        return shard.insert("users", user_data)
```

#### Database-Level Sharding

Many modern databases provide built-in sharding:

- **MongoDB**: Automatic sharding with configurable shard keys
- **PostgreSQL**: Using Citus extension or pg_partman
- **MySQL**: MySQL Cluster or ProxySQL
- **Cassandra**: Consistent hashing built-in
- **CockroachDB**: Automatic range-based sharding

### Challenges with Sharding

- **Cross-Shard Queries**: Joins across shards are expensive
- **Transactions**: Distributed transactions are complex (2PC, Saga pattern)
- **Rebalancing**: Adding/removing shards requires data migration
- **Operational Complexity**: Multiple databases to manage and monitor
- **Schema Changes**: Must be coordinated across all shards

## Partitioning

Partitioning divides a table into smaller, more manageable pieces within a single database instance. Unlike sharding, partitions remain on the same server.

### Types of Partitioning

#### 1. Horizontal Partitioning (Row-Based)

Split table rows into multiple partitions based on column values.

```sql
-- PostgreSQL example: Partition by date range
CREATE TABLE orders (
    order_id SERIAL,
    order_date DATE,
    customer_id INT,
    amount DECIMAL
) PARTITION BY RANGE (order_date);

CREATE TABLE orders_2024_q1 PARTITION OF orders
    FOR VALUES FROM ('2024-01-01') TO ('2024-04-01');

CREATE TABLE orders_2024_q2 PARTITION OF orders
    FOR VALUES FROM ('2024-04-01') TO ('2024-07-01');
```

#### 2. Vertical Partitioning (Column-Based)

Split table columns into separate tables.

```sql
-- Main table with frequently accessed columns
CREATE TABLE users (
    user_id INT PRIMARY KEY,
    username VARCHAR(50),
    email VARCHAR(100),
    created_at TIMESTAMP
);

-- Separate table for infrequently accessed columns
CREATE TABLE user_profiles (
    user_id INT PRIMARY KEY,
    bio TEXT,
    preferences JSONB,
    last_login TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(user_id)
);
```

#### 3. List Partitioning

Partition by discrete values.

```sql
CREATE TABLE sales (
    sale_id SERIAL,
    region VARCHAR(50),
    amount DECIMAL
) PARTITION BY LIST (region);

CREATE TABLE sales_north PARTITION OF sales
    FOR VALUES IN ('US-EAST', 'US-WEST', 'CANADA');

CREATE TABLE sales_europe PARTITION OF sales
    FOR VALUES IN ('UK', 'GERMANY', 'FRANCE');
```

#### 4. Hash Partitioning

Partition using a hash function for even distribution.

```sql
CREATE TABLE events (
    event_id BIGSERIAL,
    event_type VARCHAR(50),
    payload JSONB
) PARTITION BY HASH (event_id);

CREATE TABLE events_p0 PARTITION OF events
    FOR VALUES WITH (MODULUS 4, REMAINDER 0);

CREATE TABLE events_p1 PARTITION OF events
    FOR VALUES WITH (MODULUS 4, REMAINDER 1);
```

### Benefits of Partitioning

- **Query Performance**: Partition pruning reduces scanned data
- **Maintenance**: Easier to archive, backup, or drop old data
- **Parallel Processing**: Different partitions can be processed simultaneously
- **Index Size**: Smaller indexes per partition improve cache efficiency
- **Lock Contention**: Operations on one partition don't lock others

### Partitioning vs Sharding

| Aspect | Partitioning | Sharding |
|--------|--------------|----------|
| **Scope** | Single database instance | Multiple database instances |
| **Scaling** | Vertical scaling primarily | Horizontal scaling |
| **Complexity** | Lower complexity | Higher complexity |
| **Transactions** | Full ACID support | Distributed transactions required |
| **Failover** | Shared fate with database | Independent shard failures |
| **When to use** | Improve query performance, maintenance | Scale beyond single server capacity |

## Backup Strategies

### Types of Backups

#### 1. Full Backup

Complete copy of the entire database.

```bash
# PostgreSQL full backup
pg_dump -U postgres -F c -f /backup/full_backup.dump database_name

# MySQL full backup
mysqldump -u root -p --all-databases > /backup/full_backup.sql

# MongoDB full backup
mongodump --out=/backup/full_backup --db=mydb
```

**Pros:** Simple restore process, complete data snapshot  
**Cons:** Time-consuming, large storage requirements

#### 2. Incremental Backup

Only backs up data changed since the last backup.

```bash
# PostgreSQL WAL archiving (continuous incremental)
archive_command = 'cp %p /backup/wal_archive/%f'

# MySQL incremental with binary logs
mysqlbinlog --start-datetime="2024-01-01" mysql-bin.000001 > incremental.sql
```

**Pros:** Faster backups, less storage space  
**Cons:** Complex restore (need full + all incrementals), longer restore time

#### 3. Differential Backup

Backs up changes since the last full backup.

**Pros:** Faster restore than incremental (need only full + last differential)  
**Cons:** Larger than incremental backups

#### 4. Continuous Backup (Point-in-Time Recovery)

Real-time backup of transaction logs.

```sql
-- PostgreSQL PITR setup
wal_level = replica
archive_mode = on
archive_command = 'test ! -f /backup/wal/%f && cp %p /backup/wal/%f'

-- Restore to specific point in time
restore_command = 'cp /backup/wal/%f %p'
recovery_target_time = '2024-10-23 14:30:00'
```

**Pros:** Minimal data loss, flexible recovery points  
**Cons:** Requires log shipping infrastructure, storage for logs

### Backup Storage Strategies

#### Local Backup

```bash
# Simple cron job for daily backups
0 2 * * * /usr/bin/pg_dump -U postgres mydb > /backup/daily/mydb_$(date +\%Y\%m\%d).sql
```

**Pros:** Fast backup and restore, simple setup  
**Cons:** No disaster recovery if server fails

#### Cloud Backup

```bash
# Backup to AWS S3
pg_dump -U postgres mydb | gzip | aws s3 cp - s3://my-backups/mydb_$(date +\%Y\%m\%d).sql.gz

# Backup to Google Cloud Storage
pg_dump -U postgres mydb | gzip | gsutil cp - gs://my-backups/mydb_$(date +\%Y\%m\%d).sql.gz
```

**Pros:** Disaster recovery, scalable storage, geographic redundancy  
**Cons:** Network bandwidth requirements, costs

#### 3-2-1 Backup Rule

- **3** copies of data
- **2** different storage media types
- **1** copy offsite

```
Primary Database (Production server)
    ↓
Local Backup (Same datacenter, different disk)
    ↓
Cloud Backup (AWS S3 / Google Cloud Storage)
    ↓
Archive Storage (Glacier / Coldline for long-term retention)
```

### Backup Best Practices

```bash
#!/bin/bash
# Comprehensive backup script

BACKUP_DIR="/backup"
DB_NAME="production_db"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
RETENTION_DAYS=30

# Create backup with compression
pg_dump -U postgres -F c -Z 9 $DB_NAME > $BACKUP_DIR/${DB_NAME}_${TIMESTAMP}.dump

# Verify backup integrity
pg_restore --list $BACKUP_DIR/${DB_NAME}_${TIMESTAMP}.dump > /dev/null 2>&1
if [ $? -eq 0 ]; then
    echo "Backup successful: ${DB_NAME}_${TIMESTAMP}.dump"
    
    # Upload to cloud
    aws s3 cp $BACKUP_DIR/${DB_NAME}_${TIMESTAMP}.dump s3://my-backups/
    
    # Remove old backups
    find $BACKUP_DIR -name "${DB_NAME}_*.dump" -mtime +$RETENTION_DAYS -delete
else
    echo "Backup verification failed!" | mail -s "Backup Alert" admin@example.com
fi
```

## Replication

Replication creates copies of data across multiple database servers for high availability and read scalability.

### Replication Types

#### 1. Primary-Replica (Master-Slave) Replication

One primary database handles writes, replicas handle reads.

```
                    ┌─────────────┐
                    │   Primary   │
                    │  (Writes)   │
                    └──────┬──────┘
                           │
              ┌────────────┼────────────┐
              │            │            │
              ▼            ▼            ▼
         ┌─────────┐  ┌─────────┐  ┌─────────┐
         │Replica 1│  │Replica 2│  │Replica 3│
         │ (Reads) │  │ (Reads) │  │ (Reads) │
         └─────────┘  └─────────┘  └─────────┘
```

**PostgreSQL Setup:**

```sql
-- On primary server
-- postgresql.conf
wal_level = replica
max_wal_senders = 3
wal_keep_size = 1GB

-- Create replication user
CREATE ROLE replicator WITH REPLICATION LOGIN PASSWORD 'secure_password';
```

```sql
-- On replica server
-- Set up standby
pg_basebackup -h primary_host -D /var/lib/postgresql/data -U replicator -P

-- standby.signal file indicates this is a replica
touch /var/lib/postgresql/data/standby.signal

-- postgresql.conf
primary_conninfo = 'host=primary_host port=5432 user=replicator password=secure_password'
```

**MySQL Setup:**

```sql
-- On primary server
-- my.cnf
[mysqld]
server-id = 1
log_bin = /var/log/mysql/mysql-bin.log
binlog_do_db = mydb

-- Create replication user
CREATE USER 'replicator'@'%' IDENTIFIED BY 'password';
GRANT REPLICATION SLAVE ON *.* TO 'replicator'@'%';
SHOW MASTER STATUS; -- Note file and position
```

```sql
-- On replica server
-- my.cnf
[mysqld]
server-id = 2
relay-log = /var/log/mysql/mysql-relay-bin

-- Configure replication
CHANGE MASTER TO
    MASTER_HOST='primary_host',
    MASTER_USER='replicator',
    MASTER_PASSWORD='password',
    MASTER_LOG_FILE='mysql-bin.000001',
    MASTER_LOG_POS=12345;

START SLAVE;
```

#### 2. Multi-Primary (Multi-Master) Replication

Multiple databases accept writes simultaneously.

```
     ┌─────────────┐          ┌─────────────┐
     │  Primary 1  │◄────────►│  Primary 2  │
     │  (R/W)      │          │  (R/W)      │
     └─────────────┘          └─────────────┘
           ▲                         ▲
           │                         │
           └────────────┬────────────┘
                        │
                        ▼
                 ┌─────────────┐
                 │  Primary 3  │
                 │  (R/W)      │
                 └─────────────┘
```

**Pros:** High availability, write scalability, no single point of failure  
**Cons:** Conflict resolution complexity, eventual consistency

**PostgreSQL BDR (Bi-Directional Replication):**

```sql
-- Install pglogical extension
CREATE EXTENSION pglogical;

-- On first node
SELECT pglogical.create_node(
    node_name := 'node1',
    dsn := 'host=node1.example.com port=5432 dbname=mydb'
);

-- On second node
SELECT pglogical.create_node(
    node_name := 'node2',
    dsn := 'host=node2.example.com port=5432 dbname=mydb'
);

SELECT pglogical.create_subscription(
    subscription_name := 'subscription1',
    provider_dsn := 'host=node1.example.com port=5432 dbname=mydb'
);
```

#### 3. Synchronous vs Asynchronous Replication

**Synchronous Replication:**

```sql
-- PostgreSQL synchronous replication
synchronous_commit = on
synchronous_standby_names = 'replica1,replica2'
```

- Primary waits for replica acknowledgment before committing
- **Pros:** No data loss, strong consistency
- **Cons:** Higher latency, availability depends on replicas

**Asynchronous Replication:**

```sql
-- PostgreSQL asynchronous replication (default)
synchronous_commit = off
```

- Primary commits without waiting for replicas
- **Pros:** Lower latency, better performance
- **Cons:** Potential data loss during failures

#### 4. Cascading Replication

Replicas can have their own replicas.

```
        ┌─────────────┐
        │   Primary   │
        └──────┬──────┘
               │
        ┌──────┴──────┐
        │             │
        ▼             ▼
   ┌─────────┐   ┌─────────┐
   │Replica 1│   │Replica 2│
   └────┬────┘   └─────────┘
        │
        ▼
   ┌─────────┐
   │Replica 3│
   └─────────┘
```

Reduces load on primary for many replicas.

### Replication Lag Monitoring

```sql
-- PostgreSQL: Check replication lag
SELECT 
    client_addr,
    state,
    sent_lsn,
    write_lsn,
    flush_lsn,
    replay_lsn,
    sync_state,
    pg_wal_lsn_diff(sent_lsn, replay_lsn) AS lag_bytes
FROM pg_stat_replication;

-- MySQL: Check replica lag
SHOW SLAVE STATUS\G
-- Look at Seconds_Behind_Master field
```

### Failover Strategies

#### Automatic Failover

Using tools like:
- **Patroni** (PostgreSQL)
- **Orchestrator** (MySQL)
- **ProxySQL** with monitoring
- **Stolon** (PostgreSQL)

```yaml
# Patroni configuration example
scope: postgres-cluster
name: node1

restapi:
  listen: 0.0.0.0:8008

etcd:
  host: localhost:2379

bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 1048576
    
postgresql:
  data_dir: /var/lib/postgresql/data
  parameters:
    max_connections: 100
    shared_buffers: 256MB
```

## Hybrid Deployments

Hybrid deployments combine on-premises and cloud infrastructure for flexibility, compliance, and cost optimization.

### Architecture Patterns

#### 1. Primary On-Prem, Replica in Cloud

```
┌─────────────────────────────────────────┐
│         On-Premises Datacenter          │
│                                         │
│  ┌─────────────────────────────────┐    │
│  │  Primary Database (Production)  │    │
│  └──────────────┬──────────────────┘    │
└─────────────────┼───────────────────────┘
                  │
                  │ Replication
                  │
┌─────────────────▼───────────────────────┐
│           Cloud (AWS/Azure/GCP)         │
│                                         │
│  ┌─────────────────────────────────┐    │
│  │   Replica Database (DR/Read)    │    │
│  └─────────────────────────────────┘    │
│                                         │
│  ┌─────────────────────────────────┐    │
│  │   Analytics/Reporting Queries   │    │
│  └─────────────────────────────────┘    │
└─────────────────────────────────────────┘
```

**Use Cases:**
- Disaster recovery
- Compliance with data residency
- Cloud-based analytics without impacting production
- Gradual cloud migration

#### 2. Primary in Cloud, Replica On-Prem

```
┌─────────────────────────────────────────┐
│           Cloud (Primary)               │
│                                         │
│  ┌─────────────────────────────────┐   │
│  │   Primary Database              │   │
│  └──────────────┬──────────────────┘   │
└─────────────────┼───────────────────────┘
                  │
                  │ Replication
                  │
┌─────────────────▼───────────────────────┐
│         On-Premises Datacenter          │
│                                         │
│  ┌─────────────────────────────────┐    │
│  │   Replica Database (Backup)     │    │
│  └─────────────────────────────────┘    │
└─────────────────────────────────────────┘
```

**Use Cases:**
- Cloud-first strategy with on-prem backup
- Low-latency local reads
- Regulatory backup requirements

#### 3. Multi-Cloud Hybrid

```
┌─────────────────┐     ┌─────────────────┐
│   AWS Region    │     │   Azure Region  │
│                 │     │                 │
│  ┌──────────┐   │     │  ┌──────────┐   │
│  │Primary DB│◄─ ┼─────┼─►│Replica DB│   │
│  └──────────┘   │     │  └──────────┘   │
└────────┬────────┘     └─────────────────┘
         │
         │ Replication
         │
┌────────▼────────┐
│  On-Premises    │
│                 │
│  ┌──────────┐   │
│  │Replica DB│   │
│  └──────────┘   │
└─────────────────┘
```

### Challenges and Solutions

#### Network Connectivity

**Challenge:** Reliable, secure connection between on-prem and cloud

**Solutions:**
- VPN tunnels (IPSec, WireGuard)
- Dedicated connections (AWS Direct Connect, Azure ExpressRoute, GCP Interconnect)
- SD-WAN for multi-cloud

```bash
# Example: Set up WireGuard VPN
# On-prem server
[Interface]
PrivateKey = <on-prem-private-key>
Address = 10.0.0.1/24
ListenPort = 51820

[Peer]
PublicKey = <cloud-public-key>
Endpoint = cloud-ip:51820
AllowedIPs = 10.0.1.0/24
PersistentKeepalive = 25
```

#### Latency and Bandwidth

**Challenge:** Replication lag due to distance and network limitations

**Solutions:**
- Asynchronous replication for non-critical data
- Compress replication traffic
- Use CDN or edge caching for read queries
- Batch replication during off-peak hours

```sql
-- PostgreSQL compression
wal_compression = on
```

#### Data Consistency

**Challenge:** Ensuring data consistency across environments

**Solutions:**
- Monitoring replication lag
- Automated failover with health checks
- Conflict resolution policies for multi-master

```python
# Example monitoring script
import psycopg2

def check_replication_lag():
    conn = psycopg2.connect("dbname=postgres user=monitor")
    cur = conn.cursor()
    
    cur.execute("""
        SELECT pg_wal_lsn_diff(sent_lsn, replay_lsn) AS lag_bytes
        FROM pg_stat_replication
        WHERE application_name = 'cloud_replica'
    """)
    
    lag = cur.fetchone()[0]
    if lag > 10485760:  # 10MB threshold
        alert_ops_team(f"Replication lag: {lag} bytes")
    
    conn.close()
```

#### Security

**Challenge:** Securing data in transit and at rest across environments

**Solutions:**
- TLS/SSL for all database connections
- Encryption at rest
- Network segmentation
- IAM and RBAC policies

```sql
-- PostgreSQL SSL configuration
ssl = on
ssl_cert_file = '/path/to/server.crt'
ssl_key_file = '/path/to/server.key'
ssl_ca_file = '/path/to/ca.crt'
```

## Cross-Region Architecture

Deploying databases across geographic regions provides low latency for global users and disaster recovery.

### Single-Region Architecture

```
        ┌───────────────────────────┐
        │      US-East-1 Region     │
        │                           │
        │  ┌──────────────────┐     │
        │  │  Primary DB      │     │
        │  └────────┬─────────┘     │
        │           │               │
        │     ┌─────┴─────┐         │
        │     │           │         │
        │  ┌──▼────┐   ┌──▼────┐    │
        │  │Replica│   │Replica│    │
        │  └───────┘   └───────┘    │
        └───────────────────────────┘
```

**Pros:** Simple, low latency within region, strong consistency  
**Cons:** Single point of failure (region outage), high latency for distant users

### Active-Passive Cross-Region

```
┌───────────────────────────┐       ┌───────────────────────────┐
│    US-East (Primary)      │       │    EU-West (Standby)      │
│                           │       │                           │
│  ┌──────────────────┐     │       │  ┌──────────────────┐     │
│  │  Primary DB      │─────┼───────┼─►│  Replica DB      │     │
│  │  (Active)        │     │       │  │  (Passive)       │     │
│  └──────────────────┘     │       │  └──────────────────┘     │
│                           │       │                           │
│  ┌──────────────────┐     │       │                           │
│  │  Application     │     │       │                           │
│  └──────────────────┘     │       │                           │
└───────────────────────────┘       └───────────────────────────┘
```

**Use Case:** Disaster recovery with minimal RTO/RPO

**Failover Process:**
1. Detect primary failure
2. Promote replica to primary
3. Redirect application traffic
4. Update DNS or load balancer

```bash
# Promote PostgreSQL replica to primary
pg_ctl promote -D /var/lib/postgresql/data

# Or using Patroni
patronictl failover postgres-cluster
```

### Active-Active Cross-Region

```
┌───────────────────────────┐       ┌───────────────────────────┐
│      US-East Region       │       │      EU-West Region       │
│                           │       │                           │
│  ┌──────────────────┐     │       │  ┌──────────────────┐     │
│  │  Primary DB      │◄────┼───────┼─►│  Primary DB      │     │
│  │  (Active)        │     │       │  │  (Active)        │     │
│  └────────┬─────────┘     │       │  └────────┬─────────┘     │
│           │               │       │           │               │
│  ┌────────▼─────────┐     │       │  ┌────────▼─────────┐     │
│  │  Application     │     │       │  │  Application     │     │
│  └──────────────────┘     │       │  └──────────────────┘     │
└───────────────────────────┘       └───────────────────────────┘
```

**Use Case:** Global application with local writes and reads

**Challenges:**
- Conflict resolution for concurrent writes
- Network latency between regions
- Eventual consistency trade-offs

**Databases Supporting Active-Active:**
- CockroachDB
- MongoDB (with sharding)
- Cassandra
- PostgreSQL (with BDR/pglogical)
- MySQL Group Replication

### Regional Sharding

Route users to nearest region with data sharded by geography.

```
                    ┌─────────────────┐
                    │  Global Router  │
                    │  (GeoDNS/CDN)   │
                    └────────┬────────┘
                             │
        ┌────────────────────┼────────────────────┐
        │                    │                    │
        ▼                    ▼                    ▼
┌───────────────┐    ┌───────────────┐    ┌───────────────┐
│   US Region   │    │   EU Region   │    │  APAC Region  │
│               │    │               │    │               │
│ ┌───────────┐ │    │ ┌───────────┐ │    │ ┌────────────┐│
│ │US Users DB│ │    │ │EU Users DB│ │    │ │APAC UsersDB││
│ └───────────┘ │    │ └───────────┘ │    │ └────────────┘│
└───────────────┘    └───────────────┘    └───────────────┘
```

**Implementation Example:**

```python
# GeoDNS routing logic
def get_database_connection(user_location):
    region_map = {
        'US': 'us-east-db.example.com',
        'EU': 'eu-west-db.example.com',
        'APAC': 'ap-south-db.example.com'
    }
    
    db_host = region_map.get(user_location, 'us-east-db.example.com')
    return connect_to_database(db_host)
```

### CRDT-Based Replication

Conflict-free Replicated Data Types ensure eventual consistency without conflicts.

**Example Databases:**
- Riak
- Redis with CRDTs module
- AntidoteDB

```javascript
// CRDT counter example
{
  "node1": {"increments": 5, "decrements": 2},
  "node2": {"increments": 3, "decrements": 1},
  // Merge: (5+3) - (2+1) = 5
  "merged_value": 5
}
```

### Cross-Region Best Practices

1. **Measure and Monitor Latency**
```sql
-- Monitor cross-region replication lag
SELECT 
    application_name,
    client_addr,
    pg_wal_lsn_diff(sent_lsn, replay_lsn) / 1024 / 1024 AS lag_mb,
    extract(epoch from (now() - pg_last_xact_replay_timestamp())) AS lag_seconds
FROM pg_stat_replication;
```

2. **Implement Circuit Breakers**
```python
from pybreaker import CircuitBreaker

db_breaker = CircuitBreaker(fail_max=5, timeout_duration=60)

@db_breaker
def query_remote_database(query):
    return remote_db.execute(query)
```

3. **Use Read Replicas in Each Region**
```
US Region: Primary + 2 Read Replicas
EU Region: Replica (from US) + 2 Read Replicas (from EU replica)
```

4. **Implement Caching Layers**
```python
import redis

# Regional cache
cache = redis.Redis(host='local-redis.region', port=6379)

def get_user_data(user_id):
    # Try cache first
    cached = cache.get(f"user:{user_id}")
    if cached:
        return cached
    
    # Fall back to database
    data = db.query(f"SELECT * FROM users WHERE id = {user_id}")
    cache.setex(f"user:{user_id}", 3600, data)
    return data
```

## Best Practices

### Capacity Planning

#### 1. Establish Baselines

```sql
-- PostgreSQL: Monitor current database size
SELECT 
    pg_size_pretty(pg_database_size(current_database())) AS database_size;

-- Monitor table sizes
SELECT 
    schemaname,
    tablename,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS size
FROM pg_tables
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC
LIMIT 10;

-- Monitor query performance
SELECT 
    query,
    calls,
    total_exec_time,
    mean_exec_time,
    max_exec_time
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 20;
```

#### 2. Project Growth

```python
# Simple growth projection
import pandas as pd
from datetime import datetime, timedelta

def project_database_growth(current_size_gb, daily_growth_gb, days=365):
    dates = [datetime.now() + timedelta(days=x) for x in range(days)]
    sizes = [current_size_gb + (daily_growth_gb * x) for x in range(days)]
    
    df = pd.DataFrame({'date': dates, 'size_gb': sizes})
    
    # Find when you'll hit capacity
    capacity_gb = 1000  # Your current capacity
    if df['size_gb'].max() > capacity_gb:
        exceed_date = df[df['size_gb'] > capacity_gb].iloc[0]['date']
        print(f"Will exceed capacity on: {exceed_date}")
    
    return df
```

#### 3. Set Up Alerts

```yaml
# Prometheus alerting rules
groups:
  - name: database_alerts
    rules:
      - alert: DatabaseDiskSpaceRunningOut
        expr: node_filesystem_avail_bytes{mountpoint="/var/lib/postgresql"} / node_filesystem_size_bytes < 0.1
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Database disk space running out"
          
      - alert: ReplicationLagHigh
        expr: pg_replication_lag_seconds > 300
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Replication lag exceeds 5 minutes"
          
      - alert: DatabaseConnectionsHigh
        expr: pg_stat_database_numbackends / pg_settings_max_connections > 0.8
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Database connections above 80%"
```

### Query Optimization

#### 1. Index Strategy

```sql
-- Identify missing indexes
SELECT 
    schemaname,
    tablename,
    seq_scan,
    seq_tup_read,
    idx_scan,
    seq_tup_read / seq_scan AS avg_seq_read
FROM pg_stat_user_tables
WHERE seq_scan > 0
ORDER BY seq_tup_read DESC
LIMIT 20;

-- Create appropriate indexes
CREATE INDEX CONCURRENTLY idx_users_email ON users(email);
CREATE INDEX CONCURRENTLY idx_orders_user_date ON orders(user_id, order_date);

-- Partial indexes for specific queries
CREATE INDEX idx_active_users ON users(created_at) 
WHERE status = 'active';
```

#### 2. Query Analysis

```sql
-- Enable query analysis
EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON) 
SELECT u.username, COUNT(o.id) as order_count
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
WHERE u.created_at > '2024-01-01'
GROUP BY u.username
HAVING COUNT(o.id) > 10;

-- Identify slow queries
SELECT 
    substring(query, 1, 100) AS short_query,
    round(total_exec_time::numeric, 2) AS total_time,
    calls,
    round(mean_exec_time::numeric, 2) AS mean_time,
    round((100 * total_exec_time / sum(total_exec_time) OVER ())::numeric, 2) AS percentage
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 20;
```

### Connection Pooling

```python
# Using connection pool (psycopg2)
from psycopg2 import pool

connection_pool = pool.SimpleConnectionPool(
    minconn=1,
    maxconn=20,
    host='localhost',
    database='mydb',
    user='myuser',
    password='mypassword'
)

def execute_query(query):
    conn = connection_pool.getconn()
    try:
        cursor = conn.cursor()
        cursor.execute(query)
        result = cursor.fetchall()
        return result
    finally:
        connection_pool.putconn(conn)
```

**Using PgBouncer:**

```ini
# pgbouncer.ini
[databases]
mydb = host=localhost port=5432 dbname=mydb

[pgbouncer]
listen_addr = 0.0.0.0
listen_port = 6432
auth_type = md5
auth_file = /etc/pgbouncer/userlist.txt
pool_mode = transaction
max_client_conn = 1000
default_pool_size = 20
reserve_pool_size = 5
reserve_pool_timeout = 3
```

### Monitoring and Observability

```python
# Example monitoring script using multiple data sources
import psycopg2
import time
from prometheus_client import Gauge, start_http_server

# Define metrics
db_connections = Gauge('db_connections', 'Number of database connections')
replication_lag = Gauge('replication_lag_seconds', 'Replication lag in seconds')
db_size = Gauge('db_size_bytes', 'Database size in bytes')

def collect_metrics():
    conn = psycopg2.connect("dbname=postgres user=monitor")
    cur = conn.cursor()
    
    # Connection count
    cur.execute("SELECT count(*) FROM pg_stat_activity")
    db_connections.set(cur.fetchone()[0])
    
    # Replication lag
    cur.execute("""
        SELECT EXTRACT(EPOCH FROM (now() - pg_last_xact_replay_timestamp()))
        FROM pg_stat_replication
    """)
    result = cur.fetchone()
    if result:
        replication_lag.set(result[0] or 0)
    
    # Database size
    cur.execute("SELECT pg_database_size(current_database())")
    db_size.set(cur.fetchone()[0])
    
    conn.close()

# Start metrics server
start_http_server(8000)

# Collect metrics every 30 seconds
while True:
    collect_metrics()
    time.sleep(30)
```

### Testing Strategies

#### 1. Load Testing

```python
# Using locust for database load testing
from locust import User, task, between
import psycopg2

class DatabaseUser(User):
    wait_time = between(1, 3)
    
    def on_start(self):
        self.conn = psycopg2.connect(
            "dbname=testdb user=testuser password=testpass"
        )
    
    @task(3)
    def read_query(self):
        cursor = self.conn.cursor()
        cursor.execute("SELECT * FROM users WHERE id = %s", (random.randint(1, 10000),))
        cursor.fetchall()
    
    @task(1)
    def write_query(self):
        cursor = self.conn.cursor()
        cursor.execute("INSERT INTO logs (message) VALUES (%s)", (f"Test message {time.time()}",))
        self.conn.commit()
```

#### 2. Failover Testing

```bash
#!/bin/bash
# Chaos engineering script for database failover

echo "Starting failover test..."

# Kill primary database
docker stop postgres-primary

# Wait for failover
sleep 10

# Verify replica promoted
docker exec postgres-replica psql -U postgres -c "SELECT pg_is_in_recovery();"

# Check application connectivity
curl -f http://app-server/health || echo "Application health check failed!"

# Measure recovery time
echo "Failover completed in ${SECONDS} seconds"
```

### Security Best Practices

```sql
-- 1. Use SSL/TLS
ssl = on
ssl_cert_file = '/etc/ssl/certs/server.crt'
ssl_key_file = '/etc/ssl/private/server.key'

-- 2. Implement row-level security
CREATE POLICY user_data_isolation ON user_data
    FOR ALL
    TO app_user
    USING (user_id = current_setting('app.current_user_id')::int);

-- 3. Encrypt sensitive columns
CREATE EXTENSION pgcrypto;

INSERT INTO users (email, password) 
VALUES ('user@example.com', crypt('password123', gen_salt('bf')));

-- Verify password
SELECT * FROM users 
WHERE email = 'user@example.com' 
  AND password = crypt('password123', password);

-- 4. Regular security audits
SELECT 
    usename,
    valuntil,
    usesuper,
    usecreatedb
FROM pg_user
WHERE valuntil < now() OR valuntil IS NULL;
```

## Cost Optimization

### Storage Optimization

```sql
-- Identify bloated tables
SELECT 
    schemaname,
    tablename,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS size,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename) - pg_relation_size(schemaname||'.'||tablename)) AS external_size
FROM pg_tables
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC;

-- Vacuum and analyze
VACUUM FULL ANALYZE users;

-- Archive old data
CREATE TABLE orders_archive (LIKE orders INCLUDING ALL);

INSERT INTO orders_archive 
SELECT * FROM orders 
WHERE order_date < '2023-01-01';

DELETE FROM orders 
WHERE order_date < '2023-01-01';
```

### Compute Optimization

```yaml
# Right-size database instances based on usage
# Example AWS RDS instance sizing decision tree

If: Average CPU < 20% AND Average Memory < 30%
  Action: Downgrade instance size
  
If: Average CPU > 80% OR Average Memory > 90%
  Action: Upgrade instance size OR scale horizontally
  
If: Peak CPU > 90% but Average CPU < 40%
  Action: Consider adding read replicas instead of upgrading
```

### Reserved Capacity

```python
# Calculate savings with reserved instances
def calculate_reserved_savings(
    on_demand_hourly_cost,
    reserved_upfront_cost,
    reserved_hourly_cost,
    months=12
):
    hours = months * 730  # Average hours per month
    
    on_demand_total = on_demand_hourly_cost * hours
    reserved_total = reserved_upfront_cost + (reserved_hourly_cost * hours)
    
    savings = on_demand_total - reserved_total
    savings_percent = (savings / on_demand_total) * 100
    
    return {
        'on_demand_cost': on_demand_total,
        'reserved_cost': reserved_total,
        'savings': savings,
        'savings_percent': savings_percent
    }

# Example
result = calculate_reserved_savings(
    on_demand_hourly_cost=1.50,
    reserved_upfront_cost=5000,
    reserved_hourly_cost=0.75,
    months=12
)
print(f"Annual savings: ${result['savings']:.2f} ({result['savings_percent']:.1f}%)")
```

## Decision Matrix

### When to Use Each Strategy

| Requirement | Recommended Approach |
|-------------|---------------------|
| **Growing dataset (< 1TB)** | Vertical scaling + Partitioning |
| **Read-heavy workload** | Primary-Replica replication + Read replicas |
| **Write-heavy workload** | Sharding + Multiple primaries |
| **Global user base** | Cross-region replication + Geographic sharding |
| **High availability (99.99%)** | Multi-AZ deployment + Automatic failover |
| **Disaster recovery** | Cross-region async replication + Regular backups |
| **Compliance requirements** | Hybrid deployment + On-prem primary |
| **Cost optimization** | Appropriate instance sizing + Storage tiering |
| **Low latency (<50ms)** | Regional deployment + Caching + Read replicas |
| **Massive scale (>10TB)** | Horizontal sharding + Distributed database |

## Common Pitfalls and Solutions

### Pitfall 1: Over-Engineering Too Early

**Problem:** Implementing complex sharding when vertical scaling would suffice

**Solution:** Start simple, scale as needed
```
1. Single database instance
2. Add read replicas
3. Implement caching
4. Partition tables
5. Finally, consider sharding
```

### Pitfall 2: Ignoring Replication Lag

**Problem:** Read replicas serving stale data

**Solution:** Monitor and handle lag
```python
def read_with_lag_tolerance(query, max_lag_seconds=5):
    lag = check_replication_lag()
    
    if lag > max_lag_seconds:
        # Read from primary for consistency
        return primary_db.execute(query)
    else:
        # Use replica for performance
        return replica_db.execute(query)
```

### Pitfall 3: No Backup Testing

**Problem:** Backups exist but restore procedures are untested

**Solution:** Regular backup drills
```bash
#!/bin/bash
# Monthly backup restore test

# Restore to test environment
pg_restore -d test_restore /backups/latest.dump

# Verify data integrity
psql -d test_restore -c "SELECT COUNT(*) FROM users;"

# Test application against restored database
./run_integration_tests.sh --database=test_restore

# Document results
echo "Backup restore test completed successfully" >> /var/log/backup_tests.log
```

### Pitfall 4: Insufficient Capacity Planning

**Problem:** Running out of resources unexpectedly

**Solution:** Proactive monitoring and alerting
```sql
-- Create capacity planning view
CREATE VIEW capacity_metrics AS
SELECT 
    'connections' AS metric,
    count(*) AS current_value,
    (SELECT setting::int FROM pg_settings WHERE name = 'max_connections') AS max_value,
    (count(*)::float / (SELECT setting::int FROM pg_settings WHERE name = 'max_connections')) * 100 AS usage_percent
FROM pg_stat_activity
UNION ALL
SELECT 
    'disk_space',
    pg_database_size(current_database()),
    pg_tablespace_size('pg_default') AS max_value,
    (pg_database_size(current_database())::float / pg_tablespace_size('pg_default')) * 100
;
```

## Conclusion

Database scaling is a journey that evolves with your application's needs. The key principles to remember:

1. **Start Simple**: Don't over-engineer. Vertical scaling and read replicas can take you far.

2. **Monitor Everything**: You can't optimize what you don't measure. Implement comprehensive monitoring from day one.

3. **Plan for Failure**: Failures will happen. Design your architecture to handle node failures, network partitions, and region outages gracefully.

4. **Test Regularly**: Regularly test backups, failovers, and disaster recovery procedures. The middle of an incident is not the time to discover your backup strategy doesn't work.

5. **Choose the Right Tools**: Different databases excel at different scales and patterns. PostgreSQL, MySQL, MongoDB, Cassandra, and CockroachDB each have their strengths.

6. **Balance Trade-offs**: The CAP theorem is real. Understand what consistency, availability, and partition tolerance mean for your application.

7. **Automate Operations**: Manual operations don't scale. Invest in automation for deployments, backups, monitoring, and failovers.

By understanding these concepts and applying them thoughtfully, you can build database infrastructure that scales from prototype to billions of requests per day.

## Resources

### Documentation
- [PostgreSQL High Availability](https://www.postgresql.org/docs/current/high-availability.html)
- [MySQL Replication](https://dev.mysql.com/doc/refman/8.0/en/replication.html)
- [MongoDB Sharding](https://docs.mongodb.com/manual/sharding/)
- [Cassandra Architecture](https://cassandra.apache.org/doc/latest/architecture/)

### Tools
- **Patroni**: PostgreSQL HA solution
- **ProxySQL**: MySQL load balancer and query router
- **PgBouncer**: PostgreSQL connection pooler
- **Vitess**: MySQL sharding middleware
- **Flyway**: Database migration tool
- **Liquibase**: Database version control

### Books
- "Designing Data-Intensive Applications" by Martin Kleppmann
- "Database Reliability Engineering" by Laine Campbell & Charity Majors
- "The Art of PostgreSQL" by Dimitri Fontaine

### Monitoring
- **Prometheus + Grafana**: Metrics and visualization
- **Datadog**: Full-stack observability
- **pganalyze**: PostgreSQL performance monitoring
- **Percona Monitoring**: MySQL and MongoDB monitoring

---

*Questions or suggestions? Open an issue or contribute to improve this guide!*
