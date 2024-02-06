# patroni-postgres-etcd-haproxy-keepalived
### stable database with three nodes of etcd and postgres which are always sync and haproxy and keepalived for stability
### (For focal ubuntu)
### first you gotta update and upgrade your 
```bash
sudo apt update
sudo apt upgrade
```
### Edit all /etc/hosts For example for node1:
```bash
10.130.4.150  node1
10.130.4.151  node2
10.130.4.152  node3
```
### Install postgresql server software on node1,node2 and node3. Also stop postgresql service after installation:
```bash
sudo apt install postgresql-14 postgresql-contrib-14 -y
sudo systemctl stop postgresql
```
### Create a symlink on all 3 nodes between /usr/lib/postgresql/14/bin/ and /usr/sbin for patroni requirement:
```bash
sudo ln -s /usr/lib/postgresql/14/bin/* /usr/sbin
```
### Install patroni on node1,node2,node3
```bash
sudo apt -y install python3-pip python3-dev libpq-dev
pip3 install --upgrade pip
sudo pip install patroni
sudo pip install python-etcd
sudo pip install psycopg2
```
### we will install etcd on docker just intstall etcd to use etcd libraries
```bash
sudo apt -y install etcd
sudo systemctl stop etcd
sudo systemctl disable etcd
```
### Install haproxy on all three nodes
```bash
sudo apt -y install haproxy
```
### Configuration of patroni on node1 && node2 && node3
### Create /etc/patroni.yml and add below lines to patroni.yml
```bash
sudo vi /etc/patroni.yml
```
```bash
scope: postgres
namespace: /db/
name: node1
restapi:
  listen: 10.130.4.150:8008
  connect_address: 10.130.4.150:8008
etcd:
  host: 10.130.4.150:2379
bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 1048576
    postgresql:
    use_pg_rewind: true
  initdb:
    - encoding: UTF8
    - data-checksums
  pg_hba:
    - host replication replicator   127.0.0.1/32 md5
    - host replication replicator   10.130.4.150/0   md5
    - host replication replicator   10.130.4.151/0   md5
    - host replication replicator   10.130.4.152/0   md5
    - host all all   0.0.0.0/0   md5
  users:
    admin:
       password: admin
       options:
       - createrole
       - createdb
postgresql:
   listen: 10.130.4.150:5432
   connect_address: 10.130.4.150:5432
   data_dir:     /data/patroni
   pgpass:     /tmp/pgpass
   authentication:
    replication:
      username:   replicator
      password:     "A1qaz2wsx3edc"
    superuser:
      username:   postgres
      password:     "B1qaz2wsx3edc"
      parameters:
      unix_socket_directories:  '.'
tags:
   nofailover:   false
   noloadbalance:   false
   clonefrom:   false
   nosync:   false
   ```

   ### Create patroni data directory on node1,node2 and node3:
   ```bash
   sudo mkdir -p  /data/patroni
   sudo chown postgres:postgres /data/patroni/
   sudo chmod 700 /data/patroni/
   ```
   ###  Create systemd file for patroni on node1,node2,node3:
   #### sudo vi  /etc/systemd/system/patroni.service
   ```bash
   [Unit]
Description=Patroni Orchestration
After=syslog.target network.target
[Service]
Type=simple
User=postgres
Group=postgres
ExecStart=/usr/local/bin/patroni /etc/patroni.yml
KillMode=process
TimeoutSec=30
Restart=no
[Install]
WantedBy=multi-user.targ
```
## Then start patroni
```bash
sudo systemctl daemon-reload
sudo systemctl start patroni
```
### Add below lines in /etc/haproxy/haproxy.cfg on haproxy node1 and node2 and node3 
```bash
listen stats
      mode http
      bind *:7000
      stats enable
      stats uri /
listen postgres
      bind *:5000
      mode tcp
      option httpchk
      http-check expect status 200
      default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions
      server node1 10.130.4.150:5432 maxconn 100   check   port 8008
      server node2 10.130.4.151:5432 maxconn 100   check   port 8008
      server node3 10.130.4.152:5432 maxconn 100   check   port 8008
```
### Then restart the haproxy service
```bash
sudo systemctl restart haproxy
```
### And see the keepalived configuration for being sure about the stability of our three haproxy
```bash
vrrp_script chk_postgresql {
  script "killall -0 postgres"
  interval 2
  weight 2
}

vrrp_instance VI_1 {
  state MASTER
  interface ens160
  virtual_router_id 51
  priority 100
  advert_int 1
  authentication {
    auth_type PASS
    auth_pass YourAuthenticationPassword
  }
  virtual_ipaddress {
    10.130.4.169/24   # Virtual IP address for PostgreSQL access
  }
  track_script {
    chk_postgresql
  }
}
```

