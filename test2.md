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
