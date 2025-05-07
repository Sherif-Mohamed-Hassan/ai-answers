I need to create a Postgresql cluster as follows:

1. the Main site: 3 Postgresql servers
2. The Backup site: 2 Postgresql servers. but need to have one of them a read/write server, and replicate from the main site, it is a local hyber-V site act as backup when the main site is unavailable to the users in this site because of inernet unavailabilty not because the unavailability of the main site.
3. The DR site: 1 Postgresql server that replicate from the main and in case of unavailability of the main site it can act as read/write server.
   Can you help in designing this cluster ?

Below are the full configuration files for the three PostgreSQL servers in the **Main Site** of the cluster: **M1 (Primary)**, **M2 (Synchronous Standby)**, and **M3 (Asynchronous Standby)**. These configurations assume PostgreSQL 14 or later, streaming replication, and a secure setup with replication slots. The configurations include `postgresql.conf`, `pg_hba.conf`, and `standby.signal` (for standbys), tailored to the architecture described previously.

Assumptions:

- IP addresses:
  - M1: `192.168.1.10`
  - M2: `192.168.1.11`
  - M3: `192.168.1.12`
- Replication user: `replicator` with password `secure_password`.
- Data directory: `/var/lib/postgresql/14/main` (adjust as needed).
- PostgreSQL is installed on all nodes, and initial setup (e.g., `initdb`) is complete on M1.

---

### **M1: Primary Server**

# /var/lib/postgresql/14/main/postgresql.conf

# Network settings

listen_addresses = '\*' # Listen on all interfaces
port = 5432 # Default PostgreSQL port

# Replication settings

wal_level = replica # Enable streaming replication
max_wal_senders = 5 # Support up to 5 replication connections (M2, M3, B1, D1, +1 spare)
max_replication_slots = 5 # Support replication slots for standbys
synchronous_commit = on # Enable synchronous replication
synchronous_standby_names = 'M2' # M2 is the synchronous standby
wal_compression = on # Compress WAL to reduce network load
archive_mode = on # Enable WAL archiving (optional for backups)
archive_command = 'test ! -f /path/to/archive/%f && cp %p /path/to/archive/%f' # Adjust for your backup system

# Performance tuning (adjust based on hardware)

max_connections = 100 # Maximum client connections
shared_buffers = 512MB # 25% of RAM (assuming 2GB RAM for example)
work_mem = 16MB # Memory for query operations
maintenance_work_mem = 128MB # Memory for maintenance tasks
effective_cache_size = 1536MB # 75% of RAM
wal_buffers = 16MB # WAL buffer size
checkpoint_timeout = 10min # Checkpoint frequency
min_wal_size = 1GB # Minimum WAL size
max_wal_size = 4GB # Maximum WAL size

# Logging

log*destination = 'stderr' # Log to stderr
logging_collector = on # Collect logs
log_directory = 'log' # Log directory
log_filename = 'postgresql-%Y-%m-%d*%H%M%S.log' # Log file name format
log_min_messages = info # Log level
log_replication_commands = on # Log replication commands for debugging

# /var/lib/postgresql/14/main/pg_hba.conf

# Local connections

local all all trust

# Host-based connections (adjust subnet as needed)

host all all 127.0.0.1/32 md5
host all all 192.168.1.0/24 md5

# Replication connections

host replication replicator 192.168.1.11/32 md5 # M2
host replication replicator 192.168.1.12/32 md5 # M3
host replication replicator 192.168.2.0/24 md5 # Backup site (B1)
host replication replicator 192.168.3.0/24 md5 # DR site (D1)

---

### **M2: Synchronous Standby Server**

**Setup Steps** (before applying configs):

1. Initialize M2’s data directory using `pg_basebackup`:
   ```bash
   pg_basebackup -h 192.168.1.10 -D /var/lib/postgresql/14/main -U replicator -P --wal-method=stream
   ```
2. Apply the configuration files below and start the server.

# /var/lib/postgresql/14/main/postgresql.conf

# Network settings

listen_addresses = '\*' # Listen on all interfaces
port = 5432 # Default PostgreSQL port

# Replication settings

wal_level = replica # Enable streaming replication (for potential cascading replication)
max_wal_senders = 3 # Support potential downstream standbys
max_replication_slots = 3 # Support replication slots
hot_standby = on # Enable read queries on standby
hot_standby_feedback = on # Report conflicts to primary
wal_receiver_status_interval = 1s # Report status to primary frequently

# Performance tuning (same as M1 for consistency)

max_connections = 100
shared_buffers = 512MB
work_mem = 16MB
maintenance_work_mem = 128MB
effective_cache_size = 1536MB
wal_buffers = 16MB
checkpoint_timeout = 10min
min_wal_size = 1GB
max_wal_size = 4GB

# Logging

log*destination = 'stderr'
logging_collector = on
log_directory = 'log'
log_filename = 'postgresql-%Y-%m-%d*%H%M%S.log'
log_min_messages = info
log_replication_commands = on

# /var/lib/postgresql/14/main/pg_hba.conf

# Local connections

local all all trust

# Host-based connections

host all all 127.0.0.1/32 md5
host all all 192.168.1.0/24 md5

# Replication connections (if M2 becomes primary)

host replication replicator 192.168.1.10/32 md5 # M1
host replication replicator 192.168.1.12/32 md5 # M3
host replication replicator 192.168.2.0/24 md5 # Backup site (B1)
host replication replicator 192.168.3.0/24 md5 # DR site (D1)

# /var/lib/postgresql/14/main/standby.signal

standby_mode = on
primary_conninfo = 'host=192.168.1.10 port=5432 user=replicator password=secure_password application_name=M2'
primary_slot_name = 'm2_slot'

---

### **M3: Asynchronous Standby Server**

**Setup Steps** (before applying configs):

1. Initialize M3’s data directory using `pg_basebackup`:
   ```bash
   pg_basebackup -h 192.168.1.10 -D /var/lib/postgresql/14/main -U replicator -P --wal-method=stream
   ```
2. Apply the configuration files below and start the server.

# /var/lib/postgresql/14/main/postgresql.conf

# Network settings

listen_addresses = '\*' # Listen on all interfaces
port = 5432 # Default PostgreSQL port

# Replication settings

wal_level = replica # Enable streaming replication
max_wal_senders = 3 # Support potential downstream standbys
max_replication_slots = 3 # Support replication slots
hot_standby = on # Enable read queries on standby
hot_standby_feedback = on # Report conflicts to primary
wal_receiver_status_interval = 1s # Report status to primary frequently

# Performance tuning (same as M1 for consistency)

max_connections = 100
shared_buffers = 512MB
work_mem = 16MB
maintenance_work_mem = 128MB
effective_cache_size = 1536MB
wal_buffers = 16MB
checkpoint_timeout = 10min
min_wal_size = 1GB
max_wal_size = 4GB

# Logging

log*destination = 'stderr'
logging_collector = on
log_directory = 'log'
log_filename = 'postgresql-%Y-%m-%d*%H%M%S.log'
log_min_messages = info
log_replication_commands = on

# /var/lib/postgresql/14/main/pg_hba.conf

# Local connections

local all all trust

# Host-based connections

host all all 127.0.0.1/32 md5
host all all 192.168.1.0/24 md5

# Replication connections (if M3 becomes primary)

host replication replicator 192.168.1.10/32 md5 # M1
host replication replicator 192.168.1.11/32 md5 # M2
host replication replicator 192.168.2.0/24 md5 # Backup site (B1)
host replication replicator 192.168.3.0/24 md5 # DR site (D1)

# /var/lib/postgresql/14/main/standby.signal

standby_mode = on primary_conninfo = 'host=192.168.1.10 port=5432 user=replicator password=secure_password application_name=M3' primary_slot_name = 'm3_slot'

---

### **Additional Setup Instructions**

1. **Create Replication User on M1**:
   Connect to M1 and run:

   ```sql
   CREATE ROLE replicator WITH REPLICATION LOGIN PASSWORD 'secure_password';
   ```

2. **Create Replication Slots on M1**:
   For M2 and M3:

   ```sql
   SELECT pg_create_physical_replication_slot('m2_slot');
   SELECT pg_create_physical_replication_slot('m3_slot');
   ```

3. **Start PostgreSQL**:

   - On M1: `pg_ctl -D /var/lib/postgresql/14/main start`
   - On M2 and M3 (after `pg_basebackup` and applying configs): `pg_ctl -D /var/lib/postgresql/14/main start`

4. **Verify Replication**:
   On M1, check replication status:

   ```sql
   SELECT * FROM pg_stat_replication;
   ```

   Ensure M2 and M3 are listed with `sync_state` as `sync` (M2) and `async` (M3).

5. **Secure Passwords**:
   Store the `replicator` password securely (e.g., in a `.pgpass` file on M2 and M3):

   ```bash
   # /home/postgres/.pgpass
   192.168.1.10:5432:replication:replicator:secure_password
   ```

   Set permissions: `chmod 0600 /home/postgres/.pgpass`.

6. **Firewall and Network**:

   - Open port 5432 on all servers for PostgreSQL.
   - Ensure secure connections (consider SSL for replication).

7. **Monitoring**:
   - Monitor replication lag with `pg_stat_replication` on M1.
   - Set up alerts for replication delays or node failures.

---

### **Notes**

- **Performance Tuning**: Adjust `shared_buffers`, `work_mem`, etc., based on your server’s RAM and workload. The values provided are for a modest server (e.g., 2GB RAM).
- **Backup Integration**: Update `archive_command` in `M1_postgresql.conf` to integrate with a backup tool like `pg_backrest` or `barman`.
- **Failover**: For automated failover, consider using **Patroni** with **etcd** or **Consul**. Manual failover can be done with `pg_ctl promote` on M2 or M3.
- **Scalability**: M3 can handle read queries to offload M1. Use **pg_bouncer** or **Pgpool-II** to route read traffic.

If you need configuration files for the Backup or DR sites, or scripts for automation (e.g., Patroni setup), let me know!
