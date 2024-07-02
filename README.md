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

# Step 2: Configure MySQL Master and Slaves

# Initialize MySQL Data Directories for Slaves
mkdir -p /opt/homebrew/var/mysql_slave1
mkdir -p /opt/homebrew/var/mysql_slave2
mysqld --initialize-insecure --datadir=/opt/homebrew/var/mysql_slave1
mysqld --initialize-insecure --datadir=/opt/homebrew/var/mysql_slave2

# Edit MySQL Configuration Files

# Create /opt/homebrew/etc/my.cnf for the master
echo "[mysqld]
server-id=1
log-bin=mysql-bin
binlog_do_db=test_db" > /opt/homebrew/etc/my.cnf

# Create /opt/homebrew/etc/my_slave1.cnf for the first slave
echo "[mysqld]
server-id=2
relay-log=mysql-relay-bin
log-bin=mysql-bin
binlog_do_db=test_db" > /opt/homebrew/etc/my_slave1.cnf

# Create /opt/homebrew/etc/my_slave2.cnf for the second slave
echo "[mysqld]
server-id=3
relay-log=mysql-relay-bin
log-bin=mysql-bin
binlog_do_db=test_db" > /opt/homebrew/etc/my_slave2.cnf

# Start MySQL Instances
mysqld --defaults-file=/opt/homebrew/etc/my.cnf &
mysqld --defaults-file=/opt/homebrew/etc/my_slave1.cnf &
mysqld --defaults-file=/opt/homebrew/etc/my_slave2.cnf &

# Configure Replication

# Connect to the master and set up replication user
mysql -u root -p -e "
CREATE USER 'replica'@'%' IDENTIFIED BY 'ComplexPassword123!';
GRANT REPLICATION SLAVE ON *.* TO 'replica'@'%';
FLUSH PRIVILEGES;
SHOW MASTER STATUS;"

# Configure slaves
mysql -u root -p -h 127.0.0.1 -P 3307 -e "
STOP SLAVE;
CHANGE MASTER TO 
  MASTER_HOST='127.0.0.1', 
  MASTER_USER='replica', 
  MASTER_PASSWORD='ComplexPassword123!', 
  MASTER_LOG_FILE='mysql-bin.000001', 
  MASTER_LOG_POS=834;
START SLAVE;"

mysql -u root -p -h 127.0.0.1 -P 3308 -e "
STOP SLAVE;
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
echo "global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'mysql'
    static_configs:
      - targets: ['localhost:9104']" > /opt/homebrew/etc/prometheus.yml

# Start Prometheus
brew services start prometheus

# Install and Configure MySQL Exporter

# Download and move the MySQL exporter binary
curl -LO https://github.com/prometheus/mysqld_exporter/releases/download/v0.14.0/mysqld_exporter-0.14.0.darwin-amd64.tar.gz
tar xvf mysqld_exporter-0.14.0.darwin-amd64.tar.gz
sudo mv mysqld_exporter-0.14.0.darwin-amd64/mysqld_exporter /usr/local/bin/

# Create a .my.cnf file for the exporter
echo "[client]
user=exporter
password=ComplexPassword123!" > ~/.my.cnf

# Run MySQL Exporter
mysqld_exporter --config.my-cnf /usr/local/bin/.my.cnf &

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

# Start Prometheus and Grafana
brew services start prometheus
brew services start grafana

# Verify Setup
# Check MySQL master and slave status
# Ensure Prometheus is running at http://localhost:9090
# Ensure Grafana is running at http://localhost:3000
