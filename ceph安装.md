ceph安装



```
ssh-keygen -t rsa
ssh-copy-id -i root@ceph-node02
ssh-copy-id -i root@ceph-node03
```



```
vim /etc/yum.repos.d/ceph.repo
```



```
[Ceph]
name=Ceph packages for $basearch
baseurl=http://mirrors.aliyun.com/ceph/rpm-pacific/el8/$basearch
enabled=1
gpgcheck=1
type=rpm-md
gpgkey=https://mirrors.aliyun.com/ceph/keys/release.asc

[Ceph-noarch]
name=Ceph noarch packages
baseurl=http://mirrors.aliyun.com/ceph/rpm-pacific/el8/noarch
enabled=1
gpgcheck=1
type=rpm-md
gpgkey=https://mirrors.aliyun.com/ceph/keys/release.asc

[ceph-source]
name=Ceph source packages
baseurl=http://mirrors.aliyun.com/ceph/rpm-pacific/el8/SRPMS
enabled=1
gpgcheck=1
type=rpm-md
gpgkey=https://mirrors.aliyun.com/ceph/keys/release.asc

```

```
rpm --import 'https://mirrors.aliyun.com/ceph/keys/release.asc'
```



```
dnf install ceph-mon
```

```
uuidgen
```

```
vim /etc/ceph/ceph.conf
```



```shell
#######
[global]#全局设置
fsid = 22923de1-c933-4452-8627-9712075e6b80    #集群标识ID
mon host = 192.168.10.101,192.168.10.102,192.168.10.103       #monitor IP 地址
mon_initial_members = node01, node02, node03
auth_cluster_required = cephx                  #集群认证
auth_service_required = cephx                  #服务认证
auth_client_required = cephx                   #客户端认证
osd_pool_default_size = 2                      #osd的默认副本数，如果不设置默认是3份。
osd_pool_default_min_size = 1        #PG 处于 degraded 状态不影响其 IO 能力,min_size是一个PG能接受IO的最小副本数
osd_pool_default_pg_num = 128        #pool的pg数量
osd_pool_default_pgp_num = 128       #pool的pgp数量
public_network = 192.168.10.0/24         #公共网络(monitorIP段)


#######
[mon]
mon_clock_drift_allowed = 1    #默认值0.05，monitor间时钟漂移
mon_clock_drift_warn_backoff = 10       
mon_osd_min_down_reporters = 1    #默认值1，向monitor报告down的最小OSD数


#######
[osd]
osd_journal_size = 10240         #默认5120，osd journal大小
osd_mount_options_xfs = rw, noatime, inode64, logbufs=8    #安装{fs-type}类型的Ceph文件存储OSD时使用的选项。
osd_mkfs_options_xfs = -f -d agcount=24          #创建{fs-type}类型的新Ceph文件存储OSD时使用的选项。
osd_journal = /data/osd/$cluster-$id/journal #osd journal日志位置
osd_scrub_begin_hour = 22    #清洗开始时间为晚上22点 
osd_scrub_end_hour = 7    #清洗结束时间为早上7点



```

```
sudo ceph-authtool --create-keyring /data/ceph.mon.keyring --gen-key -n mon. --cap mon 'allow *'
```

```
sudo ceph-authtool --create-keyring /etc/ceph/ceph.client.admin.keyring --gen-key -n client.admin --cap mon 'allow *' --cap osd 'allow *' --cap mds 'allow *' --cap mgr 'allow *'
```

```
sudo ceph-authtool --create-keyring /var/lib/ceph/bootstrap-osd/ceph.keyring --gen-key -n client.bootstrap-osd --cap mon 'profile bootstrap-osd' --cap mgr 'allow r'

```

```
sudo ceph-authtool /data/ceph.mon.keyring --import-keyring /etc/ceph/ceph.client.admin.keyring
sudo ceph-authtool /data/ceph.mon.keyring --import-keyring /var/lib/ceph/bootstrap-osd/ceph.keyring
```

```
sudo chown ceph:ceph /data/ceph.mon.keyring
```



```
monmaptool --create --add node01 192.168.10.101 --add node02 192.168.10.102 --add node03 192.168.10.103 --fsid 22923de1-c933-4452-8627-9712075e6b80 /data/monmap

```

```
sudo -u ceph mkdir /var/lib/ceph/mon/node01
```



```
sudo -u ceph ceph-mon --mkfs -i node01 --monmap /data/monmap --keyring /data/ceph.mon.keyring
```

```
sudo systemctl start ceph-mon@node01
sudo systemctl enable ceph-mon@node01
sudo systemctl status ceph-mon@node01
```





# node02、node03安装mon

```
# node01执行
ceph auth get mon. -o /tmp/ceph.mon.keyring
ceph mon getmap -o /tmp/monmap

scp /tmp/ceph.mon.keyring /tmp/monmap root@node02:/tmp/
scp /etc/ceph/ceph.conf root@node02:/etc/ceph/
```



```

dnf install -y ceph-mon
sudo -u ceph mkdir /var/lib/ceph/mon/ceph-`hostname -s`


sudo -u ceph ceph-mon --mkfs -i `hostname -s` --monmap /data/monmap --keyring /data/ceph.mon.keyring
ls /var/lib/ceph/mon/ceph-`hostname -s`/
```

```
sudo systemctl start ceph-mon@`hostname -s`
sudo systemctl enable ceph-mon@`hostname -s`
sudo systemctl status ceph-mon@`hostname -s`
```

















```
dnf install -y ceph-osd  ceph-volume
```

```shell
lsblk -l

[root@ceph-node03 ~]# lsblk -l
NAME    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda       8:0    0   20G  0 disk
sda1      8:1    0  512M  0 part /boot
sda2      8:2    0 19.5G  0 part
sdb       8:16   0   50G  0 disk
sr0      11:0    1 1024M  0 rom
cl-root 253:0    0   18G  0 lvm  /
cl-swap 253:1    0  1.5G  0 lvm  [SWAP]

```

```
# 独立osd节点需要拷贝证书
scp /var/lib/ceph/bootstrap-osd/ceph.keyring root@osd:/var/lib/ceph/bootstrap-osd/ceph.keyring
```



```
ceph-volume lvm create --data /dev/sdb
```

```
ceph osd tree

```



# 部署mgr和dashboard服务



```
dnf install -y ceph-mgr ceph-mgr-dashboard
```



```
sudo -u ceph mkdir /var/lib/ceph/mgr/ceph-`hostname -s`
```

```
ceph-authtool --create-keyring /etc/ceph/ceph.mgr.`hostname -s`.keyring --gen-key -n mgr.`hostname -s` --cap mon 'allow profile mgr' --cap osd 'allow *' --cap mds 'allow *'
```

```
ceph auth get-or-create mgr.`hostname -s` mon 'allow profile mgr' osd 'allow *' mds 'allow *' > keyring
```

```
ceph auth import -i /etc/ceph/ceph.mgr.`hostname -s`.keyring
```

```
ceph auth get-or-create mgr.`hostname -s` -o /var/lib/ceph/mgr/ceph-`hostname -s`/keyring
```

```
systemctl start ceph-mgr@`hostname -s`
systemctl enable ceph-mgr@`hostname -s`
systemctl status ceph-mgr@`hostname -s`
```









```
[root@ceph-node01 ~]# ceph -s
  cluster:
    id:     81630091-0509-4eb7-80c1-09b82af7b379
    health: HEALTH_WARN
            mons are allowing insecure global_id reclaim
            3 monitors have not enabled msgr2

  services:
    mon: 3 daemons, quorum ceph-node01,ceph-node02,ceph-node03 (age 13m)
    mgr: ceph-node01(active, since 5m), standbys: ceph-node02, ceph-node03
    osd: 3 osds: 3 up (since 11m), 3 in (since 11m)

  data:
    pools:   1 pools, 1 pgs
    objects: 0 objects, 0 B
    usage:   15 MiB used, 150 GiB / 150 GiB avail
    pgs:     1 active+clean

```





```
ceph mgr module ls | grep dashboard
ceph mgr module enable dashboard --force
```



```
ceph dashboard create-self-signed-cert
```

```
ceph config set mgr mgr/dashboard/server_addr 0.0.0.0
ceph config set mgr mgr/dashboard/ssl_server_port 8443
```

```
echo 'Test123456' > password.txt
ceph dashboard ac-user-create admin  administrator -i password.txt
```

```
ceph mgr services
```



```
# 禁用不安全模式
ceph config set mon auth_allow_insecure_global_id_reclaim false

ceph orch set backend
```

```
# 启用msgr2
ceph mon enable-msgr2
```





```
ceph osd pool create k8s-rdb 128 128
rbd pool init k8s-rdb

ceph osd pool get k8s-rdb pg_num

ceph osd pool set k8s-rdb pg_num 128
```

```


[root@ceph-node01 ~]# ceph auth get-or-create client.kubernetes mon 'profile rbd' osd 'profile rbd pool=k8s-rdb' mgr 'profile rbd pool=k8s-rdb'
[client.kubernetes]
        key = AQDKuS9kDjfhHxAAPPAPwW5y+dWU7PJ3SNc2hQ==

```

```

https://github.com/ceph/ceph-csi/tree/devel
```

```
rdb对接k8s
https://blog.csdn.net/weixin_42340926/article/details/123931137
```



```
# 删除pool操作
ceph --admin-daemon /var/run/ceph/ceph-mon.ceph-node02.asok config set mon_allow_pool_delete true


ceph osd pool rm cephfs_metadata cephfs_metadata --yes-i-really-really-mean-it
```



```
dnf install ceph-mds
```

```
mkdir -p /var/lib/ceph/mds/ceph-ceph-node01
ceph-authtool --create-keyring /var/lib/ceph/mds/ceph-ceph-node01/keyring --gen-key -n mds.ceph-node01

ceph auth add mds.ceph-node01 osd "allow rwx" mds "allow *" mon "allow profile mds" -i /var/lib/ceph/mds/ceph-ceph-node01/keyring
```

```

systemctl start ceph-mds@ceph-node01
systemctl enable ceph-mds@ceph-node01

ceph auth get mds.ceph-node01
```

```

ceph osd pool create cephfs_data 32 32
ceph osd pool create cephfs_metadata 64 64
ceph fs new k8s-fs cephfs_metadata cephfs_data
```

```


1.查询存储池副本数
ceph osd pool get onepool size
2、设置
ceph osd pool set cephfs_metadata size 2
```



```
1、停止mds服务

 systemctl stop ceph-mds@bd-server-5.service
1
2、删除该mds在集群里面的认证信息

ceph auth del mds.bd-server-5
1
3、禁用该mds服务（如果没有此步骤，下次开机时会自动启动该mds服务）

[root@bd-server-5 ~]# systemctl disable ceph-mds@bd-server-5.service
Removed symlink /etc/systemd/system/ceph-mds.target.wants/ceph-mds@bd-server-5.service.
1
2
4、删除该mds服务的相关数据

[root@bd-server-5 ~]# rm -rf /var/lib/ceph/mds/ceph-bd-server-5/
```





```
目的
删除 cephfs pool

检查
当前 cephfs name: noah_fs

[root@ns-storage-020100 ~]# ceph fs ls
name: noah_fs, metadata pool: cephfs_metadata, data pools: [cephfs_data ]
1
2
需要停止所有的 mds 服务
检查

[root@ns-storage-020100 ~]# ceph mds stat
noah_fs-1/1/1 up  {0=ns-storage-020100=up:active}
1
2
关闭服务

# systemctl stop ceph-mds@ns-storage-020100 
# ceph mds fail 0
failed mds gid 399478
1
2
3
检查

# ceph mds stat
noah_fs-0/1/1 up , 1 failed
1
2
删除
先删除 cephfs
# ceph fs rm noah_fs --yes-i-really-mean-it
1
再删除 pool
# ceph osd pool rm cephfs_data cephfs_data  --yes-i-really-really-mean-it
pool 'cephfs_data' removed
# ceph osd pool rm cephfs_metadata cephfs_metadata  --yes-i-really-really-mean-it
pool 'cephfs_metadata' removed
```





```
ceph-mds --cluster ceph -i ceph-node03 -m 192.168.10.103
```

