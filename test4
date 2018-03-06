Siebel IP17 Install on Docker
=============================
*Dennis Waterham <dennis.waterham@oracle.com>, Oracle Advanced Customer Services*

## Introduction

### Goal
Goal of this exercise is to create a working Siebel IP2017 Enterprise on Docker, running on a single server.  According to best practices, every function (Application Interface, Gateway, Siebel Server) should be created in a separate image, so it can run in its own container. Images should be as small as possible, so the bare minimum of packages and software is installed and unnecessary files are removed after installation.

The scripts and Docker files are designed to be repeatable, meaning they need to result in the same image every time it's build with the same input. The Siebel software on the resulting images should not be patched or no other software should be installed manually. When there's a need to do so, a new image should be created.

Images should not have environment specific configuration or files, such as `tnsnames.ora`. Specific environment details should be entered at run time using environment variables, so images can be reused for multiple environments.

### Prerequisites
A Red Hat or Oracle Linux system is required to host the Docker images and containers. An Oracle 12c database is assumed to be already installed and up and running with the Siebel schema installed and `grantusr.sql` executed. The database can be located on the Docker host or any other server in the network. 

A web server will be needed to host the software installation files. If no web server is available, a Docker container can be used for this purpose.

To download the software images and to install packages from the Oracle Yum repository an internet connection is required. This can either be a direct connection or via a proxy server, when on the Oracle network.

To run the Siebel images, keystores and truststores should be created beforehand and located on the Docker host. Instructions can be found on Oracle Support. Keystores and truststores will be attached to the Siebel containers at startup.

### Limitations
With the set up in this document configuration data in containers is not persisted, so when a container stops, all configuration data is lost. Volumes can be added to the containers to store persistent data. In a future version of this document, steps will be added to automatically configure the Siebel Enterprise and store persistent data, so the containers can be stopped and restarted without losing data. While not currently used, `confd` is installed on the images to automatically configure the Siebel servers in a later phase.

The Siebel Server image will be installed without the database configuration utilities. These utilities are only needed for upgrading and installing the Siebel database, and are very large, so they are excluded to keep the image as small as possible.

## Docker Installation
Skip this section if Docker is already set up and running.

#### Update Yum repository file

As root, change directory to /etc/yum.repos.d.

    $ cd /etc/yum.repos.d

Use the wget utility to download the latest repository configuration file.

    $ wget http://yum.oracle.com/public-yum-ol7.repo

#### Install Docker

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

#### Test Docker
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

## Web Server Installation
For installing Oracle and Siebel software in the Docker images, it is best to use a web server to download the files from, rather than copying the install files to the image. By using the same `RUN` step to download the install files, do the install and remove the files, the images are kept as small as possible. 

If a web server to host the files is not available, a Docker container can be used, which will only be needed to build the images. The Docker container can be stopped afterwards. 

#### Pull Apache Image
To use a web server container, pull the standard Apache 2.4 image as follows:

    $ docker pull httpd:2.4
    
The image comes with an Apache configuration file, for this purpose, we will slightly modify it so MD5 checksums are generated, which are used to check if files are transferred correctly.

#### Web Server Configuration
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
    
#### Start Web Server
 
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

#### Test Web Server
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
    
## Software Preparation
Before creating the Docker images, all required software needs to be downloaded first, unpacked and prepared. All software will be stored locally except for Linux packages, which will be downloaded from the Oracle Yum repository servers during build time.

### Oracle Linux Image
All Siebel images will be based on the official Oracle Linux Docker image. This image can be pulled from the Oracle Container Registry or from the Docker hub. In this case, the official Oracle Container Registry is used. In a web browser, navigate to https://container-registry.oracle.com and sign in using your Oracle SSO account.

Use the web interface to accept the Oracle Standard Terms and Restrictions for the `oraclelinux` image, which be found in the OS section. After accepting the terms and restrictions, the image should be pulled in the next 8 hours. If not pulled in this time window, the acceptance process needs to be repeated.

On the host system, use the docker login command to authenticate against the Oracle Container Registry using the same Oracle credentials:

    $ docker login container-registry.oracle.com

After logging in, pull the Java and Linux images as follows:

    $ docker pull container-registry.oracle.com/os/oraclelinux:7.4

### Siebel
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

#### Web Server Download Script
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

#### Dockerfile

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
