# Siebel IP17 Install on Docker

## Prerequisits

An Oracle Linux 7 system is required to host the docker images and containers. Only application containers will be created, an Oracle database is assumed to be already installed and up and running.


## Docker Installation

### Update Yum repository file

As root, change directory to /etc/yum.repos.d.

    cd /etc/yum.repos.d

Use the wget utility to download the latest repository configuration file.

    wget http://yum.oracle.com/public-yum-ol7.repo

### Install Docker

Next you simply use yum install to start the installation.

    # yum --enablerepo='ol7_addons' install docker-engine.x86_64
    Loaded plugins: ulninfo
    Resolving Dependencies
    --> Running transaction check
    ---> Package docker-engine.x86_64 0:17.06.2.ol-1.0.1.el7 will be installed
    --> Processing Dependency: container-selinux >= 2.9 for package: docker-engine-17.06.2.ol-1.0.1.el7.x86_64
    --> Running transaction check
    ---> Package docker-engine-selinux.noarch 0:17.03.1.ce-3.0.1.el7 will be installed
    --> Finished Dependency Resolution

    Dependencies Resolved

    ==========================================================================================================================
     Package                               Arch               Version                        Repository                 Size
    ==========================================================================================================================
    Installing:
     docker-engine                         x86_64             17.06.2.ol-1.0.1.el7           ol7_addons                 21 M
    Installing for dependencies:
     container-selinux                     noarch             2:2.21-1.el7                   ol7_addons                 28 k
    
    Transaction Summary
    ==========================================================================================================================
    Install  1 Package (+1 Dependent packages)
    
    Total download size: 22 M
    Installed size: 75 M
    Is this ok [y/d/N]: y
    Downloading packages:
    (1/2): container-selinux-2:2.21-1.el7.noarch.rpm                                                   |  28 kB  00:00:00
    (2/2): docker-engine-17.06.2.ol-1.0.1.el7.x86_64.rpm                                               |  21 MB  00:00:48
    --------------------------------------------------------------------------------------------------------------------------
    Total                                                                                     413 kB/s |  22 MB  00:00:48
    Running transaction check
    Running transaction test
    Transaction test succeeded
    Running transaction
      Installing : container-selinux-2:2.21-1.el7.noarch                                                                  1/2
    libsemanage.semanage_direct_install_info: Overriding docker module at lower priority 100 with module at priority 400.
      Installing : docker-engine-17.06.2.ol-1.0.1.el7.x86_64                                                              2/2
      Verifying  : container-selinux-2:2.21-1.el7.noarch                                                                  1/2
      Verifying  : docker-engine-17.06.2.ol-1.0.1.el7.x86_64                                                              2/2

    Installed:
      docker-engine.x86_64 0:17.03.1.ce-3.0.1.el7

    Dependency Installed:
      container-selinux-2:2.21-1.el7
    
    Complete!

Once the installation is completed, the Docker service can be started.

    # systemctl enable docker
    # systemctl start docker
    Created symlink from /etc/systemd/system/multi-user.target.wants/docker.service to /usr/lib/systemd/system/docker.service.

The Docker environment is now ready to be used.

### Pull standard Oracle Docker images

In a web browser, navigate to https://container-registry.oracle.com and sign in using your Oracle SSO account.

Use the web interface to accept the Oracle Standard Terms and Restrictions for the following images:

  * Java/serverjre
  * OS/oraclelinux

Acceptance of the Oracle Standard Terms and Restrictions is valid only for 8 hours. If not pulled within the valid period for acceptance, the acceptance process needs to be repeated before pulling the image.

On the host system, use the docker login command to authenticate against the Oracle Container Registry using the same Oracle credentials:

    docker login container-registry.oracle.com

After logging in, pull the Java and Linux images as follows:

    docker pull container-registry.oracle.com/os/oraclelinux:7-slim
    docker pull container-registry.oracle.com/java/serverjre:8

### Pull Apache image

For installing Oracle and Siebel software in the Docker images, it is best to use a local web server to download the files from, rather than copying the install files to the image. This will keep the image files as small as possible, if downloading the install files, the installation and removal of install files is done in the same RUN step.

If a local web server is not available, a Docker container can be used. Pull the standard Apache 2.4 image as follows:

    docker pull httpd:2.4

Start the container as follows. This will start the Apache image using the name 'localrepo' and will use the local path '/u01/files' as the web server root. Port 8080 on the local system will be forwarded to port 80 on the web server.

    docker run --name localrepo -dti -p 8080:80 -v /u01/files:/usr/local/apache2/htdocs/ httpd:2.4


## Software Preparation

Download Siebel IP17 base installer and patch and use snic.sh to create the images. Select Linux as platform and only select Siebel Enterprise Server.

You should have following directories:

  * 17.0.0.0
  * 17.5.0.0.patchset5

Create a tar file for the patch installation:

    tar cvpf sbl_17_5.tar 17.5.0.0.patchset5

Installation components for the Siebel Database Configuration Utilities are very large, so for the base installer, create two separate tar files, one with ses.db components and one without:

    tar cvpf sbl_17_full.tar 17.0.0.0

    tar cvpf sbl_17_slim.tar 17.0.0.0 --exclude 'ses.db.*'

    $ ll sbl*.tar
    -rw-r--r-- 1 oracle oinstall  738304000 Feb 15 14:16 sbl_17_5.tar
    -rw-r--r-- 1 oracle oinstall 2222295040 Feb 15 14:15 sbl_17_full.tar
    -rw-r--r-- 1 oracle oinstall  938280960 Feb 15 14:16 sbl_17_slim.tar

Move tar files to a directory on a web server, or use the localrepo container.


Create network:

docker network sblip17



drwxr-xr-x 3 oracle oinstall         18 Feb 15 14:07 17.0.0.0
-rw-r--r-- 1 oracle oinstall 2222295040 Feb 15 14:15 sbl_17_full.tar
-rw-r--r-- 1 oracle oinstall  938280960 Feb 15 14:16 sbl_17_slim.tar
-rw-r--r-- 1 oracle oinstall  738304000 Feb 15 14:16 sbl_17_5.tar
-rw-r--r-- 1 oracle oinstall         55 Feb 15 23:36 sbl_17_5.tar.sha1
-rw-r--r-- 1 oracle oinstall          0 Feb 15 23:36 sbl_17_5_full.tar.sha1
-rw-r--r-- 1 oracle oinstall         58 Feb 15 23:37 sbl_17_full.tar.sha1
-rw-r--r-- 1 oracle oinstall         58 Feb 15 23:37 sbl_17_slim.tar.sha1
-rw-r--r-- 1 oracle oinstall   80062269 Feb 16 00:50 jre-8u161-linux-x64.tar.gz
-rw-r--r-- 1 oracle oinstall   27464683 Feb 17 14:20 basiclite.rpm
-rw-r--r-- 1 oracle oinstall      28714 Feb 17 14:35 jdk-8u161-linux-x64.rpm
-rw-r--r-- 1 oracle oinstall     282110 Feb 17 14:36 jre-8u161-linux-x64.rpm
-rw-r--r-- 1 oracle oinstall        246 Feb 17 14:36 headers
-rwxr-xr-x 1 oracle oinstall      15816 Feb 17 18:02 hostname
drwxr-xr-x 2 oracle oinstall          6 Feb 19 02:38 cache
drwxr-xr-x 3 oracle oinstall         26 Feb 19 19:20 database
drwxr-xr-x 2 oracle oinstall         21 Feb 20 05:28 test
-rw-r--r-- 1 oracle oinstall   54792288 Feb 20 06:16 server-jre-8u161-linux-x64.tar.gz
drwxr-xr-x 4 oracle oinstall         30 Feb 20 06:17 ..
drwxr-xr-x 6 oracle oinstall       4096 Feb 20 08:37 .
-- 1 oracle oinstall   54792288 Feb 20 06:16 server-jre-8u161-linux-x64.tar.gz
-rw-r--r-- 1 oracle oinstall   59115950 Feb 20 08:37 oracle-instantclient12.1-basic-12.1.0.2.0-1.i386.rpm
-rw-r--r-- 1 oracle oinstall   27464683 Feb 20 08:37 oracle-instantclient12.1-basiclite-12.1.0.2.0-1.i386.rpm
-rw-r--r-- 1 oracle oinstall   30940809 Feb 20 08:37 oracle-instantclient12.1-basiclite-12.1.0.2.0-1.x86_64.rpm
-rw-r--r-- 1 oracle oinstall     824138 Feb 20 08:37 oracle-instantclient12.1-sqlplus-12.1.0.2.0-1.i386.rpm

	

