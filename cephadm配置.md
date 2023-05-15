### ceph基础环境配置

#### 1、yum源配置

```
rpm -Uvh https://mirrors.aliyun.com/ceph/rpm-17.2.6/el8/noarch/ceph-release-1-1.el8.noarch.rpm
```

```
sed -i "s/download.ceph.com/mirrors.aliyun.com\/ceph/g" /etc/yum.repos.d/ceph.repo
```

```
rpm --import "https://mirrors.aliyun.com/ceph/keys/release.asc"
```

```
dnf install chrony python3 systemd lvm2
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



hwclock --show
hwclock --systohc
```

```
vim /etc/hosts

192.168.10.101  ceph-node01
192.168.10.102  ceph-node02
192.168.10.103  ceph-node03

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



```shell
 #安装所需的软件包。yum-utils 提供了 yum-config-manager ，并且 device mapper 存储驱动程序需要 device-mapper-persistent-data 和 lvm2。
 yum install -y yum-utils  device-mapper-persistent-data lvm2
 
 # 选择国内的一些源地址：
 yum-config-manager --add-repo  http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
 
 # 安装最新版本的 Docker Engine-Community 和 containerd
 yum install docker-ce docker-ce-cli containerd.io -y
 
# 启动
systemctl enable --now docker
systemctl enable --now containerd
 
# 配置阿里镜像源，日志大小，cgroup
mkdir -p /etc/docker

cat > /etc/docker/daemon.json << EOF
{
  "registry-mirrors": ["https://b9pmyelo.mirror.aliyuncs.com"],
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF

# 加载配置文件重启
systemctl daemon-reload

# 重启
systemctl restart docker

# 查看docker 配置信息
docker info
```



```
ssh-keygen -t rsa
ssh-copy-id -i root@node02
ssh-copy-id -i root@node03
```





```
dnf install -y cephadm ceph-common
```

```
docker pull quay.io/ceph/ceph:v16
```



```

cat <<EOF > initial-ceph.conf
[global]

mon_clock_drift_allowed = 1
mon_clock_drift_warn backoff = 10
 
osd_pool_default_size = 2
osd_pool_default_min_size = 1
osd_pool_default_pg_num = 128
osd_pool_default_pgp_num = 128

EOF
```



```
cephadm bootstrap --mon-ip 192.168.10.101 --config  initial-ceph.conf --no-minimize-config


```



```
ssh-copy-id -f -i /etc/ceph/ceph.pub root@ceph-node02
ssh-copy-id -f -i /etc/ceph/ceph.pub root@ceph-node03
```

```
ceph orch host add ceph-node02 192.168.10.102
ceph orch host add ceph-node04 192.168.10.104 _no_schedule

ceph orch daemon add osd ceph-node01:/dev/sdb


# 禁用不安全模式
ceph config set mon auth_allow_insecure_global_id_reclaim false

```





```
ceph config assimilate-conf -i /etc/ceph/ceph.conf

ceph config dump



ceph mgr module enable cephadm
ceph orch set backend cephadm
```



```
集群重新扫描
ceph orch host rescan  ceph-node01 --with-summary
 
```







```
ceph osd crush class ls
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



