---
layout:     post
title:      GlusterFs和Heketi的安装部署以及在k8s中的使用
subtitle:   
date:       2020-04-21
author:     Chymy
header-img: img/post-bg-article.jpg
catalog: true
tags:
    - GlusterFs
    - heketi
    - kubernetes
---

本环境部署是基于通过KVM创建的虚机，系统是CentOS7

### 环境说明
| IP            | 主机名        | 说明                                                         |
| ------------- | ------------- | ------------------------------------------------------------ |
| 192.168.1.189 | glusterfs-189 | glusterfs服务端，存在未初始化的卷，如/dev/sdb<br>**该节点也安装了heketi** |
| 192.168.1.190 | glusterfs-190 | glusterfs服务端，存在未初始化的卷，如/dev/sdb    |
| 192.168.1.181 | glusterfs-181 | glusterfs服务端，存在未初始化的卷，如/dev/sdb    |

上面未初始化的卷，如/dev/sdb，需要在KVM上的节点上创建，此时在节点上通过 lsblk查看到该磁盘的存在，且没有初始化。

创建好卷之后，**需要把虚拟机关机再开机才能识别。**
### 安装依赖
```shell
unset http_proxy
unset https_proxy
yum install -y psmisc firewalld-filesystem
```
**以在192.168.1.189上安装为例**

#### 配置glusterfs源
```shell
cd /etc/yum.repos.d/ && mkdir bak && mv *.repo
```

vi glusterfs.repo

```ini
[centos-gluster313]
name=CentOS-$releasever - Gluster 3.13
baseurl=http://buildlogs.centos.org/centos/7/storage/x86_64/gluster-3.13/
gpgcheck=1
enabled=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-SIG-Storage
[centos-gluster313-test]
name=CentOS-$releasever - Gluster 3.13 Testing
baseurl=http://buildlogs.centos.org/centos/7/storage/x86_64/gluster-3.13/
gpgcheck=0
enabled=0
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-SIG-Storage
```

#### 安装

```shell
yum install -y glusterfs glusterfs-server --nogpgcheck
yum install -y glusterfs-fuse --nogpgcheck
```

#### 启动

```shell
systemctl daemon-reload
systemctl enable glusterd.service
systemctl start glusterd.service
```

修改/etc/sudoers文件，在第56行的“Defaults    requiretty“开头加上#注释掉


```shell
#
# Disable "ssh hostname sudo <cmd>", because it will show the password in clear.
#         You have to run "ssh -t hostname sudo <cmd>".
#
#Defaults    requiretty
```

**注：可能centos系统需要特殊处理**

以上操作需要在192.168.1.190,192.168.1.181节点上执行一遍。若是通过KVM创建的虚机，其实拷贝一份189机器的qcow2文件，然后用该拷贝的qcow2创建190虚机，之后修改主机名和ip地址，再重启该续集的网卡即可，也省了上面步骤的重新安装。

注意：

```shell
#若拷贝虚拟机，添加peer节点时，报错
# gluster peer probe mystorage4
peer probe: failed: Peer uuid (host mystorage4) is same as local uuid

# 每隔节点可执行如下命令
# gluster system uuid reset
```

#### 组建集群

在192.168.1.189机器上用root用户执行

```shell
# gluster peer probe 192.168.1.181
# gluster peer probe 192.168.1.190
验证
# gluster peer status
Number of Peers: 2

Hostname: 192.168.1.181
Uuid: a94cd3ab-15d4-45be-b470-40b4eb1537b4
State: Peer in Cluster (Connected)

Hostname: 192.168.1.190
Uuid: 65f27c31-0d85-4b98-87e4-166754b21def
State: Peer in Cluster (Connected)
```
### heketi安装

#### 安装

使用安装glusterfs的yum源，执行如下命令：

```shell
yum -y install heketi heketi-client --nogpgcheck
```

当然可以在https://github.com/heketi/heketi/releases下载对应版本

#### 配置heketi.json

vi /etc/heketi/heketi.json

```json
{
  "_port_comment": "Heketi Server Port Number",
  "port": "8080",
// 默认值false，不需要认证
  "_use_auth": "Enable JWT authorization. Please enable for deployment",
  "use_auth": true,
  "_jwt": "Private keys for access",
  "jwt": {
    "_admin": "Admin has access to all APIs",
    "admin": {
      "key": "admin123456"  // 当上面use_auth设置为true时候，此为admin的用户密码
    },
    "_user": "User only has access to /volumes endpoint",
    "user": {
      "key": "123456" //// 当上面use_auth设置为true时候，此为普通用户的密码
    }
  },

  "_glusterfs_comment": "GlusterFS Configuration",
  "glusterfs": {
    "_executor_comment": [
      "Execute plugin. Possible choices: mock, ssh",
      "mock: This setting is used for testing and development.",
      "      It will not send commands to any node.",
      "ssh:  This setting will notify Heketi to ssh to the nodes.",
      "      It will need the values in sshexec to be configured.",
      "kubernetes: Communicate with GlusterFS containers over",
      "            Kubernetes exec api."
    ],
     // mock：测试环境下创建的volume无法挂载；
    // kubernetes：在GlusterFS由kubernetes创建时采用
    "executor": "ssh",

      // 当设置为ssh的时候需要设置如下：heketi免密访问glusterFs集群
    "_sshexec_comment": "SSH username and private key file information",
    "sshexec": {
      "keyfile": "/root/.ssh/id_rsa", // ssh的秘钥，注意该秘钥heketi无法识别，下面会讲到解决方法
      "user": "root",// 
      "port": "22",// 
      "fstab": "/etc/fstab"
    },

    "_kubeexec_comment": "Kubernetes configuration",
    "kubeexec": {
      "host" :"https://kubernetes.host:8443",
      "cert" : "/path/to/crt.file",
      "insecure": false,
      "user": "kubernetes username",
      "password": "password for kubernetes user",
      "namespace": "OpenShift project or Kubernetes namespace",
      "fstab": "Optional: Specify fstab file on node.  Default is /etc/fstab"
    },

    "_db_comment": "Database file name",
      // 定义heketi数据库文件位置
    "db": "/var/lib/heketi/heketi.db",

    "_loglevel_comment": [
      "Set log level. Choices are:",
      "  none, critical, error, warning, info, debug",
      "Default is warning"
    ],
      // 默认设置为debug，不设置时的默认值即是warning;
// 日志信息输出在/var/log/message
    "loglevel" : "debug"
  }
}
```
#### 设置heketi免密访问GlusterFS

```shell
# 选择ssh执行器，heketi服务器需要免密登陆GlusterFS集群的各节点；
# -t：秘钥类型；
# -q：安静模式；
# -f：指定生成秘钥的目录与名字，注意与heketi.json的ssh执行器中"keyfile"值一致；
# -N：秘钥密码，””即为空
[root@heketi ~]# ssh-keygen -m PEM -t rsa -b 4096 -q -N ''

```

注意：以上ssh-keygen生成秘钥的时候不能使用`ssh-keygen -t rsa -q  -N ""`，否则heketi通过`"executor": "ssh"`启动的时候，会报错，可能因为ssh版本的问题吧。（https://github.com/heketi/heketi/issues/1666）

报错如下：

```
#######
./heketi --config=heketi.json
[cmdexec] ERROR 2019/10/19 13:47:38 heketi/pkg/remoteexec/ssh/ssh.go:87:ssh.NewSshExecWithKeyFile: Unable to get keyfile: ssh: unhandled key type
[cmdexec] ERROR 2019/10/19 13:47:38 heketi/executors/sshexec/sshexec.go:126:sshexec.NewSshExecutor: Unable to read private key file
[heketi] ERROR 2019/10/19 13:47:38 heketi/apps/glusterfs/app.go:158:glusterfs.(*App).setup: Unable to read private key file
ERROR: Unable to start application: Unable to read private key file
############
```

> 若使用`ssh-keygen -t rsa -q  -N ""`方式，启动heketi没有报错，或启动成功，则可直接使用`ssh-keygen -t rsa -q  -N ""`，否则使用` ssh-keygen -m PEM -t rsa -b 4096 -q -N ''`方式



我的ssh版本如下：

```shell
# ssh -V
OpenSSH_7.9p1, OpenSSL 1.0.2k-fips  26 Jan 2017
```

分发公钥:

```shell
ssh-copy-id -i 192.168.1.189
ssh-copy-id -i 192.168.1.190
ssh-copy-id -i 192.168.1.181
```

#### 配置heketi.service

vi /usr/lib/systemd/system/heketi.service

```ini
[Unit]
Description=Heketi Server

[Service]
Type=simple
User=root
WorkingDirectory=/var/lib/heketi
ExecStart=/usr/bin/heketi --config=/etc/heketi/heketi.json
Restart=on-failure
RestartSec=10s
LimitNOFILE=40000

[Install]
WantedBy=multi-user.target
```

#### 启动

```shell
systemctl daemon-reload
systemctl start heketi.service
systemctl enable heketi.service
systemctl status heketi.service

验证：
curl http://192.168.1.189:8080/hello
返回：Hello from Heketi
```

#### 通过heketi设置GlusterFS集群

**创建topology.json文件**

```shell
# 通过topology.json文件定义组建GlusterFS集群；
# topology指定了层级关系：clusters-->nodes-->node/devices-->hostnames/zone；
# node/hostnames字段的manage填写主机ip，指管理通道，在heketi服务器不能通过hostname访问GlusterFS节点时不能填写hostname；
# node/hostnames字段的storage填写主机ip，指存储数据通道，与manage可以不一样；
# node/zone字段指定了node所处的故障域，heketi通过跨故障域创建副本，提高数据高可用性质，如可以通过rack的不同区分zone值，创建跨机架的故障域；
# devices字段指定GlusterFS各节点的盘符（可以是多块盘），必须是未创建文件系统的裸设备
```

vi /etc/heketi/topology.json

```json
{
    "clusters": [
        {
            "nodes": [
                {
                    "node": {
                        "hostnames": {
                            "manage": [
                                "192.168.1.189"
                            ],
                            "storage": [
                                "192.168.1.189"
                            ]
                        },
                        "zone": 1
                    },
                    "devices": [
                        "/dev/sdb"
                    ]
                },
                {
                    "node": {
                        "hostnames": {
                            "manage": [
                                "192.168.1.190"
                            ],
                            "storage": [
                                "192.168.1.190"
                            ]
                        },
                        "zone": 1
                    },
                    "devices": [
                        "/dev/sdb"
                    ]
                },
                {
                    "node": {
                        "hostnames": {
                            "manage": [
                                "192.168.1.181"
                            ],
                            "storage": [
                                "192.168.1.181"
                            ]
                        },
                        "zone": 1
                    },
                    "devices": [
                        "/dev/sdb"
                    ]
                }
            ]
        }
    ]
}
```

> manage和storage的值建议不要用主机名，否则在k8s集群使用的时候无法解析主机名，而创建pv失败。



 **初始化**

通过Heketi命令初始化GlusterFS各节点的存储

```shell
# GlusterFS集群各节点的glusterd服务已正常启动，但不必组建受信存储池；
# heketi-cli命令行也可手动逐层添加cluster，node，device，volume等；
# “--server http://localhost:8080”：localhost执行heketi-cli时，可不指定；
# ”--user admin --secret admin123456 “：如果heketi.json中设置了认证，执行heketi-cli时需要带上认证信息，否则报”Error: Invalid JWT token: Unknown user”错

# heketi-cli --server http://localhost:8080 --user admin --secret admin123456 topology load --json=/etc/heketi/topology.json
Creating cluster ... ID: f7871c30c88b3a0d03a6b326c17fdabb
        Allowing file volumes on cluster.
        Allowing block volumes on cluster.
        Creating node 192.168.1.189 ... ID: 1d83636bcf879c817e1d4c6d5c00a8ae
                Adding device /dev/sdb ... OK
        Creating node 192.168.1.190 ... ID: ad99623064dd44b13551d48cd95cec98
                Adding device /dev/sdb ... OK
        Creating node 192.168.1.181 ... ID: 269b6953da4690c1b6fe1a217dba77a1
                Adding device /dev/sdb ... OK

```

如果出现如下错误：

```shell
Adding device /dev/sdb ... Unable to add device: Device /dev/sdb not found (or ignored by filtering).
```

修改：vim /etc/lvm/lvm.conf

在global_filter添加"a|/dev/sdb|"

```shell
# a:表示允许，r：表示拒绝
global_filter = [ "a|^/dev/sda2$|", "a|^/dev/vda2$|",  "a|^/dev/xvda2$|","a|/dev/sdb|","r|.*/|" ]
```



这里会生成cluster id，这个后面要用到。

查看消息

```shell
# 查看heketi topology信息，此时volume与brick等未创建；
# 通过”heketi-cli cluster info [clusterId]“可以查看集群相关信息；
# 通过”heketi-cli node info [nodeid]“可以查看节点相关信息；
# 通过”heketi-cli device info [deviceid]“可以查看device相关信息

# heketi-cli topology info
Cluster Id: f7871c30c88b3a0d03a6b326c17fdabb

    File:  true
    Block: true

    Volumes:

    Nodes:

        Node Id: 1d83636bcf879c817e1d4c6d5c00a8ae
        State: online
        Cluster Id: f7871c30c88b3a0d03a6b326c17fdabb
        Zone: 1
        Management Hostnames: 192.168.1.189
        Storage Hostnames: 192.168.1.189
        Devices:
                Id:1aac11877365750a9f905a0cd1b43335   Name:/dev/sdb            State:online    Size (GiB):19      Used (GiB):0       Free (GiB):19
                        Bricks:

        Node Id: 269b6953da4690c1b6fe1a217dba77a1
        State: online
        Cluster Id: f7871c30c88b3a0d03a6b326c17fdabb
        Zone: 1
        Management Hostnames: 192.168.1.181
        Storage Hostnames: 192.168.1.181
        Devices:
                Id:4bcc57c89a9a9789be4eb12b57f8dbc6   Name:/dev/sdb            State:online    Size (GiB):19      Used (GiB):0       Free (GiB):19
                        Bricks:

        Node Id: ad99623064dd44b13551d48cd95cec98
        State: online
        Cluster Id: f7871c30c88b3a0d03a6b326c17fdabb
        Zone: 1
        Management Hostnames: 192.168.1.190
        Storage Hostnames: 192.168.1.190
        Devices:
                Id:41b01e0a29ac681edcac8e4dcc049c1b   Name:/dev/sdb            State:online    Size (GiB):19      Used (GiB):0       Free (GiB):19
                        Bricks:
```
#### 创建卷

**这里仅仅是做一个测试，实际使用中，会由kubernetes自动创建pvc**,创建一个2G的2副本的复制卷

```shell
# alias heketi-cli='heketi-cli --server "http://127.0.0.1:8080"'
# heketi-cli volume create --size=2  --replica=2 --clusters=e1869421376a5a13d2b5e53a506de858
Name: vol_3b2ce55ba93b2cc83eeb92ef63932cc8
Size: 2
Volume Id: 3b2ce55ba93b2cc83eeb92ef63932cc8
Cluster Id: ce4314bbefd243a83fa0682c77efaf90
Mount: glusterfs-190:vol_3b2ce55ba93b2cc83eeb92ef63932cc8
Mount Options: backup-volfile-servers=glusterfs-189
Block: false
Free Size: 0
Block Volumes: []
Durability Type: replicate

```

Heketi会为每个卷起一个uuid的名字，可以通过`heketi-cli volume list`查看。使用gluster命令也可以create volume，但是默认没有start，需要用gluster start命令启动volume.

```shell
# heketi-cli volume list
Id:3b2ce55ba93b2cc83eeb92ef63932cc8    Cluster:ce4314bbefd243a83fa0682c77efaf90    Name:vol_3b2ce55ba93b2cc83eeb92ef63932cc8
```

### 参考

https://www.cnblogs.com/netonline/p/10288219.html

https://www.cnblogs.com/breezey/p/8849466.html
