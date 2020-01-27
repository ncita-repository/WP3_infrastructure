# XNAT Full Install
This is a step by step walk through of installing XNAT on Ubuntu 18.04 VM running under a vSphere hypervisor. A couple of optional steps are detailed in case they are relevant. It is assumed your system is up to date with patches at the start point. The main difference between this document and the [XNAT install guide](https://wiki.xnat.org/documentation/getting-started-with-xnat/xnat-installation-guide) is that I'm giving all commands issued so you can copy and paste to your terminal.

### Conventions
Prompts used in this document are as follows:
* `$` - my regular user account.
* `xnat$` - the XNAT account, owns all XNAT directories and runs Tomcat 7.
* `postgres$` - the PostgreSQL account
* `postgres=#` - the PostgreSQL account inside `psql`.
* `=>` - Commentary on the previous command or what to put into the file being edited.

Both my user and the XNAT account have `sudo` capability and I change users with `sudo su - username`.

I use `vi` as my default text editor. Feel free to use whatever you prefer.

### Before You Start
***Consider Vagrant***

If all you need is a test system you can spin up on your laptop or desktop rather than something you can deploy into production, seriously consider using [XNAT Vagrant](https://wiki.xnat.org/documentation/getting-started-with-xnat/running-xnat-in-a-vagrant-virtual-machine). It will do all of the following for you in just a couple of commands.

***Take snapshots!***

This install was performed on a virtual machine. Given the advantages of VMs (snapshots, movability, flexibility of hardware), I'd be unlikely to use a bare metal install for XNAT. All hypervisors I've used have the capability to take a snapshot of a managed VM, take advantage of it. A snapshot allows you to rollback to a previous stable state. How often you take snapshots and how long you keep them is up to you. ***Important:*** stop PostgreSQL before taking a VM snapshot! Failing to do this can result in a database that doesn't work after the VM snapshot is restored.

***Hardware Capabilities***

The virtual hardware chosen should reflect what you plan to use the system for. A small test system on your laptop or desktop might only need 1 CPU and 2G of RAM. A production system with lots of users will benefit from more resources. I'm using 4 CPUs and 16G of RAM as this is the spec of our existing production systems. You can always change the virtual hardware later.

***Pre-existing Systems***

My installation requires integration with our Research Data Storage (RDS) system that holds our existing data. It will be mounted via NFS and thus user and group IDs already exist and must be accounted for:

`UID: 2048 (xnat)`

`GID: 2049 (clinmag)`

## Pre-requisites
### User/Group
Create the user, set password, shell and sudo privileges.

***Simple Version***

    $ sudo groupadd xnat
    $ sudo useradd -c XNAT -g xnat -m xnat

***NFS Version***

	$ sudo groupadd -g 2048 clinmag
	$ sudo useradd -c XNAT -u 2049 -g clinmag -m xnat

***Common***

	$ sudo passwd xnat
	$ sudo vi /etc/passwd
		=> Change xnat shell to /bin/bash
	$ sudo vi /etc/group
		=> Add xnat to sudo

### PostgreSQL
Ubuntu's base install doesn't have the PostgreSQL repositories by default.

	$ sudo curl https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -
		=> Adds repo key
	$ sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt/ $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'
		=> Grab distros
	$ sudo apt-get update
	$ sudo apt-get install postgresql-9.6
	$ sudo passwd postgres
	$ sudo -u postgres psql
	    => Invokes the PostgreSQL client as the postgres user. Use Ctrl-D to quit.
	postgres=# \password postgres
		=> Enter password

Create the XNAT user and an empty database.

	$ su - postgres
	postgres$ createuser -D xnat
	postgres$ createdb -O xnat xnat
	postgres$ psql -c "ALTER USER xnat WITH PASSWORD 'password';"
	
### Java
	$ sudo apt-get install openjdk-8-jre-headless

### NFS
***Optional step***. Where possible, use the most specific directory you can from the server to mount the XNAT data. XNAT only needs to see its own data, not everyone else's as well.

	$ sudo apt-get install nfs-common
	$ sudo vi /etc/fstab
		=> rds.icr.ac.uk:/rdsfs01/DATA/DRI/UCCI/XNAT /rds nfs defaults 0 0
	$ sudo mount /rds

### Tomcat 7
Ubuntu's base install does not include Tomcat 7 by default.

	$ sudo vi /etc/apt/sources.list
		=> deb http://archive.ubuntu.com/ubuntu/ xenial main
		=> deb http://archive.ubuntu.com/ubuntu/ xenial universe
	$ sudo apt-get update
	$ sudo apt-get install tomcat7
	$ sudo systemctl stop tomcat7

Configure Tomcat 7 for use by XNAT. Heap memory minimum and maximum are set to match my virtual hardware: `-Xms4g -Xmx12g`. Note the maximum heap is smaller than physical RAM. 

***NFS version***: `TOMCAT7_GROUP=clinmag`.

	$ sudo vi /etc/default/tomcat7
		=> TOMCAT7_USER=xnat
		=> TOMCAT7_GROUP=xnat
		=> JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64
		=> JAVA_OPTS="-Djava.awt.headless=true"
		=> # Set CATALINA_OPTS for Tomcat-specific operations, mainly service start and run.
		=> CATALINA_OPTS="-Xms4g -Xmx12g -XX:+UseConcMarkSweepGC -XX:-OmitStackTraceInFastThrow"
		=> CATALINA_OPTS="${CATALINA_OPTS} -XX:+CMSClassUnloadingEnabled"
		=> CATALINA_OPTS="${CATALINA_OPTS} -Dxnat.home=/data/xnat/home"
		=> CATALINA_OPTS="${CATALINA_OPTS} -agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=8000"

Ensure that the XNAT account that will run Tomcat has ownership of the Tomcat directories. 

***NFS version***: replace `xnat.xnat` with `xnat.clinmag`.

	$ sudo chown -RH --dereference xnat.xnat /var/lib/tomcat7
	$ sudo chown --no-dereference xnat.xnat /var/lib/tomcat7/*
		=> Does not fully change ownership on link targets
	$ sudo su - xnat
	xnat$ cd /var/lib/tomcat7/conf; sudo chown -R xnat.xnat *
	xnat$ cd /var/lib/tomcat7/logs; sudo chown -R xnat.xnat *
	xnat$ cd /var/lib/tomcat7/work; sudo chown -R xnat.xnat *
	$ sudo systemctl start tomcat7

### Nginx
Nginx is the proxy that sits in front of XNAT.

	$ sudo apt-get install nginx
	$ sudo vi /etc/nginx/sites-available/xnat
		=> server {
		=>     listen 80;
		=>
		=>     location / {
		=>         proxy_pass                          http://localhost:8080;
		=>         proxy_redirect                      http://localhost:8080 $scheme://localhost;
		=>         proxy_set_header Host               $host;
		=>         proxy_set_header X-Real-IP          $remote_addr;
		=>         proxy_set_header X-Forwarded-Host   $host;
		=>         proxy_set_header X-Forwarded-Server $host;
		=>         proxy_set_header X-Forwarded-For    $proxy_add_x_forwarded_for;
		=>         proxy_connect_timeout               150;
		=>         proxy_send_timeout                  100;
		=>         proxy_read_timeout                  100;
		=>         proxy_buffers                       4 32k;
		=>         client_max_body_size                0;
		=>         client_body_buffer_size             128k;
		=>     }
		=> }
	$ cd /etc/nginx/sites-enabled
	$ sudo ln -s ../sites-available/xnat xnat
	$ sudo rm default
	    => Removes the default configuration
	$ sudo systemctl restart nginx

## XNAT
All required war and jar files have been downloaded to XNAT's home directory `~`.

I prefer Tomcat to be stopped while I configure XNAT: `sudo systemctl stop tomcat7`

***Simple Version***

    $ sudo mkdir -p /data/xnat/home/config
    $ sudo mkdir /data/xnat/home/logs
    $ sudo mkdir /data/xnat/home/plugins
    $ sudo mkdir /data/xnat/home/work
    $ sudo mkdir /data/xnat/archive
    $ sudo mkdir /data/xnat/build
    $ sudo mkdir /data/xnat/cache
    $ sudo mkdir /data/xnat/ftp
    $ sudo mkdir /data/xnat/pipeline
    $ sudo mkdir /data/xnat/prearchive
    $ sudo chown -R xnat:xnat /data

***NFS Version***

	xnat$ sudo mkdir -p /data
	xnat$ cd /data
	xnat$ sudo ln -s ../rds/ncitavs/xnaticrcollab xnat

The directory structure for the simple version already exists in `/rds/ncitavs/xnaticrcollab`.

***Common***

    xnat$ cd /data/xnat/home/config
	xnat$ vi xnat-conf.properties
		=> datasource.driver=org.postgresql.Driver
		=> datasource.url=jdbc:postgresql://localhost/xnat
		=> datasource.username=xnat
		=> datasource.password=password
		=> 
		=> hibernate.dialect=org.hibernate.dialect.PostgreSQLDialect
		=> hibernate.hbm2ddl.auto=update
		=> hibernate.show_sql=false
		=> hibernate.cache.use_second_level_cache=true
		=> hibernate.cache.use_query_list=true
	xnat$ chmod 600 xnat-conf.properties
	    => Only the XNAT user has any business reading the password for the database.

Install the two plugin jars for the OHIF Viewer. This will be a production system so we're not installing the bleeding edge single jar from the dev branch.

	xnat$ cd ../plugins
	xnat$ cp ~/ohif-viewer-2.1.0.jar .
	xnat$ cp ~/xnat-roi-2.2.0.jar .

Install the main XNAT war file and the pipeline engine. `XNAT_ICR_COLLABORATIONS` is the name of this particular instance, name yours appropriately.

	xnat$ cd /var/lib/tomcat7/webapps
	xnat$ rm -r ROOT
	xnat$ cp ~/xnat-web-1.7.6.war XNAT_ICR_COLLABORATIONS.war
	xnat$ cd ~
	xnat$ unzip xnat-pipeline-1.7.6.zip
	xnat$ cd xnat-pipeline
	xnat$ cp sample.gradle.properties gradle.properties
	xnat$ vi gradle.properties
		=> xnatUrl=http://localhost:8080/XNAT_ICR_COLLABORATIONS
		=> siteName=XNAT_ICR_COLLABORATIONS
		=> adminEmail=simon.doran@icr.ac.uk
		=> smtpServer=serverIP
		=> destination=/data/xnat/pipeline
	xnat$ ./gradlew

***Snapshot***

With all the pre-requisites and XNAT installed, now is a good moment to take a snapshot: before we spin up the XNAT service itself and it makes changes e.g. to the database.

***Startup***

Finally start up your shiny new XNAT instance.

    $ sudo systemctl restart tomcat7; tail -f /var/log/tomcat7/catalina.out

Once you see `INFO: Server startup in XYZ ms` in the log, you can cancel the `tail` and log into the web interface.

### Migrating an Existing Database

***Caveat:*** I have not yet managed to sucessfully migrate our database from the production system to this clone so approach this section with care. And a snapshot.

    xnat$ cd ~
	xnat$ sudo su - postgres
	postgres$ psql
	postgres=# ALTER ROLE xnat WITH PASSWORD 'password';
	postgres=# DROP DATABASE xnat;
	postgres=# CREATE DATABASE xnaticrcollab WITH OWNER xnat;
	xnat$ psql -U xnat -d xnaticrcollab -f ~/xnaticrcollab.20200118_010001
	    => xnaticrcollab.20200118_010001 is the name of the database dump file from production
	xnat$ sudo systemctl restart tomcat7; tail -f /var/log/tomcat7/catalina.out

The production system did not use `/data/xnat/home` as its base directory so alterations had to be made to the paths so they matched the ones on production. 

### Docker

Conspicuously absent from the set up so far is Docker. This is because it can break the networking on Ubuntu 18.04. In our case, it did. Docker's default internal bridge network address, `172.17.0.1/16`, is in use on the institution's network and so it broke Ubuntu's routing. While there is a workaround for this problem, it is not tested. This section will be updated in the future once we have a stable install of Docker that plays nice with our network.
