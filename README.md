# 🔄 MySQL Replication with Fallback Server (DB01)

This project demonstrates a production-grade MySQL replication setup with a fallback mechanism using a *replica of a replica*. It is designed to improve data availability and disaster recovery in environments with hybrid infrastructure (AWS + on-premises).

---

## 📘 Project Overview

**Scenario:**

- A primary MySQL database runs on AWS (`DB-AWS`).
- An on-premises replica (`DB02`) is used internally by teams.
- A fallback server (`DB01`) replicates from `DB02` to ensure continuity.

If `DB02` fails, `DB01` can quickly take over by assuming its IP and stopping replication.

---

## 🗺️ Architecture Diagram
```
    AWS
+---------+
| DB-AWS | <--- Primary MySQL Server
+---------+
     |
     v
+---------+
| DB02 | <--- On-prem Replica
+---------+
     |
     v
+---------+
| DB01 | <--- Replica of Replica (Fallback)
+---------+
```
---

## 🧰 Tools & Technologies

- **MySQL** (5.7 or 8.x)
- [`mydumper`](https://github.com/mydumper/mydumper) – Fast backup/restore tool
- Linux (Debian/Ubuntu assumed)
- SSH/SCP
- Static/floating IP or internal DNS (optional)

---

## 🛠️ Setup Guide

### 1. Create MySQL users on both replicas server for connection and check if you LOG turned ON - on both replicas!

```
MySQL user
CREATE USER 'root'@'192.168.2.X' IDENTIFIED BY 'yoursecretpassword';
GRANT ALL PRIVILEGES ON *.* TO 'root'@'192.168.2.X' WITH GRANT OPTION;
FLUSH PRIVILEGES; - Re-sync what’s in RAM with what’s on disk.


nano /etc/mysql/mysql.conf.d/mysqld.cnf
Add lines under [mysqld] section
binlog-format           = ROW
log-slave-updates       = 1
sync-binlog             = 1
log_bin = /var/log/mysql/binlogs/mysql-bin.log
binlog_expire_logs_seconds = 864000
max_binlog_size         = 200M
server-id               = 30006
----
Save and restart service sudo systemctl restart mysql

Log to MySQL and check if LOG is turned ON 
mysql -u root -p -h IP Address
SHOW VARIABLES LIKE 'log_bin';

On DB02 check -> SHOW MASTER STATUS; - it will show you current position for sync DB before DUMP. NOTED IT! Important for resync DB after LOAD on replica server.
```
### 2. On Server DBO1 - replica of replica create in /opt/backup-db folder
```
mkdir /opt/backup-db
```
### 3. Backup DB02 Using MyDumper
```
mydumper -v 3 --regex '^(?!(mysql|test))' --host=IPAddress --user=MySQLUser --password=MySQLPassword --outputdir=backup-db --rows=500000 --compress --build-empty-files --threads=6 --compress-protocol
```
```
mydumper \
  -v 3 \                                 # Verbosity level (debug/info)
  --regex '^(?!(mysql|test))' \          # Exclude system databases (mysql, test)
  --host=192.168.2.X \                 # Source host (DB02 in your setup)
  --user=root \                          # MySQL username
  --password= \                          # MySQL password (left blank here—use env var or .my.cnf ideally)
  --outputdir=backup-db \                # Directory to store dump
  --rows=500000 \                        # Split large tables into chunks of ~500k rows
  --compress \                           # Compress the output files (.gz)
  --build-empty-files \                  # Include empty table structure files
  --threads=4 \                          # Use 4 parallel threads for dumping - really depends on processing power. Can delay replica of master -> parameter Seconds_Behind_Master
  --compress-protocol                    # Use compression for client/server protocol
```
### 4. Load DB02 Using MyLoader
```
myloader --host=IPAddress --regex '^(?!(sys))' --user=MySQLUser --password=MySQLPassword --directory=./backup-db --queries-per-transaction=50000 --threads=12 --compress-protocol --verbose=3

```
```
myloader \
  --host=192.168.2.X \                   # Target MySQL server (e.g. DB01)
  --regex='^(?!(sys))' \                 # Exclude the 'sys' database from restore
  --user=root \                          # MySQL user
  --password= \                          # MySQL password (left blank here)
  --directory=./backup-db \              # Path to backup directory from mydumper
  --queries-per-transaction=50000 \      # Commit every 50k queries for performance
  --threads=12 \                         # Use 12 threads to parallelize restore
  --compress-protocol \                  # Use compression in client/server communication
  --verbose=3                            # Set verbosity (debug/info)
```
### 5. Configure Replication
If you're not using GTID-based replication, configure the replica using the binary log file and position.
```
On DB02 check -> SHOW MASTER STATUS; -it will show you position and 

Log to MySQL and check if LOG is turned ON 
mysql -u root -p -h IP Address
SHOW VARIABLES LIKE 'log_bin';

Restart MySQL, then run:

STOP SLAVE;

CHANGE MASTER TO
  MASTER_HOST='DB02_IP',
  MASTER_USER='repl_user',
  MASTER_PASSWORD='your_password',
  MASTER_LOG_FILE='mysql-bin.000001', - log file on DB02 - replica
  MASTER_LOG_POS=3455299;

START SLAVE;
```

📌 Notes

Use read_only = 1 to protect replicas. <br>
Ensure time sync between all DB servers. <br>
Monitor replication lag via Seconds_Behind_Master. <br>
Keep repl_user with least privileges. <br>

---

**IMPORTANT <br>**
Via HEIDISQL - export DB and USER from first replica and import it via HeidiSQL to replica 2 and FLUSH PRIVILEGES; <br>
You will face with errors like -> Error 'Operation ALTER USER failed for 'replicator'@'192.168.2.x'' on query.

---
**Tuning TIP <br>**
If you are facing with high CPU utilization - tune in [**mysqld**] parameter **innodb_buffer_pool_size = XG** <br>
Fewer disk reads, Faster query responses, Lower CPU and I/O load
More consistent performance under heavy traffic <br>

“Keep more of my database in RAM, so you don’t have to read from disk as often.” :) 

---


🧑‍💻 Author
System Administrator — passionate about automation, reliability, and infrastructure resiliency.




