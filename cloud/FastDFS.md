- ### 从应用层面详解fastdfs各组件

- ### fastdfs的多服务器场景使用及部署配置说明

##  

## 相关的文章

1、单体安装教程 https://blog.csdn.net/suoyanming/article/details/88797360

2、开源中国fastdfs主页 [p/fastdfs](https://www.oschina.net/p/fastdfs)

3、github主页（不确定是否是原作者维护） [happyfish100/fastdfs](https://github.com/happyfish100/fastdfs)

4、对fastdfs-nginx-module 实现原理讲的非常清楚 **https://www.cnblogs.com/littleatp/p/4361318.html**   

 

# **一、FastDFS**

1、FastDFS是一个开源的轻量级[分布式文件系统](https://baike.baidu.com/item/分布式文件系统/1250388)，它对文件进行管理，功能包括：文件存储、文件同步、文件访问（文件上传、文件下载）等，解决了大容量存储和负载均衡的问题。特别适合以文件为载体的在线服务，如相册网站、视频网站等等。

2、FastDFS为互联网量身定制，充分考虑了冗余备份、负载均衡、线性扩容等机制，并注重高可用、高性能等指标，使用FastDFS很容易搭建一套高性能的文件服务器集群提供文件上传、下载等服务。

# **二、深入认识FastDFS**

-    任何一个中间件的应用，都必须深入了解该中间件内部各组件的承担的功能角色、运行机制，能深入了解各组件的实现原理更好。这样才能灵活应对实际应用场景、多变的业务需求、生产环境应急等问题，快速实施架构调整。
-    我们一直在使用FastDFS作为图片文件数据库，部署架构为单体（即：一个tracker、一个storage、一个group）,由于本次用于部署fastdfs的服务器硬盘空间报警，当务之急必须更改fastdfs部署架构，扩展存储。
-    下面从项目总体情况、tracker 、storage、fastdfs-nginx-module 、group 组件详细说明其功能角色及运行机制

## **1、项目总体情况**

- ​    fastdfs是开源的项目
- ​    通过github源码可看出，该项目是基于C语言开发的
- ​    fastdfs是基于操作系统OS的文件管理系统功能之上进行分布式文件管理（Linux、FreeBSD等），通过看文件在硬盘的保存方式也可以得出
- ​    提供C、Java和PHP API接口

![img](https://img-blog.csdnimg.cn/20190325165732189.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3N1b3lhbm1pbmc=,size_16,color_FFFFFF,t_70)

![img](https://img-blog.csdnimg.cn/20190325165748158.png)

## **2、tracker跟踪器**

- **主要做调度工作， 起负载均衡的作用**

- **在内存中记录集群中所有存储组group和存储服务器storage 的状态信息, 是客户端和数据服务器交互的枢纽**

- **tracker的核心工作内容**：

（1） 记录集群中有多少个group（group1\group2....）

(2) 每个group 分布在那个几个storage上，以及storage所在机器的ip，端口等信息，group之间的同步由tracker 和storage一起完成（后面细讲）

（3）如果同一个group 存在多个storage, 而这些storage又被分布在一台或多台机器上，那么对该group上传或读取文件具体落到那个机器上（即那个storage）？（有点绕）

tracker完美的解决了这个问题，即对分布式部署架构下：多group、多storage的上传和下载做负载均衡策略，通过配置tracker.conf可实现具体负载均衡策略

(4) tracker 可部署多台，多个tracker在服务器内存中记录的信息是一样的，通过nginx对tracker做负载均衡，以提高并发性能及容灾能力

（5）tracker 不去主动读取storage的相关信息，而是由storage主动推送给tracker （**这也是为什么必须先启动tracker的原因**） 

（6）以下图片摘自网上 ： 上传文件过程 、下载文件过程，通过图片可以看到，tracker的核心工作是为客户端找到一个storage, client客户端和storage进行上传下载通信。

![img](https://img-blog.csdnimg.cn/201903251658268.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3N1b3lhbm1pbmc=,size_16,color_FFFFFF,t_70)

- ###### **tracker.conf (在分布式部署架构下，通过tracker负载均衡给client端返回特定storage信息，而负载均衡的策略配置主要在tracker.conf中)**

  ## 核心参数配置说明


### （1）disabled=false

​     **#配置文件是否失效**

​     \# is this config file disabled

​    \# false for enabled

​    \# true for disabled disabled=false

​    \# is this config file disabled # false for enabled # true for disabled

### （2）port=22122

​      \#服务端口

​      \# the tracker server port

### **（3） base_path=/data/fastdfs/tracker**

​       \# 存放track 数据及日志文件目录 

​       \# the base path to store data and log files

​		work_threads=4

​     \#时线程数：一般和cpu的个数设为同一个值     

​     \# work thread count, should <= max_connections

​     \# default value is 4

​      \# since V2.00

### **（4）（重要） store_lookup=1** 

**上传文件选择哪个一个group 的 策略：0：轮询；1:指定组 ； 2： 负载均衡，选择剩余存储空间最大的组group 上传文件**

​       \# the method of selecting group to upload files

​       \# 0: round robin

​       \# 1: specify group

​      \# 2: load balance, select the max free space group to **upload file**

### **（5）（重要） store_group=group2**

​    \# 当 store_lookup=1 时，该配置有效，指定存储的组名

​     \# which group to upload file

​     \# when store_lookup set to 1, must set store_group to the group name

### **（6）（重要） store_server=0**

​    \# 应用场景： 存在多个相同的组，例如group1 ， 在多个storage 服务器上 例如：192.168.0.171 、，

​              当上传文件时优先选择那个storage的策略配置**：~ 0：轮询 ；1：按ip升序排序后选择第一个ip，即最小的那个ip (192.168.0.164)；2：按优先级排列的第一个服务器顺序，数字越小优先级越 高，storage服务器的优先在storage.conf中配置 upload_priority 参数**

![img](https://img-blog.csdnimg.cn/20190325165901337.png)

  \# **which storage server to upload file**

​    \# 0: round robin (default)

​    \# 1: the first server order by ip address

​    \# 2: the first server order by priority (the minimal)

### **（7）（重要） store_path=0**

​    \# 应用场景：选择具体一个组的那一条存储路径（**一个group有多条存储路径，一般一个服务器有两块大硬盘挂载到了两个路径下，专门用来存放文件**），~0：轮询，2：负载均衡，选择剩余空间最大的路径 

​     （**逻辑~重要**） 通过上面的配置参数确定了3件事的基础上，该配置才会起作用：

​    （1）确定了要存储在那个group上，例如group1；（2）确定上传文件要保存在那一台storage 中的group1，假如是192.168.0.171；（3） 此时如果 192.168.0.171 上的storage server中group1 有两个存储路径，即store_path0,store_path1**,(对应的文件路径即M00,M01)**

![img](https://img-blog.csdnimg.cn/20190325165922761.png)

 \# which path(means disk or mount point) of the storage server to upload file

​    \# 0: round robin

   \# 2: load balance, select the max free space path to upload file

### **（8）download_server=0**

​    \# 下载文件时存储服务器的选择策略； 应用场景：要下载的文件所在group 存在多个storage 服务器上， ~0：轮询 ；1：当前文件上载到的源存储服务器

​    \# which storage server to download file

​    \# 0: round robin (default)

​    \# 1: the source storage server which the current file uploaded to

### **（9）reserved_storage_space = 10%**

  \# 给系统或其他应用程序预留存储空间设置 

  \#（**重要**）场景：某一个group所在某一个storage服务器（可能存在多个服务器上）剩余的存储空间小于等于这个阀值时，则文件不能被保存，即使该group的其他storage服务器还有很大的存储空间

  \# reserved storage space for system or other applications.

  \# if the free(available) space of any stoarge server in

  \# a group <= reserved_storage_space,

  \# no file can be uploaded to this group.

  \# bytes unit can be one of follows:

  \### G or g for gigabyte(GB)

  \### M or m for megabyte(MB)

  \### K or k for kilobyte(KB)

  \### no unit for byte(B)

  \### XX.XX% as ratio such as reserved_storage_space = 10%

![img](https://img-blog.csdnimg.cn/20190325165954851.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3N1b3lhbm1pbmc=,size_16,color_FFFFFF,t_70)

![img](https://img-blog.csdnimg.cn/2019032517001063.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3N1b3lhbm1pbmc=,size_16,color_FFFFFF,t_70)

![img](https://img-blog.csdnimg.cn/20190325170031269.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3N1b3lhbm1pbmc=,size_16,color_FFFFFF,t_70)

## **3、storage 存储服务器**

- storage 定期向tracker发送心跳，报告自己的状态，tracker会将同组的 storage server信息返回给storage （该部分逻辑后面再细讲） 

- tracker不负责具体的文件上传、下载实现，这些都是有storage完成的

- storage保存文件和文件的属性

- storage server是基于操作系统的文件管理系统进行文件管理的（上面有提到）

- group之间文件同步由storage server 和tacker server一起完成的（该部分逻辑后面再细讲） 

- storage server的状态（7个）

- FDFS_STORAGE_STATUS_INIT :初始化，尚未得到同步已有数据的源服务器

- FDFS_STORAGE_STATUS_WAIT_SYNC :等待同步，已得到同步已有数据的源服务器

- FDFS_STORAGE_STATUS_SYNCING :同步中

- FDFS_STORAGE_STATUS_DELETED :已删除，该服务器从本组中摘除（注：本状态的功能尚未实现）

- FDFS_STORAGE_STATUS_OFFLINE :离线

- FDFS_STORAGE_STATUS_ONLINE :在线，尚不能提供服务

- FDFS_STORAGE_STATUS_ACTIVE :在线，可以提供服务

### storage.conf 核心参数配置

### （1）port=23000

   \# storage 服务端口

   \# the storage server port

### （2）base_path=/usr/local/fastdfs/fdfs_storage

   \#存放storage 服务的数据和日志

   \# the base path to store data and log files

### **（3）store_path0=/usr/local/fastdfs/fdfs_storage**

   \# 存储路径配置，可以配置多个，对应的 store_path_count=1 参数需要累加

   \# store_path#, based 0, if store_path0 not exists, it's value is base_path

   \# the paths must be exist

   \#store_path1=/home/yuqing/fastdfs2

### **（4）tracker_server=192.168.0.171:22122**

   \#tracker 服务的 ip和端口， ip替换为域名也可以，可以配置多个 

   \# tracker_server can ocur more than once, and tracker_server format is

   \# "host:port", host can be hostname or ip address

### **（5）file_distribute_path_mode=0**

  \# 分布式存储文件策略： 当storage下有多个存储路径时，该配置起作用 ~ # 0: 轮询   # 1: 根据文件名hash结果随机存储

  \# the mode of the files distributed to the data path

  \# 0: round robin(default)

  \# 1: random, distributted by hash code 

### （6）upload_priority=10 （在tracker.conf 中有提到）

\# 上传文件事，同组内的storage 服务器优先级设置，且当 tracker.conf 中store_server= 2时 起作用，值越小，优先级越高。

\# the priority as a source server for uploading file.

\# the lower this value, the higher its uploading priority.

\# default value is 10

## **4、group**

 

-  group 分组是fastdfs应对大流量应用系统中处理高并发、高容灾的经典设计，并且group还起到了应用隔离的功能
-  一个group可以存在多个storage中（在storage中也可以提到）
- 根据client端的请求分配到不同的group，文件系统具备直接的负载均衡；
- group内有storage服务节点坏掉时，需从其他group内恢复数据 

## **5、 fastdfs-nginx-module** 

- fastdfs 中storage、tracker 均提供的http服务，可以直接下载文件，但考虑到性能及负载实现难易度的问题，一般都用web服务器来下载文件，例如nginx、apache
- fastdfs-nginx-module 就是fastdfs基于ngnix实现文件http传输的组件，以nginx module的方式添加到nginx 程序中
- 每个storage 均需安装 fastdfs-nginx-module 、Nginx ，当前storage找不到文件时，向**源storage**主机发起redirect重定向或proxy转发代理动作
- fastdfs-nginx-module 安装后目录结构如下图

   说明及图片 摘自：https://www.cnblogs.com/littleatp/p/4361318.html    

​     (1)ngx_http_fastdfs_module.c  ~ nginx 模块接口实现文件，用于向nginx 接入fastdfs-module核心模块逻辑

​    （2）common.c  ~ fastdfs-module核心模块，实现了初始化、文件下载的主要逻辑

​    （3）config   ~ 编译模块所用的配置，里面定义了一些重要的常量，调用fastdfs基础组件功能，以及扩展配置文件路径、文件下载chunk大小

 

 

```html
   （4）mod_fastdfs.conf  ~扩展配置文件的demo，一般会将该文件拷贝到config指定的目录下 例如：/etc/fdfs
```

![img](https://img-blog.csdnimg.cn/20190325170102450.png)

- 初始化： nginx启动时，  fastdfs-nginx-module 要完成初始化如下图 ，我们一般在mod_fastdfs.conf配置参数，如下图

![img](https://img-blog.csdnimg.cn/20190325170132699.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3N1b3lhbm1pbmc=,size_16,color_FFFFFF,t_70)

![img](https://img-blog.csdnimg.cn/2019032517015280.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3N1b3lhbm1pbmc=,size_16,color_FFFFFF,t_70)

![img](https://img-blog.csdnimg.cn/20190325170205559.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3N1b3lhbm1pbmc=,size_16,color_FFFFFF,t_70)

**（重要）： fastdfs-nginx-module 初始化的过程要加载mod_fastdfs.conf参数，如果本机器下存在多个storage,且有多个group（group1、group2）,则 mod_fastdfs.conf 配置需做如下变动**

###  

**（1）组名：group_name=group1/group2   多个用/区分开**

**（2）设置组个数：group_count = 4**

**（4）设置各group信息：**

**[group1]**

**group_name=group1**

**storage_server_port=23000**

**store_path_count=1**

**store_path0=/usr/local/fastdfs/storage**

**[group2]**

**group_name=group2**

***\*storage_server_port=23010\****

**store_path_count=1**

**store_path0=/usr/local/fastdfs/storage**

### **（重要）通过nginx 从fastdfs下载文件，详细说明可参考https://www.cnblogs.com/littleatp/p/4361318.html**   

![img](https://img-blog.csdnimg.cn/20190325170230230.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3N1b3lhbm1pbmc=,size_16,color_FFFFFF,t_70)

### 6、各组件运行机制总结**(重要)**

- ​    一个group 对应多个 storage (1:N)
- ​    一个storage对应一个group  (1:1)
- ​    一个tracker对应多个storage(1:N)
- ​    一个storage对应多个tracker(1:N) , tracker 和storage的关系是多对多（N:M）
- ​    一个storage下有多个存储路径 store_path(1:N)

![img](https://img-blog.csdnimg.cn/20190325170336507.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3N1b3lhbm1pbmc=,size_16,color_FFFFFF,t_70)

### **7、部署架构汇总**

### 1）单体部署: 单group\单storage\单tracker

![img](https://img-blog.csdnimg.cn/20190325170402345.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3N1b3lhbm1pbmc=,size_16,color_FFFFFF,t_70)

### **2）单服务器多storage部署\**（在实际生产环境中没有意义）\****

###  **多group\多storage\单tracker**

![img](https://img-blog.csdnimg.cn/20190325170430665.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3N1b3lhbm1pbmc=,size_16,color_FFFFFF,t_70)

### 3）多服务器多group且group不互备，单tracker**（我们项目本次硬盘扩展部署架构）**

   由于目前服务器资源紧缺暂不做group互备，后面需要做group互备

![img](https://img-blog.csdnimg.cn/20190325170500305.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3N1b3lhbm1pbmc=,size_16,color_FFFFFF,t_70)

- ###  **部署步骤及参数配置**

  （1）两台服务器分别为192.168.0.171、192.168.0.164， 171服务器担任的功能角色更多一些: 文件下载请求 nginx同一入口（分发到storage1、stroage2）、tracker server  、storage1 - group0（fastdfs-nginx-module）。
  
  164服务器主要负责storage1 -group2 的存储、下载功能，没有tacker server，直接连接171服务器的tracker,需要安装nginx 、 fastdfs-nginx-module

（2) 171、 164 都需要安装 fastdfs 、fastdfs-nginx-module、 nginx 安装步骤 与 知识库文档  [Centos7 上安装 FastDFS](http://192.168.0.109:8090/pages/viewpage.action?pageId=1606067) 一致 ，但注意一点164服务器不用启动及配置tracker

（ 3） 171 tracker.conf 配置

   **171 tracker.conf 核心参数配置说明，其他参数见附件**

 

```
# is this config file disabled

# false for enabled

# true for disabled

disabled=false

 

# the tracker server port

port=22122

 

# the base path to store data and log files

base_path=/data/fastdfs/tracker

 

# the method of selecting group to upload files

# 0: round robin

# 1: specify group

# 2: load balance, select the max free space group to upload file

store_lookup=1

 

# which group to upload file

# when store_lookup set to 1, must set store_group to the group name

store_group=group2

 

# which storage server to upload file

# 0: round robin (default)

# 1: the first server order by ip address

# 2: the first server order by priority (the minimal)

store_server=0

 

# which path(means disk or mount point) of the storage server to upload file

# 0: round robin

# 2: load balance, select the max free space path to upload file

store_path=0

 

 

# which storage server to download file

# 0: round robin (default)

# 1: the source storage server which the current file uploaded to

download_server=0

 

# reserved storage space for system or other applications.

# if the free(available) space of any stoarge server in

# a group <= reserved_storage_space,

# no file can be uploaded to this group.

# bytes unit can be one of follows:

### G or g for gigabyte(GB)

### M or m for megabyte(MB)

### K or k for kilobyte(KB)

### no unit for byte(B)

### XX.XX% as ratio such as reserved_storage_space = 10%

reserved_storage_space = 10%
```



### （4）171 storage.conf 配置

 

**171 storage 核心参数配置，其他参数见附件**

```
# the name of the group this storage server belongs to

#

# comment or remove this item for fetching from tracker server,

# in this case, use_storage_id must set to true in tracker.conf,

# and storage_ids.conf must be configed correctly.

group_name=group0

 

 

# the storage server port

port=23000

 

 

# the base path to store data and log files

base_path=/data/fastdfs/storage

 

 

# path(disk or mount point) count, default value is 1

store_path_count=1

 

 

# store_path#, based 0, if store_path0 not exists, it's value is base_path

# the paths must be exist

store_path0=/data/fastdfs/storage

#store_path1=/home/yuqing/fastdfs2

 

 

# tracker_server can ocur more than once, and tracker_server format is

#  "host:port", host can be hostname or ip address

tracker_server=192.168.0.171:22122

 

 

# the priority as a source server for uploading file.

# the lower this value, the higher its uploading priority.

# default value is 10

upload_priority=10
```

 

### （5）171 fastdfs_nginx_module 配置参数 （mod_fastdfs.conf）

 

**171 mod_fastdfs.conf 核心参数配置，其他参数见附件**

```
# the base path to store log files

base_path=/data/fastdfs/storage

 

 

# if load FastDFS parameters from tracker server # since V1.12 # default value is false

load_fdfs_parameters_from_tracker=true

 

 

# FastDFS tracker_server can ocur more than once, and tracker_server format is

#  "host:port", host can be hostname or ip address

# valid only when load_fdfs_parameters_from_tracker is true

tracker_server=192.168.0.171:22122

 

 

# the port of the local storage server # the default value is 23000

storage_server_port=23000

 

 

 

 

# the group name of the local storage server group_name=group0

# if the url / uri including the group name # set to false when uri like /M00/00/00/xxx # set to true when uri like ${group_name}/M00/00/00/xxx, such as group1/M00/xxx # default value is false

url_have_group_name = true

# path(disk or mount point) count, default value is 1 # must same as storage.conf store_path_count=1

# store_path#, based 0, if store_path0 not exists, it's value is base_path # the paths must be exist # must same as storage.conf

store_path0=/data/fastdfs/storage

#store_path1=/home/yuqing/fastdfs1

 

 

 

 

# set the group count # set to none zero to support multi-group

# set to 0  for single group only

# groups settings section as [group1], [group2], ..., [groupN]

# default value is 0 # since v1.14

group_count = 0

 

 

 

 

# group settings for group #1 # since v1.14

# when support multi-group, uncomment following section

#[group1] #group_name=group1 #storage_server_port=23000

#store_path_count=2

#store_path0=/home/yuqing/fastdfs

#store_path1=/home/yuqing/fastdfs1

 

# group settings for group #2

# since v1.14

# when support multi-group, uncomment following section as neccessary

#[group2] #group_name=group2

#storage_server_port=23000

#store_path_count=1

#store_path0=/home/yuqing/fastdfs
```

 

### （6）171 nginx 参数配置 nginx.conf 

**171 nginx.conf 核心参数配置，详见附件**

```
events {

    worker_connections  1024;

}

http {    

    include       mime.types;    

    default_type  application/octet-stream;

    sendfile        on;    

    #tcp_nopush     on;

        #keepalive_timeout  0;    

    keepalive_timeout  65;

        #gzip  on;

    # 192.168.0.164 storage group2 

    upstream fdfs_group2_164 { 

        server 192.168.0.164:8288 weight=1 max_fails=2 fail_timeout=30s; 

    }

 

    server {        

        listen       8070;        

        server_name  localhost,192.168.0.171;

            #charset koi8-r;

            #access_log  logs/host.access.log  main;

 

            location / {            

            root   html;            

            max_ranges 1;            

            index  index.html index.htm;        

        } 

 

 

        location /group0/M00{            

            root /data/fastdfs/storage/data;            

            ngx_fastdfs_module;            

            if ($request_method = 'OPTIONS') {                

                add_header 'Access-Control-Allow-Origin' '*';                

                add_header 'Access-Control-Allow-Headers' 'Range';               

                add_header 'Content-Type' 'text/plain charset=UTF-8';                

                add_header 'Content-Length' 0;                

                return 204;              }            

            if ($request_method = 'POST') {                

                add_header 'Access-Control-Allow-Origin' '*';                

                add_header 'Access-Control-Allow-Methods' 'GET, POST, OPTIONS';                

                add_header 'Access-Control-Allow-Headers' 'Range';              }            

            if ($request_method = 'GET') {                

                add_header 'Access-Control-Allow-Origin' '*';                

                add_header 'Access-Control-Allow-Methods' 'GET, POST, OPTIONS';                

                add_header 'Access-Control-Allow-Headers' 'Range';                

                add_header 'Access-Control-Expose-Headers' 'Accept-Ranges, Content-Encoding, Content-Length, Content-Range';              }      

            if ($arg_attname ~* \.(doc|docx|txt|pdf|zip|rar|txt|jpg|png|gif|bmp)$) {

                add_header "Content-Disposition" "attachment;filename=$arg_attname";

                    }        

            } 

 

 

        location ~* /group2/(M00|M01) {  

                proxy_next_upstream http_502 http_504 error timeout invalid_header; 

                proxy_pass http://fdfs_group2_164;

                expires 30d; 

            }

 

 

         error_page   500 502 503 504  /50x.html;        

        location = /50x.html {             root   html;         }

    }
```

 

### （7）164 storage参数配置

**164 storage.con 核心参数配置，其他参数见附件**

```
# the name of the group this storage server belongs to # # comment or remove this item for fetching from tracker server, # in this case, use_storage_id must set to true in tracker.conf,

# and storage_ids.conf must be configed correctly.

group_name=group2

 

 

 

 

# the storage server

 port port=23000

 

 

# the base path to store data and log files

base_path=/usr/local/fastdfs/fdfs_storage

 

 

# path(disk or mount point) count, default value is 1 store_path_count=1

# store_path#, based 0, if store_path0 not exists, it's value is base_path # the paths must be exist

store_path0=/usr/local/fastdfs/fdfs_storage

#store_path1=/home/yuqing/fastdfs2

 

 

# tracker_server can ocur more than once, and tracker_server format is #  "host:port", host can be hostname or ip address

tracker_server=192.168.0.171:22122

 

 

# the priority as a source server for uploading file. # the lower this value, the higher its uploading priority. # default value is 10

upload_priority=10
```

 

### （8） 164 fastdfs_nginx_module 参数配置 （mod_fastdfs.conf）

 

**164 mod_fastdfs.conf 核心参数配置，其他参数见附件**

```
# the base path to store log files

base_path=/usr/local/fastdfs/

 

 

# if load FastDFS parameters from tracker server # since V1.12 # default value is false

load_fdfs_parameters_from_tracker=true

 

 

 

 

# FastDFS tracker_server can ocur more than once, and tracker_server format is #  "host:port", host can be hostname or ip address # valid only when load_fdfs_parameters_from_tracker is true tracker_server=192.168.0.171:22122

# the port of the local storage server # the default value is 23000

storage_server_port=23000

 

 

# the group name of the local storage server

group_name=group2

 

 

 

 

# if the url / uri including the group name

# set to false when uri like /M00/00/00/xxx

# set to true when uri like ${group_name}/M00/00/00/xxx, such as group1/M00/xxx

# default value is false

url_have_group_name = true

 

 

 

# path(disk or mount point) count, default value is 1 # must same as storage.conf

store_path_count=1

# store_path#, based 0, if store_path0 not exists, it's value is base_path

# the paths must be exist # must same as storage.conf

store_path0=/usr/local/fastdfs/fdfs_storage

#store_path1=/home/yuqing/fastdfs1

 

 

 

 

# set the group count # set to none zero to support multi-group

# set to 0  for single group only

# groups settings section as [group1], [group2], ..., [groupN]

# default value is 0 # since v1.14

group_count = 0
```

 

### （9）164 nginx参数配置，nginx.conf

**164 nginx.conf 参数配置，详见附件**

```
user  www  www;

worker_processes  12;

error_log  /var/log/nginx/error.log;

 

events {

    worker_connections  1024;

}

 

 

http {

    include       mime.types;

     default_type  application/octet-stream;

       client_max_body_size 10m;

    sendfile        on;

server {

        listen       8288;

        server_name  192.168.0.164,localhost;

 

        location /group2/M00/{

            root /usr/local/fastdfs/fdfs_storage/data;

             ngx_fastdfs_module;

        if ($request_method = 'OPTIONS') {

                 add_header 'Access-Control-Allow-Origin' '*';

                 add_header 'Access-Control-Allow-Headers' 'Range';

                 add_header 'Content-Type' 'text/plain charset=UTF-8';

                 add_header 'Content-Length' 0;

                 return 204;

             }

             if ($request_method = 'POST') {

                 add_header 'Access-Control-Allow-Origin' '*';

                 add_header 'Access-Control-Allow-Methods' 'GET, POST, OPTIONS';

                 add_header 'Access-Control-Allow-Headers' 'Range';

             }

             if ($request_method = 'GET') {

                 add_header 'Access-Control-Allow-Origin' '*';

                 add_header 'Access-Control-Allow-Methods' 'GET, POST, OPTIONS';

                 add_header 'Access-Control-Allow-Headers' 'Range';

                 add_header 'Access-Control-Expose-Headers' 'Accept-Ranges, Content-Encoding, Content-Length, Content-Range';

                    }

                if ($arg_attname ~* \.(doc|docx|txt|pdf|zip|rar|txt|jpg|png|gif|bmp)$) {

                 add_header "Content-Disposition" "attachment;filename=$arg_attname";

                 }

         }  

    }

}
```

 

### **4）真正分布式集群部署：多服务器多group且group间互备，多tracker**

![img](https://img-blog.csdnimg.cn/20190325170807597.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3N1b3lhbm1pbmc=,size_16,color_FFFFFF,t_70)

 