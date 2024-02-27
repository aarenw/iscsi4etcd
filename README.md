Steps:
1. get every master node InitiatorName
```
oc debug node/$masternodename
# run following cli in debug container terminal
chroot /host
cat /etc/iscsi/initiatorname.iscsi
```
2.  config NetAPP iSISC target , control every node only access only one iSISC LUN, ask following information from storage admin
- target portal, such as 192.168.0.123:3260
- target authentication uasername and password
3. discovery the target on every master node
```
oc debug node/$masternodename
chroot /host
iscsiadm --mode discovery --type sendtargets --portal $tragetportal 
```
4. create butane format config file, replace the $username $password with what you got from storage admin
97-iscsi-master.bu
```
variant: openshift
version: 4.12.0
metadata:
  name: 97-iscsi-master 
  labels:
    machineconfiguration.openshift.io/role: master
systemd:
  units:
    - name: iscsi.service
      enabled: true
    - name: iscsid.service
      enabled: true
storage:
  files:
    - path: /etc/iscsi/iscsid.conf
      overwrite: true
      mode: 0644
      contents:
        inline: |
          node.startup = automatic
          node.session.auth.username = $username
          node.session.auth.password = $password
```
5. generate machine config file using butane , download butane [here]((https://mirror.openshift.com/pub/openshift-v4/clients/butane/latest/) 
```
butane 97-iscsi-master.bu -o 97-iscsi-master.yaml
```
6. create manchineconfig crd
```
oc apply -f 97-iscsi-master.yaml
```
7. check , you should find new block device
```
oc debug node/$masternodename
# run following cli in debug container terminal
chroot /host
iscsiadm -m session
lsblk

```
