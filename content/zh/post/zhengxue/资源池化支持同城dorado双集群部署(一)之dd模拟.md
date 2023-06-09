+++
title = "资源池化支持同城dorado双集群部署(一)----dd模拟"
date = "2023-04-01"
tags = ["资源池化支持同城dorado双集群部署(一)----dd模拟"]
archives = "2023-04-01"
author = "shirley_zhengx"
summary = "资源池化支持同城dorado双集群部署(一)----dd模拟"
img = "/zh/post/zhengxue/title/img1.png"
times = "9:30"
+++


<!-- TOC -->
- [1. 环境描述](#1.环境描述)
  - [1.1.组网方式](#1.1.组网方式)
  - [1.2.环境配置](#1.2.环境配置)
  - [1.3.系统目录软链接](#1.3.系统目录软链接)
- [2. 编译三方库](#2.编译三方库)
  - [2.1.编译步骤综述](#2.1.编译步骤概述)
  - [2.2.依赖库](#2.2.依赖库)
  - [2.3.源码脚本修改](#2.3.源码脚本修改)
  - [2.4.三方库编译及结果](#2.4.三方库编译及结果)
- [3. 编译数据库](#3.编译数据库)
  - [3.1.准备openGauss-server源码以及代码修改](#3.1.准备openGauss-server源码以及代码修改)
  - [3.2.环境变量](#3.2.环境变量)
  - [3.3.数据库编译与打包](#3.3.数据库编译与打包)
- [4. 安装数据库](#4.安装数据库)
  - [4.1.编译安装](#4.1.编译安装)
  - [4.2.OM安装](#4.2.OM安装)
- [5. 下载链接](#5.下载链接)

<!-- /TOC -->



# 资源池化支持同城dorado双集群部署(一)----dd模拟

资源池化支持同城dorado双集群部署方式：dd模拟(手动部署)、磁阵(手动部署)、集群管理工具部署
          

## 1.环境描述

    针对dd模拟(手动部署)作出指导，环境描述如下：

### &nbsp;&nbsp;1.1.组网方式
<table>
<tbody>
    <tr>
        <td rowspan='2'>生产中心</td>
        <td rowspan='2'>主端</td>
        <td>业务计算节点0</td>
        <td rowspan='2'>主存储节点</td>
        <td rowspan='2'>Dorado</td>
    </tr>
        <td>业务计算节点1</td>
    <tr>
        <td rowspan='2'>容灾中心</td>
        <td rowspan='2'>备端</td>
        <td>业务计算节点0</td>
        <td rowspan='2'>备存储节点</td>
        <td rowspan='2'>Dorado</td>
    </tr>
        <td>业务计算节点1</td>
</tbody>
</table>

&emsp;** 缺个图，后面补充哈！！！**
### &nbsp;&nbsp;1.2.环境配置

&emsp;支持存储远程复制


## 2. 环境搭建

针对资源池化双集群部署之《资源池化dd模拟搭建(手动部署) + dorado同步复制》作出指导，环境搭建如下：

### &nbsp;&nbsp;2.1.创建lun

(1) 主存储创建lun组和lun
&emsp;主存储管控平台(DeviceManager)登录：https://主存储ip:8088
&emsp;在管控平台上创建lun组和lun，并映射到主机之后，在业务节点上查看挂好的lun

(2) 主存储对应的业务计算节点上查看挂好的lun
```
rescan-scsi-bus.sh       upadmin show vlun         lsscsi -is
```

![](../images/dd模拟/lun查询.png)

lun说明：  LUNGroup-zx01-doradoxlog 指dorado同步复制需要的lun(可以理解为共享盘，盘符/dev/sdj)，暂不关注LUNGroup-zx01-dmsdata和LUNGroup-zx01-dmsxlog0，这两个是用于磁阵搭建资源池化集群

修改盘符属组
```
chown zx:zx /dev/sgj
```

(3) 创建同步复制对
&emsp; 在主存储管控平台(DeviceManager)登录：https://主存储ip:8088
&emsp; data protection -> luns -> remote replication pairs(远程复制对) -> create ->选择主存储需要同步复制给备存储的lun -> next
&emsp; <font color='green'> 请原谅这里截图工具的搓，标记笔太难用了，画的蓝圈圈很丑但很个性！</font> 
![](../images/dd模拟/远程复制对创建.png)

选择同步 -> Automatic -> 备存储的存储池名称 -> next
![](../images/dd模拟/远程复制对创建1.png)

(4) 备存储创建lun组和lun
&emsp;备存储管控平台(DeviceManager)登录：https://备存储ip:8088
&emsp;在管控平台上创建lun组，并映射到主机

由于第(3)步创建了远程复制对，会自动在备存储上创建同步复制对应的lun，名字和主存储一致(即备存储上会有一样的lun名字为LUNGroup-zx01-doradoxlog)，在备存储管控平台上查看如下图
![](../images/dd模拟/备存储doradoxlog.png)

(5) 备存储上的lun进行映射
方式1：直接映射到业务计算节点上，不需要提前创建lun组，如果存在多个业务接待你需要映射到每一个业务节点上
选择more -> Map -> node2host01 -> ok   将dorado同步复制功能需要的lun映射到备集群业务节点上
![](../images/dd模拟/备存储映射节点.png)

&emsp;<font color='red'>@温馨提示@</font>：node2host01指为业务节点ip创建的机器名称，名字可自行定义

&emsp;方式2：在lun组中添加该lun，要提前创建lun组，直接会映射到lun组中的所有业务节点上

(6) 备存储对应的业务计算节点上查看挂好的lun
```
rescan-scsi-bus.sh       upadmin show vlun         lsscsi -is
```

![](../images/dd模拟/备存储业务节点lun查询.png)

&emsp;lun说明：  LUNGroup-zx01-doradoxlog 指dorado同步复制需要的lun(可以理解为共享盘，盘符/dev/sdi)

修改盘符属组
```
chown zx:zx /dev/sgi
```
### &nbsp;&nbsp;2.2.下载源码编译
&emsp;如果用已打包好的openGauss-server包则跳过该步骤，进行2.3，如果修改代码开发中，则进行代码更新并编译，如下步骤：

(1) 下载三方库
&emsp;根据平台操作系统下载对应三方库，三方库下载地址：https://gitee.com/opengauss/openGauss-server 主页上README.md中查找需要的三方库binarylibs

    获取master分支openEuler_x86系统对应的三方库
```
wget https://opengauss.obs.cn-south-1.myhuaweicloud.com/latest/binarylibs/openGauss-third_party_binarylibs_openEuler_x86_64.tar.gz
```
(2) 下载cbb并编译
```
git clone https://gitee.com/opengauss/CBB.git -b master cbb
cd CBB/build/linux/opengauss
sh build.sh -3rd $binarylibsDir -m Debug
```
&emsp;编译成功会自动将二进制放入三方库openGauss-third_party_binarylibs_openEuler_x86_64/kernel/component目录下
(3) 下载dss并编译
```
git clone https://gitee.com/opengauss/DSS.git -b master dss
cd CBB/build/linux/opengaussDSS
sh build.sh -3rd $binarylibsDir -m Debug
```

(4) 下载dms并编译
```
git clone https://gitee.com/opengauss/DMS.git -b master dms
cd CBB/build/linux/opengauss
sh build.sh -3rd $binarylibsDir -m Debug
```

(5) 下载openGauss-server并编译
&emsp;编译过程需要cbb、dss、dms的二进制，会从openGauss-third_party_binarylibs_openEuler_x86_64/kernel/component中获取
```
git clone https://gitee.com/opengauss/openGauss-server.git -b master openGauss-server
sh build.sh -3rd $binarylibsDir -m Debug
```
&emsp;编译完之后的二进制存放在openGauss-server/mppdb_temp_install/目录下


### &nbsp;&nbsp;2.3.环境变量
由于机器资源不足，这里以一个业务计算服务器上部署一主一备为例
(1) 主集群主节点对应的ss_env0

环境变量
```
export HOME=/opt/omm
export GAUSSHOME=${HOME}/openGauss-server/mppdb_temp_install/
export GAUSSLOG=${HOME}/cluster/gausslog0
export SS_DATA=${HOME}/cluster/ss_data
export DSS_HOME=${HOME}/cluster/ss_data/dss_home0
export LD_LIBRARY_PATH=$GAUSSHOME/lib:$LD_LIBRARY_PATH
export PATH=$GAUSSHOME/bin:$PATH
```
`Tips`: 环境变量里面一定要写export，即使`echo $GCC_PATH`存在，也要写export才能真正导入路径

参数说明：
HOME 为用户自己创建的工作目录；
GAUSSHOME 为编译完成的目标文件路径，包含openGauss的bin、lib等；
GAUSSLOG 为运行时的日志目录，包含dss、dms等日志
SS_DATA 为共享存储的根目录，即dss相关配置的根目录
DSS_HOME 为dssserver配置对应的目录

(2) 主集群备节点对应的ss_env1
环境变量
```
export HOME=/opt/omm
export GAUSSHOME=${HOME}/openGauss-server/mppdb_temp_install/
export GAUSSLOG=${HOME}/cluster/gausslog1
export SS_DATA=${HOME}/cluster/ss_data
export DSS_HOME=${HOME}/cluster/ss_data/dss_home1
export LD_LIBRARY_PATH=$GAUSSHOME/lib:$LD_LIBRARY_PATH
export PATH=$GAUSSHOME/bin:$PATH
```

(3) 备集群环境变量与主集群一样，存放在备集群的一个业务计算服务器上

### &nbsp;&nbsp;2.4.dss配置-dd模拟
配置两个节点的dss，脚本dss_autoscript.sh如下：

dss_autoscript.sh
```
#!/bin/bash
source /opt/omm/ss_env0

DSS_HOME_ONE=${SS_DATA}/dss_home0
DSS_HOME_TWO=${SS_DATA}/dss_home1

function clean_dir()
{
    ps ux | grep dssserver | grep -v grep | awk -F ' ' '{print $2}' | xargs kill -9
    rm -rf ${SS_DATA}
    mkdir -p ${SS_DATA}
    rm -rf /opt/omm/cluster/*
}

function create_one_device()
{
    mkdir -p ${DSS_HOME_ONE}
    mkdir -p ${DSS_HOME_ONE}/cfg
    mkdir -p ${DSS_HOME_ONE}/log
    touch ${DSS_HOME_ONE}/cfg/dss_vg_conf.ini
    echo "data:${DSS_HOME_ONE}/dss-data" > ${DSS_HOME_ONE}/cfg/dss_vg_conf.ini
    echo "INST_ID = 0" > ${DSS_HOME_ONE}/cfg/dss_inst.ini 
    echo "_LOG_BACKUP_FILE_COUNT = 128" > ${DSS_HOME_ONE}/cfg/dss_inst.ini 
    echo "_LOG_MAX_FILE_SIZE = 20M" > ${DSS_HOME_ONE}/cfg/dss_inst.ini 
    echo "LSNR_PATH = ${DSS_HOME_ONE}" > ${DSS_HOME_ONE}/cfg/dss_inst.ini 
    echo "_log_LEVEL = 255" > ${DSS_HOME_ONE}/cfg/dss_inst.ini
    
    dd if=/dev/zero of=${DSS_HOME_ONE}/dss-data bs=100k count=1048576 >/dev/null 2>&1
}

function create_two_device()
{
    mkdir -p ${DSS_HOME_TWO}
    mkdir -p ${DSS_HOME_TWO}/cfg
    mkdir -p ${DSS_HOME_TWO}/log
    touch ${DSS_HOME_TWO}/cfg/dss_vg_conf.ini
    echo "data:${DSS_HOME_ONE}/dss-data" > ${DSS_HOME_TWO}/cfg/dss_vg_conf.ini
    echo "INST_ID = 1" > ${DSS_HOME_TWO}/cfg/dss_inst.ini 
    echo "_LOG_BACKUP_FILE_COUNT = 128" > ${DSS_HOME_TWO}/cfg/dss_inst.ini 
    echo "_LOG_MAX_FILE_SIZE = 20M" > ${DSS_HOME_TWO}/cfg/dss_inst.ini 
    echo "LSNR_PATH = ${DSS_HOME_TWO}" > ${DSS_HOME_TWO}/cfg/dss_inst.ini 
    echo "_log_LEVEL = 255" > ${DSS_HOME_TWO}/cfg/dss_inst.ini
}

function create_vg()
{
    echo ">dsscmd cv data ${DSS_HOME_ONE}/dss-data"
    dsscmd cv -g data -v ${DSS_HOME_ONE}/dss-data -s 2048 -D {DSS_HOME_ONE}
}

function start_dsserver()
{
    dssserver -D /opt/omm/cluster/ss_data/dss_home0 &
    sleep 1
    dssserver -D /opt/omm/cluster/ss_data/dss_home1 &
    sleep 1
}

if [ "$1" == "first_create" ]; then
    clean_dir
    create_one_device
    create_two_device
    create_vg
    start_dssserver
esle
    echo "Have dd, you can reset volume"
    reset_vg
fi

```
&emsp;<font color='red'>@Notice Thing!@</font>：主备集群都执行dss_autoscript.sh脚本配置dss

### &nbsp;&nbsp;2.4 数据库部署
#### &nbsp;&nbsp;&nbsp;2.4.1 主集群(生产中心)
&emsp;(1) 主集群主节点0初始化
&emsp;<font color='blue'>@Precondition!@</font>：节点0对应的dssserver必须提前拉起，即dsserver进程存在

```
gs_initdb -D /opt/omm/cluster/dn0 --nodename=node1 -U omm -w opengauss@123 --vgname=+data --enable-dss --dma-url="0:10.10.10.10:4411,1:10.10.10.10:4412" -I 0 --socketpath='UDS:/opt/omm/cluster/ss_data/dss_home0/.dss_unix_d_socket' -d -n -g /dev/sdj
```

(2)配置主集群主节点0
&emsp;<font color='red'>postgresql.conf文件</font>
```
port = 44100
listen_address = 'localhost, 10.10.10.10'
ss_enable_reform = off
xlog_file_path = '/dev/sdj'
xlog_lock_file_path = '/opt/omm/cluster/dn0/redolog.lock'
cross_cluster_replconninfo1='localhost=10.10.10.10 localport=44100 remotehost=10.10.10.20 remoteport=44100'
cluster_run_mode = 'cluster_primary'
ha_module_debug = off
ss_log_level = 255
ss_log_backup_file_count = 100
ss_log_max_file_size = 1GB
```
&emsp;参数解释：
+ ss_enable_reform
+ xlog_file_path
+ xlog_lock_file_path
+ cross_cluster_replconninfo1
+ cluster_run_mode


&emsp;<font color='red'>pg_hba.conf文件</font>
```
host all omm 10.10.10.10/32 trust
host all omm 10.10.10.20/32 trust

host all all 10.10.10.10/32 sha256
host all all 10.10.10.20/32 sha256
```

(3)主集群备节点1初始化
```
gs_initdb -D /opt/omm/cluster/dn1 --nodename=node2 -U omm -w opengauss@123 --vgname=+data --enable-dss --dma-url="0:10.10.10.10:4411,1:10.10.10.10:4412" -I 1 --socketpath='UDS:/opt/omm/cluster/ss_data/dss_home1/.dss_unix_d_socket'
```

(4)主集群启动
```
主节点0启动
gs_ctl start -D /opt/omm/cluster/dn0 -M primary


备节点1启动
gs_ctl start -D /opt/omm/cluster/dn0
```


#### &nbsp;&nbsp;&nbsp;2.4.2 备集群(容灾中心)
&emsp;(1) 备集群首备节点0初始化
```
gs_initdb -D /opt/omm/cluster/dn0 --nodename=node1 -U omm -w opengauss@123 --vgname=+data --enable-dss --dma-url="0:10.10.10.20:4411,1:10.10.10.20:4412" -I 0 --socketpath='UDS:/opt/omm/cluster/ss_data/dss_home0/.dss_unix_d_socket' -d -n -g /dev/sdi
```

&emsp;(2) 配置备集群首备节点0

&emsp;<font color='red'>postgresql.conf文件</font>
```
port = 44100
listen_address = 'localhost, 10.10.10.20'
ss_enable_reform = off
xlog_file_path = '/dev/sdi'
xlog_lock_file_path = '/opt/omm/cluster/dn0/redolog.lock'
cross_cluster_replconninfo1='localhost=10.10.10.20 localport=44100 remotehost=10.10.10.10 remoteport=44100'
cluster_run_mode = 'cluster_standby'
ha_module_debug = off
ss_log_level = 255
ss_log_backup_file_count = 100
ss_log_max_file_size = 1GB
```
&emsp;参数解释：
+ ss_enable_reform
+ xlog_file_path
+ xlog_lock_file_path
+ cross_cluster_replconninfo1
+ cluster_run_mode


&emsp;<font color='red'>pg_hba.conf文件</font>
```
host all omm 10.10.10.10/32 trust
host all omm 10.10.10.20/32 trust

host all all 10.10.10.10/32 sha256
host all all 10.10.10.20/32 sha256
```

&emsp;(3) 首备全量build
&emsp; build之前，主集群主节点0和备集群首备必须配置流复制相关参数(cross_cluster_replconninfo1等)，即第(2)步必须在build之前操作

```
gs_ctl build -D /opt/omm/cluster/dn0 -b cross_cluster_full -g 0 --vgname=+data --enable-dss --socketpath='UDS:/opt/omm/cluster/ss_data/dss_home0/.dss_unix_d_socket' -q
```
参数解释：
+ -b cross_cluster_full
+ -g 0
+ -q

&emsp;(4)备集群从备节点1初始化
&emsp;<font color='red'>@shirley_zhengx tell you in secret that is very important!@</font>：备集群第一次初始化的时候，一定要初始化首备节点0并对首备做完build之后，再初始化备集群其它从备节点，即第(3)要在第(4)之前执行 <font color='red'>@very very important!@</font>：

```
gs_initdb -D /opt/omm/cluster/dn1 --nodename=node2 -U omm -w opengauss@123 --vgname=+data --enable-dss --dma-url="0:10.10.10.20:4411,1:10.10.10.20:4412" -I 1 --socketpath='UDS:/opt/omm/cluster/ss_data/dss_home1/.dss_unix_d_socket'
```

&emsp;(5)备集群启动
```
首备节点0启动
gs_ctl start -D /opt/omm/cluster/dn0 -M standby


从备节点1启动
gs_ctl start -D /opt/omm/cluster/dn0
```

## 3. 主备集群功能验证
### &nbsp;&nbsp;3.1.集群状态查询
```
主集群主节点0查询结果
gs_ctl query -D /opt/omm/cluster/dn0
[2023-04-03 19:29:20.472][1324519][][gs_ctl]: gs_ctl query ,datadir is /opt/omm/cluster/dn0
 HA state:
        local_role                     : Primary
        static_connections             : 1
        db_state                       : Normal
        detail_information             : Normal

 Senders info:
        sender_pid                     : 1324039
        local_role                     : Primary
        peer_role                      : StandbyCluster_Standby
        peer_state                     : Normal
        state                          : Streaming
        sender_sent_location           : 1/3049568
        sender_write_location          : 1/3049568
        sender_flush_location          : 1/3049568
        sender_replay_location         : 1/3049568
        receiver_received_location     : 1/3049568
        receiver_write_location        : 1/3049568
        receiver_flush_location        : 1/3049568
        receiver_replay_location       : 1/3049568
        sync_percent                   : 100%
        sync_state                     : Async
        sync_priority                  : 0
        sync_most_available            : Off
        channel                        : 10.10.10.10:44100-->10.10.10.20:42690

 Receiver info:
No information
```

```
主集群备节点1查询结果
gs_ctl query -D /opt/omm/cluster/dn1
[2023-04-03 19:29:20.472][2125915][][gs_ctl]: gs_ctl query ,datadir is /opt/omm/cluster/dn0
 HA state:
        local_role                     : Standby
        static_connections             : 0
        db_state                       : Normal
        detail_information             : Normal

 Senders info:
No information
 Receiver info:
No information
```

```
备集群首备节点0查询结果
gs_ctl query -D /opt/omm/cluster/dn0
[2023-04-03 19:29:20.472][2720317][][gs_ctl]: gs_ctl query ,datadir is /opt/omm/cluster/dn0
 HA state:
        local_role                     : Standby
        static_connections             : 1
        db_state                       : Normal
        detail_information             : Normal

 Senders info:
No information
 Receiver info:
        receiver_pid                   : 2720076
        local_role                     : Standby
        peer_role                      : Primary
        peer_state                     : Normal
        state                          : Normal
        sender_sent_location           : 1/3049568
        sender_write_location          : 1/3049568
        sender_flush_location          : 1/3049568
        sender_replay_location         : 1/3049568
        receiver_received_location     : 1/3049568
        receiver_write_location        : 1/3049568
        receiver_flush_location        : 1/3049568
        receiver_replay_location       : 1/3049568
        sync_percent                   : 100%
        channel                        : 10.10.10.20:39864<--10.10.10.10:44100
```

```
备集群从备节点1查询结果
gs_ctl query -D /opt/omm/cluster/dn1
[2023-04-03 19:29:20.472][2125915][][gs_ctl]: gs_ctl query ,datadir is /opt/omm/cluster/dn0
 HA state:
        local_role                     : Standby
        static_connections             : 0
        db_state                       : Normal
        detail_information             : Normal

 Senders info:
No information
 Receiver info:
No information
```

### &nbsp;&nbsp;3.1.主集群一写多读
```
主集群主节点0执行
gsql -d postgres -p 44100 -r
create table test01(id int) with(segment = on);
insert into test01 select generate_series(0,100);
```

```
主集群备节点1查询，可查询到主节点0创建的表和数据
gsql -d postgres -p 48100 -r
select * from test01;
```

### &nbsp;&nbsp;3.1.备集群只读
```
备集群首备节点0查询，可查询到主节点0创建的表和数据
gsql -d postgres -p 44100 -r
select * from test01;
```

```
备集群从备节点1查询，可查询到主节点0创建的表和数据
gsql -d postgres -p 48100 -r
select * from test01;
```


***Notice:不推荐直接用于生产环境***
