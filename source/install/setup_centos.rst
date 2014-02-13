.. _setup_centos:

#####################
Installing CentOS 6.5
#####################

==========
ISO Images
==========

Here you can find the ISO image used to install the OS:

* http://mi.mirror.garr.it/mirrors/CentOS/6.5/isos/x86_64/CentOS-6.5-x86_64-minimal.iso

=======================
Initial OS installation
=======================

When the bootload starts, select "Install or upgrade an existing system"
 - Check disk integrity if you need to, otherwise press "skip"
 - OS presentation: press Next
 - Choose language
 - Choose keyboard layout
 - Choose a device type, following the suggestions (probably "Basic storage devices" will work)
 - There will be a warning about the storage device; being on a VM, it will be empty, 
   so let's proceed selecting "Yes, discard any data"
 - Host name: specify a name here or leave the default one
 - You may now configure the network (button "config network"), 
   but we'll setup it later from a terminal.
 - Choose a city to select the time zone
 - Create a password for the root user
 - Choose the type of disk installation: "Use All Space".
    - If you need, choose "Create Custom Layout" and proceed with the related screens    
 - Click on "Write changes to disk"

The install procedure will go on installing a basic set of packages. 
At the end you will be requested to reboot the system.


=====================
Network configuration
=====================

Edit the file ``/etc/sysconfig/network-scripts/ifcfg-eth0`` paying attention to these properties::

   BOOTPROTO="static"
   ONBOOT="yes"
   IPADDR=84.33.2.27
   NETMASK=.......
   GATEWAY=.......

Edit the file ``/etc/resolv.conf`` and add your nameservers.

In the sample VM Google's DNS have been set::

   nameserver 8.8.8.8
   nameserver 8.8.4.4

Start the network service::

   service network start

Check the connection is up by pinging and external server::

   ping google.com

.. attention:: 
   Please note that in CentOS6 only ssh incoming connections are allowed; 
   all other incoming connections are disabled by default.
          
   In the paragraph related to the httpd service you can find details about
   how to enable incoming traffic. 

Note that after configuring the network, you may continue installing the system setup using a ssh connection.

========================
Installing base packages
========================

Internal clock sync
-------------------

Install the program for ntp server synchronization::

   yum install ntp

Edit ``/etc/ntp.conf`` and add the following line before the first ``server`` directive::

   server tempo.ien.it     # Galileo Ferraris

Sync with the server by issuing::

   service ntpdate start


Other utilities
---------------

Install::

  yum install man
  yum install vim
  yum install openssh-clients    # also needed for incoming scp connections
  yum install mc                 # mc (along with zip) can be used to navigate inside .war files
  yum install zip unzip
  yum install wget

=================================
Installing PostgreSQL and PostGIS
=================================

Repositories
------------

Update the packages list::

  yum check-update
  
Install the package for configuring the PGDG repository::

  wget http://yum.postgresql.org/9.2/redhat/rhel-6-x86_64/pgdg-centos92-9.2-6.noarch.rpm
  rpm -ivh pgdg-centos92-9.2-6.noarch.rpm

EPEL 6 repository will provide GDAL packages::

  wget http://mirror.i3d.net/pub/fedora-epel/6/x86_64/epel-release-6-8.noarch.rpm
  rpm -ivh epel-release-6-8.noarch.rpm

Install PG::

  yum install postgresql92-server postgis2_92

Verify::

  [root@cerco ~]# rpm -qa | grep postg
  postgresql-libs-8.4.13-1.el6_3.x86_64
  postgresql92-9.2.2-1PGDG.rhel6.x86_64
  postgresql92-server-9.2.2-1PGDG.rhel6.x86_64  
  postgresql92-libs-9.2.2-1PGDG.rhel6.x86_64
  postgis2_92-2.0.2-1.rhel6.x86_64
  [root@cerco ~]#

Init the DB::

  service postgresql-9.2 initdb
  

Setting PostgreSQL access
-------------------------

Edit the file ``/var/lib/pgsql/9.2/data/pg_hba.conf`` so that the local connection entries 
will change to::

  # "local" is for Unix domain socket connections only

  local   all             postgres                                peer
  local   all             all                                     md5

  # IPv4 local connections:

  host    all             postgres        127.0.0.1/32            ident
  host    all             all             127.0.0.1/32            md5

  # IPv6 local connections:
  host    all             postgres        ::1/128                 ident
  host    all             all             ::1/128                 md5
   

Setup automatic start
---------------------

Configure automatic service startup at boot time ::

  chkconfig --level 2345 postgresql-9.2 on
  chkconfig --add postgresql-9.2

Start the service right now ::

  service postgresql-9.2 start


=====================
Creating system users
=====================

.. _create_user_tomcat:

Create tomcat user
------------------
:: 

  [root@cerco ~]# adduser -m -s /bin/bash tomcat
  [root@cerco ~]# passwd tomcat


========================
Installing  apache httpd
========================

Apache httpd is used as entry point for web accesses. 
It will be configured as a reverse proxy for the requests to the running web applications.

Install httpd::

    yum install httpd

Create the file ``/etc/httpd/conf.d/00_servername.conf`` and configure the ``ServerName``.

If no name is assigned to the IP address assigned to this machine, we'll set the IP address here::

  ServerName 84.33.2.27:80

Configure the automatic start at boot ::

  chkconfig --level 2345 httpd on

Start the service right away ::

  service httpd start

Check if the machine is reachable from outside, pointing your browser to:: 

  http://84.33.2.27
  
If you cannot reach the machine, proceed with next section.

Configure incoming requests
---------------------------

If the machine is not reachable from the outside, allow the incoming connections by issuing this command::

  iptables -I INPUT -p tcp --dport 80 -j ACCEPT

you can then save the ``iptables`` configuration (in order to retain it through reboots) issuing ::

  service iptables save
  
Configuring httpd
-----------------

Enable gz compression
'''''''''''''''''''''

Create file ``/etc/httpd/conf.d/05_deflate.conf`` with the following content::

  SetOutputFilter DEFLATE
  AddOutputFilterByType DEFLATE text/html text/plain text/xml text/javascript text/css

===============
Installing java
===============

You can download the JDK from this page

  http://www.oracle.com/technetwork/java/javase/downloads/index.html

Oracle does not expose a URL to automatically dowload the JDK because an interactive licence acceptance is requested.  
You may start downloading the JDK RPM from a browser, and then either 
- stop the download from the browser and use on the server the dynamic download URL your browser has been assigned, or
- finish the download and transfer the JDK RPM to the server using ``scp``.   

::

  rpm -ivh jdk-7u51-linux-x64.rpm

Verify the proper installation on the JDK::

  # java -version
  java version "1.7.0_51"
  Java(TM) SE Runtime Environment (build 1.7.0_51-b13)
  Java HotSpot(TM) 64-Bit Server VM (build 24.51-b03, mixed mode) 
  # javac -version
  javac 1.7.0_51

.. _deploy_tomcat:

========================
Installing apache tomcat
========================

Download apache tomcat and install it under ``/opt``::

  wget http://mirror.nohup.it/apache/tomcat/tomcat-6/v6.0.39/bin/apache-tomcat-6.0.39.tar.gz
  tar xzvf apache-tomcat-6.0.39.tar.gz -C /opt/

Let's use a symlink to ease future upgrades::

  ln -s /opt/apache-tomcat-6.0.39/ /opt/tomcat


.. _create_catalina_base:

Creating `base/` template directory
-----------------------------------

::

  mkdir -p /var/lib/tomcat/base/{bin,conf,logs,temp,webapps,work}
  cp /opt/tomcat/conf/* /var/lib/tomcat/base/conf/

.. _cerco_apache_port:


==================
Document changelog
==================

+---------+------------+--------+------------------+
| Version | Date       | Author | Notes            |
+=========+============+========+==================+
| 1.0     | 2014-02-06 | ETj    | Initial revision |
+---------+------------+--------+------------------+