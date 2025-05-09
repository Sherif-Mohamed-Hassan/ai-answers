# PostgreSQL 16 Streaming Replication with Patroni on Rocky Linux 9

This guide sets up PostgreSQL 16 streaming replication with Patroni for automated failover on three Rocky Linux 9 servers (`10.0.1.35`, `10.0.1.141`, `10.0.1.74`).

## Step 1: Prepare the Servers
Ensure all servers are ready and can communicate.

1. **Update the system** (on all nodes):
   ```bash
   sudo dnf update -y
   ```

2. **Set hostnames** (for clarity):
   - On `10.0.1.35`:
     ```bash
     sudo hostnamectl set-hostname pg1
     ```
   - On `10.0.1.141`:
     ```bash
     sudo hostnamectl set-hostname pg2
     ```
   - On `10.0.1.74`:
     ```bash
     sudo hostnamectl set-hostname pg3
     ```

3. **Update /etc/hosts** (on all nodes):
   ```bash
   sudo nano /etc/hosts
   ```
   Add:
   ```
   10.0.1.35 pg1
   10.0.1.141 pg2
   10.0.1.74 pg3
   ```

4. **Open firewall ports** (on all nodes):
   ```bash
   sudo firewall-cmd --permanent --add-port={5432/tcp,8008/tcp,2379/tcp,2380/tcp}
   sudo firewall-cmd --reload
   ```

5. **Disable SELinux** (or configure it for PostgreSQL/Patroni, but disabling is simpler for this guide):
   ```bash
   sudo setenforce 0
   sudo sed -i 's/^SELINUX=.*/SELINUX=disabled/' /etc/selinux/config
   ```

## Step 2: Install Dependencies
Install PostgreSQL (already installed), Patroni, etcd, and Python dependencies.

1. **Install Python and pip** (on all nodes):
   ```bash
   sudo dnf install -y python3 python3-pip
   ```

2. **Install etcd** (on all nodes):
   ```bash
   sudo dnf install -y etcd
   ```

3. **Install Patroni** (on all nodes):
   ```bash
   sudo pip3 install patroni[etcd]
   ```

4. **Verify installations**:
   ```bash
   psql --version
   etcd --version
   patroni --version
   ```

## Step 3: Configure etcd
Set up an etcd cluster for Patroni to store cluster state.

1. **Edit etcd configuration** (on all nodes):
   ```bash
   sudo nano /etc/etcd/etcd.conf
   ```
   Replace the content with (adjust for each node):
   - On `pg1` (`10.0.1.35`):
     ```
     ETCD_NAME=pg1
     ETCD_DATA_DIR=/var/lib/etcd/pg1.etcd
     ETCD_LISTEN_PEER_URLS=http://10.0.1.35:2380
     ETCD_LISTEN_CLIENT_URLS=http://10.0.1.35:2379
     ETCD_INITIAL_ADVERTISE_PEER_URLS=http://10.0.1.35:2380
     ETCD_ADVERTISE_CLIENT_URLS=http://10.0.1.35:2379
     ETCD_INITIAL_CLUSTER=pg1=http://10.0.1.35:2380,pg2=http://10.0.1.141:2380,pg3=http://10.0.1.74:2380
     ETCD_INITIAL_CLUSTER_TOKEN=pg-cluster
     ETCD_INITIAL_CLUSTER_STATE=new
     ```
   - On `pg2` (`10.0.1.141`):
     ```
     ETCD_NAME=pg2
     ETCD_DATA_DIR=/var/lib/etcd/pg2.etcd
     ETCD_LISTEN_PEER_URLS=http://10.0.1.141:2380
     ETCD_LISTEN_CLIENT_URLS=http://10.0.1.141:2379
     ETCD_INITIAL_ADVERTISE_PEER_URLS=http://10.0.1.141:2380
     ETCD_ADVERTISE_CLIENT_URLS=http://10.0.1.141:2379
     ETCD_INITIAL_CLUSTER=pg1=http://10.0.1.35:2380,pg2=http://10.0.1.141:2380,pg3=http://10.0.1.74:2380
     ETCD_INITIAL_CLUSTER_TOKEN=pg-cluster
     ETCD_INITIAL_CLUSTER_STATE=new
     ```
   - On `pg3` (`10.0.1.74`):
     ```
     ETCD_NAME=pg3
     ETCD_DATA_DIR=/var/lib/etcd/pg3.etcd
     ETCD_LISTEN_PEER_URLS=http://10.0.1.74:2380
     ETCD_LISTEN_CLIENT_URLS=http://10.0.1.74:2379
     ETCD_INITIAL_ADVERTISE_PEER_URLS=http://10.0.1.74:2380
     ETCD_ADVERTISE_CLIENT_URLS=http://10.0.1.74:2379
     ETCD_INITIAL_CLUSTER=pg1=http://10.0.1.35:2380,pg2=http://10.0.1.141:2380,pg3=http://10.0.1.74:2380
     ETCD_INITIAL_CLUSTER_TOKEN=pg-cluster
     ETCD_INITIAL_CLUSTER_STATE=new
     ```

2. **Enable and start etcd** (on all nodes):
   ```bash
   sudo systemctl enable etcd
   sudo systemctl start etcd
   ```

3. **Verify etcd cluster** (from any node):
   ```bash
   etcdctl member list
   ```
   You should see all three nodes listed.

## Step 4: Configure PostgreSQL for Replication
Create a PostgreSQL user for replication and prepare the data directory.

1. **Create PostgreSQL user** (on `pg1` only):
   ```bash
   sudo -u postgres psql
   ```
   Run:
   ```sql
   CREATE ROLE replicator WITH REPLICATION LOGIN PASSWORD 'replicator_password';
   ```

2. **Stop PostgreSQL** (on all nodes):
   ```bash
   sudo systemctl stop postgresql-16
   ```

3. **Clear data directory** (on all nodes, to ensure a clean setup):
   ```bash
   sudo rm -rf /var/lib/pgsql/16/data/*
   ```

## Step 5: Configure Patroni
Create Patroni configuration files to manage PostgreSQL and replication.

1. **Create Patroni config directory** (on all nodes):
   ```bash
   sudo mkdir -p /etc/patroni
   ```

2. **Create Patroni configuration** (on all nodes):
   ```bash
   sudo nano /etc/patroni/patroni.yml
   ```
   Add the following, adjusting the `name` and `listen`/`connect_address` for each node:
   - On `pg1` (`10.0.1.35`):
     ```yaml
     name: pg1
     scope: pg-cluster
     namespace: /db/
     
     restapi:
       listen: 10.0.1.35:8008
       connect_address: 10.0.1.35:8008
     
     etcd:
       hosts: 10.0.1.35:2379,10.0.1.141:2379,10.0.1.74:2379
     
     bootstrap:
       dcs:
         ttl: 30
         loop_wait: 10
         retry_timeout: 10
         maximum_lag_on_failover: 1048576
         postgresql:
           use_pg_rewind: true
           use_slots: true
           parameters:
             wal_level: replica
             hot_standby: "on"
             max_wal_senders: 10
             max_replication_slots: 10
             wal_log_hints: "on"
     
       initdb:
         - encoding: UTF8
         - data-checksums
     
       pg_hba:
         - host replication replicator 10.0.1.0/24 md5
         - host all all 10.0.1.0/24 md5
     
     postgresql:
       listen: 10.0.1.35:5432
       connect_address: 10.0.1.35:5432
       data_dir: /var/lib/pgsql/16/data
       bin_dir: /usr/pgsql-16/bin
       authentication:
         replication:
           username: replicator
           password: replicator_password
         superuser:
           username: postgres
           password: postgres_password
       parameters:
         unix_socket_directories: /var/run/postgresql
     
     tags:
       nofailover: false
       noloadbalance: false
       clonefrom: true
       nosync: false
     ```
   - On `pg2` (`10.0.1.141`):
     ```yaml
     name: pg2
     scope: pg-cluster
     namespace: /db/
     
     restapi:
       listen: 10.0.1.141:8008
       connect_address: 10.0.1.141:8008
     
     etcd:
       hosts: 10.0.1.35:2379,10.0.1.141:2379,10.0.1.74:2379
     
     bootstrap:
       dcs:
         ttl: 30
         loop_wait: 10
         retry_timeout: 10
         maximum_lag_on_failover: 1048576
         postgresql:
           use_pg_rewind: true
           use_slots: true
           parameters:
             wal_level: replica
             hot_standby: "on"
             max_wal_senders: 10
             max_replication_slots: 10
             wal_log_hints: "on"
     
       initdb:
         - encoding: UTF8
         - data-checksums
     
       pg_hba:
         - host replication replicator 10.0.1.0/24 md5
         - host all all 10.0.1.0/24 md5
     
     postgresql:
       listen: 10.0.1.141:5432
       connect_address: 10.0.1.141:5432
       data_dir: /var/lib/pgsql/16/data
       bin_dir: /usr/pgsql-16/bin
       authentication:
         replication:
           username: replicator
           password: replicator_password
         superuser:
           username: postgres
           password: postgres_password
       parameters:
         unix_socket_directories: /var/run/postgresql
     
     tags:
       nofailover: false
       noloadbalance: false
       clonefrom: true
       nosync: false
     ```
   - On `pg3` (`10.0.1.74`):
     ```yaml
     name: pg3
     scope: pg-cluster
     namespace: /db/
     
     restapi:
       listen: 10.0.1.74:8008
       connect_address: 10.0.1.74:8008
     
     etcd:
       hosts: 10.0.1.35:2379,10.0.1.141:2379,10.0.1.74:2379
     
     bootstrap:
       dcs:
         ttl: 30
         loop_wait: 10
         retry_timeout: 10
         maximum_lag_on_failover: 1048576
         postgresql:
           use_pg_rewind: true
           use_slots: true
           parameters:
             wal_level: replica
             hot_standby: "on"
             max_wal_senders: 10
             max_replication_slots: 10
             wal_log_hints: "on"
     
       initdb:
         - encoding: UTF8
         - data-checksums
     
       pg_hba:
         - host replication replicator 10.0.1.0/24 md5
         - host all all 10.0.1.0/24 md5
     
     postgresql:
       listen: 10.0.1.74:5432
       connect_address: 10.0.1.74:5432
       data_dir: /var/lib/pgsql/16/data
       bin_dir: /usr/pgsql-16/bin
       authentication:
         replication:
           username: replicator
           password: replicator_password
         superuser:
           username: postgres
           password: postgres_password
       parameters:
         unix_socket_directories: /var/run/postgresql
     
     tags:
       nofailover: false
       noloadbalance: false
       clonefrom: true
       nosync: false
     ```

3. **Set permissions** (on all nodes):
   ```bash
   sudo chown postgres:postgres /etc/patroni/patroni.yml
   sudo chmod 600 /etc/patroni/patroni.yml
   ```

## Step 6: Set Up Patroni as a Service
Create a systemd service for Patroni to ensure it runs automatically.

1. **Create Patroni service file** (on all nodes):
   ```bash
   sudo nano /etc/systemd/system/patroni.service
   ```
   Add:
   ```
   [Unit]
   Description=Patroni PostgreSQL High Availability
   After=network.target
   
   [Service]
   User=postgres
   Group=postgres
   ExecStart=/usr/local/bin/patroni /etc/patroni/patroni.yml
   ExecReload=/bin/kill -s HUP $MAINPID
   KillMode=process
   Restart=on-failure
   TimeoutSec=300
   
   [Install]
   WantedBy=multi-user.target
   ```

2. **Enable and start Patroni** (on all nodes):
   ```bash
   sudo systemctl daemon-reload
   sudo systemctl enable patroni
   sudo systemctl start patroni
   ```

## Step 7: Verify the Cluster
Check that Patroni has set up the PostgreSQL cluster correctly.

1. **Check Patroni status** (from any node):
   ```bash
   patronictl -c /etc/patroni/patroni.yml list
   ```
   Output should show:
   - One `Leader` (primary, likely `pg1` initially).
   - Two `Replica` nodes (`pg2`, `pg3`).
   - All nodes in `running` state with minimal lag.

2. **Verify PostgreSQL** (on the primary, e.g., `10.0.1.35`):
   ```bash
   sudo -u postgres psql -h 10.0.1.35
   ```
   Run:
   ```sql
   SELECT * FROM pg_stat_replication;
   ```
   Confirm that `pg2` and `pg3` are connected as replicas.

## Step 8: Set Up a Virtual IP (VIP) for Application Connections
Use `keepalived` to manage a VIP (`10.0.1.100`) that follows the primary.

1. **Install keepalived** (on all nodes):
   ```bash
   sudo dnf install -y keepalived
   ```

2. **Configure keepalived** (on all nodes):
   ```bash
   sudo nano /etc/keepalived/keepalived.conf
   ```
   - On `pg1` (`10.0.1.35`):
     ```conf
     vrrp_instance VI_1 {
         state MASTER
         interface eth0  # Replace with your network interface (check with `ip a`)
         virtual_router_id 51
         priority 100
         advert_int 1
         unicast_src_ip 10.0.1.35
         unicast_peer {
             10.0.1.141
             10.0.1.74
         }
         authentication {
             auth_type PASS
             auth_pass mypassword
         }
         virtual_ipaddress {
             10.0.1.100/24
         }
         track_script {
             chk_patroni
         }
     }
     vrrp_script chk_patroni {
         script "/usr/bin/curl -s http://10.0.1.35:8008/leader"
         interval 2
         fall 2
         rise 2
     }
     ```
   - On `pg2` (`10.0.1.141`):
     ```conf
     vrrp_instance VI_1 {
         state BACKUP
         interface eth0
         virtual_router_id 51
         priority 90
         advert_int 1
         unicast_src_ip 10.0.1.141
         unicast_peer {
             10.0.1.35
             10.0.1.74
         }
         authentication {
             auth_type PASS
             auth_pass mypassword
         }
         virtual_ipaddress {
             10.0.1.100/24
         }
         track_script {
             chk_patroni
         }
     }
     vrrp_script chk_patroni {
         script "/usr/bin/curl -s http://10.0.1.141:8008/leader"
         interval 2
         fall 2
         rise 2
     }
     ```
   - On `pg3` (`10.0.1.74`):
     ```conf
     vrrp_instance VI_1 {
         state BACKUP
         interface eth0
         virtual_router_id 51
         priority 80
         advert_int 1
         unicast_src_ip 10.0.1.74
         unicast_peer {
             10.0.1.35
             10.0.1.141
         }
         authentication {
             auth_type PASS
             auth_pass mypassword
         }
         virtual_ipaddress {
             10.0.1.100/24
         }
         track_script {
             chk_patroni
         }
     }
     vrrp_script chk_patroni {
         script "/usr/bin/curl -s http://10.0.1.74:8008/leader"
         interval 2
         fall 2
         rise 2
     }
     ```

3. **Install curl for keepalived** (on all nodes):
   ```bash
   sudo dnf install -y curl
   ```

4. **Enable and start keepalived** (on all nodes):
   ```bash
   sudo systemctl enable keepalived
   sudo systemctl start keepalived
   ```

5. **Verify VIP**:
   - On the primary node (e.g., `pg1` initially):
     ```bash
     ip a
     ```
     You should see `10.0.1.100` assigned to the interface.
   - From another server, ping the VIP:
     ```bash
     ping 10.0.1.100
     ```

## Step 9: Connect Applications
Configure your application to connect to the VIP (`10.0.1.100:5432`). Patroni and keepalived ensure the VIP always points to the current primary.

Example connection string:
```
postgresql://postgres:postgres_password@10.0.1.100:5432/your_database
```

## Step 10: Test Failover
Simulate a primary failure to verify automatic failover.

1. **Stop Patroni on the primary** (e.g., `pg1`):
   ```bash
   sudo systemctl stop patroni
   ```

2. **Check cluster status**:
   ```bash
   patronictl -c /etc/patroni/patroni.yml list
   ```
   Confirm that `pg2` or `pg3` is now the `Leader`.

3. **Verify VIP**:
   On the new primary, check:
   ```bash
   ip a
   ```
   The VIP (`10.0.1.100`) should now be on the new primary.

4. **Test application connectivity**:
   Ensure your application still connects to `10.0.1.100:5432` without changes.

5. **Restart the old primary**:
   ```bash
   sudo systemctl start patroni
   ```
   It should rejoin as a replica.

## Step 11: Monitoring and Maintenance
- **Monitor the cluster**:
  ```bash
  patronictl -c /etc/patroni/patroni.yml list
  ```
- **View logs**:
  ```bash
  sudo journalctl -u patroni
  sudo journalctl -u etcd
  sudo journalctl -u keepalived
  ```
- **Perform switchover** (planned maintenance):
  ```bash
  patronictl -c /etc/patroni/patroni.yml switchover --master pg1 --candidate pg2
  ```

## Troubleshooting
- **etcd not starting**: Check `/var/log/etcd.log` and ensure ports 2379/2380 are open.
- **Patroni not initializing**: Verify `/etc/patroni/patroni.yml` syntax and PostgreSQL data directory permissions.
- **VIP not moving**: Ensure `keepalived` is running and `curl` can reach the Patroni API (`http://<node>:8008/leader`).