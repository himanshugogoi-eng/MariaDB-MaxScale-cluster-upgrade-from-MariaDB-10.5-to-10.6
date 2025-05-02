# MariaDB 10.5 to 10.6 Upgrade Plan (Rocky Linux, GTID Replication)
![MariaDB](https://img.shields.io/badge/MariaDB-10.5.22-blue) ![MariaDB](https://img.shields.io/badge/MariaDB-10.6.21-blue) ![MaxScale](https://img.shields.io/badge/MaxScale-24.02.4-blue) ![Rocky Linux](https://img.shields.io/badge/Rocky-Linux-9.5-green)


## Overiew
This guide details the process for upgrading a MariaDB GTID replication cluster from 10.5 to 10.6 on Rocky Linux.  
**Cluster topology:** 1 master, 2 slaves, GTID replication, 2 MaxScale proxies.

---

## 1. Pre-Upgrade Checklist

### 1.1 Notify Stakeholders

Inform all users and teams of the planned maintenance window. In my case downtime can be afforded.

### 1.2 Check Replication Health

```sh
SHOW SLAVE STATUS\G
```


Ensure `Slave_IO_Running` and `Slave_SQL_Running` are both `Yes` on all slaves.

### 1.3 Backup All Data

**Database Backup:**

```sh
mysqldump --all-databases --single-transaction --lock-tables=false > /home/all-databases-$(date +%F).sql
```
**Configuration Backup:**
```sh
cp /etc/my.cnf /etc/my.cnf.bak
cp -r /var/lib/mysql /var/lib/mysql_bak
```


---

## 2. Upgrade Procedure (Per Node)

> **Upgrade all slaves first, then the master.**

### 2.1 Stop MariaDB

```sh
sudo systemctl stop mariadb
```


### 2.2 Remove Old MariaDB Packages

```sh
sudo yum remove "mariadb-*"
sudo yum remove galera-4
```


### 2.3 Clean Up Old Repository Files if exists.

```sh
sudo mv /etc/yum.repos.d/mariadb.repo /etc/yum.repos.d/mariadb.repo.bak
```


### 2.4 Add MariaDB 10.6 Repository

```sh
 sudo yum install curl
 curl -LsSO https://r.mariadb.com/downloads/mariadb_repo_setup
 chmod +x mariadb_repo_setup
 sudo ./mariadb_repo_setup --mariadb-server-version="mariadb-10.6"
```


### 2.5 Install MariaDB 10.6

```sh
sudo yum install MariaDB-server MariaDB-backup
```


### 2.6 Start MariaDB

```sh
sudo systemctl start mariadb
```

Note: If error comes while starting the mariadb service then look for folder /run/mariadb. If it does not exist then create it and give ownership to mysql user.

```sh
mkdir -p /run/mariadb
chown mysql:mysql /run/mariadb
```
### 2.7 Run the Upgrade Script

```sh
sudo mysql_upgrade
```


### 2.8 Enable MariaDB at Boot

```sh
sudo systemctl enable mariadb
```


### 2.9 Verify Upgrade

```sh
mysql -V
```


---

## 3. Post-Upgrade Steps

### 3.1 Check Replication Status

```sh
SHOW SLAVE STATUS\G
```


### 3.2 Monitor Logs

```sh
tail -f /var/log/mariadb/mariadb.log
```


### 3.3 Test Application Connectivity

Ensure your application can connect and operate as expected.

---

## 4. Master Node Special Instructions

### 4.1 Set Master to Read-Only Before Upgrade

```sh
SET GLOBAL read_only = ON;
FLUSH TABLES WITH READ LOCK;
```


### 4.2 After Upgrade, Remove Read-Only and Unlock

```sh
SET GLOBAL read_only = OFF;
UNLOCK TABLES;
```


---

## 5. Configuration File Notes

Ensure your `/etc/my.cnf` on slaves contains:

```sh
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
```


---

## 6. Rollback Plan

If issues arise, stop MariaDB and restore from backup:

sudo systemctl stop mariadb

Restore files from your backup location

```sh
sudo cp -r /var/lib/mysql_bak/* /var/lib/mysql/
sudo cp /etc/my.cnf.bak /etc/my.cnf
sudo systemctl start mariadb
```


---

## 7. Galera Cluster Recovery (if applicable)

If MariaDB fails to start due to Galera errors, follow these steps:

On the node with the highest sequence number:

```sh
cat /var/lib/mysql/grastate.dat
```
If safe_to_bootstrap: 0, set to 1

```sh
sudo sed -i 's/safe_to_bootstrap: 0/safe_to_bootstrap: 1/' /var/lib/mysql/grastate.dat
```
Bootstrap the cluster

```sh
sudo galera_new_cluster
```
Start MariaDB on other nodes

```sh
sudo systemctl start mariadb
```


---

## 8. References

- [MariaDB Official Upgrade Guide](https://mariadb.com/kb/en/upgrading/)
- [MariaDB Repository Setup](https://mariadb.com/downloads/)
- [MariaDB MaxScale Documentation](https://mariadb.com/kb/en/mariadb-maxscale-25-upgrade/)

---

**Test this process in a staging environment before applying to production.**
