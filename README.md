# MySQL Cluster Setup with Grafana Monitoring and Load Testing

## Introduction

This project sets up a MySQL cluster with 1 master and 2 slaves in a local environment. Additionally, Grafana is configured to monitor the MySQL cluster, and load testing is added to put some traffic into the cluster.

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
mysql_secure_installation (password: ComplexPassword123!)

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
sudo nano /opt/homebrew/etc/my_slave2.cnf
[mysqld]
server-id=3
datadir=/opt/homebrew/var/mysql_slave1
port=3308
relay-log=relay-log
socket=/tmp/mysql_slave1.sock
mysqlx=0

# Start MySQL Instances
mysqld --defaults-file=/opt/homebrew/etc/my.cnf &
mysqld --defaults-file=/opt/homebrew/etc/my_slave1.cnf &
mysqld --defaults-file=/opt/homebrew/etc/my_slave2.cnf &

# Configure Replication

# Connect to the master and set up replication user
mysql -u root -p
CREATE USER 'replica'@'%' IDENTIFIED BY 'ComplexPassword123!';
GRANT REPLICATION SLAVE ON *.* TO 'replica'@'%';
FLUSH PRIVILEGES;
SHOW MASTER STATUS;"
+------------------+----------+--------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000001 |      834 |              |                  |                   |
+------------------+----------+--------------+------------------+-------------------+

# Configure slaves
mysql -u root -h 127.0.0.1 -P 3307
STOP SLAVE;
ALTER USER 'root'@'localhost' IDENTIFIED BY 'root_password';
CHANGE MASTER TO 
  MASTER_HOST='127.0.0.1', 
  MASTER_USER='replica', 
  MASTER_PASSWORD='ComplexPassword123!', 
  MASTER_LOG_FILE='mysql-bin.000001', 
  MASTER_LOG_POS=834;
START SLAVE;"

mysql -u root -h 127.0.0.1 -P 3308 -e "
STOP SLAVE;
ALTER USER 'root'@'localhost' IDENTIFIED BY 'root_password';
CHANGE MASTER TO 
  MASTER_HOST='127.0.0.1', 
  MASTER_USER='replica', 
  MASTER_PASSWORD='ComplexPassword123!', 
  MASTER_LOG_FILE='mysql-bin.000001', 
  MASTER_LOG_POS=834;
START SLAVE;"

# Verify replication
mysql -u root -p -h 127.0.0.1 -P 3307 -e "SHOW SLAVE STATUS\G;"
mysql -u root -p -h 127.0.0.1 -P 3308 -e "SHOW SLAVE STATUS\G;"

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

# Start Prometheus
brew services start prometheus

# Install and Configure MySQL Exporter

# Download and move the MySQL exporter binary
curl -LO https://github.com/prometheus/mysqld_exporter/releases/download/v0.14.0/mysqld_exporter-0.14.0.darwin-amd64.tar.gz
tar xvf mysqld_exporter-0.14.0.darwin-amd64.tar.gz
sudo mv mysqld_exporter-0.14.0.darwin-amd64/mysqld_exporter /usr/local/bin/

# Create a MySQL User for Exporter:
CREATE USER 'exporter'@'localhost' IDENTIFIED BY 'ComplexPassword123!';
GRANT PROCESS, REPLICATION CLIENT, SELECT ON *.* TO 'exporter'@'localhost';
FLUSH PRIVILEGES;

# Create a .my.cnf file for the exporter
sudo nano /usr/local/bin/.my.cnf
user=exporter
password=ComplexPassword123!

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
sysbench --mysql-host=127.0.0.1 --mysql-port=3306 --mysql-user=root --mysql-password=ComplexPassword123! --mysql-db=test_db --table-size=100000 --tables=10 --threads=6 --time=60 --events=0 --report-interval=10 oltp_read_write prepare

# Run the Load Test

sysbench --mysql-host=127.0.0.1 --mysql-port=3306 --mysql-user=root --mysql-password=ComplexPassword123! --mysql-db=test_db --table-size=100000 --tables=10 --threads=6 --time=60 --events=0 --report-interval=10 oltp_read_write run

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
