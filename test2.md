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
