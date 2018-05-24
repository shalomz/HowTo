

[Source](http://www.archaeogeek.com/blog/2011/08/11/setting-up-a-postgresql-standby-servers/ "Permalink to setting up a postgresql standby servers")

# setting up a postgresql standby servers

Over the last couple of months I have been investigating options for setting up a standby server for PostgreSQL, you know, the sort of magical thing that stops your day/night/week being totally wrecked when, to quote [Joel Spolsky][1]  you "go crying to the system administrator and asking piercingly sad questions about _why_ the backup system is "temporarily" out of commission and has been for the last eight months". As a relative beginner to all of this, if you look in the PostgreSQL documentation, you will get totally overwhelmed and confused (well I did, anyway) as the information you need is spread across several chapters and is dense, even by the PostgreSQL documentation's standards. So, when I found some documentation that was actually quite clear and concise, and got me through the process without too much angst, I thought I'd record my notes here. **Big disclaimer- these are by no means comprehensive or complete. No flaming!**

**Useful links**

[Easier to follow but less comprehensive guide][2]

**Basic Idea** We are setting up a **warm standby** postgresql server with **streaming replication**. In the event of a failure of the primary server, the standby server will take over but will not be queryable until that point. Streaming replication reduces the window of data loss between a primary server failing and the standby server being in a position to take over.

**Important Notes** Additional network and operating system specific software will also be required to handle the change in IP address and the trigger to move the standby server into read/write mode (see later). The two servers must be as identical as possible. In particular the same version of postgresql must be installed and the servers must have the same architecture (32 or 64bit).

**Primary Server Preparation: postgresql.conf**

Set up continuous archiving on the primary server by setting the following parameters in postgresql.conf (note that the archive_command line has been split over two lines for readability- in real life that would be one line):
    
    
     archive_command = on
     archive_command = 'cp %p
    
    
    
    
       /location/where/write-ahead-logfiles/should/go/%f'
     wal_level = archive
     max_wal_senders = 5
     wal_keep_segments = 32
     listen_addresses='*'
    

The location of the write-ahead-logfiles (WAL) should be accessible to both the primary and the standby server, and preferably remote from the primary server (for obvious reasons). The %p and %f symbols are specific to the archive_command and are substituted for the path and the file name respectively when the command is executed.

**Primary Server Preparation:** **pg_hba.conf**

Add a connection from the standby server to the main server as follows:

host replication [user] [addressrange] md5

The database name "replication" should not be changed- it's a pseudo-database connection specifically to allow replication to take place and does not reflect an actual database.

**Standby server configuration**

Ensure that the location of the WAL files are accessible from the standby server and that a connection can be made to the database on the primary server.

Create a file called "recovery.conf" in the data directory for the standby server. This should have the following parameters as a minimum (the primary_conninfo command has been split over two lines here for readability):
    
    
     standby_mode='on'
     primary_conninfo = 'host=primaryhostIP user=databaseuser
    
    
    
    
       password=databasepassword port=port'
     trigger_file='/tmp/psql.trigger'
     restore_command = 'cp /path/to/WALfiles/%f %p'
    

Substitute the correct connection details in primary_conninfo, and the path to the remote WAL logs as appropriate. Also substitute operating-system specific file copy commands in the restore_command entry. The trigger file does not have to exist at this point. Its purpose is to tell the standby server when to move into read/write mode (see later). You may also wish to replicate the settings from the primary server's pg_hba.conf to ensure that all connections will be allowed in the event that the standby server is used.

**Take a base backup of the primary server**

A base backup is not the same as a database dump. It is a backup of all the files in the postgresql data directory. The location of this can vary. In linux, you can find it by running the following command at a command prompt:
    
    
     ps auxw | grep postgres | grep -- -D
    

In any operating system, if you can connect to the database server (eg with pgadmin3) then enter the following SQL command:
    
    
     SHOW data_directory;
    

You can take a backup of the data directory when the server is running, or when it is stopped. It is simplest to take it when the database server is stopped (and it need not be stopped for very long).

**Backing up data directory when postgresql is stopped**

* Stop the database in the appropriate way (eg /etc/init.d/postgresql stop on linux or stop the service in windows).
* Backup the entirety of the data directory as found above, using whatever method you like, such as tar.
* Copy the backup file to somewhere accessible to the standby server
* Restart the database

**Backing up the data directory without stopping postgresql**

* Connect to the database server as the postgres user and issue the following command (where 'label' is anything you want to label the backup in the log files with):

SELECT pg_start_backup('label',true);

* Disconnect from the server and back up the contents of the data directory as above
* Connect to the database server as the postgres user and issue the following command:

SELECT pg_stop_backup();

* Copy the backup file to somewhere accessible to the standby server

**Restoring the backup to the standby server**

* Stop the postgresql service on the standby server by the appropriate means.
* Replace the contents of the database directory with that from the backup, being sure not to overwrite recovery.conf
* Start the postgresql service.

If you watch the log files on the standby server you should see that it will reach a stage where it is successfully connected to the primary server. Errors about the cp command not being able to find archive files are usually non-fatal, as are entries about zero-length logfiles. However, if your log files show other errors, then it is likely that the base backup has not been correctly restored. Try redoing this with the postgresql service stopped to avoid issues.

**Note**

We have set up a warm standby server in this scenario. This means that you cannot connect to the standby database with psql or pgadmin3 to check that it is working! The log files will simply show a cycle of synchronising with the WAL files from the remote location, and then attempting replication via TCP with the primary server.

**How to bring up the standby server**

In the event of a failure in the primary server, something (e.g. a heartbeat process) needs to create the trigger_file in the location specified in the standby server recovery.conf file. This can be an empty file, so on linux a simple "touch" command will be enough. Once the standbyserver recognises this file, it will switch to read/write mode. This is clear from the log files, and database connections will be allowed.

The recovery.conf file will be automatically renamed recovery.done once successful recovery has taken place.

In the event that the primary server is reinstated, it will be necessary to synchronise the two databases- the base backup procedure run on the standby server should be sufficient.

[1]: http://www.joelonsoftware.com/
[2]: http://eggie5.com/15-setting-up-pg9-streaming-replication

  
