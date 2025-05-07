To completely remove PostgreSQL 16 from Rocky Linux 9, including its packages, configuration files, and data, follow these steps:

1. **Stop the PostgreSQL 16 Service**:
   Ensure the PostgreSQL 16 service is stopped:

   ```bash
   sudo systemctl stop postgresql-16
   ```

2. **Remove PostgreSQL 16 Packages**:
   Uninstall the PostgreSQL 16 server, client, and related packages:

   ```bash
   sudo dnf remove -y postgresql16-server postgresql16-contrib postgresql16*
   ```

3. **Remove the PostgreSQL Repository**:
   Remove the PostgreSQL repository to prevent accidental reinstallation:

   ```bash
   sudo dnf remove -y pgdg-redhat-repo
   ```

4. **Delete PostgreSQL Data and Configuration Files**:
   Remove the PostgreSQL 16 data directory and configuration files:

   ```bash
   sudo rm -rf /var/lib/pgsql/16
   sudo rm -rf /var/lib/pgsql/.psql_history
   ```

   Check for any remaining PostgreSQL files in `/usr/pgsql-16` and remove them:

   ```bash
   sudo rm -rf /usr/pgsql-16
   ```

5. **Remove PostgreSQL User and Group** (Optional):
   If no other PostgreSQL versions are installed, you can remove the `postgres` user and group:

   ```bash
   sudo userdel -r postgres
   ```

   **Note**: Skip this step if other PostgreSQL versions are still in use, as they may rely on the `postgres` user.

6. **Clean Up Dependencies**:
   Remove any unused dependencies installed with PostgreSQL:

   ```bash
   sudo dnf autoremove -y
   ```

7. **Remove Firewall Rules** (If Added):
   If you added a firewall rule for PostgreSQL (port 5432), remove it:

   ```bash
   sudo firewall-cmd --remove-port=5432/tcp --permanent
   sudo firewall-cmd --reload
   ```

8. **Verify Removal**:
   Check if PostgreSQL 16 is completely removed:

   ```bash
   rpm -qa | grep postgresql
   ```

   Ensure no `postgresql16` packages are listed. Also, verify the service is gone:

   ```bash
   systemctl list-units | grep postgresql
   ```

9. **Optional: Clean Up DNF Cache**:
   Clear the DNF cache to remove any cached PostgreSQL packages:
   ```bash
   sudo dnf clean all
   ```

**Notes**:

- Be cautious when deleting the data directory (`/var/lib/pgsql/16`), as it contains your databases and cannot be recovered unless backed up.
- If SELinux was configured for PostgreSQL, you may need to reset SELinux contexts for affected directories using `restorecon` if you plan to reuse them.
- If you disabled the default PostgreSQL module earlier, you can re-enable it if needed:
  ```bash
  sudo dnf module reset postgresql
  ```

PostgreSQL 16 and its associated files are now completely removed from Rocky Linux 9!
