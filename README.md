
# Implementing High available multi master (active/active) Postgres SQL Database
Note: These instructions are allowing any IPs to access the server so I used 0.0.0.0/0 for sake of making the deployment smooth

Download Source Code Installation version PostgreSql 9.4.21 Bdr plugin 1.0.7

Server-01 IP 172.16.16.71

Server-02 IP 172.16.16.72

Create User postgres on both servers

```
  	adduser postgres
	adduser postgres sudo 
	sudo usermod -a -G sudo postgres
	sudo mkdir -p /var/lib/postgresql
	sudo chown postgres:postgres /var/lib/postgresql
	sudo usermod -d /var/lib/postgresql postgres
```

Then switch to that postgres user 

```
	su -l postgres
```

```
wget https://github.com/2ndQuadrant/bdr-postgres/archive/bdr-pg/REL9_4_21-1.tar.gz && tar -xvzf REL9_4_21-1.tar.gz && rm -f REL9_4_21-1.tar.gz & wget https://github.com/2ndQuadrant/bdr/archive/bdr-plugin/1.0.7.tar.gz && tar -xvzf 1.0.7.tar.gz && rm -f 1.0.7.tar.gz
sudo apt-get -y install apt-transport-https && sudo apt-get -y install curl ca-certificates  && curl https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -
```
```
sudo su 
sudo touch /etc/apt/sources.list.d/pgdg.list
cat > /etc/apt/sources.list.d/pgdg.list <<EOF
deb http://apt.postgresql.org/pub/repos/apt/ bionic-pgdg main
deb-src http://apt.postgresql.org/pub/repos/apt/ bionic-pgdg main
EOF

```
	su -l postgres
```


sudo apt-get update && sudo apt-get -y build-dep postgresql-9.4
cd bdr-postgres-bdr-pg-REL9_4_21-1 && sudo ./configure --prefix=/usr/lib/postgresql/9.4 --enable-debug --with-openssl
sudo make check
sudo make -j4 -s install-world
sudo apt-get update
```

Then Install BDR
  ```
cd /home/ubuntu/bdr-bdr-plugin-1.0.7
sudo PATH=/usr/lib/postgresql/9.4/bin:"$PATH" ./configure
   sudo make -j4 -s all
   sudo make -s install

  
	su -l postgres
	export PATH=/usr/lib/postgresql/9.4/bin:$PATH
	 mkdir -p $HOME/9.4-bdr
	initdb -D ~/9.4-bdr -A trust
```
	

Edit the postgresql.conf file for both nodes/instances:
```
vi $HOME/9.4-bdr/postgresql.conf
	listen_addresses = '*' 
    shared_preload_libraries = 'bdr'
    wal_level = 'logical'
    track_commit_timestamp = on
    max_connections = 100
    max_wal_senders = 10
    max_replication_slots = 10
    # Make sure there are enough background worker slots for BDR to run
    max_worker_processes = 10

    # These aren't required, but are useful for diagnosing problems
    #log_error_verbosity = verbose
    #log_min_messages = debug1
    #log_line_prefix = 'd=%d p=%p a=%a%q '

    # Useful options for playing with conflicts
    #bdr.default_apply_delay=2000   # milliseconds
    #bdr.log_conflicts_to_table=on
   ```
```
vi $HOME/9.4-bdr/pg_hba.conf
	
	host replication bdrsync 0.0.0.0/0 password
	host replication bdrsync 0.0.0.0/0 password
    local   replication   postgres                  trust
    host    replication   postgres     0.0.0.0/0 trust
    host    replication   postgres     ::1/128      trust

	host replication bdrsync 0.0.0.0/0 password
```
Now repeat above steps to the second sever
!!!!!!IMPORTANT!!!!!
Do not start the server unless you completed above and the second server is done
    	
		
If everything is ready now let us start the server
```
su -l postgres
export PATH=/usr/lib/postgresql/9.4/bin:$PATH
pg_ctl -l ~/log -D ~/9.4-bdr start
psql -c "CREATE USER bdrsync superuser;"
psql -c "ALTER USER bdrsync WITH PASSWORD '123';"
```




===============ON Server 1 & 2==========================
```
createuser test_user
createdb -O test_user test_db
psql test_db -c 'CREATE EXTENSION btree_gist;'
psql test_db -c 'CREATE EXTENSION bdr;'
```
===============On Server 1 - Master Node Only ===============
```
psql
\c test_db
SELECT bdr.bdr_group_create(
    local_node_name := 'node1',
    node_external_dsn := 'host=172.16.16.71 user=bdrsync dbname=test_db password=123'
);
```

==============On Server 2 =======Join it to master node1===============
```
psql
\c test_db
SELECT bdr.bdr_group_join(
    local_node_name := 'node2',
    node_external_dsn := 'host=172.16.16.72 user=bdrsync dbname=test_db password=123',
    join_using_dsn := 'host=172.16.16.71 user=bdrsync dbname=test_db password=123'
);
```


 
Verify 
```
SELECT bdr.bdr_node_join_wait_for_ready();
select * from bdr.bdr_nodes;
select * from bdr.bdr_connections;
```

2.7. Testing your BDR-enabled system
Create a table and insert rows from your first node/instance:

```
      CREATE TABLE t1bdr (c1 INT, PRIMARY KEY (c1));
      INSERT INTO t1bdr VALUES (1);
      INSERT INTO t1bdr VALUES (2);
      -- you will see two rows
      SELECT * FROM t1bdr;
```
