What is Streaming Replication in Postgresql ? and how to deploy it on three posgresql servers running on Rocky Linux 9 and their IP addresses are: 10.16.34.60,61,62 ?

### What is Streaming Replication in PostgreSQL?

**Streaming Replication** is a feature in PostgreSQL that allows a primary database server to send transaction log (WAL - Write-Ahead Log) records to one or more standby servers in real-time. This enables the standby servers to maintain an up-to-date replica of the primary database, ensuring high availability, load balancing, or disaster recovery. Key characteristics include:

- **Asynchronous by Default**: Changes are sent to replicas with minimal delay, but there’s no guarantee of immediate consistency.
- **Synchronous Option**: Ensures data is written to at least one standby before committing on the primary, reducing data loss risk.
- **Hot Standby**: Standby servers can serve read-only queries while replicating, useful for load balancing.
- **WAL-Based**: Uses PostgreSQL’s Write-Ahead Logging to replicate changes.

Streaming Replication is ideal for scenarios requiring high availability or read scalability, as it allows failover to a standby server if the primary fails.

---

### Deploying Streaming Replication on Three PostgreSQL Servers

We’ll set up **Streaming Replication** on three PostgreSQL servers running on Rocky Linux 9 with IPs `10.16.34.60` (primary), `10.16.34.61` (standby 1), and `10.16.34.62` (standby 2). The primary server will replicate to both standbys, which will be hot standbys for read-only queries.

#### Prerequisites

- PostgreSQL 14 or later installed on all three servers (Rocky Linux 9 repositories provide PostgreSQL 13, so we’ll use the official PostgreSQL repository for a newer version).
- Servers can communicate over the network (ports `5432` for PostgreSQL and any replication-specific ports).
- Root or sudo access on all servers.
- Firewall and SELinux configured to allow PostgreSQL traffic.

#### Step-by-Step Deployment

##### 1. Install PostgreSQL on All Servers

Install PostgreSQL 15 (or desired version) on each server (`10.16.34.60`, `10.16.34.61`, `10.16.34.62`).

```bash
# Add PostgreSQL repository
sudo dnf install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-9-x86_64/pgdg-redhat-repo-latest.noarch.rpm

# Install PostgreSQL 15
sudo dnf install -y postgresql15-server postgresql15-contrib

# Initialize the database
sudo /usr/pgsql-15/bin/postgresql-15-setup initdb

# Enable and start PostgreSQL
sudo systemctl enable postgresql-15
sudo systemctl start postgresql-15
```

Repeat on all three servers.

##### 2. Configure the Primary Server (`10.16.34.60`)

a. **Edit PostgreSQL Configuration**
Edit `/var/lib/pgsql/15/data/postgresql.conf` to enable replication:

```bash
sudo vi /var/lib/pgsql/15/data/postgresql.conf
```

Add or modify:

```ini
listen_addresses = 'localhost,10.16.34.60'  # Listen on server IP
wal_level = replica                        # Enable replication
max_wal_senders = 5                        # Max number of WAL senders
wal_keep_size = 128                        # Retain WAL logs for standbys (in MB)
hot_standby = on                           # Allow read-only queries on standbys
synchronous_commit = on                    # Optional: for synchronous replication
```

b. **Configure Authentication**
Edit `/var/lib/pgsql/15/data/pg_hba.conf` to allow replication connections from standbys:

```bash
sudo vi /var/lib/pgsql/15/data/pg_hba.conf
```

Add:

```ini
# Allow replication from standbys
host replication replicator 10.16.34.61/32 scram-sha-256
host replication replicator 10.16.34.62/32 scram-sha-256
# Allow client connections (optional, for queries)
host all all 10.16.34.0/24 scram-sha-256
```

c. **Create a Replication User**
Log in to PostgreSQL as the default superuser and create a replication user:

```bash
sudo -u postgres psql
```

Run:

```sql
CREATE ROLE replicator WITH REPLICATION LOGIN PASSWORD 'secure_password';
```

d. **Restart PostgreSQL**
Apply changes:

```bash
sudo systemctl restart postgresql-15
```

##### 3. Configure Standby Servers (`10.16.34.61` and `10.16.34.62`)

For each standby server, perform the following:

a. **Stop PostgreSQL**

```bash
sudo systemctl stop postgresql-15
```

b. **Backup Primary Database**
Clear the standby’s data directory and copy the primary’s data:

```bash
sudo -u postgres rm -rf /var/lib/pgsql/15/data/*
```

Use `pg_basebackup` to copy the primary’s data:

```bash
sudo -u postgres pg_basebackup -h 10.16.34.60 -D /var/lib/pgsql/15/data -U replicator -P --wal-method=stream
```

Enter the `replicator` user’s password (`secure_password`) when prompted.

c. **Configure Standby Settings**
Edit `/var/lib/pgsql/15/data/postgresql.conf`:

```bash
sudo vi /var/lib/pgsql/15/data/postgresql.conf
```

Add or modify:

```ini
hot_standby = on
primary_conninfo = 'host=10.16.34.60 port=5432 user=replicator password=secure_password'
```

d. **Create Standby Signal File**
Create a file to indicate this is a standby:

```bash
sudo -u postgres touch /var/lib/pgsql/15/data/standby.signal
```

e. **Start PostgreSQL**

```bash
sudo systemctl start postgresql-15
```

Repeat steps a–e on `10.16.34.62`.

##### 4. Configure Firewall and SELinux

On all servers, allow PostgreSQL traffic:

```bash
# Firewall
sudo firewall-cmd --permanent --add-port=5432/tcp
sudo firewall-cmd --reload

# SELinux (if enforcing)
sudo setsebool -P httpd_can_network_connect_db 1
sudo chcon -R -t postgresql_db_t /var/lib/pgsql/15/data
```

##### 5. Verify Replication

On the **primary** (`10.16.34.60`):

```bash
sudo -u postgres psql -c "SELECT * FROM pg_stat_replication;"
```

You should see entries for both standbys (`10.16.34.61` and `10.16.34.62`) with `state` as `streaming`.

On each **standby**:

```bash
sudo -u postgres psql -c "SELECT pg_is_in_recovery();"
```

Should return `t` (true), indicating it’s in recovery mode.

Test read-only queries on a standby:

```bash
psql -h 10.16.34.61 -U postgres -d postgres -c "SELECT * FROM pg_stat_activity;"
```

##### 6. (Optional) Configure Synchronous Replication

For zero data loss, configure synchronous replication on the primary (`10.16.34.60`).

Edit `/var/lib/pgsql/15/data/postgresql.conf`:

```ini
synchronous_standby_names = 'ANY 1 (standby1, standby2)'
```

Edit `/var/lib/pgsql/15/data/pg_hba.conf` to name standbys:

```ini
host replication replicator 10.16.34.61/32 scram-sha-256 standby1
host replication replicator 10.16.34.62/32 scram-sha-256 standby2
```

Restart the primary:

```bash
sudo systemctl restart postgresql-15
```

##### 7. Test Failover (Optional)

To simulate a failover:

1. Stop the primary:
   ```bash
   sudo systemctl stop postgresql-15
   ```
2. Promote a standby (e.g., `10.16.34.61`):
   ```bash
   sudo -u postgres pg_ctl promote -D /var/lib/pgsql/15/data
   ```
3. Verify the new primary:
   ```bash
   psql -h 10.16.34.61 -U postgres -c "SELECT pg_is_in_recovery();"
   ```
   Should return `f` (false).

Reconfigure the old primary and other standby to replicate from the new primary after recovery.

---

### Notes

- **Backup**: Regularly back up the primary using `pg_dump` or physical backups to avoid data loss.
- **Monitoring**: Use tools like `pg_stat_replication` or external monitoring (e.g., Nagios, Zabbix) to track replication lag.
- **Security**: Use SSL for replication (`ssl=on` in `postgresql.conf`) and secure passwords.
- **Logs**: Check `/var/lib/pgsql/15/data/log/` for errors if replication fails.

If you encounter issues (e.g., connection errors, replication lag), share relevant logs or error messages for further troubleshooting.
