```
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
```
