To install PostgreSQL 16 specifically on Rocky Linux 9, follow these steps:

1. **Update the System**:

   ```bash
   sudo dnf update -y
   ```

2. **Add the PostgreSQL Repository**:
   Rocky Linux 9's default repositories may not include PostgreSQL 16. Add the official PostgreSQL repository:

   ```bash
   sudo dnf install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-9-x86_64/pgdg-redhat-repo-latest.noarch.rpm
   ```

3. **Disable Default PostgreSQL Module**:
   Prevent conflicts with the default Rocky Linux PostgreSQL module:

   ```bash
   sudo dnf -qy module disable postgresql
   ```

4. **Install PostgreSQL 16 Server**:
   Install the PostgreSQL 16 server and client packages:

   ```bash
   sudo dnf install -y postgresql16-server postgresql16-contrib
   ```

5. **Initialize the Database**:
   Initialize the PostgreSQL 16 database cluster:

   ```bash
   sudo /usr/pgsql-16/bin/postgresql-16-setup initdb
   ```

6. **Start and Enable PostgreSQL Service**:
   Start the PostgreSQL 16 service and enable it to run on boot:

   ```bash
   sudo systemctl start postgresql-16
   sudo systemctl enable postgresql-16
   ```

7. **Verify Installation**:
   Check the PostgreSQL service status:

   ```bash
   sudo systemctl status postgresql-16
   ```

   Confirm the version:

   ```bash
   /usr/pgsql-16/bin/psql --version
   ```

8. **Optional: Configure PostgreSQL**:

   - The default data directory is `/var/lib/pgsql/16/data/`.
   - Edit `/var/lib/pgsql/16/data/postgresql.conf` for configuration (e.g., listen addresses).
   - Edit `/var/lib/pgsql/16/data/pg_hba.conf` for client authentication.
   - Restart PostgreSQL after changes:
     ```bash
     sudo systemctl restart postgresql-16
     ```

9. **Access PostgreSQL**:
   Switch to the `postgres` user and access the PostgreSQL 16 prompt:

   ```bash
   sudo -u postgres /usr/pgsql-16/bin/psql
   ```

   Set a password for the `postgres` user:

   ```sql
   \password postgres
   ```

10. **Optional: Firewall Configuration**:
    Allow PostgreSQL (default port 5432) for remote access:
    ```bash
    sudo firewall-cmd --add-port=5432/tcp --permanent
    sudo firewall-cmd --reload
    ```

**Notes**:

- Ensure SELinux is configured if you face permission issues (`semanage` and `restorecon` may be needed).
- The PostgreSQL 16 binaries are located in `/usr/pgsql-16/bin/`.
- Secure the database by configuring `pg_hba.conf` and using strong passwords for production.

PostgreSQL 16 is now installed and running on Rocky Linux 9!
