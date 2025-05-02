# MariaDB 10.5 to 10.6 Upgrade Plan (Rocky Linux, GTID Replication)

# MariaDB 10.5 to 10.6 Upgrade Plan (Rocky Linux, GTID Replication)

This guide details the process for upgrading a MariaDB GTID replication cluster from 10.5 to 10.6 on Rocky Linux.  
**Cluster topology:** 1 master, 2 slaves, GTID replication, 2 MaxScale proxies.

---

## 1. Pre-Upgrade Checklist

### 1.1 Notify Stakeholders

Inform all users and teams of the planned maintenance window. In my case downtime can be afforded.

### 1.2 Check Replication Health

SHOW SLAVE STATUS\G

text
Ensure `Slave_IO_Running` and `Slave_SQL_Running` are both `Yes` on all slaves.

### 1.3 Backup All Data

**Database Backup:**
mysqldump --all-databases --single-transaction --quick --lock-tables=false > /root/all-databases-$(date +%F).sql

text

**Configuration Backup:**
cp /etc/my.cnf /etc/my.cnf.bak
cp -r /var/lib/mysql /var/lib/mysql_bak

text

---

## 2. Upgrade Procedure (Per Node)

> **Upgrade all slaves first, then the master.**

### 2.1 Stop MariaDB

sudo systemctl stop mariadb

text

### 2.2 Remove Old MariaDB Packages

sudo yum remove "MariaDB-*"
sudo yum remove galera-4

text

### 2.3 Clean Up Old Repository Files

sudo mv /etc/yum.repos.d/mariadb.repo /etc/yum.repos.d/mariadb.repo.bak

text

### 2.4 Add MariaDB 10.6 Repository

sudo yum install curl
curl -LsSO https://r.mariadb.com/downloads/mariadb_repo_setup
chmod +x mariadb_repo_setup
sudo ./mariadb_repo_setup --mariadb-server-version="mariadb-10.6"

text

### 2.5 Install MariaDB 10.6

sudo yum install MariaDB-server MariaDB-backup

text

### 2.6 Start MariaDB

sudo systemctl start mariadb

text

### 2.7 Run the Upgrade Script

sudo mysql_upgrade

text

### 2.8 Enable MariaDB at Boot

sudo systemctl enable mariadb

text

### 2.9 Verify Upgrade

mysql -V

text

---

## 3. Post-Upgrade Steps

### 3.1 Check Replication Status

SHOW SLAVE STATUS\G

text

### 3.2 Monitor Logs

tail -f /var/log/mariadb/mariadb.log

text

### 3.3 Test Application Connectivity

Ensure your application can connect and operate as expected.

---

## 4. Master Node Special Instructions

### 4.1 Set Master to Read-Only Before Upgrade

SET GLOBAL read_only = ON;
FLUSH TABLES WITH READ LOCK;

text

### 4.2 After Upgrade, Remove Read-Only and Unlock

SET GLOBAL read_only = OFF;
UNLOCK TABLES;

text

---

## 5. Configuration File Notes

Ensure your `/etc/my.cnf` on slaves contains:

[mysqld]
datadir=/var/lib/mysql
socket=/var/lib/mysql/mysql.sock
log-error=/var/log/mariadb/mariadb.log
pid-file=/run/mariadb/mariadb.pid
binlog_format=ROW
gtid_strict_mode=ON
log_slave_updates=ON
log_bin=mariadb-bin
log_bin_index=mariadb-bin.index
relay_log=relay-bin
relay_log_index=relay-bin.index
relay_log_info_file=relay-bin.info
server_id=2
gtid_domain_id=2
#shutdown_wait_for_slaves=ON
expire_logs_days=7
session_track_system_variables=last_gtid
log-basename=slave2
binlog_expire_logs_seconds=604800

text

---

## 6. Rollback Plan

If issues arise, stop MariaDB and restore from backup:

sudo systemctl stop mariadb

Restore files from your backup location
sudo cp -r /var/lib/mysql_bak/* /var/lib/mysql/
sudo cp /etc/my.cnf.bak /etc/my.cnf
sudo systemctl start mariadb

text

---

## 7. Galera Cluster Recovery (if applicable)

If MariaDB fails to start due to Galera errors, follow these steps:

On the node with the highest sequence number:
cat /var/lib/mysql/grastate.dat

If safe_to_bootstrap: 0, set to 1
sudo sed -i 's/safe_to_bootstrap: 0/safe_to_bootstrap: 1/' /var/lib/mysql/grastate.dat

Bootstrap the cluster
sudo galera_new_cluster

Start MariaDB on other nodes
sudo systemctl start mariadb

text

---

## 8. References

- [MariaDB Official Upgrade Guide](https://mariadb.com/kb/en/upgrading/)
- [MariaDB Repository Setup](https://mariadb.com/downloads/)
- [MariaDB MaxScale Documentation](https://mariadb.com/kb/en/mariadb-maxscale-25-upgrade/)

---

**Test this process in a staging environment before applying to production.**
