I need your help with details to configure streaming replication between 3 Postgresql-16 installed on Rocky linux 9, use the subnet 10.0.1.0/24

Configuring streaming replication for PostgreSQL 16 on three Rocky Linux 9 servers within the subnet 10.0.1.0/24 involves setting up one primary server and two standby servers. Below is a detailed guide to achieve this, assuming the servers are:

- **Primary**: 10.0.1.10
- **Standby 1**: 10.0.1.11
- **Standby 2**: 10.0.1.12

### Prerequisites

- PostgreSQL 16 is installed on all three servers (`sudo dnf install postgresql16-server postgresql16-contrib`).
- All servers are running Rocky Linux 9, and the PostgreSQL service is initialized (`/usr/pgsql-16/bin/postgresql-16-setup initdb`).
- Firewall rules allow traffic on port 5432 between the servers (`firewall-cmd --add-port=5432/tcp --permanent; firewall-cmd --reload`).
- The subnet 10.0.1.0/24 is configured, and servers can communicate.
- You have root or sudo access and the `postgres` user is available.

### Step-by-Step Configuration

#### 1. Configure the Primary Server (10.0.1.10)

1. **Edit PostgreSQL Configuration (`postgresql.conf`)**

   - File location: `/var/lib/pgsql/16/data/postgresql.conf`
   - Modify the following settings (uncomment and set as needed):
     ```conf
     listen_addresses = '*'  # Allow connections from all IPs
     wal_level = replica    # Enable replication
     max_wal_senders = 5    # Allow up to 5 standby servers
     wal_keep_size = 256MB  # Retain enough WAL for standbys
     max_replication_slots = 5  # Allow replication slots
     hot_standby = on       # Enable read-only queries on standbys
     synchronous_commit = on  # Ensure commits are safe
     ```
   - Save and exit.

2. **Configure Host-Based Authentication (`pg_hba.conf`)**

   - File location: `/var/lib/pgsql/16/data/pg_hba.conf`
   - Add the following lines at the end to allow replication from standbys:
     ```conf
     host replication rep_user 10.0.1.11/32 scram-sha-256
     host replication rep_user 10.0.1.12/32 scram-sha-256
     ```
   - Save and exit.

3. **Create a Replication User**

   - Switch to the `postgres` user: `sudo su - postgres`
   - Run `psql` and create a user for replication:
     ```sql
     CREATE ROLE rep_user WITH REPLICATION LOGIN PASSWORD 'secure_password';
     ```
   - Exit psql: `\q`

4. **Restart PostgreSQL**

   - Restart the service to apply changes:
     ```bash
     sudo systemctl restart postgresql-16
     ```

5. **Create Replication Slots**
   - In `psql` on the primary, create slots for each standby:
     ```sql
     SELECT pg_create_physical_replication_slot('standby1_slot');
     SELECT pg_create_physical_replication_slot('standby2_slot');
     ```
   - Verify slots:
     ```sql
     SELECT * FROM pg_replication_slots;
     ```

#### 2. Configure Standby Servers (10.0.1.11 and 10.0.1.12)

Perform the following steps on both standby servers.

1. **Stop PostgreSQL Service**

   - Ensure the PostgreSQL service is stopped:
     ```bash
     sudo systemctl stop postgresql-16
     ```

2. **Clear Data Directory**

   - Remove existing data to prepare for the primary’s data:
     ```bash
     sudo rm -rf /var/lib/pgsql/16/data/*
     ```

3. **Copy Data from Primary**

   - Use `pg_basebackup` to copy the primary’s data directory:
     - For Standby 1 (10.0.1.11):
       ```bash
       sudo -u postgres pg_basebackup -h 10.0.1.10 -D /var/lib/pgsql/16/data -U rep_user -P -v -R -X stream -C -S standby1_slot
       ```
     - For Standby 2 (10.0.1.12):
       ```bash
       sudo -u postgres pg_basebackup -h 10.0.1.10 -D /var/lib/pgsql/16/data -U rep_user -P -v -R -X stream -C -S standby2_slot
       ```
   - Enter the `rep_user` password (`secure_password`) when prompted.
   - The `-R` option creates a `standby.signal` file and appends connection settings to `postgresql.auto.conf`.

4. **Edit PostgreSQL Configuration (`postgresql.conf`)**

   - File location: `/var/lib/pgsql/16/data/postgresql.conf`
   - Ensure the following is set:
     ```conf
     hot_standby = on  # Enable read-only queries
     ```
   - Save and exit.

5. **Verify Standby Configuration**

   - Check `postgresql.auto.conf` in `/var/lib/pgsql/16/data/`:
     ```conf
     primary_conninfo = 'user=rep_user password=secure_password host=10.0.1.10 port=5432 sslmode=prefer sslcompression=0 krbsrvname=postgres target_session_attrs=any'
     primary_slot_name = 'standby1_slot'  # or 'standby2_slot' for 10.0.1.12
     ```
   - Ensure the `standby.signal` file exists: `ls /var/lib/pgsql/16/data/standby.signal`.

6. **Set File Permissions**

   - Ensure the `postgres` user owns the data directory:
     ```bash
     sudo chown -R postgres:postgres /var/lib/pgsql/16/data
     sudo chmod -R 700 /var/lib/pgsql/16/data
     ```

7. **Start PostgreSQL**
   - Start the service:
     ```bash
     sudo systemctl start postgresql-16
     ```

#### 3. Verify Replication

1. **On the Primary (10.0.1.10)**

   - Check the replication status:
     ```bash
     sudo -u postgres psql -c "SELECT * FROM pg_stat_replication;"
     ```
   - Look for `state = streaming` and `sync_state = async` (or `sync` if configured for synchronous replication) for both standbys (client_addr: 10.0.1.11 and 10.0.1.12).
   - Verify replication slots:
     ```bash
     sudo -u postgres psql -c "SELECT * FROM pg_replication_slots;"
     ```
   - Ensure `active` is `t` (true) for both slots.

2. **On Standby Servers (10.0.1.11 and 10.0.1.12)**

   - Confirm the server is in recovery mode:
     ```bash
     sudo -u postgres psql -c "SELECT pg_is_in_recovery();"
     ```
   - Output should be `t` (true).
   - Check WAL receiver status:
     ```bash
     sudo -u postgres psql -c "SELECT * FROM pg_stat_wal_receiver;"
     ```
   - Look for `status = streaming`.

3. **Test Data Replication**
   - On the primary, create a test database and table:
     ```bash
     sudo -u postgres psql -c "CREATE DATABASE testdb;"
     psql -d testdb -c "CREATE TABLE test_table (id SERIAL PRIMARY KEY, data TEXT); INSERT INTO test_table (data) VALUES ('Test Data');"
     ```
   - On each standby, verify the data:
     ```bash
     sudo -u postgres psql -d testdb -c "SELECT * FROM test_table;"
     ```
   - You should see the row with `Test Data`.

#### 4. Optional: Configure Synchronous Replication

To ensure commits wait for at least one standby, configure synchronous replication on the primary.

1. **Edit `postgresql.conf` on Primary**

   - Add or modify:
     ```conf
     synchronous_standby_names = 'standby1_slot,standby2_slot'
     ```

2. **Restart Primary**

   - ```bash
     sudo systemctl restart postgresql-16
     ```

3. **Verify Synchronous Replication**
   - Check `pg_stat_replication` on the primary:
     ```bash
     sudo -u postgres psql -c "SELECT * FROM pg_stat_replication;"
     ```
   - Ensure `sync_state = sync` for at least one standby.

#### 5. Monitoring and Maintenance

- **Monitor Replication Lag**:
  - On the primary:
    ```sql
    SELECT application_name, client_addr, write_lag, flush_lag, replay_lag FROM pg_stat_replication;
    ```
  - On standbys:
    ```sql
    SELECT pg_last_wal_receive_lsn(), pg_last_wal_replay_lsn();
    ```
- **Log Files**:
  - Check logs for errors: `/var/lib/pgsql/16/data/log/`.
- **Failover**:
  - To promote a standby (e.g., 10.0.1.11) to primary:
    ```bash
    sudo -u postgres pg_ctl promote -D /var/lib/pgsql/16/data
    ```
  - Reconfigure the other servers accordingly.

#### 6. Troubleshooting

- **WAL Segment Errors**:
  - If you see `requested WAL segment has already been removed`, increase `wal_keep_size` or use archive mode (`archive_mode = on`, `archive_command`).
- **Connection Issues**:
  - Verify firewall settings and `pg_hba.conf` entries.
  - Ensure `rep_user` password is correct in `primary_conninfo`.
- **Replication Not Starting**:
  - Check `standby.signal` and `postgresql.auto.conf` on standbys.
  - Ensure replication slots are active.

### Notes

- Replace `secure_password` with a strong password.
- Replication slots prevent WAL removal, reducing the risk of standby falling behind.
- For production, consider setting up WAL archiving for additional safety.
- Ensure backups are in place before making changes.

This setup provides a robust streaming replication configuration for PostgreSQL 16 on Rocky Linux 9. For further details, refer to the PostgreSQL documentation: https://www.postgresql.org/docs/16/streaming-replication.html.[](https://www.postgresql.org/docs/current/runtime-config-replication.html)[](https://www.postgresql.org/docs/current/protocol-replication.html)
