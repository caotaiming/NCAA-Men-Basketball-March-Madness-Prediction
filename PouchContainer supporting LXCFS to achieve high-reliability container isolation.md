# PouchContainer supporting LXCFS to achieve high-reliability container isolation

## Introduction
PouchContainer is an open-source runtime container software developed by Alibaba. The latest released version is 0.3.0, located at [https://github.com/alibaba/pouch](https://github.com/alibaba/pouch). PouchContainer is designed to support LXCFS to realize highly reliable container separation. While Linux adopted cgroup technology to realize resource separation, the problem that users obtain host information when trying to read files in /proc/meminfo/, caused by host machine's file system still mounting in container, remains unresolved. The lack of `/proc view isolation` will cause a series of problems such as stalling or obstructing enterprise business containerization. LXCFS ([https://github.com/lxc/lxcfs](https://github.com/lxc/lxcfs)) is an open-source file system solution for resolving `/proc view isolation` issue, making the container acting like a traditional virtual machine in presentation layer. This article will first introduce the appropriate business scene for LXCFS and then introduce how LXCFS works in PouchContainer. 


## LXCFS Business Scene
In the age of physical machine and virtual machine, the Alibaba developed a internal toolbox including compiler packing, application deployment, unified monitoring. These tools have been providing stable services for applications that deploy physical machines and virtual machines. 


### Monitoring and Operational tools
Most monitoring tools rely on the /proc file system to retrieve system information. In the example of Alibaba, part of the infrastructural monitoring tools collect informations through tsar（[https://github.com/alibaba/tsar](https://github.com/alibaba/tsar)). However, collecting memory and CPU information in tsar depends on /proc file system. We can download tsar source code to learn how tsar uses files under /proc. 

```
$ git remote -v
origin https://github.com/alibaba/tsar.git (fetch)
origin https://github.com/alibaba/tsar.git (push)
$ grep -r cpuinfo .
./modules/mod_cpu.c:    if ((ncpufp = fopen("/proc/cpuinfo", "r")) == NULL) {
:tsar letty$ grep -r meminfo .
./include/define.h:#define MEMINFO "/proc/meminfo"
./include/public.h:#define MEMINFO "/proc/meminfo"
./info.md:内存的计数器在/proc/meminfo,里面有一些关键项
./modules/mod_proc.c:    /* read total mem from /proc/meminfo */
./modules/mod_proc.c:    fp = fopen("/proc/meminfo", "r");
./modules/mod_swap.c: * Read swapping statistics from /proc/vmstat & /proc/meminfo.
./modules/mod_swap.c:    /* read /proc/meminfo */
$ grep -r diskstats .
./include/public.h:#define DISKSTATS "/proc/diskstats"
./info.md:IO的计数器文件是:/proc/diskstats,比如:
./modules/mod_io.c:#define IO_FILE "/proc/diskstats"
./modules/mod_io.c:FILE *iofp;                     /* /proc/diskstats*/
./modules/mod_io.c:    handle_error("Can't open /proc/diskstats", !iofp);
```

It is obvious that tsar's monitoring of process, IO, and cpu relies on /proc file system. 

When the information provided by /proc file system is from host machine, these monitorings cannot monitor the information in container. To satisfy business demand to appropriate container monitoring, it is even nessary to develop another set of monitoring tools specifically for a container. This issue will, in nature, stall or even obstruct the containerization of enterprise existing business. Therefore, container technology must have the compatibility of original existing monitoring tools to avoid develop new tools everytime and preserve nice user interface that customs to Engineers' user habits. 

PouchContainer is a tool that support LXCFS and get rid of issues listed above. PouchContainer resolves listed issues by transparentizing the monitoring and operational tools that depend on /proc file system deployed in container or on the host. Existing monitoring and operational tools will be able to transition into container to achieve in-container monitoring and operations without appropriating structure or re-developing.

Next, let's see from an example of installing PouchContainer 0.3.0 in Ubuntu:



```
# uname -a
Linux p4 4.13.0-36-generic #40~16.04.1-Ubuntu SMP Fri Feb 16 23:25:58 UTC 2018 x86_64 x86_64 x86_64 GNU/Linux
```
systemd invoke pouchd in default mode that does not start LXCFS. Now, the created container cannot use LXCFS functionalities. Let's see the contents under /proc in container:

```
# systemctl start pouch
# head -n 5 /proc/meminfo
MemTotal:        2039520 kB
MemFree:          203028 kB
MemAvailable:     777268 kB
Buffers:          239960 kB
Cached:           430972 kB
root@p4:~# cat /proc/uptime
2594341.81 2208722.33
# pouch run -m 50m -it registry.hub.docker.com/library/busybox:1.28
/ # head -n 5 /proc/meminfo
MemTotal:        2039520 kB
MemFree:          189096 kB
MemAvailable:     764116 kB
Buffers:          240240 kB
Cached:           433928 kB
/ # cat /proc/uptime
2594376.56 2208749.32
```
It is obvious to see the consistency between the outputs from /proc/meminfo、uptime files and those from the host machine. Although we designated 50 M memory in start time, /proc/meminfo files does not demonstrate the memory limit in containers. 

Starting LXCFS service inside the host machine, manually invoking pouchd process and designating related relative LXCFS parameters:


```
# systemctl start lxcfs
# pouchd -D --enable-lxcfs --lxcfs /usr/bin/lxcfs >/tmp/1 2>&1 &
[1] 32707
# ps -ef |grep lxcfs
root       698     1  0 11:08 ?        00:00:00 /usr/bin/lxcfs /var/lib/lxcfs/
root       724 32144  0 11:08 pts/22   00:00:00 grep --color=auto lxcfs
root     32707 32144  0 11:05 pts/22   00:00:00 pouchd -D --enable-lxcfs --lxcfs /usr/bin/lxcfs
```

Start the container and get the corresponding file content:

```
# pouch run --enableLxcfs -it -m 50m registry.hub.docker.com/library/busybox:1.28
/ # head -n 5 /proc/meminfo
MemTotal:          51200 kB
MemFree:           50804 kB
MemAvailable:      50804 kB
Buffers:               0 kB
Cached:                4 kB
/ # cat /proc/uptime
10.00 10.00
```

Using LXCFS started container and reading in-container /proc files to obtain relative information in container.

### Business Applications
Configuration for most applications which rely heavily on the operation system, the startup program of applications needs to obtain information about the system's memory, CPU, and so on.
When the '/proc' file in the container does not accurately reflect the resources condition of the container, it will cause significant effects of the application.

For example, when some Java applications dynamically allocate the stack size of the running program by checking /proc/meminfo and the container memory limit is less than the host memory, program startup failure will appear because of failed memory allocation.

For DPDK related applications, the application tools needs to get CPU information and the CPU logic core used by initialization in the EAL layer from /proc/cpuinfo. 
If the above information cannot be accurately obtained in the container, the DPDK application needs to modify the corresponding tool.


## PouchContainer integrated LXCFS
PouchContainer supports LXCFS from version 0.1.0, please check this instance:[https://github.com/alibaba/pouch/pull/502](https://github.com/alibaba/pouch/pull/502) 

In short, when the container starts, it will through -v mount the LXCFS mount point on the host  /var/lib/lxc/lxcfs/proc/  to the virtual filesystem directory  /proc inside the container. 

At this point you can see in the /proc directory of the container, some proc files include meminfo, uptime, swaps, stat, diskstats, cpuinfo and so on.
The parameters are as follows:


```
-v /var/lib/lxc/:/var/lib/lxc/:shared
-v /var/lib/lxc/lxcfs/proc/uptime:/proc/uptime 
-v /var/lib/lxc/lxcfs/proc/swaps:/proc/swaps 
-v /var/lib/lxc/lxcfs/proc/stat:/proc/stat 
-v /var/lib/lxc/lxcfs/proc/diskstats:/proc/diskstats 
-v /var/lib/lxc/lxcfs/proc/meminfo:/proc/meminfo 
-v /var/lib/lxc/lxcfs/proc/cpuinfo:/proc/cpuinfo
```

To simplify usage, the pouch create and run command lines provide parameters  `--enableLxcfs`. If specify above parameters when creating the container, you can omit the complicated `-v` parameter.

After a period of using and testing, we found after lxcfs restarts, proc and cgroup will be rebuilt. That will cause a `connect failed` error when users access /proc in the container.

In order to enhance the stability of LXCFS, in pull request:[https://github.com/alibaba/pouch/pull/885](https://github.com/alibaba/pouch/pull/885，refined the management method of LXCFS to systemd guarantee. In order to do that, use remote operation by adding ExecStartPost to lxcfs.service, traverse LXCFS container, and mount again in the container.

## Summary

PouchContainer supports LXCFS implement view isolation for /proc filesystems within containers which will reduce the original tool chain as well as operation and maintenance habits in the process of enterprise storage application containerization, and speed up the process. PouchContainer will support strongly for the smooth transition of enterprises from traditional virtualization to container virtualization.
