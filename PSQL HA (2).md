

[Source](http://pgsnaga.blogspot.co.ke/2010/05/5-steps-to-implement-postgresql.html "Permalink to 5 steps to implement a PostgreSQL replication system")

# 5 steps to implement a PostgreSQL replication system

![][1]

Yesterday, I tried to build a PostgreSQL master-slave replication system with new feature "Streaming Replication (SR)", which is coming in the next PostgreSQL major release 9.0 as a built-in replication solution, and it worked well on my servers.

I feel it is very easy to configure, so I have decided to make some memos here for building a PostgreSQL master-slave replication system.

Here are 5 steps to implement a PostgreSQL replication system.  

1. Build and install binaries.
2. Initialize a database cluster and duplicate it.
3. Configure the master node.
4. Configure the slave node(s).
5. Start the master and slave node(s).
Ok, let's start.

(This photo was taken at the PostgreSQL 9 TestFest Japan.)  
  
_**How it works**_

Taking a minute to understand how SR works before building it could be very helpful to you.  

* SR enables to have a single master (read-write) node, and multiple slave (read-only) nodes.
* The master node sends transactional log (WAL) records generated on the master to the slave node(s).
* The slave is waiting for the log records and applying them continuously. (hot standby mode)
* During the hot standby mode, the slave can also serve to client applications to process read-only queries.
So you have to configure one master node, and one or more slave nodes to implement the SR.

_**Step 1. Build and install binaries**_

At first, you have to install PostgreSQL binaries on each node. To install them, you can use build commands as usual.  

    
    
    $ ./configure –-prefix=/usr/local/pgsql90b1
    $ make ; make check
    $ su
    # make install

_**Step 2. Initialize a database cluster and duplicate it**_

In the 2nd step, you have to initialize a database cluster (database directory) on the master node, and duplicate it on the slave node(s).

After initializing a database cluster on the master node, you have to make a base backup to duplicate it on the slave(s). "A base backup" means a whole dump of the database cluster which is taken in the PostgreSQL manner. If you are not familiar with the PostgreSQL base backup, see "24.3.2. Making a Base Backup" in the PostgreSQL official manual for more details.  

So the protocol is shown in below. ($PGDATA is a database cluster directory you can specify as you like.)
    
    
    master$ initdb –D $PGDATA –-no-locale –-encoding=UTF8
    master$ pg_ctl –D $PGDATA start
    master$ psql –c "SELECT pg_start_backup('initial backup for SR')" template1
    master$ tar cvf pg_base_backup.tar $PGDATA
    master$ psql –c "SELECT pg_stop_backup()" template1

After taking the base backup on the master node, you have to duplicate (copy and extract) it on the slave node. In addition, you have to remove "postmaster.pid" file on the slave.
    
    
    slave$ tar xvf pg_base_backup.tar
    slave$ rm –f $PGDATA/postmaster.pid

_**Step 3. Configure the master node**_

On the master node, you have to configure two files, "postgresql.conf" and "pg_hba.conf".

_**postgresql.conf**_

Here is 6 entries in postgresql.conf to enable the master node.  

    
    
    listen_addresses = '*' # to accept a connection from the slave
    wal_level = hot_standby     # to generate WAL records for SR purpose.
    archive_mode = on           # to enable the archiving log mode.
    archive_command = 'cp %p /home/snaga/pgdata90b1/pg_xlogarch/%f'
                                # to specify the log archiving command.
    max_wal_senders = 5         # to specify the max number of the slave(s).
    wal_keep_segments = 32      # to specify the number of the previous WAL files to hold on the master.

See the official manual for more details.  
  
_**pg_hba.conf**_

You have to add an entry to accept a connection from the slave in pg_hba.conf. The database name must be "replication" here, and you have to specify IP addresses of the slave nodes.  

    
    
    host   replication    all    10.0.2.42/32        trust

  
_**Step 4. Configure the slave node(s)**_

You also have to modify two files on the slave node, "postgresql.conf" and "recovery.conf".

_**postgresql.conf**_

You have only one entry in postgresql.conf to modify.  

    
    
    hot_standby = on       # to allow read-only queries on the slave (standby) node.

_**recovery.conf**_

To enable the standby mode on the slave node, you have to make a "recovery.conf" file as below.  

    
    
    standby_mode = 'on'      # to enable the standby (read-only) mode.
    primary_conninfo = 'host=10.0.2.41 port=5432 user=snaga'
                             # to specify a connection info to the master node.
    trigger_file = '/tmp/pg_failover_trigger'
                             # to specify a trigger file to recognize a fail over.
    restore_command = 'cp /home/snaga/pgdata90b/pg_xlogarch/%f "%p"'
                             # to specify a recovery command.

Finally, the configuration is finished, and you can start both master and slave nodes.

_**Step 5. Start the master and slave node(s)**_

Start your master PostgreSQL and slave PostgreSQL. After starting both, you may find a server log record on each.

On the master node, you may find a server log record as below.  

    
    
    LOG: replication connection authorized: user=snaga host=10.0.2.42 port=55811

And on the slave node, you can find a record as below.
    
    
    LOG: streaming replication successfully connected to primary

If you can find them, congratulations! Well done, and it's time to take a break with some coffee. :-)

Enjoy your PostgreSQL!

**_More resources_**  

[1]: http://4.bp.blogspot.com/__8NhvlYdsEo/S-5KedWe7ZI/AAAAAAAAAIw/uPH5mf5Wok8/s320/aaa.jpg

  
