

```
ceph osd crush class ls
ceph osd crush class create ssd
```

```
ceph osd tree
```

```
ceph osd crush add-bucket test root
ceph osd crush add-bucket test01 host
ceph osd crush move test01 root=test
```

```
ceph osd crush create-or-move osd.0 1.00000 host=test01
```

```
ceph osd crush rm-device-class osd.0
ceph osd crush set-device-class ssd osd.0
```

```
ceph osd crush rule create-replicated ssd_rule default host ssd

ceph osd crush rule list

ceph osd pool create cache 64 64 ssd_rule

ceph osd pool get cache crush_rule
```

```
ceph osd tier add data cache

ceph osd tier cache-mode cache writeback
ceph osd tier set-overlay data cache
```

```
# 将 cache pool 放置到 data pool 前端
ceph osd tier add data cache
# 设置缓存模式为 readonly
ceph osd tier cache-mode cache readonly
```

