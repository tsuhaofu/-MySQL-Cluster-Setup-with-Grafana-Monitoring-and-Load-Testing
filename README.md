# MySQL Cluster Setup with Grafana Monitoring and Load Testing

## Introduction

This project sets up a MySQL cluster with 1 master and 2 slaves in a local environment. Prometheus scrapes the master through `mysqld_exporter`, Grafana renders it on dashboard 7362, and sysbench puts load through the master so the dashboard has something to show.

**This repository is a runbook, not an application.** It contains no code, configs or
dashboards — everything needed is in the commands below, which were written and run on
macOS with Homebrew. Follow them in order and you will end up with the cluster described.

Passwords below are written as placeholders (`<ROOT_PASSWORD>`, `<REPLICATION_PASSWORD>`,
`<EXPORTER_PASSWORD>` — three distinct accounts). Substitute your own; they are local-only
credentials for a cluster you can tear down.

> ### ⚠️ Read this before you start: MySQL version
>
> This runbook was written and run against **MySQL 8.0**. The replication commands it
> uses — `CHANGE MASTER TO`, `START SLAVE`, `STOP SLAVE`, `SHOW SLAVE STATUS` — were
> deprecated in 8.0.22–8.0.23 and **removed in MySQL 8.4**. `brew install mysql` today
> installs **9.x**, on which Step 2 fails outright.
>
> Both forms are given below: the 8.0 command as originally run, and the 8.4+
> equivalent. Use whichever matches your server (`mysql --version`).
>
> | 8.0 (as run here) | 8.4 and later |
> |---|---|
> | `CHANGE MASTER TO` | `CHANGE REPLICATION SOURCE TO` |
> | `MASTER_HOST` / `MASTER_USER` / `MASTER_PASSWORD` | `SOURCE_HOST` / `SOURCE_USER` / `SOURCE_PASSWORD` |
> | `MASTER_LOG_FILE` / `MASTER_LOG_POS` | `SOURCE_LOG_FILE` / `SOURCE_LOG_POS` |
> | `START SLAVE` / `STOP SLAVE` | `START REPLICA` / `STOP REPLICA` |
> | `SHOW SLAVE STATUS` | `SHOW REPLICA STATUS` |
> | `SHOW MASTER STATUS` | `SHOW BINARY LOG STATUS` |
> | `GRANT REPLICATION SLAVE` | unchanged |

## Installation

### Prerequisites

- Homebrew installed on macOS
- MySQL
- Prometheus
- Grafana
- Sysbench

### Steps to Set Up the Environment

```bash
# Step 1: Install MySQL
brew install mysql
brew services start mysql
mysql_secure_installation   # set a root password when prompted

# Step 2: Configure MySQL Master and Slaves

# Initialize MySQL Data Directories for Slaves
mkdir -p /opt/homebrew/var/mysql_slave1
mkdir -p /opt/homebrew/var/mysql_slave2
mysqld --initialize-insecure --datadir=/opt/homebrew/var/mysql_slave1
mysqld --initialize-insecure --datadir=/opt/homebrew/var/mysql_slave2

# Edit MySQL Configuration Files

# Create /opt/homebrew/etc/my.cnf for the master
sudo nano /opt/homebrew/etc/my.cnf
[mysqld]
server-id=1
log-bin=mysql-bin

# Create /opt/homebrew/etc/my_slave1.cnf for the first slave
sudo nano /opt/homebrew/etc/my_slave1.cnf
[mysqld]
server-id=2
datadir=/opt/homebrew/var/mysql_slave1
port=3307
relay-log=relay-log
socket=/tmp/mysql_slave1.sock
mysqlx=0

# Create /opt/homebrew/etc/my_slave2.cnf for the second slave
# NOTE: datadir and socket must differ from slave 1, or the second instance
# will not start -- it would be pointed at the first slave's files.
sudo nano /opt/homebrew/etc/my_slave2.cnf
[mysqld]
server-id=3
datadir=/opt/homebrew/var/mysql_slave2
port=3308
relay-log=relay-log
socket=/tmp/mysql_slave2.sock
mysqlx=0

# Start MySQL Instances
mysqld --defaults-file=/opt/homebrew/etc/my.cnf &
mysqld --defaults-file=/opt/homebrew/etc/my_slave1.cnf &
mysqld --defaults-file=/opt/homebrew/etc/my_slave2.cnf &

# Configure Replication

# Connect to the master and set up replication user
mysql -u root -p -e "
CREATE USER 'replica'@'%' IDENTIFIED BY '<REPLICATION_PASSWORD>';
GRANT REPLICATION SLAVE ON *.* TO 'replica'@'%';
FLUSH PRIVILEGES;
SHOW MASTER STATUS;"        # 8.4+: SHOW BINARY LOG STATUS;
+------------------+----------+--------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000001 |      834 |              |                  |                   |
+------------------+----------+--------------+------------------+-------------------+

# The File and Position above are from one particular run. Read your own values
# from this output and use them in the two CHANGE MASTER TO statements below --
# 834 will not be your position.

# Configure slaves
mysql -u root -h 127.0.0.1 -P 3307 -e "
STOP SLAVE;
ALTER USER 'root'@'localhost' IDENTIFIED BY '<ROOT_PASSWORD>';
CHANGE MASTER TO 
  MASTER_HOST='127.0.0.1', 
  MASTER_USER='replica', 
  MASTER_PASSWORD='<REPLICATION_PASSWORD>', 
  MASTER_LOG_FILE='mysql-bin.000001', 
  MASTER_LOG_POS=834;
START SLAVE;"

# On MySQL 8.4 or later, the same step reads:
# mysql -u root -h 127.0.0.1 -P 3307 -e "
# STOP REPLICA;
# ALTER USER 'root'@'localhost' IDENTIFIED BY '<ROOT_PASSWORD>';
# CHANGE REPLICATION SOURCE TO
#   SOURCE_HOST='127.0.0.1',
#   SOURCE_USER='replica',
#   SOURCE_PASSWORD='<REPLICATION_PASSWORD>',
#   SOURCE_LOG_FILE='mysql-bin.000001',
#   SOURCE_LOG_POS=834;
# START REPLICA;"

mysql -u root -h 127.0.0.1 -P 3308 -e "
STOP SLAVE;
ALTER USER 'root'@'localhost' IDENTIFIED BY '<ROOT_PASSWORD>';
CHANGE MASTER TO 
  MASTER_HOST='127.0.0.1', 
  MASTER_USER='replica', 
  MASTER_PASSWORD='<REPLICATION_PASSWORD>', 
  MASTER_LOG_FILE='mysql-bin.000001', 
  MASTER_LOG_POS=834;
START SLAVE;"

# Verify replication   (8.4+: SHOW REPLICA STATUS)
mysql -u root -p -h 127.0.0.1 -P 3307 -e "SHOW SLAVE STATUS\G"
mysql -u root -p -h 127.0.0.1 -P 3308 -e "SHOW SLAVE STATUS\G"

# Step 3: Install and Configure Prometheus and Grafana

# Install Prometheus
brew install prometheus

# Configure Prometheus
sudo nano /opt/homebrew/etc/prometheus.yml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'mysql'
    static_configs:
      - targets: ['localhost:9104']

# Note: this is a single exporter, and the .my.cnf below sets no port, so it
# connects to 3306 -- the master only. The two slaves are not scraped. To cover
# the whole cluster you would run one exporter per instance on separate ports
# and list all three as targets here.

# Start Prometheus
brew services start prometheus

# Install and Configure MySQL Exporter

# Download and move the MySQL exporter binary
curl -LO https://github.com/prometheus/mysqld_exporter/releases/download/v0.14.0/mysqld_exporter-0.14.0.darwin-amd64.tar.gz
tar xvf mysqld_exporter-0.14.0.darwin-amd64.tar.gz
sudo mv mysqld_exporter-0.14.0.darwin-amd64/mysqld_exporter /usr/local/bin/

# Create a MySQL User for Exporter:
CREATE USER 'exporter'@'localhost' IDENTIFIED BY '<EXPORTER_PASSWORD>';
GRANT PROCESS, REPLICATION CLIENT, SELECT ON *.* TO 'exporter'@'localhost';
FLUSH PRIVILEGES;

# Create a .my.cnf file for the exporter
sudo nano /usr/local/bin/.my.cnf
user=exporter
password=<EXPORTER_PASSWORD>

# Run MySQL Exporter
mysqld_exporter --config.my-cnf /usr/local/bin/.my.cnf &

# Check Prometheus Targets:
Open your web browser and go to http://localhost:9090/targets. You should see both the Prometheus and MySQL targets listed and their status as “UP”.

# Install Grafana
brew install grafana
brew services start grafana

# Configure Grafana
# Open Grafana at http://localhost:3000
# Log in with default credentials (admin/admin)
# Add Prometheus as a data source (http://localhost:9090)
# Import a MySQL dashboard using ID 7362

# Step 4: Load Testing with Sysbench

# Install Sysbench
brew install sysbench

# Prepare the Test Database
sysbench --mysql-host=127.0.0.1 --mysql-port=3306 --mysql-user=root --mysql-password=<ROOT_PASSWORD> --mysql-db=test_db --table-size=100000 --tables=10 --threads=6 --time=60 --events=0 --report-interval=10 oltp_read_write prepare

# Run the Load Test

sysbench --mysql-host=127.0.0.1 --mysql-port=3306 --mysql-user=root --mysql-password=<ROOT_PASSWORD> --mysql-db=test_db --table-size=100000 --tables=10 --threads=6 --time=60 --events=0 --report-interval=10 oltp_read_write run

# Step 5: Monitor the Dashboard

# Open the MySQL Dashboard in Grafana
# While the load test is running, monitor your MySQL metrics on the imported Grafana dashboard
# Observe metrics such as QPS (Queries Per Second), active connections, and other performance indicators

# Step 6: Closing the Setup

# Stop MySQL Slaves
mysql -u root -p -h 127.0.0.1 -P 3307 -e "STOP SLAVE;"
mysql -u root -p -h 127.0.0.1 -P 3308 -e "STOP SLAVE;"

# Stop MySQL Master and Slave Instances
mysqladmin -u root -p -h 127.0.0.1 -P 3306 shutdown
mysqladmin -u root -p -h 127.0.0.1 -P 3307 shutdown
mysqladmin -u root -p -h 127.0.0.1 -P 3308 shutdown

# Stop Prometheus and Grafana
brew services stop prometheus
brew services stop grafana

# Step 7: Reopening the Setup

# Start MySQL Master
mysqld --defaults-file=/opt/homebrew/etc/my.cnf &

# Start MySQL Slaves
mysqld --defaults-file=/opt/homebrew/etc/my_slave1.cnf &
mysqld --defaults-file=/opt/homebrew/etc/my_slave2.cnf &

# Connect and Start Slaves
mysql -u root -p -h 127.0.0.1 -P 3307 -e "START SLAVE;"
mysql -u root -p -h 127.0.0.1 -P 3308 -e "START SLAVE;"

# Start MySQL_Exporter, Prometheus and Grafana
mysqld_exporter --config.my-cnf /usr/local/bin/.my.cnf &
brew services start prometheus
brew services start grafana

# Verify Setup
# Check MySQL master and slave status
# Ensure Prometheus is running at http://localhost:9090
# Ensure Grafana is running at http://localhost:3000
