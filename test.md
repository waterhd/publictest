Siebel IP17 Install on Docker
=============================
*Dennis Waterham <dennis.waterham@oracle.com>, Oracle Advanced Customer Services*

##Introduction

###Goal
Goal of this exercise is to create a working Siebel IP2017 Enterprise on Docker, running on a single server.  According to best practices, every function (Application Interface, Gateway, Siebel Server) should be created in a separate image, so it can run in its own container. Images should be as small as possible, so the bare minimum of packages and software is installed and unnecessary files are removed after installation.

The scripts and Docker files are designed to be repeatable, meaning they need to result in the same image every time it's build with the same input. The Siebel software on the resulting images should not be patched or no other software should be installed manually. When there's a need to do so, a new image should be created.

Images should not have environment specific configuration or files, such as `tnsnames.ora`. Specific environment details should be entered at run time using environment variables, so images can be reused for multiple environments.

###Prerequisites
A Red Hat or Oracle Linux system is required to host the Docker images and containers. An Oracle 12c database is assumed to be already installed and up and running with the Siebel schema installed and `grantusr.sql` executed. The database can be located on the Docker host or any other server in the network. 

A web server will be needed to host the software installation files. If no web server is available, a Docker container can be used for this purpose.

To download the software images and to install packages from the Oracle Yum repository an internet connection is required. This can either be a direct connection or via a proxy server, when on the Oracle network.

To run the Siebel images, keystores and truststores should be created beforehand and located on the Docker host. Instructions can be found on Oracle Support. Keystores and truststores will be attached to the Siebel containers at startup.

###Limitations
With the set up in this document configuration data in containers is not persisted, so when a container stops, all configuration data is lost. Volumes can be added to the containers to store persistent data. In a future version of this document, steps will be added to automatically configure the Siebel Enterprise and store persistent data, so the containers can be stopped and restarted without losing data. While not currently used, `confd` is installed on the images to automatically configure the Siebel servers in a later phase.

The Siebel Server image will be installed without the database configuration utilities. These utilities are only needed for upgrading and installing the Siebel database, and are very large, so they are excluded to keep the image as small as possible.

##Docker Installation
Skip this section if Docker is already set up and running.

####Update Yum repository file

As root, change directory to /etc/yum.repos.d.

    $ cd /etc/yum.repos.d

Use the wget utility to download the latest repository configuration file.

    $ wget http://yum.oracle.com/public-yum-ol7.repo

####Install Docker

Next you simply use yum install to start the installation.

    $ yum --enablerepo='ol7_addons' install docker-engine.x86_64

    Loaded plugins: ulninfo
    Resolving Dependencies
    --> Running transaction check
    ---> Package docker-engine.x86_64 0:17.06.2.ol-1.0.1.el7 will be installed
    --> Processing Dependency: container-selinux >= 2.9 for package: docker-engine...
    --> Running transaction check
    ---> Package docker-engine-selinux.noarch 0:17.03.1.ce-3.0.1.el7 will be installed
    --> Finished Dependency Resolution

    Dependencies Resolved

    ========================================================================================
     Package                     Arch       Version                  Repository        Size
    ========================================================================================
    Installing:
     docker-engine               x86_64     17.06.2.ol-1.0.1.el7     ol7_addons        21 M
    Installing for dependencies:
     container-selinux           noarch     2:2.21-1.el7             ol7_addons        28 k

    Transaction Summary
    ========================================================================================
    Install  1 Package (+1 Dependent packages)

    Total download size: 22 M
    Installed size: 75 M
    Is this ok [y/d/N]: y
    Downloading packages:
    (1/2): container-selinux-2:2.21-1.el7.noarch.rpm                   |  28 kB  00:00:00
    (2/2): docker-engine-17.06.2.ol-1.0.1.el7.x86_64.rpm               |  21 MB  00:00:48
    ----------------------------------------------------------------------------------------
    Total                                                     413 kB/s |  22 MB  00:00:48
    Running transaction check
    Running transaction test
    Transaction test succeeded
    Running transaction
      Installing : container-selinux-2:2.21-1.el7.noarch                                1/2
      Installing : docker-engine-17.06.2.ol-1.0.1.el7.x86_64                            2/2
      Verifying  : container-selinux-2:2.21-1.el7.noarch                                1/2
      Verifying  : docker-engine-17.06.2.ol-1.0.1.el7.x86_64                            2/2

    Installed:
      docker-engine.x86_64 0:17.03.1.ce-3.0.1.el7

    Dependency Installed:
      container-selinux-2:2.21-1.el7

    Complete!
    
Once the installation is completed, the Docker service can be enabled and started.

#### Configure Proxy
During build time, an internet connection is required to download Docker images and to download packages from the Oracle Yum repository. If the host is on the Oracle network, a direct internet connection is not available, so the Docker daemon should be configured to use a proxy server.

The nearest proxy server can be determined as follows:

    $ curl -s http://wpad/wpad.dat | sed -En 's/^.*proxies = "PROXY ([^;" ]*).*$/\1/p'
    www-proxy-ams.nl.oracle.com:80
    
As root, add the proxy server to the Docker configuration file:
 
    $ echo 'HTTP_PROXY=http://www-proxy-ams.nl.oracle.com:80' >>/etc/sysconfig/docker

#### Start Docker Daemon
As root, use `systemctl` to enable and start the Docker daemon:

    $ systemctl enable docker
    
    $ systemctl start docker
    Created symlink from /etc/systemd/system/multi-user.target.wants/docker.service to /usr/lib/systemd/system/docker.service.

Docker can normally be used by the root account only, so all Docker commands should be executed using sudo. Alternatively, to execute without sudo, add your user account to the `docker` group, log out and log in again:

    $ sudo usermod -aG docker $(whoami)

The Docker environment is now ready to be used.

####Test Docker
Using the `hello-world` image, Docker can be tested to check if it's fully functional:

    $ docker run --rm hello-world

    Unable to find image 'hello-world:latest' locally
    Using default tag: latest
    latest: Pulling from library/hello-world
    ca4f61b1923c: Pull complete 
    Digest: sha256:083de497cff944f969d8499ab94f07134c50bcf5e6b9559b27182d3fa80ce3f7
    Status: Downloaded newer image for hello-world:latest

    Hello from Docker!
    This message shows that your installation appears to be working correctly.

    To generate this message, Docker took the following steps:
     1. The Docker client contacted the Docker daemon.
     2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
        (amd64)
     3. The Docker daemon created a new container from that image which runs the
        executable that produces the output you are currently reading.
     4. The Docker daemon streamed that output to the Docker client, which sent it to your terminal.

    To try something more ambitious, you can run an Ubuntu container with:
     $ docker run -it ubuntu bash

    Share images, automate workflows, and more with a free Docker ID:
     https://cloud.docker.com/

    For more examples and ideas, visit:
     https://docs.docker.com/engine/userguide/

##Web Server Installation
For installing Oracle and Siebel software in the Docker images, it is best to use a web server to download the files from, rather than copying the install files to the image. By using the same `RUN` step to download the install files, do the install and remove the files, the images are kept as small as possible. 

If a web server to host the files is not available, a Docker container can be used, which will only be needed to build the images. The Docker container can be stopped afterwards. 

####Pull Apache Image
To use a web server container, pull the standard Apache 2.4 image as follows:

    $ docker pull httpd:2.4
    
The image comes with an Apache configuration file, for this purpose, we will slightly modify it so MD5 checksums are generated, which are used to check if files are transferred correctly.

####Web Server Configuration
Create a directory on the host (e.g. `/u01/localrepo`), which will contain all web server files:

    $ mkdir -p /u01/localrepo
    
In this directory create a file called `httpd.conf` with the following contents:

    ServerRoot "/usr/local/apache2"
    Listen 80

    LoadModule mpm_event_module modules/mod_mpm_event.so
    LoadModule authn_file_module modules/mod_authn_file.so
    LoadModule authn_core_module modules/mod_authn_core.so
    LoadModule authz_host_module modules/mod_authz_host.so
    LoadModule authz_groupfile_module modules/mod_authz_groupfile.so
    LoadModule authz_user_module modules/mod_authz_user.so
    LoadModule authz_core_module modules/mod_authz_core.so
    LoadModule access_compat_module modules/mod_access_compat.so
    LoadModule auth_basic_module modules/mod_auth_basic.so
    LoadModule reqtimeout_module modules/mod_reqtimeout.so
    LoadModule filter_module modules/mod_filter.so
    LoadModule mime_module modules/mod_mime.so
    LoadModule headers_module modules/mod_headers.so
    LoadModule version_module modules/mod_version.so
    LoadModule unixd_module modules/mod_unixd.so
    LoadModule dir_module modules/mod_dir.so

    User daemon
    Group daemon
    ServerName localrepo
    ServerAdmin you@example.com

    LogLevel warn
    ErrorLog /proc/self/fd/2
    TypesConfig conf/mime.types

    DocumentRoot "/usr/local/apache2/htdocs/"
    ContentDigest On
    
Create another directory on the host where all install files will be located. This directory will be the web server root.

    $ mkdir /u01/localrepo/files
    
####Start Web Server
 
Now, start the container by running the following command:

    $ docker run --name localrepo -dti \
        -v /u01/localrepo/files:/usr/local/apache2/htdocs/ \
        -v /u01/localrepo/httpd.conf:/usr/local/apache2/conf/httpd.conf \
        httpd:2.4

Replace paths if necessary.

The following commands can be used to check if the container was started successfully and to inspect the logs:

    $ docker ps
    CONTAINER ID   IMAGE       COMMAND             CREATED         STATUS         PORTS     NAMES
    558cd1ae149e   httpd:2.4   "httpd-foreground"  3 seconds ago   Up 2 seconds   80/tcp    localrepo

    $ docker logs localrepo
    [Thu Mar 01 10:16:09 2018] [mpm_event:notice] ... AH00489: Apache/2.4.29 (Unix) configured -- resuming normal operations
    [Thu Mar 01 10:16:09 2018] [core:notice] ... AH00094: Command line: 'httpd -D FOREGROUND'

Once the container is running, its IP address can be found using the following command:

    $ docker inspect --format='{{ .NetworkSettings.IPAddress }}' localrepo
    172.17.0.2
    
This IP address should be used as the `REPO_SERVER` build argument when building the Siebel images.

####Test Web Server
To test the web server, create a file in the web server root:

    $ echo 'Hello, world!' > /u01/localrepo/files/helloworld.txt
    
The `curl` utility can be used to request this file from the web server:
 
    $ curl -v http://${LOCALREPO}/helloworld.txt
    * Hostname was NOT found in DNS cache
    *   Trying 172.17.0.2...
    * Connected to 172.17.0.2 (172.17.0.2) port 80 (#0)
    > GET /helloworld.txt HTTP/1.1
    > User-Agent: curl/7.35.0
    > Host: 172.17.0.2
    > Accept: */*
    > 
    /1.1 200 OK
    < Date: Thu, 01 Mar 2018 12:39:38 GMT
    * Server Apache/2.4.29 (Unix) is not blacklisted
    < Server: Apache/2.4.29 (Unix)
    < Last-Modified: Thu, 01 Mar 2018 12:39:31 GMT
    < ETag: "e-5665926be34d8"
    < Accept-Ranges: bytes
    < Content-Length: 14
    < Content-MD5: dGMIgpV14XwzMbvLAMCJiw==
    < Content-Type: text/plain
    < 
    Hello, world!
    * Connection #0 to host 172.17.0.2 left intact
    
##Software Preparation
Before creating the Docker images, all required software needs to be downloaded first, unpacked and prepared. All software will be stored locally except for Linux packages, which will be downloaded from the Oracle Yum repository servers during build time.

###Oracle Linux Image
All Siebel images will be based on the official Oracle Linux Docker image. This image can be pulled from the Oracle Container Registry or from the Docker hub. In this case, the official Oracle Container Registry is used. In a web browser, navigate to https://container-registry.oracle.com and sign in using your Oracle SSO account.

Use the web interface to accept the Oracle Standard Terms and Restrictions for the `oraclelinux` image, which be found in the OS section. After accepting the terms and restrictions, the image should be pulled in the next 8 hours. If not pulled in this time window, the acceptance process needs to be repeated.

On the host system, use the docker login command to authenticate against the Oracle Container Registry using the same Oracle credentials:

    $ docker login container-registry.oracle.com

After logging in, pull the Java and Linux images as follows:

    $ docker pull container-registry.oracle.com/os/oraclelinux:7.4

###Siebel
Download Siebel IP17 base software from Oracle eDelivery and the most recent patch set (currently, PS5) from Oracle Support. After unpacking, use `snic.sh` to create the install images. Select Linux as platform and only select Siebel Enterprise Server. 

After creating the network images, the directory structure will look as follows:

    siebel_image
    ├── 17.0.0.0
    │   └── Linux
    │       └── Server
    │           └── Siebel_Enterprise_Server
    │               └── Disk1
    │                   ├── install
    │                   │   └── ...
    │                   └── stage
    │                       └── ...
    └── 17.5.0.0.patchset5
        └── Linux
            └── Server
                └── Siebel_Enterprise_Server
                    └── Disk1
                        ├── install
                        │   └── ...
                        └── stage
                            └── ...

To make the download easier and faster, we will create tarballs so all installation files can be downloaded by the Docker daemon as a single file.

Navigate to the Siebel image directory and create a tarball for the Siebel patchset:

    $ cd /siebel_image

    $ tar -cvpf sbl_17_5.tar 17.5.0.0.patchset5

For the base installer, create a tarball without the components for the database configuration utilities. These files are very large and the database configuration utilities will not be needed for now. 

    $ tar -cvpf sbl_17.tar 17.0.0.0 --exclude 'ses.db.*'

Make sure the files have read permissions for all users:

    $ chmod a+r sbl*.tar

Check the files, file sizes should be more or less the same:

    $ ls -ltrh sbl*.tar
    -rw-r--r-- 1 dennis dennis 705M Feb 28 23:40 sbl_17_5.tar
    -rw-r--r-- 1 dennis dennis 895M Feb 28 23:40 sbl_17.tar

Move the tarballs to the web server root.

### Oracle Client

Download the Oracle 12.1.0.2 instant client from Oracle Technology Network (http://www.oracle.com/technetwork/topics/linuxsoft-082809.html).

Accept the license agreement and select the following files:

- oracle-instantclient12.1-basic-12.1.0.2.0-1.i386.rpm
- oracle-instantclient12.1-basiclite-12.1.0.2.0-1.i386.rpm
- oracle-instantclient12.1-sqlplus-12.1.0.2.0-1.i386.rpm

To keep the images as small as possible, only the `basiclite` rpm will be installed. If SQL*Plus is needed (not required for Siebel), then the `basic` and `sqlplus` rpms will need to be installed instead. This can be specified in the Dockerfile later on.

After downloading, move the files to the web server root.

### Confd
For populating configuration files from templates, download the latest `confd` release from the `kelseyhightower/confd` repository on Github (https://github.com/kelseyhightower/confd/releases). Select the following file (for release v0.15.0):

- confd-0.15.0-linux-amd64

After downloading, move the file to the web server root.

### Result
    
After preparing all software, the directory should now look as follows:

    u01
    └── localrepo
        └── files
            ├── confd-0.15.0-linux-amd64
            ├── confd-0.15.0-linux-amd64
            ├── oracle-instantclient12.1-basic-12.1.0.2.0-1.i386.rpm
            ├── oracle-instantclient12.1-basiclite-12.1.0.2.0-1.i386.rpm
            ├── oracle-instantclient12.1-sqlplus-12.1.0.2.0-1.i386.rpm
            ├── sbl_17_5.tar
            └── sbl_17.tar


## Siebel Images
There will be a number of different Siebel images, one for each specific Siebel module (Application Interface, Gateway, Application Server). The installation will be different for each image, however, there are a number of things that the images have in common, so a base image will be created.

The image hierarchy will look as follows:

    Oracle Linux image (oraclelinux:7.4)
    └── Siebel Base (sbl_base)
        ├── Siebel Application Interface (sbl_ai)
        ├── Siebel Gateway (sbl_gtwysrvr)
        └── Siebel Server (sbl_siebsrvr)

The names between the brackets will be used as Docker tags, appended with a version number and 'latest', to specify this version to be the most recent. For example, the Siebel base image will be tagged as 'sbl_base:1.0.0' and 'sbl_base:latest', the Siebel Gateway image for Siebel IP17 with patchset 5 applied, will be tagged as `sbl_gtwysrvr:17.5.0` and `sbl_gtwysrvr:latest`.

Create two directories to host the Siebel images, called `sbl_base` and `sbl_install`.

### Base Image
The Siebel base image is based on the official Oracle Linux image, adding the necessary 32-bit packages for Siebel as well as installing the 32-bit Oracle instant client, `confd` and setting appropriate environment variables. Existing 64-bit packages `libgcc`, `libstdc++` and `glibc` packages are upgraded first to avoid multilib issues, without having to upgrade all existing packages.

Save all files in the sections below to the `sbl_base` directory.

####Web Server Download Script
To download files from the web server, a script is used which takes a URL as input, downloads the file, checks the digest to see if the file was downloaded successfully and if the file is a tarball, extracts it. If not downloaded successfully, the script will end with exit status 1, which will cause the image build to fail.

Create the script with following contents and save as `getfromrepo`:

    #!/bin/bash -ae

    URL=${1:?'URL required.'}       # Url for download
    FILE=${2:-${URL##*/}}           # File name where to save
    CHMOD=${3:-'755'}               # File mode
    EXT=${FILE##*.}                 # File extension
    HEADERS=headers.$$              # Temporary headers file

    echo "Downloading ${FILE} from ${URL}..."
    curl -f -L -# -o "${FILE}" -D ${HEADERS} ${URL}

    # Show headers
    tail -vn +1 ${HEADERS}

    # Check for digest in the headers, if found compare with file
    DIGEST=$(sed -n 's/^Content-MD5: \([A-Za-z0-9+=/]*\).*/\1/p' ${HEADERS})
    [[ ${DIGEST} ]] && MD5=$(echo ${DIGEST} | base64 -d | xxd -plain)
    [[ ${MD5} ]] && echo "${MD5} *${FILE}" | md5sum -c

    # Delete headers
    rm -v ${HEADERS}

    # Set file mode
    chmod -vc ${CHMOD} "${FILE}"

    # If not at a tarball, exit
    [[ ! ${EXT} = 'tar' ]] && exit 0

    # Untar
    tar -xapf ${FILE}
    rm -v ${FILE}

Make the script executable:

    $ chmod a+x getfromrepo

####Dockerfile

Create a file called `Dockerfile` with following contents:

    FROM    oraclelinux:7.4

    LABEL   maintainer="Dennis Waterham <dennis.waterham@oracle.com>"

    ARG     REPO_SERVER=172.17.0.2

    ARG     CONFD=confd-0.15.0-linux-amd64
    ARG     ORACLI=oracle-instantclient12.1-basiclite-12.1.0.2.0-1.i386.rpm
    ARG     KEYSTORE_PW=SiebelIP2017

    ENV     ORACLE_HOME=/usr/lib/oracle/12.1/client \
            RESOLV_MULTI=off \
            NLS_LANG=AMERICAN_AMERICA.AL32UTF8 \
            LANG=en_US.UTF-8 \
            SES_BASE=/u01/app/siebel \
            SIEBEL_FS=/mnt/siebelfs

    ENV     SES_HOME=${SES_BASE}/ses \
            CONFD_DIR=${SES_BASE}/confd

    ENV     AC_HOME=${SES_HOME}/applicationcontainer

    ENV     KEYSTORE=${AC_HOME}/siebelcerts/keystore.jks \
            TRUSTSTORE=${AC_HOME}/siebelcerts/truststore.jks \
            KEYSTORE_PW=${KEYSTORE_PW} \
            TRUSTSTORE_PW=${KEYSTORE_PW}

    ADD     getfromrepo /usr/local/bin/

    SHELL   [ "bash" , "-ec" ]

    RUN     yum -y --errorlevel=0 upgrade \
            libgcc \
            libstdc++ \
            glibc \
     &&     yum -y --errorlevel=0 install \
            libgcc.i686 \
            libstdc++.i686 \
            glibc.i686 \
            libaio.i686 \
            libX11.i686 \
            libXext.i686 \
            vim-common \
            gettext \
            ksh \
            tcsh \
            vim-common \
            net-tools \
            sudo \
     &&     getfromrepo http://${REPO_SERVER}/${ORACLI} oracle_client.rpm \
     &&     yum -y --errorlevel=0 localinstall oracle_client.rpm \
     &&     rm -v oracle_client.rpm \
     &&     getfromrepo http://${REPO_SERVER}/${CONFD} /usr/local/bin/confd 755 \
     &&     cd ${ORACLE_HOME}/lib \
     &&     ln -srv libclntsh.so.12.1 libclntsh.so \
     &&     ln -srv libocci.so.12.1 libocci.so \
     &&     pwd > /etc/ld.so.conf.d/oracle-instantclient12.1.conf \
     &&     rm -v *.jar \
     &&     ldconfig \
     &&     yum clean all | head -n -1 \
     &&     echo -e '%siebel\tALL=(ALL)\tNOPASSWD: ALL' >>/etc/sudoers \
     &&     rm -rf /var/cache/yum

    RUN     groupadd -g 54330 siebel \
     &&     useradd -m -g siebel -u 54330 siebel \
     &&     mkdir -pv ${SES_BASE} ${SIEBEL_FS} \
     &&     chown -v siebel:siebel ${SES_BASE} ${SIEBEL_FS}

    USER    siebel

    WORKDIR ${SES_BASE}
    
### Build Arguments
    
When building the image, a number of variables can be supplied if the value to be used is different from the default value. Each `ARG` instruction above is such a build argument. 

The build arguments for the base image are:

| Argument      | Description   | Default Value |
| ------------- | ------------- | ------------- |
| REPO_SERVER   | IP address (or host name) of the web server | 172.17.0.2 |
| KEYSTORE_PW   | Keystore password to be used | SiebelIP2017 |
| CONFD         | the name of the *confd* binary | confd-0.15.0-linux-amd64 |
| ORACLI        | the name of the Oracle instant client package | oracle-instantclient12.1-basiclite-12.1.0.2.0-1.i386.rpm |

### Build Base Image

To build the image, change to the `sbl_base` directory and execute the `docker build` command:

    $ cd /u01/docker/sbl_base
	
    $ docker build -t sbl_base:1.0.0 -t sbl_base:latest .
    
Use the `--build-arg` option to enter build arguments. For example, to set the web server host to 192.168.56.101, use the following command:

    $ docker build --build-arg REPO_SERVER=192.168.56.101 -t sbl_base:1.0.0 -t sbl_base:latest .
   
The output should like this:

    Sending build context to Docker daemon  11.26kB
    Step 1/16 : FROM    oraclelinux:7.4
     ---> 738555b05d00
    Step 2/16 : LABEL   maintainer="Dennis Waterham <dennis.waterham@oracle.com>"
     ---> Running in 1f3d58ae0d45
    Removing intermediate container 1f3d58ae0d45
     ---> cce7898bfda8
    Step 3/16 : ARG     REPO_SERVER=172.17.0.2
     ---> Running in 8090804a7343
    Removing intermediate container 8090804a7343
     ---> f7886418e100
    Step 4/16 : ARG     CONFD=confd-0.15.0-linux-amd64
     ---> Running in a2868c6f6fa3
    Removing intermediate container a2868c6f6fa3
     ---> 9d9a09b025be
    Step 5/16 : ARG     ORACLI=oracle-instantclient12.1-basiclite-12.1.0.2.0-1.i386.rpm
     ---> Running in 71fbe66b073a
    Removing intermediate container 71fbe66b073a
     ---> ff6c71cdf244
    Step 6/16 : ARG     KEYSTORE_PW=SiebelIP2017
     ---> Running in a5a9ec067f59
    Removing intermediate container a5a9ec067f59
     ---> 913e4fb4640f
    Step 7/16 : ENV     ORACLE_HOME=/usr/lib/oracle/12.1/client         RESOLV_MULTI=off         NLS_LANG=AMERICAN_AMERICA.AL32UTF8         LANG=en_US.UTF-8         SES_BASE=/u01/app/siebel         SIEBEL_FS=/mnt/siebelfs
     ---> Running in 90e32c2202b7
    Removing intermediate container 90e32c2202b7
     ---> d4b44d28916c
    Step 8/16 : ENV     SES_HOME=${SES_BASE}/ses         CONFD_DIR=${SES_BASE}/confd
     ---> Running in 500ab25580cc
    Removing intermediate container 500ab25580cc
     ---> 64b84864a562
    Step 9/16 : ENV     AC_HOME=${SES_HOME}/applicationcontainer
     ---> Running in 7ecfcaf19d4a
    Removing intermediate container 7ecfcaf19d4a
     ---> bc318da64a1d
    Step 10/16 : ENV     KEYSTORE=${AC_HOME}/siebelcerts/keystore.jks         TRUSTSTORE=${AC_HOME}/siebelcerts/truststore.jks         KEYSTORE_PW=${KEYSTORE_PW}         TRUSTSTORE_PW=${KEYSTORE_PW}
     ---> Running in d9167e646ae8
    Removing intermediate container d9167e646ae8
     ---> a3ae5ff04978
    Step 11/16 : ADD     getfromrepo /usr/local/bin/
     ---> 7ffc31a03be8
    Step 12/16 : SHELL   [ "bash" , "-ec" ]
     ---> Running in 08e335501a4b
    Removing intermediate container 08e335501a4b
     ---> 73eaca888097
    Step 13/16 : RUN     yum -y --errorlevel=0 upgrade             libgcc             libstdc++             glibc  &&     yum -y --errorlevel=0 install             libgcc.i686             libstdc++.i686             glibc.i686             libaio.i686             libX11.i686             libXext.i686             vim-common             gettext             ksh             tcsh             vim-common             net-tools             sudo  &&     getfromrepo http://${REPO_SERVER}/${ORACLI} oracle_client.rpm  &&     yum -y --errorlevel=0 localinstall oracle_client.rpm  &&     rm -v oracle_client.rpm  &&     getfromrepo http://${REPO_SERVER}/${CONFD} /usr/local/bin/confd 755  &&     cd ${ORACLE_HOME}/lib  &&     ln -srv libclntsh.so.12.1 libclntsh.so  &&     ln -srv libocci.so.12.1 libocci.so  &&     pwd > /etc/ld.so.conf.d/oracle-instantclient12.1.conf  &&     rm -v *.jar  &&     ldconfig  &&     yum clean all | head -n -1  &&     echo -e '%siebel\tALL=(ALL)\tNOPASSWD: ALL' >>/etc/sudoers  &&     rm -rf /var/cache/yum
     ---> Running in cc585b746f5a
    Loaded plugins: ovl, ulninfo
    Resolving Dependencies
    --> Running transaction check
    ---> Package libgcc.x86_64 0:4.8.5-16.el7_4.1 will be updated
    ---> Package libgcc.x86_64 0:4.8.5-16.0.1.el7_4.1 will be an update
    ---> Package libstdc++.x86_64 0:4.8.5-16.el7_4.1 will be updated
    ---> Package libstdc++.x86_64 0:4.8.5-16.0.1.el7_4.1 will be an update
    --> Finished Dependency Resolution

    Dependencies Resolved

    ================================================================================
     Package         Arch         Version                    Repository        Size
    ================================================================================
    Updating:
     libgcc          x86_64       4.8.5-16.0.1.el7_4.1       ol7_latest        98 k
     libstdc++       x86_64       4.8.5-16.0.1.el7_4.1       ol7_latest       301 k

    Transaction Summary
    ================================================================================
    Upgrade  2 Packages

    Total download size: 399 k
    Downloading packages:
    Delta RPMs disabled because /usr/bin/applydeltarpm not installed.
    --------------------------------------------------------------------------------
    Total                                              1.6 MB/s | 399 kB  00:00     
    Running transaction check
    Running transaction test
    Transaction test succeeded
    Running transaction
      Updating   : libgcc-4.8.5-16.0.1.el7_4.1.x86_64                           1/4 
      Updating   : libstdc++-4.8.5-16.0.1.el7_4.1.x86_64                        2/4 
      Cleanup    : libstdc++-4.8.5-16.el7_4.1.x86_64                            3/4 
      Cleanup    : libgcc-4.8.5-16.el7_4.1.x86_64                               4/4 
      Verifying  : libgcc-4.8.5-16.0.1.el7_4.1.x86_64                           1/4 
      Verifying  : libstdc++-4.8.5-16.0.1.el7_4.1.x86_64                        2/4 
      Verifying  : libstdc++-4.8.5-16.el7_4.1.x86_64                            3/4 
      Verifying  : libgcc-4.8.5-16.el7_4.1.x86_64                               4/4 

    Updated:
      libgcc.x86_64 0:4.8.5-16.0.1.el7_4.1  libstdc++.x86_64 0:4.8.5-16.0.1.el7_4.1 

    Complete!
    Loaded plugins: ovl, ulninfo
    Resolving Dependencies
    --> Running transaction check
    ---> Package gettext.x86_64 0:0.19.8.1-2.el7 will be installed
    --> Processing Dependency: gettext-libs(x86-64) = 0.19.8.1-2.el7 for package: gettext-0.19.8.1-2.el7.x86_64
    --> Processing Dependency: libgomp.so.1(GOMP_1.0)(64bit) for package: gettext-0.19.8.1-2.el7.x86_64
    --> Processing Dependency: libunistring.so.0()(64bit) for package: gettext-0.19.8.1-2.el7.x86_64
    --> Processing Dependency: libgettextsrc-0.19.8.1.so()(64bit) for package: gettext-0.19.8.1-2.el7.x86_64
    --> Processing Dependency: libgettextlib-0.19.8.1.so()(64bit) for package: gettext-0.19.8.1-2.el7.x86_64
    --> Processing Dependency: libcroco-0.6.so.3()(64bit) for package: gettext-0.19.8.1-2.el7.x86_64
    --> Processing Dependency: libgomp.so.1()(64bit) for package: gettext-0.19.8.1-2.el7.x86_64
    ---> Package glibc.i686 0:2.17-196.el7_4.2 will be installed
    --> Processing Dependency: libfreebl3.so for package: glibc-2.17-196.el7_4.2.i686
    --> Processing Dependency: libfreebl3.so(NSSRAWHASH_3.12.3) for package: glibc-2.17-196.el7_4.2.i686
    ---> Package ksh.x86_64 0:20120801-34.el7 will be installed
    ---> Package libX11.i686 0:1.6.5-1.el7 will be installed
    --> Processing Dependency: libX11-common >= 1.6.5-1.el7 for package: libX11-1.6.5-1.el7.i686
    --> Processing Dependency: libxcb.so.1 for package: libX11-1.6.5-1.el7.i686
    ---> Package libXext.i686 0:1.3.3-3.el7 will be installed
    ---> Package libaio.i686 0:0.3.109-13.el7 will be installed
    ---> Package libgcc.i686 0:4.8.5-16.0.1.el7_4.1 will be installed
    ---> Package libstdc++.i686 0:4.8.5-16.0.1.el7_4.1 will be installed
    ---> Package net-tools.x86_64 0:2.0-0.22.20131004git.el7 will be installed
    ---> Package sudo.x86_64 0:1.8.19p2-11.el7_4 will be installed
    ---> Package tcsh.x86_64 0:6.18.01-15.el7 will be installed
    ---> Package vim-common.x86_64 2:7.4.160-2.el7 will be installed
    --> Processing Dependency: vim-filesystem for package: 2:vim-common-7.4.160-2.el7.x86_64
    --> Running transaction check
    ---> Package gettext-libs.x86_64 0:0.19.8.1-2.el7 will be installed
    ---> Package libX11-common.noarch 0:1.6.5-1.el7 will be installed
    ---> Package libcroco.x86_64 0:0.6.11-1.el7 will be installed
    ---> Package libgomp.x86_64 0:4.8.5-16.0.1.el7_4.1 will be installed
    ---> Package libunistring.x86_64 0:0.9.3-9.el7 will be installed
    ---> Package libxcb.i686 0:1.12-1.el7 will be installed
    --> Processing Dependency: libXau.so.6 for package: libxcb-1.12-1.el7.i686
    ---> Package nss-softokn-freebl.i686 0:3.28.3-8.0.1.el7_4 will be installed
    ---> Package vim-filesystem.x86_64 2:7.4.160-2.el7 will be installed
    --> Running transaction check
    ---> Package libXau.i686 0:1.0.8-2.1.el7 will be installed
    --> Finished Dependency Resolution

    Dependencies Resolved

    ================================================================================
     Package              Arch     Version                       Repository    Size
    ================================================================================
    Installing:
     gettext              x86_64   0.19.8.1-2.el7                ol7_latest   1.0 M
     glibc                i686     2.17-196.el7_4.2              ol7_latest   4.2 M
     ksh                  x86_64   20120801-34.el7               ol7_latest   883 k
     libX11               i686     1.6.5-1.el7                   ol7_latest   610 k
     libXext              i686     1.3.3-3.el7                   ol7_latest    38 k
     libaio               i686     0.3.109-13.el7                ol7_latest    24 k
     libgcc               i686     4.8.5-16.0.1.el7_4.1          ol7_latest   106 k
     libstdc++            i686     4.8.5-16.0.1.el7_4.1          ol7_latest   314 k
     net-tools            x86_64   2.0-0.22.20131004git.el7      ol7_latest   305 k
     sudo                 x86_64   1.8.19p2-11.el7_4             ol7_latest   1.1 M
     tcsh                 x86_64   6.18.01-15.el7                ol7_latest   338 k
     vim-common           x86_64   2:7.4.160-2.el7               ol7_latest   5.9 M
    Installing for dependencies:
     gettext-libs         x86_64   0.19.8.1-2.el7                ol7_latest   500 k
     libX11-common        noarch   1.6.5-1.el7                   ol7_latest   163 k
     libXau               i686     1.0.8-2.1.el7                 ol7_latest    28 k
     libcroco             x86_64   0.6.11-1.el7                  ol7_latest   104 k
     libgomp              x86_64   4.8.5-16.0.1.el7_4.1          ol7_latest   154 k
     libunistring         x86_64   0.9.3-9.el7                   ol7_latest   289 k
     libxcb               i686     1.12-1.el7                    ol7_latest   226 k
     nss-softokn-freebl   i686     3.28.3-8.0.1.el7_4            ol7_latest   199 k
     vim-filesystem       x86_64   2:7.4.160-2.el7               ol7_latest   9.3 k

    Transaction Summary
    ================================================================================
    Install  12 Packages (+9 Dependent packages)

    Total download size: 16 M
    Installed size: 57 M
    Downloading packages:
    --------------------------------------------------------------------------------
    Total                                              6.4 MB/s |  16 MB  00:02     
    Running transaction check
    Running transaction test
    Transaction test succeeded
    Running transaction
      Installing : 2:vim-filesystem-7.4.160-2.el7.x86_64                       1/21 
      Installing : libX11-common-1.6.5-1.el7.noarch                            2/21 
      Installing : libgcc-4.8.5-16.0.1.el7_4.1.i686                            3/21 
      Installing : glibc-2.17-196.el7_4.2.i686                                 4/21 
      Installing : nss-softokn-freebl-3.28.3-8.0.1.el7_4.i686                  5/21 
      Installing : libgomp-4.8.5-16.0.1.el7_4.1.x86_64                         6/21 
      Installing : libunistring-0.9.3-9.el7.x86_64                             7/21 
      Installing : libcroco-0.6.11-1.el7.x86_64                                8/21 
      Installing : gettext-libs-0.19.8.1-2.el7.x86_64                          9/21 
      Installing : gettext-0.19.8.1-2.el7.x86_64                              10/21 
    install-info: /usr/share/info/dir: could not read (No such file or directory) and could not create (No such file or directory)
      Installing : tcsh-6.18.01-15.el7.x86_64                                 11/21 
      Installing : 2:vim-common-7.4.160-2.el7.x86_64                          12/21 
      Installing : net-tools-2.0-0.22.20131004git.el7.x86_64                  13/21 
      Installing : sudo-1.8.19p2-11.el7_4.x86_64                              14/21 
      Installing : ksh-20120801-34.el7.x86_64                                 15/21 
    failed to link /usr/share/man/man1/ksh.1.gz -> /etc/alternatives/ksh-man: No such file or directory
      Installing : libXau-1.0.8-2.1.el7.i686                                  16/21 
      Installing : libxcb-1.12-1.el7.i686                                     17/21 
      Installing : libX11-1.6.5-1.el7.i686                                    18/21 
      Installing : libXext-1.3.3-3.el7.i686                                   19/21 
      Installing : libstdc++-4.8.5-16.0.1.el7_4.1.i686                        20/21 
      Installing : libaio-0.3.109-13.el7.i686                                 21/21 
      Verifying  : nss-softokn-freebl-3.28.3-8.0.1.el7_4.i686                  1/21 
      Verifying  : libX11-common-1.6.5-1.el7.noarch                            2/21 
      Verifying  : libXext-1.3.3-3.el7.i686                                    3/21 
      Verifying  : glibc-2.17-196.el7_4.2.i686                                 4/21 
      Verifying  : tcsh-6.18.01-15.el7.x86_64                                  5/21 
      Verifying  : libxcb-1.12-1.el7.i686                                      6/21 
      Verifying  : libstdc++-4.8.5-16.0.1.el7_4.1.i686                         7/21 
      Verifying  : gettext-0.19.8.1-2.el7.x86_64                               8/21 
      Verifying  : libgomp-4.8.5-16.0.1.el7_4.1.x86_64                         9/21 
      Verifying  : 2:vim-common-7.4.160-2.el7.x86_64                          10/21 
      Verifying  : libunistring-0.9.3-9.el7.x86_64                            11/21 
      Verifying  : libXau-1.0.8-2.1.el7.i686                                  12/21 
      Verifying  : libX11-1.6.5-1.el7.i686                                    13/21 
      Verifying  : net-tools-2.0-0.22.20131004git.el7.x86_64                  14/21 
      Verifying  : libaio-0.3.109-13.el7.i686                                 15/21 
      Verifying  : gettext-libs-0.19.8.1-2.el7.x86_64                         16/21 
      Verifying  : libgcc-4.8.5-16.0.1.el7_4.1.i686                           17/21 
      Verifying  : libcroco-0.6.11-1.el7.x86_64                               18/21 
      Verifying  : sudo-1.8.19p2-11.el7_4.x86_64                              19/21 
      Verifying  : 2:vim-filesystem-7.4.160-2.el7.x86_64                      20/21 
      Verifying  : ksh-20120801-34.el7.x86_64                                 21/21 

    Installed:
      gettext.x86_64 0:0.19.8.1-2.el7                                               
      glibc.i686 0:2.17-196.el7_4.2                                                 
      ksh.x86_64 0:20120801-34.el7                                                  
      libX11.i686 0:1.6.5-1.el7                                                     
      libXext.i686 0:1.3.3-3.el7                                                    
      libaio.i686 0:0.3.109-13.el7                                                  
      libgcc.i686 0:4.8.5-16.0.1.el7_4.1                                            
      libstdc++.i686 0:4.8.5-16.0.1.el7_4.1                                         
      net-tools.x86_64 0:2.0-0.22.20131004git.el7                                   
      sudo.x86_64 0:1.8.19p2-11.el7_4                                               
      tcsh.x86_64 0:6.18.01-15.el7                                                  
      vim-common.x86_64 2:7.4.160-2.el7                                             

    Dependency Installed:
      gettext-libs.x86_64 0:0.19.8.1-2.el7                                          
      libX11-common.noarch 0:1.6.5-1.el7                                            
      libXau.i686 0:1.0.8-2.1.el7                                                   
      libcroco.x86_64 0:0.6.11-1.el7                                                
      libgomp.x86_64 0:4.8.5-16.0.1.el7_4.1                                         
      libunistring.x86_64 0:0.9.3-9.el7                                             
      libxcb.i686 0:1.12-1.el7                                                      
      nss-softokn-freebl.i686 0:3.28.3-8.0.1.el7_4                                  
      vim-filesystem.x86_64 2:7.4.160-2.el7                                         

    Complete!
    Downloading oracle_client.rpm from http://172.17.0.2/oracle-instantclient12.1-basiclite-12.1.0.2.0-1.i386.rpm...
    ######################################################################## 100.0%
    ==> headers.60 <==
    HTTP/1.1 200 OK
    Date: Tue, 06 Mar 2018 12:17:43 GMT
    Server: Apache/2.4.29 (Unix)
    Last-Modified: Wed, 28 Feb 2018 22:39:54 GMT
    ETag: "1a313eb-5664d6c0c5898"
    Accept-Ranges: bytes
    Content-Length: 27464683
    Content-MD5: nX97wYPgO2vdRu3PY+pVTg==

    oracle_client.rpm: OK
    removed ‘headers.60’
    mode of ‘oracle_client.rpm’ changed from 0644 (rw-r--r--) to 0755 (rwxr-xr-x)
    Loaded plugins: ovl, ulninfo
    Examining oracle_client.rpm: oracle-instantclient12.1-basiclite-12.1.0.2.0-1.i386
    Marking oracle_client.rpm to be installed
    Resolving Dependencies
    --> Running transaction check
    ---> Package oracle-instantclient12.1-basiclite.i386 0:12.1.0.2.0-1 will be installed
    --> Finished Dependency Resolution

    Dependencies Resolved

    ================================================================================
     Package                             Arch  Version        Repository       Size
    ================================================================================
    Installing:
     oracle-instantclient12.1-basiclite  i386  12.1.0.2.0-1   /oracle_client   69 M

    Transaction Summary
    ================================================================================
    Install  1 Package

    Total size: 69 M
    Installed size: 69 M
    Downloading packages:
    Running transaction check
    Running transaction test
    Transaction test succeeded
    Running transaction
      Installing : oracle-instantclient12.1-basiclite-12.1.0.2.0-1.i386         1/1 
      Verifying  : oracle-instantclient12.1-basiclite-12.1.0.2.0-1.i386         1/1 

    Installed:
      oracle-instantclient12.1-basiclite.i386 0:12.1.0.2.0-1                        

    Complete!
    removed ‘oracle_client.rpm’
    Downloading /usr/local/bin/confd from http://172.17.0.2/confd-0.15.0-linux-amd64...
    ######################################################################## 100.0%
    ==> headers.74 <==
    HTTP/1.1 200 OK
    Date: Tue, 06 Mar 2018 12:17:45 GMT
    Server: Apache/2.4.29 (Unix)
    Last-Modified: Wed, 28 Feb 2018 22:39:51 GMT
    ETag: "10cc900-5664d6bd903f4"
    Accept-Ranges: bytes
    Content-Length: 17615104
    Content-MD5: 1+nzew0upoL7JI5dJKmgrw==

    /usr/local/bin/confd: OK
    removed ‘headers.74’
    mode of ‘/usr/local/bin/confd’ changed from 0644 (rw-r--r--) to 0755 (rwxr-xr-x)
    ‘libclntsh.so’ -> ‘libclntsh.so.12.1’
    ‘libocci.so’ -> ‘libocci.so.12.1’
    removed ‘ojdbc6.jar’
    removed ‘ojdbc7.jar’
    removed ‘xstreams.jar’
    Loaded plugins: ovl, ulninfo
    Cleaning repos: ol7_UEKR4 ol7_latest
    Cleaning up everything
    Removing intermediate container cc585b746f5a
     ---> 7c241151cbb3
    Step 14/16 : RUN     groupadd -g 54330 siebel  &&     useradd -m -g siebel -u 54330 siebel  &&     mkdir -pv ${SES_BASE} ${SIEBEL_FS}  &&     chown -v siebel:siebel ${SES_BASE} ${SIEBEL_FS}
     ---> Running in de289610bcf7
    mkdir: created directory ‘/u01’
    mkdir: created directory ‘/u01/app’
    mkdir: created directory ‘/u01/app/siebel’
    mkdir: created directory ‘/mnt/siebelfs’
    changed ownership of ‘/u01/app/siebel’ from root:root to siebel:siebel
    changed ownership of ‘/mnt/siebelfs’ from root:root to siebel:siebel
    Removing intermediate container de289610bcf7
     ---> 2f8ff40ff292
    Step 15/16 : USER    siebel
     ---> Running in cedeec6b4720
    Removing intermediate container cedeec6b4720
     ---> 92ac835d7d64
    Step 16/16 : WORKDIR ${SES_BASE}
    Removing intermediate container 1d53dd22d1ae
     ---> 69f508cebce0
    Successfully built 69f508cebce0
    Successfully tagged sbl_base:1.0.0
    Successfully tagged sbl_base:latest

### Siebel Images
All Siebel images will be based on the above base image. 

Save all files in the sections below to the `sbl_install` directory.

#### Install Script
Create a script with following contents and save as `install.sh`.

    #!/bin/bash -aeu

    # Create temporary install dir
    mkdir -v tempinst
    cd tempinst

    # Set variables
    URL=${1:?'URL required.'}                        # URL of Siebel install tarball
    VER_BASE=${2:-'17'}                              # Siebel base version to install (17)
    VER_PATCH=${3:-'0'}                              # Siebel patch version (0 for base)
    ORACLE_HOME_NAME="Siebel_Home_${SES_HOME//\//_}" # Oracle Home name
    INSTALLGROUP=$(id -gn)                           # Install group
    CATALINA_OUT="${PWD}/catalina.out"               # Catalina log file

    # Set version, Disk1 directory based on Siebel version
    VERSION="${VER_BASE}.${VER_PATCH}.0.0.0"
    DISK1="${VER_BASE}.${VER_PATCH}.0.${VER_PATCH/[^0]/0.patchset${VER_PATCH}}/Linux/Server/Siebel_Enterprise_Server/Disk1"

    # Unset Oracle Home
    unset ORACLE_HOME

    # Select Siebel modules
    IS_AI='false'
    IS_GTWYSRVR='false'
    IS_SIEBSRVR='false'
    export IS_${ROLE^^}='true'

    # Creating empty truststore, keystore and Catalina log file
    touch {keystore,truststore}.jks ${CATALINA_OUT}

    # Download and extract tarball
    getfromrepo ${URL}

    # Create Oracle inventory location file
    cat > orainst.loc <<EOF
    inventory_loc=${SES_BASE}/oraInventory
    inst_group=${INSTALLGROUP}
    EOF

    # Create and display response file
    envsubst < ../install_template.rsp > install.rsp
    tail -vn +1 install.rsp

    # Start the installer
    ${DISK1}/install/runInstaller -silent -waitforcompletion -responseFile ${PWD}/install.rsp -invPtrLoc ${PWD}/orainst.loc

    # Wait for AI to initialize
    echo "Waiting for application container to finish initializing..."
    (timeout 5m tail -f ${CATALINA_OUT} &) | grep -q "org.apache.catalina.startup.Catalina.start Server startup in"

    # Shut down AI
    ${AC_HOME}/bin/shutdown.sh

    # Clean up
    rm -rf ${PWD}

Make the script executable:

    $ chmod a+x install.sh
	
#### Response File

Create a response file for the Siebel installers with the following contents and save as `install_template.rsp`:

    RESPONSEFILE_VERSION=2.2.1.0.0
    s_shiphomeLocation="${PWD}/${DISK1}"
    FROM_LOCATION="${PWD}/${DISK1}/stage/products.xml"
    s_topLevelComp="oracle.siebel.ses"
    s_SiebelVersion="${VERSION}"
    s_installType="New Installation"
    ORACLE_HOME="${SES_HOME}"
    ORACLE_HOME_NAME="${ORACLE_HOME_NAME}"

    b_isGatewayInstalled="${IS_GTWYSRVR}"
    b_isSiebsrvrInstalled="${IS_SIEBSRVR}"
    b_isDBInstalled="false"
    b_isEAIInstalled="false"
    b_isApplicationInterfaceInstalled="${IS_AI}"
    b_isEnterpriseCacheInstalled="false"
    b_isConstraintEngineInstalled="false"
    selectedLangs="[English]"

    s_Redirectport="9701"
    s_Httpport="9702"
    s_Shutdownport="9703"
    s_TLSPort="9704"

    s_txtUserName="admin"
    s_txtPassword="oracle"

    s_txtKeyStoreName="${PWD}/keystore.jks"
    s_txtKeyStorePassword="${KEYSTORE_PW}"
    s_txtKeyStoreType="JKS"
    s_txtTrustStoreName="${PWD}/truststore.jks"
    s_txtTrustStorePassword="${TRUSTSTORE_PW}"
    s_txtTrustStoreType="JKS"

    DEINSTALL_LIST={"oracle.siebel.ses","${VERSION}"}
    SHOW_DEINSTALL_CONFIRMATION=true
    SHOW_DEINSTALL_PROGRESS=true

While a few values are hardcoded, such as port numbers and initial passwords, most values are configurable using build arguments at build time or environment variables at run time. Before installation, the install script parses the response file and replaces the environment variables with the actual values.

The same response file is used for Siebel base and patch, so the above response file is a superset of both.
 
####Server Start Script
Create a script with following contents and save as `server_start.sh`:

    #!/bin/bash -aeu

    # Run confd
    confd -onetime -backend env -confdir ${CONFD_DIR}

    # Show config files
    tail -vn +1 ${AC_HOME}/webapps/*.properties

    # Start Application Interface
    cd /
    ${AC_HOME}/bin/catalina.sh run

Make the script executable as follows:

    $ chmod a+x server_start.sh

####Dockerfile

Create a file called `Dockerfile` with following contents:

    FROM    sbl_base:latest

    LABEL   maintainer="Dennis Waterham <dennis.waterham@oracle.com>"

    ARG     REPO_SERVER=172.17.0.2

    ARG     ROLE=ai

    ARG     VER_BASE=17
    ARG     VER_PATCH=5

    ARG     TAR_BASE=sbl_${VER_BASE}.tar
    ARG     TAR_PATCH=sbl_${VER_BASE}_${VER_PATCH}.tar

    ENV     VER_BASE=${VER_BASE} \
            VER_PATCH=${VER_PATCH} \
            ROLE=${ROLE}

    ADD     install.sh install_template.rsp ./

    RUN     ./install.sh http://${REPO_SERVER}/${TAR_BASE} ${VER_BASE} \
     &&     ./install.sh http://${REPO_SERVER}/${TAR_PATCH} ${VER_BASE} ${VER_PATCH} \
     &&     rm -rf \
                ${SES_HOME}/{ps_backup,OPatch} \
                ${AC_HOME}/siebelcerts/*.jks \
                ${AC_HOME}/webapps/{examples,docs,siebel} \
                ${AC_HOME}/bin/*.{exe,bat,dll,tar.gz}

    ADD     server_start.sh ./

    EXPOSE  9701-9704

    VOLUME  [ "/mnt/siebelfs" ]

    CMD     [ "./server_start.sh" ]

### Build Arguments
    
When building the main images, a number of variables can be supplied if the value to be used is different from the default value. Each `ARG` instruction above is such a build argument. 

The build arguments for the base image are:

| Argument      | Description   | Default Value |
| ------------- | ------------- | ------------- |
| REPO_SERVER   | IP address (or host name) of the web server | 172.17.0.2 |
| ROLE          | Role of the Siebel image (ai, gtwysrvr, siebsrvr) | ai |
| VER_BASE      | Siebel base version | 17 |
| VER_PATCH     | Siebel patch version | 5 |
| TAR_BASE      | Siebel main installer tarball | sbl_17.tar |
| TAR_PATCH     | Siebel patch installer tarball | sbl_17_5.tar |

### Build Main Images
    
Use the following commands to build the Siebel images for each server role:

    $ docker build --build-arg ROLE=ai -t sbl_ai:17.5.0 -t sbl_ai:latest .

    $ docker build --build-arg ROLE=gtwysrvr -t sbl_gtwysrvr:17.5.0 -t sbl_gtwysrvr:latest .

    $ docker build --build-arg ROLE=siebsrvr -t sbl_siebsrvr:17.5.0 -t sbl_siebsrvr:latest .

The output will look as follows, for the *ai* role:

    Sending build context to Docker daemon  79.87kB
    Step 1/15 : FROM    sbl_base:latest
     ---> 69f508cebce0
    Step 2/15 : LABEL   maintainer="Dennis Waterham <dennis.waterham@oracle.com>"
     ---> Running in 016a8729ead2
    Removing intermediate container 016a8729ead2
     ---> 1131289d9140
    Step 3/15 : ARG     REPO_SERVER=172.17.0.2
     ---> Running in b1a036359ca9
    Removing intermediate container b1a036359ca9
     ---> b118699c9d44
    Step 4/15 : ARG     ROLE=ai
     ---> Running in b400f7c028cb
    Removing intermediate container b400f7c028cb
     ---> f1a16057f182
    Step 5/15 : ARG     VER_BASE=17
     ---> Running in cd7d61c04c8f
    Removing intermediate container cd7d61c04c8f
     ---> 5208e5d22bbd
    Step 6/15 : ARG     VER_PATCH=5
     ---> Running in a9f11514787d
    Removing intermediate container a9f11514787d
     ---> ee12cf17a5b9
    Step 7/15 : ARG     TAR_BASE=sbl_${VER_BASE}.tar
     ---> Running in 8eab38c00273
    Removing intermediate container 8eab38c00273
     ---> cb4ae1a96ab9
    Step 8/15 : ARG     TAR_PATCH=sbl_${VER_BASE}_${VER_PATCH}.tar
     ---> Running in 9ef6dce2aac4
    Removing intermediate container 9ef6dce2aac4
     ---> 691663e6c72a
    Step 9/15 : ENV     VER_BASE=${VER_BASE}         VER_PATCH=${VER_PATCH}         ROLE=${ROLE}
     ---> Running in cbf0eeaba1f7
    Removing intermediate container cbf0eeaba1f7
     ---> 2d9df9c9b13d
    Step 10/15 : ADD     install.sh install_template.rsp ./
     ---> 9eedc2ea4a1f
    Step 11/15 : RUN     ./install.sh http://${REPO_SERVER}/${TAR_BASE} ${VER_BASE}  &&     ./install.sh http://${REPO_SERVER}/${TAR_PATCH} ${VER_BASE} ${VER_PATCH}  &&     rm -rf             ${SES_HOME}/{ps_backup,OPatch}             ${AC_HOME}/siebelcerts/*.jks             ${AC_HOME}/webapps/{examples,docs,siebel}             ${AC_HOME}/bin/*.{exe,bat,dll,tar.gz}
     ---> Running in 02379d3e4059
    mkdir: created directory ‘tempinst’
    Downloading sbl_17.tar from http://172.17.0.2/sbl_17.tar...
    ###############################################################           88.2%==> headers.12 <==
    HTTP/1.1 200 OK
    Date: Tue, 06 Mar 2018 12:30:40 GMT
    Server: Apache/2.4.29 (Unix)
    Last-Modified: Wed, 28 Feb 2018 22:40:40 GMT
    ETag: "84758800-5664d6ec04d73"
    Accept-Ranges: bytes
    Content-Length: 2222295040
    Content-MD5: jwwPk8AQ/vfkMWAvt7rdHg==
    Content-Type: application/x-tar

    ######################################################################## 100.0%
    sbl_17.tar: OK
    removed ‘headers.12’
    mode of ‘sbl_17.tar’ changed from 0644 (rw-r--r--) to 0755 (rwxr-xr-x)
    removed ‘sbl_17.tar’
    ==> install.rsp <==
    RESPONSEFILE_VERSION=2.2.1.0.0
    s_shiphomeLocation="/u01/app/siebel/tempinst/17.0.0.0/Linux/Server/Siebel_Enterprise_Server/Disk1"
    FROM_LOCATION="/u01/app/siebel/tempinst/17.0.0.0/Linux/Server/Siebel_Enterprise_Server/Disk1/stage/products.xml"
    s_topLevelComp="oracle.siebel.ses"
    s_SiebelVersion="17.0.0.0.0"
    s_installType="New Installation"
    ORACLE_HOME="/u01/app/siebel/ses"
    ORACLE_HOME_NAME="Siebel_Home__u01_app_siebel_ses"

    b_isGatewayInstalled="false"
    b_isSiebsrvrInstalled="false"
    b_isDBInstalled="false"
    b_isEAIInstalled="false"
    b_isApplicationInterfaceInstalled="true"
    b_isEnterpriseCacheInstalled="false"
    b_isConstraintEngineInstalled="false"
    selectedLangs="[English]"

    s_Redirectport="9701"
    s_Httpport="9702"
    s_Shutdownport="9703"
    s_TLSPort="9704"

    s_txtUserName="admin"
    s_txtPassword="oracle"

    s_txtKeyStoreName="/u01/app/siebel/tempinst/keystore.jks"
    s_txtKeyStorePassword="SiebelIP2017"
    s_txtKeyStoreType="JKS"
    s_txtTrustStoreName="/u01/app/siebel/tempinst/truststore.jks"
    s_txtTrustStorePassword="SiebelIP2017"
    s_txtTrustStoreType="JKS"

    DEINSTALL_LIST={"oracle.siebel.ses","17.0.0.0.0"}
    SHOW_DEINSTALL_CONFIRMATION=true
    SHOW_DEINSTALL_PROGRESS=true
    Starting Oracle Universal Installer...

    Preparing to launch Oracle Universal Installer from /tmp/OraInstall2018-03-06_12-31-39PM. Please wait ...
    ...
    The installation of Siebel Enterprise Server - enu was successful.
    Please check '/u01/app/siebel/oraInventory/logs/silentInstall2018-03-06_12-31-46-PM.log' for more details.
    ...
    Extracting war files.
    reading placeholderforwebxml.txt file
    Updating web.xml file
    Deploying siebel.war file
    Executing Command: /bin/csh -c chmod -R 755 /u01/app/siebel/ses
    Webserver startup file: /u01/app/siebel/ses/applicationcontainer/bin/startup.sh
    Executing Command: /bin/csh -c /u01/app/siebel/ses/applicationcontainer/bin/startup.sh
    Webserver Started successfully.
    Waiting for application container to finish initializing...
    Inside setenv.sh
    Using JRE_HOME as /u01/app/siebel/ses/applicationcontainer/bin/./../../jre
    mkdir: created directory ‘tempinst’
    Downloading sbl_17_5.tar from http://172.17.0.2/sbl_17_5.tar...
    #####################################################                     73.7%==> headers.585 <==
    HTTP/1.1 200 OK
    Date: Tue, 06 Mar 2018 12:33:54 GMT
    Server: Apache/2.4.29 (Unix)
    Last-Modified: Wed, 28 Feb 2018 22:40:07 GMT
    ETag: "2c01a000-5664d6cccab09"
    Accept-Ranges: bytes
    Content-Length: 738304000
    Content-MD5: BBaZ7gHVx2bHLvXiWiZd/Q==
    Content-Type: application/x-tar

    ##############################################################            87.3%sbl_17_5.tar: OK
    removed ‘headers.585’
    mode of ‘sbl_17_5.tar’ changed from 0644 (rw-r--r--) to 0755 (rwxr-xr-x)
    ######################################################################## 100.0%
    removed ‘sbl_17_5.tar’
    ==> install.rsp <==
    RESPONSEFILE_VERSION=2.2.1.0.0
    s_shiphomeLocation="/u01/app/siebel/tempinst/17.5.0.0.patchset5/Linux/Server/Siebel_Enterprise_Server/Disk1"
    FROM_LOCATION="/u01/app/siebel/tempinst/17.5.0.0.patchset5/Linux/Server/Siebel_Enterprise_Server/Disk1/stage/products.xml"
    s_topLevelComp="oracle.siebel.ses"
    s_SiebelVersion="17.5.0.0.0"
    s_installType="New Installation"
    ORACLE_HOME="/u01/app/siebel/ses"
    ORACLE_HOME_NAME="Siebel_Home__u01_app_siebel_ses"

    b_isGatewayInstalled="false"
    b_isSiebsrvrInstalled="false"
    b_isDBInstalled="false"
    b_isEAIInstalled="false"
    b_isApplicationInterfaceInstalled="true"
    b_isEnterpriseCacheInstalled="false"
    b_isConstraintEngineInstalled="false"
    selectedLangs="[English]"

    s_Redirectport="9701"
    s_Httpport="9702"
    s_Shutdownport="9703"
    s_TLSPort="9704"

    s_txtUserName="admin"
    s_txtPassword="oracle"

    s_txtKeyStoreName="/u01/app/siebel/tempinst/keystore.jks"
    s_txtKeyStorePassword="SiebelIP2017"
    s_txtKeyStoreType="JKS"
    s_txtTrustStoreName="/u01/app/siebel/tempinst/truststore.jks"
    s_txtTrustStorePassword="SiebelIP2017"
    s_txtTrustStoreType="JKS"

    DEINSTALL_LIST={"oracle.siebel.ses","17.5.0.0.0"}
    SHOW_DEINSTALL_CONFIRMATION=true
    SHOW_DEINSTALL_PROGRESS=true
    Starting Oracle Universal Installer...

    Preparing to launch Oracle Universal Installer from /tmp/OraInstall2018-03-06_12-34-21PM. Please wait ...
    ...
    ...
    The installation of Siebel SES PatchSet - enu was successful.
    ...
    Patch Installation Completed Successfully
    Before starting Post install process ..... /u01/app/siebel/ses
    After version
    After toplevel
    After component
    cp  /u01/app/siebel/ses/applicationinterface/applicationinterface.properties /u01/app/siebel/ses/applicationcontainer/webapps
    Extracting war files.
    Web.xml file location : /u01/app/siebel/ses/applicationinterface/applicationinterface.properties
    Copying web.xml file .....
    cp  /u01/app/siebel/ses/ps_backup/17.5.0.0.0/applicationcontainer/webapps/siebel/WEB-INF/web.xml /u01/app/siebel/ses/siebel/WEB-INF/web.xml
    Deploying siebel.war file
    Copying server.xml from ... /u01/app/siebel/ses/ps_backup/17.5.0.0.0/applicationcontainer/conf/server.xml TO /u01/app/siebel/ses/applicationcontainer/conf/server.xml
    cp  /u01/app/siebel/ses/ps_backup/17.5.0.0.0/applicationcontainer/conf/server.xml /u01/app/siebel/ses/applicationcontainer/conf/server.xml
    Copying properties files .....
    cp  /u01/app/siebel/ses/ps_backup/17.5.0.0.0/applicationcontainer/webapps/*.properties /u01/app/siebel/ses/applicationcontainer/webapps
    Copying setenv file .....
    cp  /u01/app/siebel/ses/ps_backup/17.5.0.0.0/applicationcontainer/bin/setenv.sh /u01/app/siebel/ses/applicationcontainer/bin
    Creating /u01/app/siebel/DB2_JAR Directory!
    Webserver startup file: /u01/app/siebel/ses/applicationcontainer/bin/startup.sh
    After starting Post install process ..... /u01/app/siebel/ses
    Waiting for application container to finish initializing...
    Inside setenv.sh
    Using JRE_HOME as /u01/app/siebel/ses/applicationcontainer/bin/./../../jre
    Removing intermediate container 02379d3e4059
     ---> 701c8c00aafa
    Step 12/15 : ADD     server_start.sh ./
     ---> 3a85bd8c81c7
    Step 13/15 : EXPOSE  9701-9704
     ---> Running in c30da378cb6f
    Removing intermediate container c30da378cb6f
     ---> 181c92f7c46a
    Step 14/15 : VOLUME  [ "/mnt/siebelfs" ]
     ---> Running in 7134eb97ccce
    Removing intermediate container 7134eb97ccce
     ---> 712f1f38cca0
    Step 15/15 : CMD     [ "./server_start.sh" ]
     ---> Running in e7522610a2d0
    Removing intermediate container e7522610a2d0
     ---> 2efb4d6983e3
    Successfully built 2efb4d6983e3
    Successfully tagged sbl_ai:17.5.0
    Successfully tagged sbl_ai:latest

##Start Siebel Containers
After building all the images, the containers can be started. A Docker network will be created so all Siebel containers can communicate with each other. Host names that are used below are examples, other host names can be used if needed.

###Create Network
Enter the following command to create a network called `local`:

    $ docker network create local

    1dcac61216f890cb85e647e49ff02f26663f8462f050f48b5ac3493bfe68f06d

The number returned is the ID of the network, by which it can be referenced besides the name.

Network settings can be retrieved using the `network inspect` command:

    $ docker network inspect local
    
    [
        {
	    "Name": "local",
	    "Id": "1dcac61216f890cb85e647e49ff02f26663f8462f050f48b5ac3493bfe68f06d",
	    "Created": "2018-03-02T12:47:10.036945182+01:00",
	    "Scope": "local",
	    "Driver": "bridge",
	    "EnableIPv6": false,
	    "IPAM": {
	        "Driver": "default",
	        "Options": {},
	        "Config": [
	            {
	                "Subnet": "172.18.0.0/16",
	                "Gateway": "172.18.0.1"
	            }
	        ]
	    },
	    "Internal": false,
	    "Attachable": false,
	    "Ingress": false,
	    "ConfigFrom": {
	        "Network": ""
	    },
	    "ConfigOnly": false,
	    "Containers": {},
	    "Options": {},
	    "Labels": {}
        }
    ]


###Start Siebel Containers

Use the following commands to start the Siebel containers:

    $ docker run --name sblai1 -dti -p 9701:9701 -h sblai1.local --network local --privileged \
        -v /u01/docker/keys/sblai1.jks:/u01/app/siebel/ses/applicationcontainer/siebelcerts/keystore.jks:ro \
        -v /u01/docker/keys/sblai1.jks:/u01/app/siebel/ses/applicationcontainer/siebelcerts/truststore.jks:ro \
        sbl_ai:latest

    $ docker run --name sblgw -dti -h sblgw.local --network local --privileged \
        -v /u01/docker/keys/sblgw.jks:/u01/app/siebel/ses/applicationcontainer/siebelcerts/keystore.jks:ro \
        -v /u01/docker/keys/sblgw.jks:/u01/app/siebel/ses/applicationcontainer/siebelcerts/truststore.jks:ro \
        sbl_gtwysrvr:latest

    $ docker run --name sblapp1 -dti -h sblapp1.local --network local --privileged \
        -v /u01/docker/keys/sblapp1.jks:/u01/app/siebel/ses/applicationcontainer/siebelcerts/keystore.jks:ro \
        -v /u01/docker/keys/sblapp1.jks:/u01/app/siebel/ses/applicationcontainer/siebelcerts/truststore.jks:ro \
        sbl_siebsrvr:latest

Make sure to use the correct host names and locations of keystore and truststore files.

On the Application Interface container, port 9701 is published so it is available from the Docker host.

###Launch Siebel Management Console

The Siebel Management Console can now be started by navigating to the following URL:

    http://dockerhost:9701/
    
Where *dockerhost* is the name or IP address of the Docker host.

The Siebel environment can now be set up. When asked for database location, where normally a `tnsnames.ora` alias would be used, use the following connect string instead:

    IP address:Port/Service
    
For example:

    192.168.56.160:1521/pdb1

This avoids the need for a `tnsnames.ora` file on the image.
