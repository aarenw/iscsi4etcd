`This solution is not offical supported by Red Hat, it's a potential altalative when you have ETCD storage performance issue, please check with your Red Hat rep before implement in product env.`

# Purpose
This is a PoC to test useing external ISCSI storage to replace Nutanix software defined storage for ETCD.

# ETCD 
## What's ETCD
[etcd](https://etcd.io/) is a strongly consistent, distributed key-value store that provides a reliable way to store data that needs to be accessed by a distributed system or cluster of machines. It gracefully handles leader elections during network partitions and can tolerate machine failure, even in the leader node.   It manages the configuration data, state data, and metadata for Kubernetes .
## ETCD Storage requirement
In Kubernetes, the ETCD store data on local block storage of master node directly other than workload that leverage PV to persistent data .
In terms of latency, run etcd on top of a block device that can write at least 50 IOPS of 8000 bytes long sequentially. That is, with a latency of **10ms**, keep in mind that uses fdatasync to synchronize each write in the WAL. For heavy loaded clusters, sequential 500 IOPS of 8000 bytes (2 ms) are recommended. To measure those numbers, you can use a benchmarking tool, such as fio.

To achieve such performance, run etcd on machines that are backed by SSD or NVMe disks with low latency and high throughput. Consider single-level cell (SLC) solid-state drives (SSDs), which provide 1 bit per memory cell, are durable and reliable, and are ideal for write-intensive workloads. 

## Why does ETCD need low latency block storage
Because etcd writes data to disk and persists proposals on disk, its performance depends on disk performance. Although etcd is not particularly I/O intensive, it requires a low latency block device for optimal performance and stability. 

Because etcd’s consensus protocol depends on persistently storing metadata to a log (WAL), etcd is sensitive to disk-write latency. Slow disks and disk activity from other processes can cause long fsync latencies.

Those latencies can cause etcd to miss heartbeats, not commit new proposals to the disk on time, and ultimately experience request timeouts and temporary leader loss. High write latencies also lead to an OpenShift API slowness, which affects cluster performance. Because of these reasons, avoid colocating other workloads on the control-plane nodes that are I/O sensitive or intensive and share the same underlying I/O infrastructure.

When you mornitor the ETCD from Openshift console, the following metric should meet the requirment:
### Metrics
<https://etcd.io/docs/v3.4/metrics/>
### `etcd_disk_backend_commit_duration`
 `etcd_disk_backend_commit_duration` 99th should be lower than `25ms`
```
histogram_quantile(0.99, irate(etcd_disk_backend_commit_duration_seconds_bucket[5m]))
```
###  `etcd_disk_wal_fsync_duration`
`etcd_disk_wal_fsync_duration` 99th and 99.9th should be lower than `10ms`
```text
histogram_quantile(0.99, irate(etcd_disk_wal_fsync_duration_seconds_bucket[5m])) 
histogram_quantile(0.999, irate(etcd_disk_wal_fsync_duration_seconds_bucket[5m])) 
```

### metric for cpu `iowait`
Values in that graph should be below `4.0`
```
(sum(irate(node_cpu_seconds_total {mode="iowait"} [2m])) without (cpu)) / count(node_cpu_seconds_total) without (cpu) * 100 AND on (instance) label_replace( kube_node_role{role="master"}, "instance", "$1", "node", "(.+)" )
```
### `etcd_network_peer_round_trip_time`
`etcd_network_peer_round_trip_time` 99th should be lower than `50ms`
```
histogram_quantile(0.99, irate(etcd_network_peer_round_trip_time_seconds_bucket[5m]))
```
More information about recomend practice and how to mornitor, trouble shooting ETCD performance, please refer:

- [Recommended etcd practices](https://docs.openshift.com/container-platform/4.13/scalability_and_performance/recommended-performance-scale-practices/recommended-etcd-practices.html)
- [Hardware recommendations](https://etcd.io/docs/v3.3/op-guide/hardware/)
- [Understanding etcd and the tunables/conditions affecting performance](https://access.redhat.com/articles/7010406)
- [Storage recommendations for optimal etcd performance](https://access.redhat.com/solutions/5060021)
- [Backend Performance Requirements for OpenShift ETCD](https://access.redhat.com/solutions/4770281)
- [Diagnosing etcdGRPCRequestsSlow Alert in OpenShift Container Platform 4](https://access.redhat.com/solutions/6985052)
- [ETCD performance troubleshooting guide for OpenShift Container Platform](https://access.redhat.com/articles/6271341)

# Nutanix Storage issue
[Nutanix](https://www.nutanix.com/) is one leading private cloud solution provider, it provide Hyper Converged Infrastructure, it include software defined compute, storage and network together. It have may deployment model. If you are using this deployment model:

In this deployment model, one Nutanix only support one shared software definded storage pool, this pool export NFS to vsphere datastore, then the VM consume backend storage though vSphere Datastore->NFS->Nutanix SDS->Physical Storage.
With this deployment, all workload are shareing the same storage pool, there is no QoS or IO isolation to garantee the IO performance to specific requirement.  If unlucky , some workload consumed the IO heavyly , that will impact Openshift ETCD. 
For Nutanix managed all disk that attached on the baremetal , so there is not anyway to config local disk or FC SAN volume to VM through Raw Device Mapping.
One option is attaching iSCSI volume to the Openshift Master Nodes directly, and using them as ETCD storage, so that isolution them with the shared Nutanix storage pool.
 
# Replace ETCD storage with extal iSCSI volume
## Create external iSCSI volume and authorize it to OCP master node
If you have enterpise SAN storage, most of them support export iSCSI target, otherwise you could use targetcli export iSCSI from RHEL for testing.
For all three master nodes, export one target; create three volumes under this target, using ACL to control every volume could be accessed by only one master node.
### create iSCSI server using targetcli on RHEL
1. install targetcli
```
sudo dnf -y install targetcli
```
2. create backend storage that used to create LUN，fileio simulated block disk using file, Backstore/fileio 下创建预定大小的文件
```
targetcli
/> cd backstores/fileio
/backstores/fileio> create targetdisk1 /var/targetdisk01/targetdisk0.img 5G
/backstores/fileio> create targetdisk1 /var/targetdisk01/targetdisk1.img 5G
/backstores/fileio> create targetdisk1 /var/targetdisk01/targetdisk2.img 5G
/backstores/fileio> ls
```
3. create iSCSI target
iscsi目标，命名约定是标准的，如下所示：
`[ iqn.(year)-(month).(reverse of domain name):(any name you prefer) ]`
创建target同时也创建了portal, 位于生产环境中的服务器上可能有多块网卡，那么到底是由哪个网卡或IP地址对外提供共享存储资源呢？这就需要我们在配置文件中手动定义iSCSI服务端的信息，即在portals参数目录中写上服务器的IP地址。
```
/backstores/fileio> cd /iscsi
/iscsi> create iqn.2024-02.bastion.snn2f.sandbox211.opentlc.com:geekstarget1
/iscsi> ls
/iscsi/iqn.20…et1/tpg1> cd portals   
/iscsi/iqn.20…et1/tpg1/portals> create 192.168.245.128 ip_port=3260
```
3. create three LUN,  the lun maps with the backend storage that you created in step2.
iSCSI LUN是存储的逻辑单元，目标可以向iSCSI客户端提供一个或多个LUN，iSCSI客户端启动与iSCSI服务器的连接，导航到在上一个命令中创建的目标门户网站组（TPG）
先进入target的lun目录，然后从之前创建的后台存储创建LUN，在LUN和后台存储之间建立映射
```
/iscsi> cd iqn.2024-02.bastion.snn2f.sandbox211.opentlc.com:geekstarget1/tpg1/luns
/iscsi/iqn.20…et1/tpg1/luns> create /backstores/fileio/targetdisk0
/iscsi/iqn.20…et1/tpg1/luns> create /backstores/fileio/targetdisk1
/iscsi/iqn.20…et1/tpg1/luns> create /backstores/fileio/targetdisk2
```
4. create ACL
the ACL control which initiator(client) could access the LUN ，make sure the innitiator name is same as the `InitiatorName` in master node /etc/iscsi/initiatorname.iscsi file. After you create the ACL, every initiator could access three LUN, you should enter the initiator directory and deleter the other two LUN.
导航至ACL
```
/iscsi/iqn.20…starget1/tpg1> cd acls
/iscsi/iqn.20…et1/tpg1/acls> create iqn.1994-05.com.redhat:master0
/iscsi/iqn.20…et1/tpg1/acls> create iqn.1994-05.com.redhat:master1
/iscsi/iqn.20…et1/tpg1/acls> create iqn.1994-05.com.redhat:master2
```
5. setup the initiator login username and passwork, the three initiators using same username password
```
/iscsi/iqn.20…et1/tpg1/acls> cd iqn.1994-05.com.redhat:9b7bec98fb64
/iscsi/iqn.20…ks:initiator1> set auth userid=initiator1
/iscsi/iqn.20…ks:initiator1> set auth password=gai0daeNgu

```
6. After you done, under targecli
```
cd /
ls
```
Write down the target portal information such 192.168.0.146:3260 


7. exit and save
```
/iscsi> exit
```
8. start service and config firewall
```
sudo systemctl enable target
sudo firewall-cmd --add-service=iscsi-target --permanent
sudo firewall-cmd --reload
```

## Config master node to attach iSCSI volume
1. config the master node initiator name in  /etc/iscsi/initiatorname.iscsi as you used in target configuration
```bash
oc debug node/$masternodename
chroot /host
sed -i   's/InitiatorName=iqn.1994-05.com.redhat:.*/InitiatorName=$yourdesiredinitiatorname/' /etc/iscsi/initiatorname.iscsi
```
The change will be keeped in master node even upgrade. This is one time job if you don't replace the node with newone. 
You also could change it using machine config, but every master node should have different initiator name, so you have to create three machine config poor and three machine config to do this job. 

2. Discovery target on master nodes
 The discovery action will write node and taget informaton under `/var/lib/iscsi/nodes` directory
```bash
oc debug node/$masternodename
chroot /host
iscsiadm --mode discovery --type sendtargets --portal $tragetportal 
```
This can be done through machine config, create one service unit to run iscsiadm discovery command , this service should be runed before iscsi.service . There is selinux issure when run iscsiadm in systemd service, selinu will reject iscsiadm create director/files under  `/var/lib/iscsi/nodes/xxxx`. The rude approach is disable selinu before this command and enable it after this command. If your enterpise security policy doesn't allow your disable selinu,  you should create one selinu policy to authrise iscsiadm have the permission on `/var/lib/iscsi/nodes/xxxx`, I am not good at the selinux policy, didn't test.
3. change the iscsid configuration file /etc/iscsi/iscsid.conf, change related username password that your storage adm shared with you.
```
node.startup = automatic
#isns.address = 192.168.0.146
#isns.port = 3260
node.session.auth.username = initiator1
node.session.auth.password = gai0daeNgu
#discovery.sendtargets.auth.username = initiator1
#discovery.sendtargets.auth.password = gai0daeNgu
```
4. start iscsi.service , this service will login all target that you discoveryed in step 2
5. start iscsid.service
6. verify the new disk on master node, you will find the new block storage sda
```bash
lsblk
```
7. steps 2-6 could be done through machine config
- create butane format config file 97-iscsi-master.bu, replace the $username $password with what you got from storage admin 
- use butane create machine config yaml file, you could download  butane from  [here](https://mirror.openshift.com/pub/openshift-v4/clients/butane/latest/) 
```
butane 97-iscsi-master.bu -o 97-iscsi-master.yaml
```
-  create manchineconfig crd
```
oc apply -f 97-iscsi-master.yaml
```
- after machine config applied and master nodes reboot, you should find new block device
```
oc debug node/$masternodename
chroot /host
iscsiadm -m session
lsblk
```

## Replace the ETCD sorage with the new iSCSI disk
1. Create Machine Config yaml file  98-var-lib-etcd.yaml
2. create manchineconfig crd
```
oc apply -f 98-var-lib-etcd.yaml
```
This machine config create fowlling service unit to implement:
- create XFS file system on the specified disk. This sould be run after iscsi.service
-  mounts disk to /var/lib/etcd. this should be run before remote-fs.target, different with local disk mount, the mount option should be `_netdev,x-systemd.requires=iscsi.service,prjquota`
-  Sync the content from /sysroot/ostree/deploy/rhcos/var/lib/etcd syncs to /var/lib/etcd. the rsync have same selinux issue same as iscsiadm, the workaroud is same.
 

3. after machine config applied and master nodes reboot
-  check whether new block device map to /var/lib/etcd
```
oc debug node/$masternodename
chroot /host
lsblk
``` 
- check whether ETCD work well
```
oc get pods -n openshift-etcd
```
4. For create file system on new disk and sync etcd data should be run at first time, we need replace the machine config with 98-var-lib-etcd-next.yaml
```
oc replace -f 98-var-lib-etcd-next.yaml
```

## Upgrade the cluster to do further test

## Combine machine config
To simplify, you could combine replace -f 98-var-lib-etcd-next.yaml together with 98-var-lib-etcd.yaml


