### 一、ceph基础系统环境配置

#### 1、yum源配置

```shell
rpm -Uvh https://mirrors.aliyun.com/ceph/rpm-17.2.6/el8/noarch/ceph-release-1-1.el8.noarch.rpm
```

```shell
sed -i "s/download.ceph.com/mirrors.aliyun.com\/ceph/g" /etc/yum.repos.d/ceph.repo
```

```shell
rpm --import "https://mirrors.aliyun.com/ceph/keys/release.asc"
```

#### 2、配置时间同步

```shell
dnf install chrony
```

```shell
vim /etc/chrony.conf
server ntp.aliyun.com iburst
server cn.ntp.org.cn iburst

#运行chrony服务并开机自启
systemctl enable chronyd --now

# 查看同步进度
chronyc sources –v

# 同步硬件时间，osd重启一直down，可能是这个问题
hwclock --show
hwclock --systohc
```

#### 3、设置hosts

```shell
vim /etc/hosts

192.168.10.101 node01
192.168.10.102 node02
192.168.10.103 node03

```

#### 4、Ceph性能调优设置

```shell

# 设置系统的进程数量，网卡访问设置
cat << EOF >> /etc/sysctl.conf
kernel.pid_max = 4194303
vm.swappiness = 0
net.ipv4.tcp_rmem = 4096 87380 16777216 
net.ipv4.tcp_wmem = 4096 16384 16777216 
net.core.rmem_max = 16777216 
net.core.wmem_max = 16777216
EOF

sysctl -p


# I/O调度器，ssd和虚拟机（VMware自身调度）建议使用none，普通hhd磁盘使用deadline调度
# 临时设置
cat /sys/block/sd[x]/queue/scheduler
[mq-deadline] kyber bfq none

echo mq-deadline >/sys/block/sd[x]/queue/scheduler

echo none > /sys/block/sd[x]/queue/scheduler


# 开机启动，并设置调度器深度为1024
chmod +x  /etc/rc.local | cat << EOF >> /etc/rc.local
echo 1024 > /sys/block/sda/queue/nr_requests
echo none > /sys/block/sda/queue/scheduler

echo 1024 > /sys/block/sdb/queue/nr_requests
echo none > /sys/block/sdb/queue/scheduler

EOF

```



### 二、mon服务初始化

#### 1、ssh互信，初始化都在node01节点执行

```
# 配置ssh互信密钥，建议配置，方便scp同步配置文件到其它节点
ssh-keygen -t rsa
ssh-copy-id -i root@node02
ssh-copy-id -i root@node03
```

#### 2、yum安装mon包，配置ceph文件

```shell
# 所有mon节点都可以提前安装
dnf install -y ceph-mon
```

```shell
# 生成fsid
uuidgen
```



```shell
cat << EOF > /etc/ceph/ceph.conf
[global]

fsid = efc3cb49-3f70-41ef-a185-8ad1f604759a
mon_initial_members =node01, node02, node03
mon_host = 192.168.10.101, 192.168.10.102, 192.168.10.103
public_network = 192.168.10.0/24

auth_cluster_required = cephx
auth_service_required = cephx
auth_client_required = cephx

# 时钟偏移设置，默认0.05秒
mon_clock_drift_allowed = 0.5
mon_clock_drift_warn_backoff = 10

# 设置集群磁盘阈值
mon_osd_full_ratio = .80
mon_osd_backfillfull_ratio = .75
mon_osd_nearfull_ratio = .70

# 设置默认副本数与默认pg、pgp数（pg、pgp会自动根据osd与副本做调整）
osd_pool_default_size = 2
osd_pool_default_min_size = 1
osd_pool_default_pg_num = 64
osd_pool_default_pgp_num = 64

# 设置journal大小为10G，可根据ssd容量调整大小与路径
[osd]
osd_journal_size = 10240
#osd_journal =/var/lib/ceph/osd/\$cluster-\$id/journal

EOF

```



#### 3、创建集群相关密钥

##### 01、创建集群Monitor密钥

```shell
sudo ceph-authtool --create-keyring /etc/ceph/ceph.mon.keyring --gen-key -n mon. --cap mon 'allow *'
```

##### 02、创建client.admin用户，并导入集群密钥中

```shell
sudo ceph-authtool --create-keyring /etc/ceph/ceph.client.admin.keyring --gen-key -n client.admin --cap mon 'allow *' --cap osd 'allow *' --cap mds 'allow *' --cap mgr 'allow *'

sudo ceph-authtool /etc/ceph/ceph.mon.keyring --import-keyring /etc/ceph/ceph.client.admin.keyring
```

##### 03、创建client.bootstrap-osd用户密钥，并导入集群密钥中

```shell
sudo ceph-authtool --create-keyring /var/lib/ceph/bootstrap-osd/ceph.keyring  --gen-key -n client.bootstrap-osd --cap mon 'profile bootstrap-osd'

sudo ceph-authtool /etc/ceph/ceph.mon.keyring --import-keyring /var/lib/ceph/bootstrap-osd/ceph.keyring
```

##### 04、创建client.bootstrap-mgr用户密钥，并导入集群密钥中（可选）

```shell
sudo ceph-authtool --create-keyring /var/lib/ceph/bootstrap-mgr/ceph.keyring --gen-key -n client.bootstrap-mgr --cap mon 'profile bootstrap-mgr'

sudo ceph-authtool /etc/ceph/ceph.mon.keyring --import-keyring /var/lib/ceph/bootstrap-mgr/ceph.keyring
```

##### 05、创建client.bootstrap-mds用户密钥，并导入集群密钥中（可选）

```shell
sudo ceph-authtool --create-keyring /var/lib/ceph/bootstrap-mds/ceph.keyring --gen-key -n client.bootstrap-mds --cap mon 'profile bootstrap-mds'

sudo ceph-authtool /etc/ceph/ceph.mon.keyring --import-keyring /var/lib/ceph/bootstrap-mds/ceph.keyring
```

##### 06、创建client.bootstrap-rgw用户密钥，并导入集群密钥中（可选）

```
sudo ceph-authtool --create-keyring /var/lib/ceph/bootstrap-rgw/ceph.keyring --gen-key -n client.bootstrap-rgw --cap mon 'profile bootstrap-rgw'

sudo ceph-authtool /etc/ceph/ceph.mon.keyring --import-keyring /var/lib/ceph/bootstrap-rgw/ceph.keyring
```





```
monmaptool --create --add node01 192.168.10.101 --add node02 192.168.10.102 --add node03 192.168.10.103 --fsid efc3cb49-3f70-41ef-a185-8ad1f604759a /etc/ceph/monmap
```









节点安装，优先安装

```
dnf install -y ceph-mon

sudo -u ceph mkdir /var/lib/ceph/mon/ceph-`hostname -s`
```

拷贝配置到其它节点，主节点操作

```
scp /etc/ceph/ceph.conf root@node02:/etc/ceph 

scp /etc/ceph/ceph.mon.keyring root@node02:/etc/ceph

scp /etc/ceph/monmap root@node02:/etc/ceph 

scp /etc/ceph/ceph.client.admin.keyring root@node02:/etc/ceph 

scp /var/lib/ceph/bootstrap-mds/ceph.keyring root@node02:/var/lib/ceph/bootstrap-mds/ 

scp /var/lib/ceph/bootstrap-mgr/ceph.keyring root@node02:/var/lib/ceph/bootstrap-mgr/ 

scp /var/lib/ceph/bootstrap-osd/ceph.keyring root@node02:/var/lib/ceph/bootstrap-osd/ 

scp /var/lib/ceph/bootstrap-rgw/ceph.keyring root@node02:/var/lib/ceph/bootstrap-rgw/
```



从节点操作

```
chown -R ceph:ceph /etc/ceph 
chown -R ceph:ceph /var/lib/ceph
```





```
sudo -u ceph ceph-mon --mkfs -i `hostname -s` --monmap /etc/ceph/monmap --keyring /etc/ceph/ceph.mon.keyring
```

```

sudo systemctl enable ceph-mon@`hostname -s` --now
sudo systemctl status ceph-mon@`hostname -s`
```

更新monmap

```
# 所有节点执行执行
ceph mon getmap -o /etc/ceph/monmap
```



```
[root@node01 ~]# ceph -s
  cluster:
    id:     a0a8b4b2-3133-459c-831e-bb0204b9c056
    health: HEALTH_WARN
            mons are allowing insecure global_id reclaim
            3 monitors have not enabled msgr2

  services:
    mon: 3 daemons, quorum node01,node02,node03 (age 106s)
    mgr: no daemons active
    osd: 0 osds: 0 up, 0 in

  data:
    pools:   0 pools, 0 pgs
    objects: 0 objects, 0 B
    usage:   0 B used, 0 B / 0 B avail
    pgs:


```

```
# 禁用不安全模式
ceph config set mon auth_allow_insecure_global_id_reclaim false

# 启用msgr2
ceph mon enable-msgr2
```

```
[root@node01 ~]# ceph -s
  cluster:
    id:     a0a8b4b2-3133-459c-831e-bb0204b9c056
    health: HEALTH_OK

  services:
    mon: 3 daemons, quorum node01,node02,node03 (age 2s)
    mgr: no daemons active
    osd: 0 osds: 0 up, 0 in

  data:
    pools:   0 pools, 0 pgs
    objects: 0 objects, 0 B
    usage:   0 B used, 0 B / 0 B avail
    pgs:


```



### 添加osd节点

```
dnf install -y ceph-osd ceph-volume
```

```
lsblk -l

NAME    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda       8:0    0   20G  0 disk
sda1      8:1    0  512M  0 part /boot
sda2      8:2    0 19.5G  0 part
sdb       8:16   0   50G  0 disk
sdc       8:32   0   50G  0 disk
sr0      11:0    1 1024M  0 rom
cl-root 253:0    0   18G  0 lvm  /
cl-swap 253:1    0  1.5G  0 lvm  [SWAP]

```

```
scp /var/lib/ceph/bootstrap-osd/ceph.keyring root@node02:/var/lib/ceph/bootstrap-osd/

ceph-volume lvm create --data /dev/sdb
```

```
ceph osd tree
```



```
## 手动设备分类
# 创建root分类
ceph osd crush add-bucket hhd root
ceph osd crush add-bucket ssd root

# 创建host分类
ceph osd crush add-bucket node01_hhd host
ceph osd crush add-bucket node02_hhd host
ceph osd crush add-bucket node03_hhd host



# 移动host到root_hdd
ceph osd crush move node01_hhd root=hhd
ceph osd crush move node02_hhd root=hhd
ceph osd crush move node03_hhd root=hhd

# 移动磁盘到host
ceph osd crush create-or-move osd.0 1.00000 host=node01_hhd
ceph osd crush create-or-move osd.1 1.00000 host=node02_hhd
ceph osd crush create-or-move osd.2 1.00000 host=node03_hhd
ceph osd crush create-or-move osd.3 1.00000 host=node01_hhd
ceph osd crush create-or-move osd.4 1.00000 host=node02_hhd
ceph osd crush create-or-move osd.5 1.00000 host=node03_hhd



# 创建rule，
# hhd_rule:rule名称
# hhd:crush名称
# host:rule类型，对应不同的故障级别（root,rack,host）
# firstn:副本存储池选firstn，纠删码存储池选indep

ceph osd crush rule create-simple hhd_rule hhd host firstn

```



mds

```
dnf install -y ceph-mds

sudo -u ceph mkdir /var/lib/ceph/mds/ceph-`hostname -s`

sudo -u ceph ceph-authtool --create-keyring /var/lib/ceph/mds/ceph-`hostname -s`/keyring --gen-key -n mds.`hostname -s`

ceph auth add mds.`hostname -s` osd "allow rwx" mds "allow *" mon "allow profile mds" -i /var/lib/ceph/mds/ceph-`hostname -s`/keyring



```



```
vim /etc/ceph/ceph.conf

[mds.node01]
host = node01

[mds.node02]
host = node02

[mds.node03]
host = node03

systemctl enable ceph-mds@`hostname -s` --now
systemctl status ceph-mds@`hostname -s`
```



转为cephadm管理

```
ceph orch set backend cephadm
ceph orch status
ceph orch set backend -h
cephadm -v
cephadm prepare-host
cephadm ls
ceph config dump
ceph config assimilate-conf -i /etc/ceph/ceph.conf
ceph config dump
ceph mgr module enable cephadm
ceph orch set backend cephadm
ceph cephadm generate-key
ceph cephadm get-pub-key > /etc/ceph/ceph.pub
ssh-copy-id -f -i /etc/ceph/ceph.pub root@node02
ssh-copy-id -f -i /etc/ceph/ceph.pub root@node03
ssh-copy-id -f -i /etc/ceph/ceph.pub root@node01
cephadm adopt --style legacy --name mon.node01
cephadm adopt --style legacy --name mon.node02
cephadm adopt --style legacy --name mon.node03
cephadm adopt --style legacy --name mgr.node01
cephadm adopt --style legacy --name mgr.node02
cephadm adopt --style legacy --name mgr.node03
ceph orch host add node01 192.168.10.101
ceph orch host add node02 192.168.10.102
ceph orch host add node03 192.168.10.103
cephadm check-host
```



### 添加mgr节点

```
dnf install -y ceph-mgr ceph-mgr-dashboard

mkdir /var/lib/ceph/mgr/ceph-`hostname -s`

ceph auth get-or-create mgr.`hostname -s`  mon 'allow profile mgr' osd 'allow *' mds 'allow *' > /var/lib/ceph/mgr/ceph-`hostname -s`/keyring
 
chown -R ceph:ceph /var/lib/ceph/mgr
 
ceph-mgr -i `hostname -s`
 
systemctl enable ceph-mgr@`hostname -s` --now
systemctl status ceph-mgr@`hostname -s`
```

```
ceph config set mgr mgr_ttl_cache_expire_seconds 10
```







```
ceph osd pool get .mgr pg_num
ceph osd pool get .mgr pgp_num

ceph osd pool set .mgr pg_num 8
ceph osd pool set .mgr pgp_num 8

```





```
ceph -s

  cluster:
    id:     3aa46758-d9d4-4911-af25-121c46d0240b
    health: HEALTH_WARN
            mons are allowing insecure global_id reclaim
            3 monitors have not enabled msgr2
            10 daemons have recently crashed
            OSD count 0 < osd_pool_default_size 2

  services:
    mon: 3 daemons, quorum node01,node02,node03 (age 24m)
    mgr: node01(active, starting, since 2s), standbys: node02, node03
    osd: 0 osds: 0 up, 0 in

  data:
    pools:   0 pools, 0 pgs
    objects: 0 objects, 0 B
    usage:   0 B used, 0 B / 0 B avail
    pgs:


```



```
# 查看mgr节点在哪里，安装dashboard
```



```


ceph mgr module enable dashboard
ceph dashboard create-self-signed-cert

ceph config set mgr mgr/dashboard/`hostname -s`/server_addr 0.0.0.0
ceph config set mgr mgr/dashboard/`hostname -s`/ssl_server_port 8443
```

```

echo 'www.123.com' > password.txt
ceph dashboard set-login-credentials admin -i password.txt
```





