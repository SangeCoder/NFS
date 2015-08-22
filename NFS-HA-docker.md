# 一、NFS介绍 #
## 1、NFS介绍 ##
### 1.1NFS概念描述 ###

什么是NFS？NFS是NetWork File System的缩写，它的主要功能是通过网络让不同的主机系统之间可以彼此共享网络文件或目录。NFS客户端（一般为应用服务器，如web）可以通过挂载（mount）的方式将NFS服务器端共享的数据文件目录挂载到NFS客户端本地系统中（就是某个挂载点）。从NFS客户端的机器本地上看，NFS服务器端共享的目录就好像是客户端自己的磁盘分区或者目录一样，而实际上确是远端的服务器目录。

NFS网络文件系统的使用很像windows系统的网络共享、安全功能、网络驱动器映射，这也和linux里的samba服务类似。

### 1.2NFS历史介绍 ###

第一个网络文件系统称为File Access Listener，由Digital Equipment Corporation（DEC）在1976年开发。
NFS是第一个构建于IP协议之上的现代网络文件系统。在20世纪80年代，它首先作为实验的文件系统，由Sun Microsystems在内部完成开发。NFS协议归为Request for Comments（RFC）标准，并演化为NFSv2，作为一个标准，由于NFS与其他客户端和服务器的互操作能力很好而快速发展。

标准持续地演化为NFSv3，在RFC1813中有定义。这一新的协议比以前的版本更具好的可扩展性，支持大文件（超过2GB），异步写入，以及将TCP作为传输协议，为文件系统在更广泛的网络中使用铺平了道路。

### 1.3NFS在企业中的应用场景 ###

在企业集群架构的工作场景中，NFS网络文件系统一般被用来存储共享视频、图片、附件等静态资源文件（一般把网站用户上传的文件都放到NFS共享里，例如：BBS产品的图片、附件、头像，注意网站BBS程序不要放在NFS共享里），NFS是当前互联网系统架构中最常用的数据存储服务之一，特别是中小型网站公司应用频率很高。大公司或门户除了使用NFS外，还可能会使用MFS、GFS、FASTFS、TFS等分布式文件系统。

### 1.4企业生产集群为什么需要共享存储角色 ###

画图解释需要共享存储的理由：

![](http://i.imgur.com/xtw6bDN.png)


例如：男孩传图片到A，女孩访问分发到B，结果看不到图片，所以需要共享存储，共享存储有软件和硬件，互联网中小公司会用普通服务器和NFS服务实现。

中小型互联网企业一般不会买硬件存储，太贵，大公司如果业务发展很快的话，可能会临时买存储顶一下网站压力，当网站并发继续 加大后，硬件存储扩展就相对 很费劲，且价格呈几何级增长。例如：淘宝网就替换掉了很多硬件设备集群软件，用lvs+haproxy替换了netscaler负载均衡设备，用FASTFS，TFS配合PC服务器替换了netapp、emc商业存储设备。

### 1.5NFS挂载原理详细介绍 ###

当我们在NFS服务器端设置好一个共享存储目录/video后，其他的有权限访问NFS服务器端的NFS客户端可以将这个共享目录/video，挂载到NFS客户端本地系统上的某个挂载点（其实就是一个目录，这个挂载点目录可以自己随意指定），两个本地NFS客户端的挂载点分别为/v/video和/video，不同客户端的挂载点可以不同。

当客户端正确挂载完毕后，进入到指定NFS客户端的/v/video或/video目录，就可以看到NFS服务器/video共享出来的目录下的所有数据。在客户端服务器上查看，看起来NFS服务器端的/video目录就相当于NFS客户端本地的磁盘分区或目录一样，几乎感觉不到使用上的区别，根据NFS服务端授予的NFS共享权限以及共享目录的本地系统权限，只要在指定的NFS客户端操作系统挂载/v/video或/video，就可以将数据轻松的存取到NFS服务器端的/video目录中。

挂载NFS后，NFS客户端本地的挂载内容显示如下：
![](http://i.imgur.com/HmloVoy.png)

从上面可以看出，和本地的磁盘分区几乎没区别，只是文件系统的开头是IP地址形式。

NFS系统是通过网络来进行数据传输的，因此，NFS会使用一些端口来传输数据。那么，NFS到底使用哪些端口来进行数据传输的？下面是NFS服务两次向RPC服务注册的端口列表结果对比：

![](http://i.imgur.com/ZrJiLNM.png)

上面试实际测试得知，NFS在传输数据时使用的端口会随机选择。那么，NFS客户端是怎么知道NFS服务端使用的是哪个端口呢？当然是RPC（中文：远程过程调用，英文：Remote Procedure Call）协议/服务来实现的，这个RPC服务的应用在门户级的网站很多的，例如：百度。


## 2、RPC介绍 ##

### 2.1什么是RPC ###

由于NFS支持的功能相当多，而不同的功能都会使用不同的程序来启动，每启动一个功能就会启用一些端口来传输数据，因此，NFS的功能所对应的端口才无法固定，而是随机取用一些未使用的端口来作为传输之用，其中centos5.x随机端口为小于1024的，而centos6.x随机端口都是较大的。

因为端口不固定，这样一来就会造成客户端与NFS服务器端的通讯障碍，由于NFS客户端必须要知道NFS服务器端的数据传输端口才能进行通信交互数据。

解决以上问题，我们需要RPC服务来帮忙，NFS的RPC服务主要的功能是记录每个NFS功能所对应的端口号，并且在NFS客户端请求时将该端口和功能对应的信息传递给请求数据的NFS客户端，从而可以确保客户端连接正确的NFS端口上去，达到实现数据传输交互数据目的。RPC相当于NFS服务的中介。

如图所示：NFS工作流程简图

![](http://i.imgur.com/j3Joy3a.png)


大致如以下几点：

1、首先用户访问网站程序，由程序在NFS客户端上发出NFS文件存取功能的询问请求，这时NFS客户端（即执行程序的服务器）RPC服务（portmap或rpcbind服务）就会通过网络向NFS服务端的RPC服务（portmap或rpcbind）的111端口发出NFS文件存取功能的询问请求。

2、NFS服务器端的RPC服务（即portmap或rpcbind）找到对应的已注册的NFS daemon端口后，通知NFS客户端的RPC服务（即portmap或rpcbind服务）

3、此时NFS客户端就可以获取到正确的端口，然后就直接与NFS daemon联机存取数据了。

4、NFS客户端把数据存取成功后，返回给当前访问程序，告知用户存取结果，作为网站用户，我们就完成了一次存取操作。
由于NFS的各项功能都需要想RPC服务注册，所以RPC服务才能获取到NFS服务的各项功能对应的端口、PID、NFS在主机所监听的IP等，NFS客户端才能够通过向RPC服务询问才找到正确的端口。也就是说，NFS需要有RPC服务的协助才能成功对外提供服务。由上面的描述，我们不难推出：无论是NFS客户端还是NFS服务器端，当要使用NFS时，都需要首先启动RPC服务，然后在启动NFS服务，客户端可以不启动NFS服务。

# 二、NFS流程演示 #
## 2.1 NFS环境准备 ##

宿主机环境：
    
    [root@cxp ~]# cat   /etc/redhat-release 
    CentOS release 6.6 (Final)
    [root@cxp ~]# uname -r
    3.10.79-1.el6.elrepo.x86_64 #之前升级的内核，不影响

### 2.1.1 NFS部署环境准备 ###
    
    服务器环境	角色	IP
    CentOS6.6	服务端	192.168.159.143
    CentOS6.6	客户端	192.168.159.144
    
### 2.1.2 NFS软件 ###

部署NFS服务，需要安装下面的软件包：
	
	·nfs-utils  NFS服务主程序
	包括rpc.nfsd、rpc.mountd两个daemon和相关文档说明及执行命令文件等；
	·rpcbind：RPC服务的主程序（CentOS5.X是portmap）

### 2.1.3查看NFS软件包 ###

Server服务端：
    
    [root@server /]# rpm  -qa  nfs-utils  portmap  rpcbind
    [root@server /]# 没有安装

Client客户端：
    
    [root@client/]# rpm  -qa  nfs-utils  portmap  rpcbind 
    [root@client/]# 没有安装
    
### 2.1.4安装NFS及RPC ###

根据当期的系统环境，我们需要安装nfs-utils，rpcbind

Server1：
    [root@server /]# yum  install  nfs-utils  rpcbind -y   #安装服务
    [root@server /]# rpm -aq  nfs-utils rpcbind#检查服务软件
    rpcbind-0.2.0-11.el6.x86_64
    nfs-utils-1.2.3-54.el6.x86_64

Client：
    [root@client/]# yum  install  nfs-utils  rpcbind -y   #安装服务
    [root@client/]# rpm -aq  nfs-utils rpcbind#检查服务软件
    rpcbind-0.2.0-11.el6.x86_64
    nfs-utils-1.2.3-54.el6.x86_64

基本的环境和软件部署，我们已经完成了哦！

## 2.2 启动NFS相关服务 ##
### 2.2.1 server服务端 ###

（1）启动rpcbind服务：
    
    [root@server /]# /etc/init.d/rpcbind  start
    Starting rpcbind:  [  OK  ]
    
（2）检查rpcbind服务：

    [root@server /]# ps -ef|grep  rpcbind
    rpc 121  1  0 21:53 ?00:00:00 rpcbind
    root125 16  0 21:53 ?00:00:00 grep rpcbind

也可以查看状态：
    
    [root@server /]# /etc/init.d/rpcbind  status
    rpcbind (pid  121) is running...

（3）查看rpcinfo

    [root@server /]# rpcinfo -p  localhost
       program vers proto   port  service
    1000004   tcp111  portmapper
    1000003   tcp111  portmapper
    1000002   tcp111  portmapper
    1000004   udp111  portmapper
    1000003   udp111  portmapper
    1000002   udp111  portmapper

我们把rpcbind服务关闭看看：
    
    [root@server /]# /etc/init.d/rpcbind   stop 
    Stopping rpcbind:[  OK  ]
    再来查看一下：
    [root@server /]# rpcinfo  -p  localhost
    rpcinfo: can't contact portmapper: RPC: Remote system error - Connection refused
    #提示不能连接，报错；是由于我们没有开启rpcbind服务程序

（4）启动NFS服务
    
    [root@server /]# /etc/init.d/nfs  start 
    FATAL: Could not load /lib/modules/3.10.79-1.el6.elrepo.x86_64/modules.dep: No such file or directory
    Starting NFS services:   [  OK  ]
    Starting NFS mountd: [  OK  ]
    

（5）我们再一次的查看rpcinfo
    
    [root@server /]# rpcinfo -p  localhost
       program vers proto   port  service
    1000004   tcp111  portmapper
    1000003   tcp111  portmapper
    1000002   tcp111  portmapper
    1000004   udp111  portmapper
    1000003   udp111  portmapper
    1000002   udp111  portmapper
    1000051   udp  55334  mountd
    1000051   tcp  44912  mountd
    1000052   udp  50921  mountd
    1000052   tcp  36843  mountd
    1000053   udp  55650  mountd
    1000053   tcp  45212  mountd
    #发现没，多了一些端口。这表明NFS已经向RPC服务进行注册了。

（6）设置开机自启动
    
    [root@server /]# chkconfig nfs on 
    [root@server /]# chkconfig  rpcbind  on

（7）检查开机自启动
    
    [root@server /]# chkconfig  --list  nfs  
    nfs	0:off	1:off	2:on	3:on	4:on	5:on	6:off
    [root@server /]# chkconfig  --list  rpcbind
    rpcbind	0:off	1:off	2:on	3:on	4:on	5:on	6:off
    
### 2.2.2 client客户端 ###

从前面的图中，我们可以知道客户端只需要启动RPC服务，因此：

（1）启动rpcbind服务
    
    [root@client/]# /etc/init.d/rpcbind  start
    Starting rpcbind:[  OK  ]

（2）检查运行状态

    [root@client/]# /etc/init.d/rpcbind  status
    rpcbind (pid  113) is running...

（3）设置开机自启动

    [root@client/]# chkconfig  rpcbind  on 
    
（4）检查开机自启动

    [root@client/]# chkconfig  --list  rpcbind
    rpcbind	0:off	1:off	2:on	3:on	4:on	5:on	6:off
    
## 2.3配置server端相关服务 ##
### 2.3.1 编辑配置文件exports ###

（1）创建共享目录

    [root@server /]# mkdir  /data

（2）编辑配置文件/etc/exports

    [root@server /]# vim  /etc/exports 
    #shared  data for bbs by Siffre  at  20140702
    /data  192.168.159.141/24(rw, sync)
    
    解析：
    1./data  共享的目录
    2.192.168.159.141/24 共享的IP网段（写具体IP也可以），24为子网掩码
    3.（rw， sync） 权限：读写rw，sync将buffer里面的数据写入磁盘

（3）检查配置

    [root@server /]# cat  /etc/exports 
    #shared  data for bbs by Siffre  at  20140702
    /data 192.168.159.141/24(rw,sync)
    
### 2.3.2 优雅方式重启NFS ###

（1）优雅的启动NFS服务（平滑）
    
    [root@server /]# /etc/init.d/nfs reload
    [root@server /]# 
    
（2）解析reload
    
    打开启动文件并查询/reload：
    [root@server /]# vim  /etc/init.d/nfs
    。。。省略。。。
    reload | force-reload)
    /usr/sbin/exportfs -r
    [ -f /var/lock/subsys/nfs ] && touch /var/lock/subsys/nfs
    。。。省略。。。
    
    上面的脚本文件，我们可以看出：reload 等价于 /usr/sbin/exportfs -r
    
## 2.4 检查配置后的服务 ##

### 2.4.1 server服务端 ###

服务端自身的检查

    [root@server /]# showmount -e localhost
    Export list for localhost:
    /data 192.168.159.141/24

### 2.4.2 client客户端 ###

客户端的检查

    [root@client /]# showmount -e   192.168.159.141
    Export list for 192.168.159.141:
    /data 192.168.159.141/24
    
## 2.5 client 挂载到server ##

### 2.5.1 查看client磁盘情况 ###
    
    [root@client ~]# df -h
    Filesystem  Size  Used Avail Use% Mounted on
    /dev/sda318G  7.4G  9.2G  45% /
    tmpfs   997M 0  997M   0% /dev/shm
    /dev/sda1   190M   89M   88M  51% /boot
    
### 2.5.2 挂载server到client ###
    
    [root@client ~]# mount -t nfs  192.168.159.141:/data/mnt

检查
    
    [root@client ~]# df -h 
    FilesystemSize  Used Avail Use% Mounted on
    /dev/sda3  18G  7.4G  9.2G  45% /
    tmpfs 997M 0  997M   0% /dev/shm
    /dev/sda1 190M   89M   88M  51% /boot
    192.168.159.141:/data
       18G  7.4G  9.2G  45% /mnt  #已经挂载上了

## 2.6 测试结果 ##
### 2.6.1 server --> client ###

server端
    
    [root@server ~]# cd  /data
    
    [root@server data]# mkdir  app1
    [root@server data]# ls
    app1
    
client端
    
    [root@client ~]# cd  /mnt/
    
    [root@client mnt]# ls
    app1
    
    [root@client app1]# touch  test.txt
    touch: cannot touch `test.txt': Permission denied  #没有权限
    
### 2.6.2 client --->server ###

Client端
    
    [root@client ~]# cd  /mnt/
    [root@client mnt]# ls
    [root@client mnt]# mkdir  test
    mkdir: cannot create directory `test': Permission denied  #没有权限
    

上述两个角度测试都出现了没有权限的问题，我们在配置文件给予了rw（读写）权限，那么这又是怎么回事呢？原来是在linux系统中，不仅是共享的权限还有目录的权限哦！

### 2.6.3 解决测试问题 ###

我们可以通过一下方式来解决上面测试出现的问题：

解决方法一：

（1）先在server端给予共享目录/data，777 权限
    
    [root@server /]# chmod  777   /data
    [root@server /]# ls -ld   /data
    drwxrwxrwx 3 root root 4096 Jul  1 08:36 /data

（2）在client端继续测试
    
    [root@client mnt]# ls
    app1
    [root@client mnt]# mkdir  app2
    [root@client mnt]# ls
    app1  app2
    
    [root@client mnt]# ls -ld  app2
    drwxr-xr-x 2 nfsnobody nfsnobody 4096 Jul  1 08:54 app2
    
    看到没有属主属组均为：nfsnobody

解决方法二：

此方法是基于上述方法的改进，我们查看到属、属组均为nfsnobody。那么我们怎么查找到该用户呢？如下：

    [root@server /]# cat  /var/lib/nfs/etab 
    /data	192.168.159.141/24(rw,sync,wdelay,hide,nocrossmnt,secure,root_squash,no_all_squash,no_subtree_check,secure_locks,acl,anonuid=65534,anongid=65534）
    
    [root@server /]# grep  65534   /etc/passwd
    nfsnobody:x:65534:65534:Anonymous NFS User:/var/lib/nfs:/sbin/nologin
    

因此，我们同更改属主、属组的方式来实现
    
    [root@server /]# chmod  755  /data  #改回原来的权限
    [root@server /]# chown  -R  nfsnobody.nfsnobody  /data
    
    再次的测试如下：
    [root@client mnt]# ls
    app1  app2
    [root@client mnt]# mkdir app3
    [root@client mnt]# ls
    app1  app2  app3
    
到此，成功了！

# 三、深入NFS #
## 3.1 NFS服务组件 ##

（1）查看进程
    
    [root@server /]# ps -ef|grep  -E "rpc|nfs"
    rpc1707   1650  0 01:58 ?00:00:00 rpcbind 
    root   1738   1650  0 01:58 ?00:00:00 rpc.mountd#权限管理进程
    rpc2783  1  0 08:03 ?00:00:00 rpcbind
    root   2813  2  0 08:03 ?00:00:00 [rpciod]
    root   2822  1  0 08:03 ?00:00:00 rpc.rquotad #磁盘配额进程
    root   2827  1  0 08:03 ?00:00:00 rpc.mountd
    root   2833  2  0 08:03 ?00:00:00 [nfsd4]
    root   2834  2  0 08:03 ?00:00:00 [nfsd4_callbacks]
    root   2838  2  0 08:03 ?00:00:00 [nfsd]
    root   2839  2  0 08:03 ?00:00:00 [nfsd]
    root   2840  2  0 08:03 ?00:00:00 [nfsd]
    root   2841  2  0 08:03 ?00:00:00 [nfsd]
    root   2842  2  0 08:03 ?00:00:00 [nfsd]
    root   2843  2  0 08:03 ?00:00:00 [nfsd]
    root   2844  2  0 08:03 ?00:00:00 [nfsd]
    root   2845  2  0 08:03 ?00:00:00 [nfsd]
    root   2872  1  0 08:03 ?00:00:00 rpc.idmapd
    root   3827   2708  0 16:10 pts/000:00:00 grep -E rpc|nfs

（2）NFS服务启动进程说明：

1）nfsd（rpc.nfsd）：rpc.nfsd主要是管理NFS客户端是否能够登入NFS服务端主机，其中还包含登入者的ID判别等；

2）mountd（rpc.mountd）主要管理NFS文件系统，当NFS客户端顺利通过rpc.nfsd登入NFS服务端主机之后，在其允许使用服务器提供数据之前，会去读取NFS的配置文件/etc/exports来比对NFS客户端的权限，当通过这关后，还会经过NFS服务端本地文件系统的使用权限（owner、group、other）的认证程序。如果都通过，NFS客户端就可以取得使用NFS服务端文件的权限。注意：/etc/exports文件也是我们用来管理NFS共享目录的使用权限与安全设置的地方，特别强调，NFS本身设置的是网络共享权限，整个共享目录的权限还与目录自身的系统权限有关。

3）rpc.lockd：可用来锁定文件，用于多客户端同时写入

4）rpc.statd：检查文件的一致性，与rpc.lockd有关

以上的进程查看均可以执行“man 进程名”来查看进程的功能细节。


## 3.2 配置NFS服务 ##
### 3.2.1 NFS配置文件路径 ###
    
    NFS常用路径	说明
    /etc/exports	NFS服务主配置文件，配置NFS具体共享服务的地方，默认内容为空。
    /usr/sbin/exportfs	NFS服务管理命令。例如：可以加载NFS配置生效，还可以直接NFS共享目录，即无需配置/etc/exports实现共享
    
    exportfs -rv  ===  /etc/init.d/nfs  reload
    /usr/sbin/showmount	查看NFS配置及挂载结果的命令
    /var/lib/nfs/etab	NFS配置文件的完整参数设定的文件（有很多默认的参数）
    /var/lib/nfs/xtab	适合CentOS5.x
    
    
/etc/xexports配置文件格式

NFS共享目录  NFS客户端地址1（参数1，参数2...） 客户端地址2（参数1，参数2...）
NFS共享目录 NFS客户端地址（参数1，参数2...）

    [root@server /]# cat  /etc/exports 
    #shared  /data  for  bbs  by Siffre
    /data  192.168.159.141/24(rw,sync)
    
    查看帮助方法：man  exports

参数含义如下：

1）NFS共享目录：为NFS服务端要共享的实际目录，要用绝对路径。
NFS客户端地址：为NFS服务器端授权的可以访问共享目录的NFS客户端地址，可以为单独的IP地址或主机、域名等，也可以为整个网段地址，还可以用“*”来匹配所有客户端服务器可以访问，客户端指的是前端的业务服务器。

指定NFS客户端地址的配置详细说明：
    
    客户端地址	实例	说明
    授权单一客户端访问NFS	10.0.0.30	一般情况，生产环境中此配置不多
    x	10.0.0.0/24	其中24等同于255.255.255.0，指定网段为生产环境中最常见的配置
    授权整个网段可访问的NFS	10.0.0.*	指定网段的另外写法
    授权某个域名客户端访问NFS	nfs.Siffre.cc	此方法生产环境中一般情况不常用
    授权整个域名客户端访问NFS	*.Siffre.cc	不常用
    

### 3.2.2 NFS配置参数权限设置 ###

NFS配置文件权限参数说明，即/etc/exports文件配置格式中小括号()里的参数：
#     
        参数名	说明
    rw	read-write，读写权限
    ro	read-only，只读权限
    sync	同步，请求或写入数据时，数据同步写入到NFS Server的硬盘后才返回
    async	异步，请求或写入数据时，先返回请求，再将数据写入到内存缓存和硬盘中，即异步写入数据。此参数可以提升NFS性能，但是会降低数据的安全性。因此，一般情况下建议不用，如果NFS处于瓶颈状态，并且允许数据丢失的话可以打开此参数提升性能。
    写入时会先放到内存缓冲区，等硬盘有空档再写入磁盘，这样可以提升写入效率，风险：服务器宕机或不正常关机，会损失缓冲区中未写入磁盘的数据。
    no_root_squash	访问NFS Server共享目录的用户，如果是root，它对该共享目录具有root权限，这个配置原本为无盘客户端准备的，用户应避免使用。
    root_squash	对于访问NFS Server共享目录的用户，如果是root，则它的权限将被压缩成匿名用户，同时它的UID和GID通常会变成nobody或nfsnobody账号身份。
    all_squash	不管访问NFS Server共享目录的用户身份如何，它的权限都将被压缩成匿名用户，同时他的UID和GID都会变成nobody或nfsnobody账号身份，在多个NFS客户端同时读写NFS Server数据时，这个参数很有用。
    anonuid	参数以anon*开头是指anonymous匿名用户，这个用户的UID设置值通常为nobody和nfsnobody的UID值，当然我们也可以自行设置这个UID值。但是UID必须存在于/etc/passwd中。在多个NFS Client时，如多台web server共享一个NFS目录时，通过这个参数可以使得不同的NFS Client写入的数据对所有的NFS  Client保持同样的用户权限，即为配置的匿名UID对应用户权限，这个参数很有用。
    anongid	同上，区别就在于uid和gid
    secure	不允许Client使用大于1024的端口号，也就是从Server传递的资料到Client端的目标port要小于1024，此时，Client端一定要使用root账号才能mount远端NFS Server，建议使用insecure
    insecure	允许Client端自行决定个你自己机器使用的port，通常都会设置这个，如此非root账号的client端才能 mount NFS Server
    nohide	当export出两个目录，而其中一个目录是另外一个目录的子目录，例如：我们使用虚拟目录的例子，此时我们mount跟目录时，会自动把所有子目录mount起来。建议使用这个选项比较方便，尤其是在NFSv4有虚拟目录的情形。
    hide	当mount跟目录时，export出的子目录需要自己明确的再挂载
    subtree_check	当分享的目录是某个档案系统的子目录，选用这个可以确定父目录的权限让NFS Server分享使用。
    no_subtree_check	刚好和上面的相反，因为不做权限测试，效能比较好
    fsid=0	定义NFSv4中的目录，只能有一个 #
    
    
如图所示：    
    
![](http://i.imgur.com/UdITVOP.png)


### 3.2.3 NFS客户端mount共享目录知识 ###

（1）NFS客户端挂载命令格式
    
    挂载命令	文件格式	服务端共享目录	客户端挂载目录
    mount	-t  nfs	10.0.0.12：/data	/mnt
    完整命令： mount -t nfs 10.0.0.12：/data  /mnt   #在客户端执行

（2）执行挂载过程
    
    [root@cxp ~]# df -h   #查看磁盘情况
    Filesystem  Size  Used Avail Use% Mounted on
    /dev/sda318G  7.4G  9.2G  45% /
    tmpfs   997M 0  997M   0% /dev/shm
    /dev/sda1   190M   89M   88M  51% /boot
    
    [root@cxp ~]# mount -t nfs  192.168.159.141:/data  /mnt  #挂载命令
    
    [root@cxp ~]# df -h  #磁盘情况
    FilesystemSize  Used Avail Use% Mounted on
    /dev/sda3  18G  7.4G  9.2G  45% /
    tmpfs 997M 0  997M   0% /dev/shm
    /dev/sda1 190M   89M   88M  51% /boot
    192.168.159.141:/data
       18G  7.4G  9.2G  45% /mnt  #挂载结果
    
    [root@cxp ~]# grep  mnt   /proc/mounts
    192.168.159.141:/data /mnt nfs4 rw,relatime,vers=4.0,rsize=262144,wsize=262144,namlen=255,hard,proto=tcp,port=0,timeo=600,retrans=2,sec=sys,clientaddr=192.168.159.144,local_lock=none,addr=192.168.159.141 0 0
    
（3）开机自动挂载

方法一：放在/etc/rc.local里面

    [root@cxp ~]# vim  /etc/rc.local 
    #!/bin/sh
    #
    # This script will be executed *after* all the other init scripts.
    # You can put your own initialization stuff in here if you don't
    # want to do the full Sys V style init stuff.
    
    touch /var/lock/subsys/local
    /bin/mount  -t nfs 192.168.159.141:/data  /mnt
    #尽量用全路径命令

方法二：写入fstab
    
    [root@cxp ~]# vim   /etc/fstab 
    
    #
    # See man pages fstab(5), findfs(8), mount(8) and/or blkid(8) for more info
    #
    UUID=51b98897-1b10-406c-a94c-52f2bddfe47d /   ext4defaults1 1
    UUID=b2169e1e-8088-4e06-a38c-f740e9e06073 /boot   ext4defaults1 2
    UUID=6c911b4d-ad07-413f-82c8-63f584981cd4 swapswapdefaults0 0
    tmpfs   /dev/shmtmpfs   defaults0 0
    devpts  /dev/ptsdevpts  gid=5,mode=620  0 0
    sysfs   /syssysfs   defaults0 0
    proc/proc   procdefaults0 0
    192.168.159.141:/data/mntnfs  default   0 0
    #后面两个0 0 表示：是否备份， 是否检查；
    注意：第二个不能写1，写了就不能启动计算机了（后面会演示写1怎么解决）
    提示：加载网络文件时，最好不要写在fstab，可能导致机器起不来，fstab加载优先于网络。
    
（4）企业生产环境NFS客户端挂载建议

1）把NFS rpc服务的启动命令和挂载命令均放入/etc/rc.local，然后在通过nagios监控软件监控开机后的挂载情况。

2）如果决定考虑把挂载命令放入/etc/fstab里，那么关键是第5,6列的数字要为0，即不备份，不做磁盘 检查。

### 3.2.4 NFS客户端mount挂载参数说明 ###

    （1）通过man  8  mount   查看命令帮助
    [root@cxp ~]# man  8  mount
    
    MOUNT(8)   Linux Programmer’s Manual  MOUNT(8)
    
    NAME
       mount - mount a filesystem
    
    SYNOPSIS
       mount [-lhV]
    
       mount  -a  [-fFnrsvw] [-t vfstype] [-O
       optlist]
    
       mount  [-fnrsvw]   [-o
       option[,option]...]  device|dir
    
       mount   [-fnrsvw]   [-t  vfstype]  [-o
       options] device dir
    
    。。。省略。。。

（2）mount -o选项后的常用参数（只有出现在/etc/fstab才有效）
    
    1)async：涉及到文件系统I/O的操作都是异步处理，即不会同步写入磁盘，参数提高性能，但是会降低数据的安全系数。
    
    2）atime：每次访问时，同步更新每访问的inode时间，默认选项。高并发情况下，可通过加noatime，来取消默认选项，以达到提升IO性能，达到优化IO目的。
    
    3）auto：通过-a选项可被自动挂载
    
    4)defaults:包括：rw，suid，dev，exec，auto，nouser，async
    
    5）exec：允许执行二进制文件，取消这个选项，可以提升系统安全
    
    6）nodiratime：不更新文件系统上的directory inode访问时间，高并发环境，可以提升系统的I/O性能。
    
    7）remount：尝试从新挂载一个已经挂载了的文件系统，通常被用来改变一个文件系统的挂载标志，从而使得一个只读文件系统变的可写，这个动作不会改变设备或者挂载点。
    
    8）ro：只读
    
    9）rw：读写
    
### 3.2.5 NFS客户端mount挂载优化 ###

（一）有关系统安全挂载参数选项

在企业场景，一般来说，NFS共享服务器的只是普通静态数据（图片、附件、视频），不需要执行suid、exec等权限，挂载的这个文件系统只能作为数据数据存储之用，无法执行程序，对于客户端来讲增加了安全性。
    
    安全挂载参数：
    mount  -t nfs -o nosuid,noexec,nodev,rw,192.168.159.141:/data  /mnt

测试NFS服务器端默认共享参数选项：

（1）测试环境：

    [root@cxp ~]# cat  /etc/redhat-release 
    CentOS release 6.6 (Final)
    [root@cxp ~]# uname -r
    3.10.79-1.el6.elrepo.x86_64

（2）NFS  Server  192.168.159.141
    
    [root@cxp ~]# cat  /etc/exports 
    #shared  /data  for  bbs  by Siffre
    /data  192.168.159.141/24(rw,sync)
    
    [root@cxp ~]# exportfs  -rv
    exporting 192.168.159.141/24:/data

（3）NFS Client 192.168.159.144
    
    [root@cxp ~]# showmount -e   192.168.159.141
    Export list for 192.168.159.141:
    /data 192.168.159.141/24
    
    [root@cxp ~]# mount  -t nfs  192.168.159.141:/data  /mnt
    [root@cxp ~]# df -h
    FilesystemSize  Used Avail Use% Mounted on
    /dev/sda3  18G  7.4G  9.2G  45% /
    tmpfs 997M 0  997M   0% /dev/shm
    /dev/sda1 190M   89M   88M  51% /boot
    192.168.159.141:/data
       18G  7.4G  9.2G  45% /mnt

NFS Client默认挂载参数
    
    [root@cxp ~]# grep  mnt   /proc/mounts
    192.168.159.141:/data /mnt nfs4 rw,relatime,vers=4.0,rsize=262144,wsize=262144,namlen=255,hard,proto=tcp,port=0,timeo=600,retrans=2,sec=sys,clientaddr=192.168.159.144,local_lock=none,addr=192.168.159.141 0 0
    
（4）开始测试安全参数

1）先来一个小注意事项吧：
    
    [root@cxp mnt]# umount  /mnt
    umount.nfs: /mnt: device is busy
    umount.nfs: /mnt: device is busy
    #设备很忙，这是由于我们在当前目录，退出即可
    
    [root@cxp mnt]# cd ..
    [root@cxp /]# umount  /mnt

那么在当前目录可以下载吗？当然是可以的啦！

    [root@cxp mnt]# df  -h
    FilesystemSize  Used Avail Use% Mounted on
    /dev/sda3  18G  7.4G  9.2G  45% /
    tmpfs 997M 0  997M   0% /dev/shm
    /dev/sda1 190M   89M   88M  51% /boot
    192.168.159.141:/data
       18G  7.4G  9.2G  45% /mnt
    
    [root@cxp /]# cd  /mnt
    [root@cxp mnt]# umount  -lf /mnt   #加两个参数搞定
    [root@cxp mnt]# df -h
    Filesystem  Size  Used Avail Use% Mounted on
    /dev/sda318G  7.4G  9.2G  45% /
    tmpfs   997M 0  997M   0% /dev/shm
    /dev/sda1   190M   89M   88M  51% /boot


2）从新挂载并添加参数测试

    [root@cxp /]# mount  -t nfs  -o nosuid,noexec,nodev,rw 192.168.159.141:/data /mnt
    [root@cxp /]# df -h
    FilesystemSize  Used Avail Use% Mounted on
    /dev/sda3  18G  7.4G  9.2G  45% /
    tmpfs 997M 0  997M   0% /dev/shm
    /dev/sda1 190M   89M   88M  51% /boot
    192.168.159.141:/data
       18G  7.4G  9.2G  45% /mnt
    
    [root@cxp mnt]# grep  mnt   /proc/mounts
    192.168.159.141:/data /mnt nfs4 rw,nosuid,nodev,noexec,relatime,vers=4.0,rsize=262144,wsize=262144,namlen=255,hard,proto=tcp,port=0,timeo=600,retrans=2,sec=sys,clientaddr=192.168.159.144,local_lock=none,addr=192.168.159.141 0 0


3）测试shell执行

    [root@cxp mnt]# echo  'echo `pwd`' > test.sh 
    [root@cxp mnt]# cat   test.sh 
    echo `pwd`
    [root@cxp mnt]# cd  ..
    [root@cxp /]# sh   /mnt/test.sh  #可以执行，可以tab补全
    /
    [root@cxp /]#  /mnt/test.sh#不能执行，由于noexec参数，不能tab补全
    -bash: /mnt/test.sh: Permission denied
    [root@cxp /]# chmod  4755  /mnt/test.sh #给予suid权限
    [root@cxp /]# ls -l  /mnt/test.sh 
    -rwsr-xr-x 1 nfsnobody nfsnobody 11 Jul  4 18:35 /mnt/test.sh
    [root@cxp /]# /mnt/test.sh
    -bash: /mnt/test.sh: Permission denied #nosuid参数限制，不能tab补全

4）测试php

    [root@cxp mnt]# vim  test.php
    <?php
    $get_value="My name is Siffre.";
    echo $get_value "\n";
    ?>
    
    [root@cxp mnt]# /application/php/bin/php  /mnt/test.php
    My name is Siffre.

小结：
1.nosuid，noexec对于shell脚本，php脚本的执行也生效

2.对于二进制程序，如cat，生效
注意：通过sh test.sh，以及/application/php/bin/php test.php依然可以执行程序，不带解释器的情况如：/mnt/test.sh ,/mnt/test.php即使有执行权限也是无法执行的。


**（二）有关系统性能的挂载参数选项**

（1）NFS Server默认的挂载参数

    [root@cxp ~]# cat  /var/lib/nfs/etab
    /data	192.168.159.141/24(rw,sync,wdelay,hide,nocrossmnt,secure,root_squash,no_all_squash,no_subtree_check,secure_locks,acl,anonuid=65534,anongid=65534)

根据前面的测试我们知道NFS Client默认的挂载参数

    [root@cxp ~]# grep  mnt   /proc/mounts
    192.168.159.141:/data /mnt nfs4 rw,relatime,vers=4.0,rsize=262144,wsize=262144,namlen=255,hard,proto=tcp,port=0,timeo=600,retrans=2,sec=sys,clientaddr=192.168.159.144,local_lock=none,addr=192.168.159.141 0 0
    
    mount挂载默认参数中的rsize=65536，wsize=65536就是很重要的性能优化参数。
    rsize和wsize设定了NFS Server 和NFS Client之间来往数据块的大小，rsize，wsize大小最好是1024的倍数，对于NFSv4可以到65536.
    如果在客户端挂载时使用了这两个参数，可以让客户端读取和写入数据时，一次性可以读写更多的数据库包，因此，可以提升访问性能和效率。
    除此之外，还有noatime，nodiratime选项，这两个选项是说在读写磁盘的时候，不更新文件和目录的时间戳（即不更新文件系统中文件对应的inode信息），这样就可以减少和磁盘系统的交互，提升读取和写入的效率，因为磁盘是机械的，每次读写都会消耗磁盘IO，而更新文件时间戳对于工作数据必要性不大，最大的问题是增加了访问磁盘的IO次数，拖慢系统性能。
    
企业生产环境NFS性能优化挂载例子：
    
    mount -t nfs  -o noatime,nodiratime 192.168.159.141:/data 
    mount -t nfs  -o nosuid,noexec,nodev,noatime,nodiratime,intr,rsize=65536,wsize=65536 192.168.159.141:/data 
    mount -t nfs  192.168.159.141:/data

（2）测试

    [root@cxp mnt]# grep  mnt  /proc/mounts
    192.168.159.141:/data /mnt nfs4 rw,nosuid,nodev,noexec,relatime,vers=4.0,rsize=262144,wsize=262144,namlen=255,hard,proto=tcp,port=0,timeo=600,retrans=2,sec=sys,clientaddr=192.168.159.144,local_lock=none,addr=192.168.159.141 0 0
    
    [root@cxp mnt]# time  dd  if=/dev/zero of=/mnt/testfile1  bs=9k count=2000
    2000+0 records in
    2000+0 records out
    18432000 bytes (18 MB) copied, 1.55156 s, 11.9 MB/s
    
    real	0m1.584s
    user	0m0.002s
    sys	0m0.419s

将rsize和wsize调小

    [root@cxp mnt]# mount -t nfs  -o noexec,nosuid,rsize=1024,wsize=1024 192.168.159.141:/data  /mnt
    [root@cxp mnt]# time  dd  if=/dev/zero of=/mnt/testfile2  bs=9k count=2000
    2000+0 records in
    2000+0 records out
    18432000 bytes (18 MB) copied, 3.34387 s, 5.5 MB/s
    
    real	0m3.352s
    user	0m0.002s
    sys	0m0.489s
    发现没？性能降低了！
    现在我们少加几个参数在来测试一下：
    [root@cxp mnt]# mount -t nfs  -o rsize=1024,wsize=1024 192.168.159.141:/data  /mnt
    [root@cxp mnt]# time  dd  if=/dev/zero of=/mnt/testfile3  bs=9k count=2000
    2000+0 records in
    2000+0 records out
    18432000 bytes (18 MB) copied, 2.65285 s, 6.9 MB/s
    
    real	0m2.659s
    user	0m0.007s
    sys	0m0.516s
    
    相比第二个，性能有所提升了！
    
    通过此命令，我们可以做各种想要的测试！记住：数据说话！
    
**（三）NFS网络文件系统优化挂载参数建议：**
    
    mount -t nfs -o noatime,nodiratime,rsize=65536,wsize=65536 192.168.159.141:/data /mnt

非性能的参数越多，速度越慢

官方建议：
    内核的优化
    Cat >>/etc/sysctl.conf<<EOF
    >net.core.wmen_default = 8388608  #指定发送套接字缓冲区大小的缺省值
    >net.core.rmen_default = 8388608  #指定接收套接字缓冲区大小的缺省值
    >net.core.rmen_max = 16777216   #指定接收套接字缓冲区大小的最大值
    >net.core.wmen_max = 16777216   #指定发送套接字缓冲区大小的最大值
    >EOF

更多参数及优化请参考官方网站！

小结：

生产场景NFS共享存储优化

1.硬件：sas/ssd磁盘，买多块，raid0/raid10，网卡号

2.服务端： /data 192.168.159.141/24(all_squash，async,anonuid=555,anongid=555)

3.客户端挂载：rsize，wsize，noatime，nodiratime，nosuid，noexec，soft（hard，intr）

4.内核优化  


### 3.2.6 exportfs命令说明 ###

语法：  

    /usr/sbin/exportfs  [-avi] [-o options,...] [client:/path ...]
    /usr/sbin/exportfs  -r [-v] 相当于 /etc/init.d/nfs reload
    /usr/sbin/exportfs  [-av] -u [client:/path ..]

命令行可以修改相关参数
    
    [root@cxp ~]# exportfs  -o rw,sync,anonuid=500,anongid=500,all_squash 192.168.159.141:/data
    
3.2.7 rpcinfo命令

命令格式：
    
    rpcinfo  [-n 端口号] -u 主机名 程序名 [版本号] 
    rpcinfo  [-n 端口号] -u 主机 程序名 [版本号] 

如下：

    [root@cxp ~]# rpcinfo -p  localhost
       program vers proto   port  service
    1000004   tcp111  portmapper
    1000003   tcp111  portmapper
    1000002   tcp111  portmapper
    1000004   udp111  portmapper
    1000003   udp111  portmapper
    1000002   udp111  portmapper
    1000111   udp875  rquotad
    1000112   udp875  rquotad
    1000111   tcp875  rquotad
    1000112   tcp875  rquotad
    1000051   udp  41244  mountd
    1000051   tcp  47912  mountd
    1000052   udp  49709  mountd
    1000052   tcp  38983  mountd
    1000053   udp  51165  mountd
    1000053   tcp  40519  mountd
    1000032   tcp   2049  nfs
    1000033   tcp   2049  nfs
    1000034   tcp   2049  nfs
    1002272   tcp   2049  nfs_acl
    1002273   tcp   2049  nfs_acl
    1000032   udp   2049  nfs
    1000033   udp   2049  nfs
    1000034   udp   2049  nfs
    1002272   udp   2049  nfs_acl
    1002273   udp   2049  nfs_acl
    1000211   udp  47190  nlockmgr
    1000213   udp  47190  nlockmgr
    1000214   udp  47190  nlockmgr
    1000211   tcp  53415  nlockmgr
    1000213   tcp  53415  nlockmgr
    1000214   tcp  53415  nlockmgr
    
### 3.2.8 NFS Server 防火墙控制 ###

注：真正的企业环境的存储服务器都属于内网环境，都无需防火墙。因此，此处可以不配置，如果需要配置的话就有两种方法：

（1）仅允许IP端访问
    
    iptables -A INPUT -s 192.168.159.0/24 -j ACCEPT

（2）允许IP段加端口访问

    iptables -A INPUT -i eth1 -p tcp -s 192.168.159.0/24 --dport 111 -j ACCEPT
    iptables -A INPUT -i eth1 -p udp -s 192.168.159.0/24 --dport 111 -j ACCEPT
    iptables -A INPUT -i eth1 -p udp -s 192.168.159.0/24 --dport 2049 -j ACCEPT
    iptables -A INPUT -i eth1 -p udp -s 192.168.159.0/24  -j ACCEPT


## 3.3 NFS 故障排除 ##

### 3.3.1 mount clntudp_create:RPC: ### Program not registered

解决：nfs和rpcbind启动顺序

### 3.3.2 RPC Error：Program not registered ###

解决：网络原因，重启nfs和rpcbind服务

### 3.3.3 cannot registered service：RPC: Uable to recive; error = Connection refused ###

解决：rpcbind没有启动，重启即可


## 3.4 NFS服务生产场景应用 ##
### 3.4.1 作用 ###

NFS服务可以让不同的客户端挂载使用同一个目录，作为共享存储使用，这样可以保证不同节点客户端数据的一致性，在集群架构环境中经常会用到。

（1）优点

1）简单，容易上手，容易掌握，数据时在文件系统上的

2）方便，部署快速，维护简单

3）可靠，从软件层面上看，数据可靠性高，经久耐用

（2）局限性

1）局限性是存在单点故障，如果nfs server宕机了所有客户端都不能访问共享目录，这个可以通过负载均衡及高可用方案弥补

2）在高并发的场合，NFS效率性能有限（一般几千万以下的PV的网站不是瓶颈，除非网站架构太差）

3）客户端认证时基于IP和主机名的，安全性一般

4）NFS数据时明文，对数据完整性不做验证

5）多台机器挂载NFS服务器时，连接管理维护麻烦，尤其NFS服务端出问题后，所有NFS客户端都挂掉状态（测试环境可以使用autofs自动挂载解决），耦合度太高

（3）生产场景

中小型网站（2000万PV以下）线上应用，都有用武之地，门户网站也会有其他方面的应用，未必是线上存储使用。

### 3.4.2 扩展：autofs自动挂载（了解） ###

Autofs可以实现当用户访问的时候在挂载，如果没有用户访问，指定之间之内，就自动挂载。可以解决NFS服务器和客户端紧密耦合的问题。缺点：用户请求才挂载，所以开始请求的瞬间效率较差。Autofs用于测试环境中，或者并发很低的生产环境（家目录LDAP）中，一般的大并发场景不用。

# 四、NFS高可用 #

# 五、NFS +Docker方案 #
## 5.1 方案一 ##
### 5.1.1 方案假设：Docker里面跑NFS服务 ###
（1）方案架构图，如图所示：

![](http://i.imgur.com/YmHmbjZ.png)

### 5.1.2方案分析 ###


## 5.2 方案二 ##
### 5.2.1 方案假设：NFS机器上架设Docker，Docker跑整体服务 ###

（1）方案架构图，如图所示

![](http://i.imgur.com/pTKsqpl.png)

### 5.1.2方案分析 ###


## 5.2 方案二 ##
### 5.2.1 方案假设：NFS机器上架设Docker，Docker跑整体服务 ###

（1）方案架构图，如图所示

![](http://i.imgur.com/RqCzcbt.png)


### 5.3.2 方案分析 ###
### 5.3.3 实战检测 ###
### 5.3.4 测试结果 ###
