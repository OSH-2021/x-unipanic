# 基于docker的部署-cephadm

参考官方文档： https://docs.ceph.com/en/latest/cephadm/install/

## 初始化配置

[IP](https://www.notion.so/e8c6166401f34811a1abbd14fb62a3e3)

### 修改每个主机的hosts以及hostname

#### ceph-node0

```bash
echo ceph-node0 | sudo tee /etc/hostname
```

修改`/etc/hosts`为

```bash
127.0.1.1       ceph-node0

172.16.75.6 ceph-node1
172.16.75.7 ceph-node2
```

#### ceph-node1

```bash
echo ceph-node1 | sudo tee /etc/hostname
```

修改`/etc/hosts`为

```bash
127.0.1.1       ceph-node1

172.16.75.5 ceph-node0
172.16.75.6 ceph-node1
172.16.75.7 ceph-node2
```

#### ceph-node2

```bash
echo ceph-node2 | sudo tee /etc/hostname
```

修改`/etc/hosts`为

```bash
127.0.1.1       ceph-node2

172.16.75.5 ceph-node0
172.16.75.6 ceph-node1
172.16.75.7 ceph-node2
```

## 安装cephadm

```bash
sudo apt install cephadm -y
```

## 创建集群

在node0上面运行

```bash
sudo cephadm bootstrap --mon-ip 172.16.75.5
```

[LOG](https://www.notion.so/LOG-2ae8619133594c9caf3796b458ece19a)

- 在node0节点上为Ceph集群创建`Mon`和`Mgr`服务
- 生成一个新的SSH Key，并将其添加到root用户/root/.ssh/authorized_keys文件中
- 将默认的配置文件写入/etc/ceph/ceph.conf
- 创建client.admin密钥的副本到/etc/ceph/ceph.client.admin.keyring
- 将public key写入到/etc/ceph/ceph.pub

之后可以在 [`https://172.16.75.5:8443`](https://172.16.75.5:8443/#/dashboard) 上面访问到ceph的控制面板，注意必须是https的协议，不然好像连不上去

### 进入 CLI

```bash
sudo cephadm shell
```

![enter_shell](https://github.com/OSH-2021/x-unipanic/blob/master/Lab4/img/enter_shell.png)



### 查看集群状态

```bash
root@ceph-node0:/# ceph -s
  cluster:
    id:     6dcb33dc-e390-11eb-acdf-23342d4d06f1
    health: HEALTH_WARN
            OSD count 0 < osd_pool_default_size 3

  services:
    mon: 1 daemons, quorum ceph-node0 (age 13m)
    mgr: ceph-node0.fsniny(active, since 13m)
    osd: 0 osds: 0 up, 0 in

  data:
    pools:   0 pools, 0 pgs
    objects: 0 objects, 0 B
    usage:   0 B used, 0 B / 0 B avail
    pgs:
```

### 设置集群默认的Mon数量

一个典型的Ceph集群具有三个或五个Mon守护程序，这些守护程序分布在不同的主机上，默认情况下，Cephadm将设置5个Mon节点，但是本篇文章只有两个，且另外一个是后面添加的，所以我们将集群Mon数量设置为默认一个即可。

```bash
ceph orch apply mon 1
```

## 添加节点

### 查看有多少节点

```bash
root@ceph-node0:/# ceph orch host ls
HOST        ADDR        LABELS  STATUS
ceph-node0  ceph-node0
```

需要将 ceph-node-0 上的 `/etc/ceph/ceph.pub` 复制到 ceph-node1 和 ceph-node2上面。由于可能没有允许 root 的 password 登录，因此需要手动复制到 authorized_keys 里面

### 在ceph中添加新节点

在ceph shell中

```bash
ceph orch host add ceph-node1 --labels=mon
ceph orch host add ceph-node2 --labels=osd
```

结果

```
root@ceph-node0:/# ceph orch host add ceph-node1 --labels=mon
Added host 'ceph-node1'
root@ceph-node0:/# ceph orch host add ceph-node2 --labels=osd
Added host 'ceph-node2'
```

在后台中可以看到

![add_node](https://github.com/OSH-2021/x-unipanic/blob/master/Lab4/img/add_node.png)

### 将节点设置为mon节点

```bash
ceph orch apply mon ceph-node1
```

在dashboard-status里面可以看到monitors变为了2

![mon_2](https://github.com/OSH-2021/x-unipanic/blob/master/Lab4/img/mon_2.png)

在hosts里面可以看到 ceph-node1 上面的service增加了

![service_inc](https://github.com/OSH-2021/x-unipanic/blob/master/Lab4/img/service_inc.png)

## 添加OSD节点

### 查看设备列表

查看所有的硬盘

```bash
root@ceph-node0:/# ceph orch device ls
Hostname    Path      Type  Serial  Size   Health   Ident  Fault  Available
ceph-node0  /dev/sdb  hdd           21.4G  Unknown  N/A    N/A    Yes
ceph-node0  /dev/sdc  hdd           21.4G  Unknown  N/A    N/A    Yes
ceph-node0  /dev/sdd  hdd           21.4G  Unknown  N/A    N/A    Yes
ceph-node1  /dev/sdb  hdd           21.4G  Unknown  N/A    N/A    Yes
ceph-node1  /dev/sdc  hdd           21.4G  Unknown  N/A    N/A    Yes
ceph-node1  /dev/sdd  hdd           21.4G  Unknown  N/A    N/A    Yes
ceph-node2  /dev/sdb  hdd           21.4G  Unknown  N/A    N/A    Yes
ceph-node2  /dev/sdc  hdd           21.4G  Unknown  N/A    N/A    Yes
ceph-node2  /dev/sdd  hdd           21.4G  Unknown  N/A    N/A    Yes
```

查看某一个设备的硬盘，通过 `--hostname` 参数

```bash
root@ceph-node0:/# ceph orch device ls --hostname "ceph-node0"
Hostname    Path      Type  Serial  Size   Health   Ident  Fault  Available
ceph-node0  /dev/sdb  hdd           21.4G  Unknown  N/A    N/A    Yes
ceph-node0  /dev/sdc  hdd           21.4G  Unknown  N/A    N/A    Yes
ceph-node0  /dev/sdd  hdd           21.4G  Unknown  N/A    N/A    Yes
```

### 添加设备到osd中

添加node0节点上面的/dev/sdb设备

```bash
ceph orch daemon add osd ceph-node0:/dev/sdb
```

添加将所有主机的所有可用设备

```bash
ceph orch apply osd --all-available-devices
```

运行以上命令后，如果集群检测到新设备将自动添加到OSD，要禁用在可用设备上自动创建OSD的功能，请使用以下unmanaged参数：

如果要避免此行为（禁用可用设备上的OSD自动创建），请使用unmanaged参数：

```bash
ceph orch apply osd --all-available-devices --unmanaged=true
```

感觉不能直接加--umanaged=true参数，要先运行第一条命令，再运行--umanaged

-dry-run参数在不实际创建OSD的情况下呈现将要发生的情况的预览

```bash
root@ceph-node0:/# ceph orch apply osd --all-available-devices --dry-run
WARNING! Dry-Runs are snapshots of a certain point in time and are bound
to the current inventory setup. If any on these conditions changes, the
preview will be invalid. Please make sure to have a minimal
timeframe between planning and applying the specs.
################
OSDSPEC PREVIEWS
################
+---------+-----------------------+------------+----------+----+-----+
|SERVICE  |NAME                   |HOST        |DATA      |DB  |WAL  |
+---------+-----------------------+------------+----------+----+-----+
|osd      |all-available-devices  |ceph-node0  |/dev/sdc  |-   |-    |
|osd      |all-available-devices  |ceph-node0  |/dev/sdd  |-   |-    |
|osd      |all-available-devices  |ceph-node1  |/dev/sdb  |-   |-    |
|osd      |all-available-devices  |ceph-node1  |/dev/sdc  |-   |-    |
|osd      |all-available-devices  |ceph-node1  |/dev/sdd  |-   |-    |
|osd      |all-available-devices  |ceph-node2  |/dev/sdb  |-   |-    |
|osd      |all-available-devices  |ceph-node2  |/dev/sdc  |-   |-    |
|osd      |all-available-devices  |ceph-node2  |/dev/sdd  |-   |-    |
+---------+-----------------------+------------+----------+----+-----+
```

查看已挂载的OSD列表

```bash
root@ceph-node0:/# ceph osd tree
ID  CLASS  WEIGHT   TYPE NAME            STATUS  REWEIGHT  PRI-AFF
-1         0.17537  root default
-3         0.05846      host ceph-node0
 0    hdd  0.01949          osd.0            up   1.00000  1.00000
 3    hdd  0.01949          osd.3            up   1.00000  1.00000
 4    hdd  0.01949          osd.4            up   1.00000  1.00000
-5         0.05846      host ceph-node1
 2    hdd  0.01949          osd.2            up   1.00000  1.00000
 5    hdd  0.01949          osd.5            up   1.00000  1.00000
 7    hdd  0.01949          osd.7            up   1.00000  1.00000
-7         0.05846      host ceph-node2
 1    hdd  0.01949          osd.1            up   1.00000  1.00000
 6    hdd  0.01949          osd.6            up   1.00000  1.00000
 8    hdd  0.01949          osd.8            up   1.00000  1.00000
```

### 激活现有的OSD

如果重新安装了主机的操作系统，则需要再次激活现有的OSD

```bash
ceph cephadm osd activate <host>...
```

## CephFS

### 创建fs

创建一些MDS（Metadata Server）

```bash
ceph fs volume create myfs --placement="ceph-node0,ceph-node1"
```

### 查看创建的fs

```bash
root@ceph-node0:/# ceph fs ls
name: myfs, metadata pool: cephfs.myfs.meta, data pools: [cephfs.myfs.data ]
```

### 挂载cephfs

在ceph-node2上面

#### 创建挂载目录

```bash
mkdir -p /mnt/ceph/myfs
```

#### 查看key

```bash
root@ceph-node0:/# cat /etc/ceph/ceph.keyring
[client.admin]
        key = AQDAEu1gkGbHOhAABHar7V3645EtvH/qlc6iKA==
```

#### 使用mount进行挂载

在ceph-node2上执行

```bash
sudo mount -t ceph ceph-node0:6789:/ /mnt/ceph/myfs -o name=admin,secret=AQDAEu1gkGbHOhAABHar7V3645EtvH/qlc6iKA==
```

遇到了一个问题，[挂载CephFS时出现failed: No such process的问题](https://www.ichenfu.com/2019/09/07/mount-cephfs-no-such-process/) 感觉比较离谱 需要安装 keyutils 这个包

### 测试

```bash
# elsa @ ceph-node2 in /mnt/ceph/myfs [23:53:19]
$ df -h | grep ceph
ceph-node0:6789:/   54G     0   54G   0% /mnt/ceph/myfs
# elsa @ ceph-node2 in /mnt/ceph/myfs [23:53:06]
$ echo a | sudo tee ping
a

# elsa @ ceph-node2 in /mnt/ceph/myfs [23:53:16]
$ cat ping
a
```

可以看出，是能正常使用的

## Benchmark

### 创建测速pool

```bash
ceph osd pool create scbench 128 128
```

### 生成测试文件&测试写速度

```bash
rados bench -p scbench 10 write --no-cleanup
```

结果

```
root@ceph-node0:/# rados bench -p scbench 10 write --no-cleanup
hints = 1
Maintaining 16 concurrent writes of 4194304 bytes to objects of size 4194304 for up to 10 seconds or 0 objects
Object prefix: benchmark_data_ceph-node0_59
  sec Cur ops   started  finished  avg MB/s  cur MB/s last lat(s)  avg lat(s)
    0       0         0         0         0         0           -           0
    1      16        18         2   7.95231         8    0.906519    0.768087
    2      16        23         7   10.4531        20     1.20302     1.00606
    3      16        23         7   5.69969         0           -     1.00606
    4      16        24         8   4.33342         2     4.95839      1.5001
    5      16        28        12   5.72421        16     8.13396     3.65596
    6      16        28        12   5.11334         0           -     3.65596
    7      16        28        12   4.45091         0           -     3.65596
    8      14        28        14   4.74565   2.66667     9.87271     4.61819
    9      14        28        14      4.37         0           -     4.61819
   10      14        28        14   4.02868         0           -     4.61819
   11       2        28        26   6.96873        16     6.51716     7.82486
   12       1        28        27   6.78132         4     7.15459     7.80004
Total time run:         16.095
Total writes made:      28
Write size:             4194304
Object size:            4194304
Bandwidth (MB/sec):     6.95868
Stddev Bandwidth:       7.44791
Max bandwidth (MB/sec): 20
Min bandwidth (MB/sec): 0
Average IOPS:           1
Stddev IOPS:            1.92275
Max IOPS:               5
Min IOPS:               0
Average Latency(s):     8.05765
Stddev Latency(s):      5.01325
Max latency(s):         15.0132
Min latency(s):         0.629655
```

### 测试顺序读

运行命令

```bash
rados bench -p scbench 10 seq
```

结果

```
root@ceph-node0:/# rados bench -p scbench 10 seq
hints = 1
  sec Cur ops   started  finished  avg MB/s  cur MB/s last lat(s)  avg lat(s)
    0       0         0         0         0         0           -           0
Total time run:       0.733222
Total reads made:     28
Read size:            4194304
Object size:          4194304
Bandwidth (MB/sec):   152.75
Average IOPS:         38
Stddev IOPS:          0
Max IOPS:             27
Min IOPS:             27
Average Latency(s):   0.316084
Max latency(s):       0.656681
Min latency(s):       0.0277402
```

### 清理测试文件

```bash
rados -p scbench cleanup
```