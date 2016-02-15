---
layout: post
date: 2014-03-21
title: Installing MariaDB Galera Cluster on Windows Azure
---

## Install MariaDB Galera Cluster and create a VM Image

First, we need to install the MariaDB/Galera bits on a virtual machine that we will use as an image to create all the instances in our cluster. Let's create an Ubuntu 12.04 VM:

	azure vm create mariadb b39f27a8b8c64d52b05eac6a62ebad85__Ubuntu-12_04_3-LTS-amd64-server-20140130-en-us-30GB \
	--location "West Europe" --vm-size small --ssh 22 --ssh-cert myCert.pem azureuser

Then we will install the MariaDB Galera Cluster by following the [installation instructions](https://mariadb.com/kb/en/getting-started-with-mariadb-galera-cluster/):

	sudo apt-get install python-software-properties
	sudo apt-key adv --recv-keys --keyserver hkp://keyserver.ubuntu.com:80 0xcbcb082a1bb943db
	sudo add-apt-repository 'deb http://mirrors.linsrv.net/mariadb/repo/5.5/ubuntu precise main'
	sudo apt-get update
	sudo apt-get install mariadb-server

We will now deprovision, shut down and capture the VM image.

In the VM, run:

	sudo waagent -deprovision

Then log off and from your admin workstation:

	azure vm shutdown mariadb
	azure vm capture mariadb mariadb-galera-image

## Create the Virtual Machines

Now that we have a base image, let's start the virtual machines that will form the cluster. Looking at the [Galera Cluster deployment variants](http://www.codership.com/wiki/doku.php?id=galera_deployment), I am aiming for variant 3a, i.e. a three-node cluster with a distributed load-balancer, in this case using the JDBC driver built-in load balancing capabilities. However I would also like to be able to test variant 3, using the Windows Azure load-balancer. This means I need to have the VMs in the same Cloud Service (to be able to load-balance the traffic) and in a Virtual Network (so I can address each node with a stable IP).

I am going to use three nodes, as recommended in the [Getting Started](https://mariadb.com/kb/en/getting-started-with-mariadb-galera-cluster/) document to avoid "split-brain" situations. However, note that Galera also provides an [arbitrator](http://www.codership.com/wiki/doku.php?id=galera_arbitrator) that can be used as a lightweight member instead of a full node.

Let's create an Affinity Group:

	azure account affinity-group create --location "West Europe" galerawest

And a VNet:

	azure network vnet create --address-space 10.0.0.0 --cidr 8 --subnet-name mariadb --subnet-start-ip 10.0.0.0 \
	--subnet-cidr 24 --affinity-group galerawest galeravnet

And now will create three VMs for the cluster. We need to put them in the right VNet/subnet, and we use the `--connect` option to put them in the same Cloud Service.

	azure vm create -v --vm-name mariadb1 --virtual-network-name galeravnet --subnet-names mariadb --affinity-group galerawest \
	--vm-size small --ssh 22 --ssh-cert ./certs/myCert.pem --no-ssh-password mariadbcluster mariadb-galera-image tom

	azure vm create -v --vm-name mariadb2 --virtual-network-name galeravnet --subnet-names mariadb --affinity-group galerawest \
	--vm-size small --ssh 23 --ssh-cert ./certs/myCert.pem --no-ssh-password --connect mariadbcluster mariadb-galera-image tom

	azure vm create -v --vm-name mariadb3 --virtual-network-name galeravnet --subnet-names mariadb --affinity-group galerawest \
	--vm-size small --ssh 24 --ssh-cert ./certs/myCert.pem --no-ssh-password --connect mariadbcluster mariadb-galera-image tom

We now have our three nodes. For the sake of simplicity, we won't add a DNS server to the setup and just use the IP addresses of our machines. In our case, these will be 10.0.0.4, 10.0.0.5 and 10.0.0.6.

## Configuring the cluster

We are now going to configure our cluster. I used Codership's [Galera configuration page](http://www.codership.com/wiki/doku.php?id=mysql_galera_configuration) as reference, as well as a couple how-to articles:

- http://planet.mysql.com/entry/?id=616914
- https://www.digitalocean.com/community/articles/how-to-configure-a-galera-cluster-with-mariadb-on-ubuntu-12-04-servers
- http://www.severalnines.com/clustercontrol-mysql-galera-tutorial

First, stop the mysqld that is probably running on all the nodes after the initial installation:

	sudo service mysql stop

We are going to create a new configuration file for the cluster parameters, i.e.:

	vi /etc/mysql/conf.d/cluster.cnf

And here is the configuration I used:

	[mysqld]
	query_cache_size=0
	binlog_format=ROW
	default-storage-engine=innodb
	innodb_autoinc_lock_mode=2
	query_cache_type=0
	bind-address=0.0.0.0

	# Galera Provider Configuration
	wsrep_provider=/usr/lib/galera/libgalera_smm.so
	#wsrep_provider_options="gcache.size=32G"

	# Galera Cluster Configuration
	wsrep_cluster_name="test_cluster"
	wsrep_cluster_address="gcomm://10.0.0.4,10.0.0.5,10.0.0.6"

	# Galera Synchronization Congifuration
	wsrep_sst_method=rsync
	#wsrep_sst_auth=user:pass

	# Galera Node Configuration
	wsrep_node_address="10.0.0.4"
	wsrep_node_name="mariadb1"

As you can see, I gave my cluster a `wsrep_cluster_name` and listed the nodes' IP addresses in `wsrep_cluster_address`. You have copy this configuration file on all the nodes, and change the `wsrep_node_name` and `wsrep_node_address` parameters on each.

You should also copy Debian "maintenance" configuration from the first node to all the others, so that the credentials for the special `debian-sys-maint` user are the same on all nodes. Just copy the contents of `/etc/mysql/debian.cnf` to the other nodes. If you want to quickly edit a configuration file on a remote node as root, you can use:

	ssh -t 10.0.0.6 sudo vi /etc/mysql/debian.cnf

## Adding disks

I am solely focusing on testing clustering here, but at this point you should really attach a bunch of disks to each node, create a RAID array and change the `datadir` in MySQL's `my.cnf` in order to store the data properly.

Create Storage Accounts, one per VM:

	for i in {1..3}; do azure storage account create --affinity-group galerawest galerastor$i; done

Let's create 4 x 256GB disks per VM, each group of 4 going to a separate Storage Account:

	for j in {1..3}; \
	do for i in {1..4}; \
	do azure vm disk attach-new mariadb$j 256 http://galerastor$j.blob.core.windows.net/disks/mariadb-$j-disk-$i.vhd; \
	done; done

Once you have created and attached the disks, you should see four new devices appear; `/dev/sdc` to `/dev/sdf`:

	root@mariadb1:~# ls -l /dev/sd*
	brw-rw---- 1 root disk 8,  0 Mar 18 15:14 /dev/sda
	brw-rw---- 1 root disk 8,  1 Mar 18 15:14 /dev/sda1
	brw-rw---- 1 root disk 8, 16 Mar 18 15:14 /dev/sdb
	brw-rw---- 1 root disk 8, 17 Mar 18 15:14 /dev/sdb1
	brw-rw---- 1 root disk 8, 32 Mar 18 15:14 /dev/sdc
	brw-rw---- 1 root disk 8, 48 Mar 18 15:14 /dev/sdd
	brw-rw---- 1 root disk 8, 64 Mar 18 15:14 /dev/sde
	brw-rw---- 1 root disk 8, 80 Mar 18 15:14 /dev/sdf

We are now going to [create a RAID striped array using `mdadm`](https://raid.wiki.kernel.org/index.php/RAID_setup).

On each machine, install `mdadm`:

	apt-get install mdadm

On Ubuntu, APT insists on installing Postfix as a dependency; just select "No configuration" in the Postfix configuration screen that pops up (you can always run `dpkg-reconfigure postfix` later).

Now let's create the stripez:

	mdadm --create --verbose /dev/md0 --level=stripe --raid-devices=4 /dev/sdc /dev/sdd /dev/sde /dev/sdf

Save the configuration:

	mdadm --detail --scan >> /etc/mdadm/mdadm.conf

Create the filesystem, the mount point, and mount the disk:

	mkfs.ext4 /dev/md0
	mkdir /data
	mount -t ext4 /dev/md0 /data

If you already have some data, you will need to [move it to the new location](http://askubuntu.com/questions/137424/moving-mysql-datadir). I will go for the easy option of moving the whole datadir and pointing to the new location with a symlink. You should not have to reconfigure AppArmor as instructed in the link above.

	service mysql stop
	mv /var/lib/mysql /data
	ln -s /data/mysql /var/lib/mysql
	service mysql start

You can do this operation on each node, one by one.

## Starting the cluster

Now let's start the cluster on the first node, using `--wsrep-new-cluster` to create the cluster.

	sudo service mysql start --wsrep-new-cluster

And on the other nodes:

	sudo service mysql start

You should now be able to see the three nodes in the cluster:

	mysql -u root -p -e 'SELECT VARIABLE_VALUE as "cluster size" FROM INFORMATION_SCHEMA.GLOBAL_STATUS WHERE VARIABLE_NAME="wsrep_cluster_size"'
	+--------------+
	| cluster size |
	+--------------+
	| 3            |
	+--------------+

## Testing the cluster

Let's create a test database and table:

	mysql -u root -p -e 'CREATE DATABASE playground;'
	mysql -u root -p -e 'CREATE TABLE playground.jdbctests ( id INT NOT NULL AUTO_INCREMENT, t VARCHAR(64), h VARCHAR(64), d DATE, PRIMARY KEY(id) );'

Now you should be able to connect to any node, insert data in the table, select from another node, etc., and make sure the replication works. If you need to allow access to the database:

	grant all on playground.* to 'root'@'%' identified by '<password>';

Let's use a little test Java program too see how load-balancing works. You can download the [MySQL JDBC driver](http://dev.mysql.com/downloads/connector/j/), and read up on the [multi-host connections](http://dev.mysql.com/doc/connector-j/en/connector-j-multi-host-connections.html) capabilities.

I used some sample source code from the following blog post:

- http://openlife.cc/blogs/2012/june/howto-use-mysql-jdbc-loadbalancer-galera-multi-master-clusters

I changed the output messages somewhat and added logging of the node where the connection was established, so we can check our connections are load-balanced.

I also modified the JDBC URL:

- listed the nodes we want to load-balance
- added `loadBalanceBlacklistTimeout` (see [How to use JDBC (Connector/J) with MySQL Cluster](https://blogs.oracle.com/carriergrademysql/entry/how_to_use_jdbc_connector))
- added `connectTimeout` in case a node is down

Here is the code:

~~~java
import java.sql.*;

public class MysqlTest
{
    public static void main (String[] args)
    {
        Connection conn = null;

        while(true)
        {
            // Connect to MySQL
            try
            {
                String userName = "root";
                String password = "xxx";
                String hosts = "10.0.0.4,10.0.0.5,10.0.0.6";
                String url = "jdbc:mysql:loadbalance://" + hosts + "/playground?loadBalanceBlacklistTimeout=30000&connectTimeout=1000";
                Class.forName ("com.mysql.jdbc.Driver").newInstance ();
                conn = DriverManager.getConnection (url, userName, password);
                System.out.println ("Connected to " + conn.getMetaData().getURL());
            }
            catch (Exception e)
            {
                System.err.println ("Cannot connect to database server: " + e);
            }

            try
            {
                Statement s = conn.createStatement ();
                int count;
                count = s.executeUpdate("INSERT INTO jdbctests (t, h, d) VALUES ('Inserted from Java code.', @@hostname, now());");
                s.close ();
                //System.out.println (count + " rows were inserted");
            }
            catch (SQLException e)
            {
                System.err.println ("Error when executing SQL statement: " + e);
            }

            // Close connection
            if (conn != null)
            {
                try
                {
                    conn.close ();
                    //System.out.println ("Database connection terminated");
                }
                catch (Exception e)
                {
                    System.err.println( "Error when closing database connection: " + e );
                }
            }
            try { Thread.sleep(1000); } catch( Exception e ) { }
        }
    }
}
~~~

You can compile and run the program like this:

	javac MysqlTest.java
	java -cp ./mysql-connector-java-5.1.29/mysql-connector-java-5.1.29-bin.jar:. MysqlTest

You should see connections going out to all the nodes in the cluster at random.

You can test the behavior in case of node failure by shutting down one of the nodes (e.g. from the admin portal). You should not see any errors, just one of the nodes disappearing from the log.

### Failover

To use failover instead of load-balancing, you can change the JDBC URL to:

	jdbc:mysql://" + hosts + "/playground?loadBalanceBlacklistTimeout=30000&connectTimeout=1000

i.e. remove the "loadbalance" option. However this will use standard failover where the secondary host is considered a read-only slave.

To configure multi-master failover:

	String hosts = "address=(type=master)(host=10.0.0.4)(protocol=tcp)(port=3306),\
	address=(type=master)(host=10.0.0.5)(protocol=tcp)(port=3306),\
	address=(type=master)(host=10.0.0.6)(protocol=tcp)(port=3306)";
	String url = "jdbc:mysql://" + hosts + "/playground?autoReconnect=true&failOverReadOnly=false&connectTimeout=1000";

This extended syntax allows you to specify that the failover hosts are also masters.

## Using the Windows Azure load balancer

As discussed above, you can also take advantage of the integrated Windows Azure load balancer (WALB) to implement the connection to the cluster.

Let's create a load-balancer set:

	azure vm endpoint create --lb-set-name mysql mariadb1 3306
	azure vm endpoint create --lb-set-name mysql mariadb2 3306
	azure vm endpoint create --lb-set-name mysql mariadb3 3306

Now you can use your Cloud Service address, mariadbcluster.cloudapp.net in my case, to connect to your cluster. The connections will automatically be load-balanced on all nodes using a round-robin algorithm.

## Running some benchmarks

Here are some notes on how to benchmark your MariaDB cluster using [sysbench](http://sysbench.sourceforge.net/docs/#database_mode).

Install `sysbench`:

	sudo apt-get install sysbench
	sudo apt-get install mysql-client-core-5.5

Create a test database using the `mysql` client:

	create database sysbench;
	grant all on sysbench.* to 'root'@'%' identified by 'xxx!';

I will run the benchmark client using the load-balanced endpoint address that I created above.

Prepare the benchmark, e.g.:

	sysbench --test=oltp --oltp-table-size=1000000 --mysql-host=mariadbcluster.cloudapp.net --mysql-db=sysbench \
	--mysql-user=root --mysql-password='xxx' prepare

Run the benchmark, e.g. 1 minute, 4 threads, simple test:

	sysbench --test=oltp --oltp-table-size=1000000 --mysql-host=mariadbcluster.cloudapp.net --mysql-db=sysbench \
	--mysql-user=root --mysql-password='xxx' --max-time=60 --oltp-read-only=on --num-threads=4 --max-requests=0 run

5 minutes, 16 threads, complex test:

	sysbench --test=oltp --oltp-table-size=1000000 --mysql-host=mariadbcluster.cloudapp.net --mysql-db=sysbench \
	--mysql-user=root --mysql-password='xxx' --max-time=300 --oltp-read-only=on --num-threads=16 \
	--max-requests=0 --oltp-test-mode=complex run
