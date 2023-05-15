ceph 15.2.17

```
rpm -Uvh http://mirrors.aliyun.com/ceph/rpm-15.2.17/el8/noarch/ceph-release-1-1.el8.noarch.rpm
```

```
sed -i "s/download.ceph.com/mirrors.aliyun.com\/ceph/g" /etc/yum.repos.d/ceph.repo
```

```
rpm --import "https://mirrors.aliyun.com/ceph/keys/release.asc"
```





```
dnf install chrony
```

```
vim /etc/chrony.conf
server ntp.aliyun.com iburst
server cn.ntp.org.cn iburst
# server 192.168.10.101 iburst 其它节点同步主节点时间

# 即使server端无法从互联网同步时间，也同步本机时间至client，（chrony主节点设置，客户端禁用）
local stratum 10

# 在偏移0.5秒后，允许通过三次调整时钟偏移，修改客户端同步偏移
makestep 0.5 3

#运行chrony服务并开机自启
systemctl enable chronyd --now

# 查看同步进度
chronyc sources –v

```

```
vim /etc/hosts

192.168.10.101 node01
192.168.10.102 node02
192.168.10.103 node03

```

```
# 设置系统的进程数量
cat << EOF >> /etc/sysctl.conf
kernel.pid_max = 4194303
vm.swappiness = 0
net.ipv4.tcp_rmem = 4096 87380 16777216 
net.ipv4.tcp_wmem = 4096 16384 16777216 
net.core.rmem_max = 16777216 
net.core.wmem_max = 16777216
EOF

sysctl -p


# I/O调度，ssd和虚拟机建议使用none，普通磁盘使用deadline调度，bfq默认调度，适合大部分

cat /sys/block/sd[x]/queue/scheduler
[mq-deadline] kyber bfq none

echo mq-deadline >/sys/block/sd[x]/queue/scheduler

echo none > /sys/block/sd[x]/queue/scheduler



chmod +x  /etc/rc.local | cat << EOF >> /etc/rc.local
echo 1024 > /sys/block/sda/queue/nr_requests
echo none > /sys/block/sda/queue/scheduler

echo 1024 > /sys/block/sdb/queue/nr_requests
echo none > /sys/block/sdb/queue/scheduler

echo 1024 > /sys/block/sdc/queue/nr_requests
echo none > /sys/block/sdc/queue/scheduler
EOF

```



```
ssh-keygen -t rsa
ssh-copy-id -i root@node02
ssh-copy-id -i root@node03
```





```
dnf install -y ceph-mon
```



```
uuidgen
```



```
vim /etc/ceph/ceph.conf
```

```
[global]

fsid = efc3cb49-3f70-41ef-a185-8ad1f604759a
mon_initial_members =node01,node02,node03
mon_host = 192.168.10.101,192.168.10.102,192.168.10.103
public_network = 192.168.10.0/24

auth_cluster_required = cephx
auth_service_required = cephx
auth_client_required = cephx

mon_clock_drift_allowed = 2
mon_clock_drift_warn_backoff = 30
 
osd_pool_default_size = 2
osd_pool_default_min_size = 1
osd_pool_default_pg_num = 128
osd_pool_default_pgp_num = 128

需手动设置设备分类才启用
#osd_crush_update_on_start = false
 
```



```
sudo ceph-authtool --create-keyring /etc/ceph/ceph.mon.keyring --gen-key -n mon. --cap mon 'allow *'
 
sudo ceph-authtool --create-keyring /etc/ceph/ceph.client.admin.keyring --gen-key -n client.admin --cap mon 'allow *' --cap osd 'allow *' --cap mds 'allow *' --cap mgr 'allow *'
 
sudo ceph-authtool --create-keyring /var/lib/ceph/bootstrap-mds/ceph.keyring --gen-key -n client.bootstrap-mds --cap mon 'profile bootstrap-mds'
 
sudo ceph-authtool --create-keyring /var/lib/ceph/bootstrap-mgr/ceph.keyring --gen-key -n client.bootstrap-mgr --cap mon 'profile bootstrap-mgr'
 
sudo ceph-authtool --create-keyring /var/lib/ceph/bootstrap-osd/ceph.keyring  --gen-key -n client.bootstrap-osd --cap mon 'profile bootstrap-osd'
 
sudo ceph-authtool --create-keyring /var/lib/ceph/bootstrap-rgw/ceph.keyring --gen-key -n client.bootstrap-rgw --cap mon 'profile bootstrap-rgw'
```



```
sudo ceph-authtool /etc/ceph/ceph.mon.keyring --import-keyring /etc/ceph/ceph.client.admin.keyring
 
sudo ceph-authtool /etc/ceph/ceph.mon.keyring --import-keyring /var/lib/ceph/bootstrap-mds/ceph.keyring
 
sudo ceph-authtool /etc/ceph/ceph.mon.keyring --import-keyring /var/lib/ceph/bootstrap-mgr/ceph.keyring
 
sudo ceph-authtool /etc/ceph/ceph.mon.keyring --import-keyring /var/lib/ceph/bootstrap-osd/ceph.keyring
 
sudo ceph-authtool /etc/ceph/ceph.mon.keyring --import-keyring /var/lib/ceph/bootstrap-rgw/ceph.keyring
```

```
monmaptool --create --add node01 192.168.10.101 --add node02 192.168.10.102 --add node03 192.168.10.103 --fsid efc3cb49-3f70-41ef-a185-8ad1f604759a /etc/ceph/monmap
```



```
sudo -u ceph mkdir /var/lib/ceph/mon/ceph-`hostname -s`
```





```
chown -R ceph:ceph /etc/ceph 
chown -R ceph:ceph /var/lib/ceph


```



```
sudo -u ceph ceph-authtool /etc/ceph/ceph.mon.keyring --import-keyring /etc/ceph/ceph.client.admin.keyring
```



```
monmaptool --create --add node01 192.168.10.101 --add node02 192.168.10.102 --add node03 192.168.10.103 --fsid efc3cb49-3f70-41ef-a185-8ad1f604759a /etc/ceph/monmap
```

```
chown ceph:ceph /etc/ceph/monmap
```



```
sudo -u ceph ceph-mon --mkfs -i `hostname -s` --monmap /etc/ceph/monmap --keyring /etc/ceph/ceph.mon.keyring
```

```
sudo systemctl enable ceph-mon@`hostname -s` --now
sudo systemctl status ceph-mon@`hostname -s`
```







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

ceph mgr module enable dashboard
ceph dashboard create-self-signed-cert

ceph config set mgr mgr/dashboard/`hostname -s`/server_addr 0.0.0.0
ceph config set mgr mgr/dashboard/`hostname -s`/ssl_server_port 8443
```

```
echo 'www.123.com' > password.txt
ceph dashboard set-login-credentials admin -i password.txt
```





