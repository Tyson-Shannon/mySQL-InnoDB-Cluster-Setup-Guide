# mySQL-InnoDB-Cluster-Setup-Guide
Setup guide for a mySQL InnoDB cluster on 3 Ubuntu server VM's with mySQL workbench connection
</br>
## Pre-Req
1. 3 existing Ubuntu Server VM's
2. Take a snapshot of each VM's current state in case of errors
3. Ensure NAT is set instead of Host-Only for the first step (may require restart and setting change)

*NOTE: use only one network adapter and change between NAT and Host-Only. Using 2 network adapters may cause a DNS issue a require restoring from snapshot*

If your VM's have an existing setup like mariaDB remove it and its DB's (one command at a time)
```CLI
sudo systemctl stop mariadb
sudo apt purge mariadb-server mariadb-client mariadb-common -y
sudo apt autoremove -y
sudo rm -rf /var/lib/mysql
```
Ping google.com to ensure NAT is working
```CLI
ping google.com
```
On all nodes download mySQL and mySQL shell
```CLI
sudo apt update
sudo apt install mysql-server mysql-shell -y
```
Check install was sucssessful (optional but good practice)
```CLI
mysql --version
mysqlsh --version
```
Shut down all nodes and replace NAT with Host-Only </br>
*NOTE: allow all for promiscuous mode in adapter settings or cluster wont connect*
```CLI
sudo shutdown now
```

## Configure
Start up all nodes (VM's) in order of primary, node 2, node 3 </br>
*NOTE: wait for each node to get ip before starting next on with the 'ip a' command* </br>
Make record of each nodes IP it will be needed later but should look like 192.168.56.X </br>
From this point on it is suggested to use putty to ssh into your VM's so you can copy and paste configuration details. </br>
On all nodes set up secure installation
```CLI
sudo mysql_secure_installation
```
Recommended answers:
- Password validation -> Y -> 0
- Remove anonymous users -> Y
- Disallow root remote login -> N
- Remove test DB -> Y
- Reload privileges -> Y

Configure mySQL for InnoDB on all nodes
```CLI
sudo nano /etc/mysql/mysql.conf.d/mysqld.cnf
```
Under [mysqld] add for each host using thier ip
```CLI
report_host=192.168.56.100
```
Change bind-address to:
```CLI
bind-address=0.0.0.0
```
Copy the below into a txt document, make changes where needed to ip's, id's and cluster name, paste at bottom of the mysqld.cnf file that you opened
```txt
# UNIQUE PER NODE (VM2=2, VM3=3)
server-id=1

# REQUIRED FOR GROUP REPLICATION
log_bin=binlog
binlog_format=ROW
gtid_mode=ON
enforce_gtid_consistency=ON

# INNODB CLUSTER SETTINGS
transaction_write_set_extraction=XXHASH64

# GROUP REPLICATION (change to your cluster name
loose-group_replication_group_name="tysonshannoncluster"

# CHANGE PER NODE
loose-group_replication_local_address="192.168.56.100:33061"

# ALL NODES LISTED (change to your ip's)
loose-group_replication_group_seeds="192.168.56.100:33061,192.168.56.104:33061,192.168.56.105:33061"

loose-group_replication_start_on_boot=off
```
Exit and save</br>
*NOTE: most errors will occur here, formatting is very picky and changing a tab to a space can break the cluster*</br>
Restart mySQL on all nodes
```CLI
sudo systemctl restart mysql
```
*NOTE: if this freezes try restarting vm and trying again*</br>
Log in as root to mySQL on all nodes
```CLI
sudo mysql -u root -p
```
Create admin user on all nodes
```SQL
CREATE USER 'clusteradmin'@'%' IDENTIFIED BY 'StrongPass123!';
GRANT ALL PRIVILEGES ON *.* TO 'clusteradmin'@'%' WITH GRANT OPTION;
FLUSH PRIVILEGES;
QUIT;
```
If necessary open your ports in your firewall for all nodes
```CLI
sudo ufw allow 3306
sudo ufw allow 33061
sudo ufw reload
```
Launch mysqlsh on all nodes
```CLI
mysqlsh
```
Ensure you are on python mode (Py) (SQL mode is \sql)
```python
\py
```
Configure the nodes with this command (change the ip for each)
```python
dba.configure_instance('clusteradmin@192.168.56.100:3306')
```
Prompts:
- Enter password -> StrongPass123!
- Save password -> Y
- “Do you want to perform the required configuration changes?” -> y

## Create Cluster
On the primary node or VM 1 ONLY </br>
Connect to mySQl server
```python
shell.connect('clusteradmin@192.168.56.100:3306')
```
Create cluster
```python
cluster = dba.create_cluster('tysonshannoncluster')
```
Add other nodes to cluster (still from VM1 only)
```python
cluster.add_instance('clusteradmin@192.168.56.104:3306')
cluster.add_instance('clusteradmin@192.168.56.105:3306')
```
Select C (clone) data when prompted </br>
Verify cluster
```python
cluster.status()
```
You should see 3 nodes in the JSON that is returned with VM1 stated as primary and the others stated as secondary </br>
From this point on nodes should be shut done in reverse order (3-1)
```CLI
sudo systemctl stop mysql
sudo shutdown now
```
And started in order (1-3)
```CLI
sudo systemctl start mysql
```
If nodes don't connect automatically you can add them back or reboot whole cluster
```python
cluster.rejoinInstance('clusteradmin@192.168.56.102:3306')
dba.rebootClusterFromCompleteOutage()
```

## Testing the Cluster
Connect into primary node
```python
shell.connect('clusteradmin@192.168.56.100:3306')
session.run_sql("CREATE DATABASE testcluster;")
session.run_sql("USE testcluster;")
session.run_sql("""
CREATE TABLE users (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(50)
);
""")
session.run_sql("""
INSERT INTO users (name) VALUES 
('Tyson'), ('NodeTest'), ('ReplicationCheck');
""")
\exit
```
On other nodes
```python
shell.connect('clusteradmin@192.168.56.104:3306')
session.run_sql("USE testcluster;")
session.run_sql("SELECT * FROM users;")
\exit
```

## Connect mySQL Workbench
### Using a direct connection
In MySQL Workbench:
- Click + (New Connection)

Enter:
- Hostname: 192.168.56.100 (PRIMARY node)
- Port: 3306
- Username: clusteradmin
- Password: (store it)

Click “Test Connection”</br>
If it works click "Ok" and scroll down to open your connection and write SQL into the distributed DB!
