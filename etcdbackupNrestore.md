
# How to backup Live Deployments on K8s Cluster using ETCD(ctl) & restore deployments during Disaster event

### Scenario :
For OS patch update , Sysadmin need to reboot K8s Master Server as scheduled activity & they anticipate low risk of any data loss.
Considering this activity , DevOPS team wants to take a “Backup of the running Deployments on the K8s Cluster” – just to be on Safer side.
After the maintenance, the Cluster is UP but no “Deployments” are seen . The following article show how to recover the deployments from backup.

#### etcdctl
[etcdctl] is a command line client for etcd. It can be used in scripts or for administrators to explore an etcd cluster.

## Procedure to take backup of Live Deployments

On Kubernetes Master node , as "root" user , find the path of etcdctl:

```sh
[vagrant@kmaster ~]$ sudo find / -name etcdctl -print
/var/lib/docker/overlay2/5089447ac1df5c81b1e856dbe311e1cdf2af4162546ce1bffd9a5bcb17f26a48/diff/usr/local/bin/etcdctl
/var/lib/docker/overlay2/e32f05a91646893f86bfebc49bb904e5336484830f0cc20667786a5d3c9b3ead/merged/usr/local/bin/etcdctl
```
Copy etcdctl under /usr/bin (so we can avoid absolute path)

```sh
[vagrant@kmaster ~]$ sudo cp /var/lib/docker/overlay2/5089447ac1df5c81b1e856dbe311e1cdf2af4162546ce1bffd9a5bcb17f26a48/diff/usr/local/bin/etcdctl /usr/bin/

 ```
Check if etcdctl command works
```sh
[vagrant@kmaster ~]$ sudo ETCDCTL_API=3 etcdctl member list --cacert=/etc/kubernetes/pki/etcd/ca.crt --cert=/etc/kubernetes/pki/etcd/server.crt --key=/etc/kubernetes/pki/etcd/server.key --endpoints=127.0.0.1:2379
cbe1b3a9f7c32775, started, kmaster.mylab.com, https://172.42.42.100:2380, https://172.42.42.100:2379, false
 
 ```
Now take a backup of etcd db using etcdctl command
```sh
[vagrant@kmaster ~]$ sudo ETCDCTL_API=3 etcdctl snapshot save --cacert=/etc/kubernetes/pki/etcd/ca.crt --cert=/etc/kubernetes/pki/etcd/server.crt --key=/etc/kubernetes/pki/etcd/server.key --endpoints=127.0.0.1:2379 /tmp/snapshot-pre-boot.db

{"level":"info","ts":1605596608.7358596,"caller":"snapshot/v3_snapshot.go:119","msg":"created temporary db file","path":"/tmp/snapshot-pre-boot.db.part"}
{"level":"info","ts":"2020-11-17T07:03:28.745Z","caller":"clientv3/maintenance.go:200","msg":"opened snapshot stream; downloading"}
{"level":"info","ts":1605596608.7455497,"caller":"snapshot/v3_snapshot.go:127","msg":"fetching snapshot","endpoint":"127.0.0.1:2379"}
{"level":"info","ts":"2020-11-17T07:03:28.827Z","caller":"clientv3/maintenance.go:208","msg":"completed snapshot read; closing"}
{"level":"info","ts":1605596608.8368568,"caller":"snapshot/v3_snapshot.go:142","msg":"fetched snapshot","endpoint":"127.0.0.1:2379","size":"4.2 MB","took":0.100909395}
{"level":"info","ts":1605596608.8369892,"caller":"snapshot/v3_snapshot.go:152","msg":"saved","path":"/tmp/snapshot-pre-boot.db"}

Snapshot saved under /tmp/snapshot-pre-boot.db

[vagrant@kmaster ~]$ ls -ltr /tmp/snapshot-pre-boot.db
-rw-------. 1 root root 4206624 Nov 17 07:03 /tmp/snapshot-pre-boot.db
```
The backup in now stored in a file /tmp/snapshot-pre-boot.db

Take a screenshot of etcd process running on the Master Server , as we need few values for later usage (during recovery)
```sh
[vagrant@kmaster ~]$ sudo ps -ef | grep etcd
vagrant   3016  9678  0 07:08 pts/0    00:00:00 grep --color=auto etcd
root      4929  4868  6 03:53 ?        00:12:37 kube-apiserver --advertise-address=172.42.42.100 --allow-privileged=true --authorization-mode=Node,RBAC --client-ca-file=/etc/kubernetes/pki/ca.crt --enable-admission-plugins=NodeRestriction --enable-bootstrap-token-auth=true --etcd-cafile=/etc/kubernetes/pki/etcd/ca.crt --etcd-certfile=/etc/kubernetes/pki/apiserver-etcd-client.crt --etcd-keyfile=/etc/kubernetes/pki/apiserver-etcd-client.key --etcd-servers=https://127.0.0.1:2379 --insecure-port=0 --kubelet-client-certificate=/etc/kubernetes/pki/apiserver-kubelet-client.crt --kubelet-client-key=/etc/kubernetes/pki/apiserver-kubelet-client.key --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname --proxy-client-cert-file=/etc/kubernetes/pki/front-proxy-client.crt --proxy-client-key-file=/etc/kubernetes/pki/front-proxy-client.key --requestheader-allowed-names=front-proxy-client --requestheader-client-ca-file=/etc/kubernetes/pki/front-proxy-ca.crt --requestheader-extra-headers-prefix=X-Remote-Extra- --requestheader-group-headers=X-Remote-Group --requestheader-username-headers=X-Remote-User --secure-port=6443 --service-account-key-file=/etc/kubernetes/pki/sa.pub --service-cluster-ip-range=10.96.0.0/12 --tls-cert-file=/etc/kubernetes/pki/apiserver.crt --tls-private-key-file=/etc/kubernetes/pki/apiserver.key
root      4935  4857  2 03:53 ?        00:04:08 etcd --advertise-client-urls=https://172.42.42.100:2379 --cert-file=/etc/kubernetes/pki/etcd/server.crt --client-cert-auth=true --data-dir=/var/lib/etcd --initial-advertise-peer-urls=https://172.42.42.100:2380 --initial-cluster=kmaster.mylab.com=https://172.42.42.100:2380 --key-file=/etc/kubernetes/pki/etcd/server.key --listen-client-urls=https://127.0.0.1:2379,https://172.42.42.100:2379 --listen-metrics-urls=http://127.0.0.1:2381 --listen-peer-urls=https://172.42.42.100:2380 --name=kmaster.mylab.com --peer-cert-file=/etc/kubernetes/pki/etcd/peer.crt --peer-client-cert-auth=true --peer-key-file=/etc/kubernetes/pki/etcd/peer.key --peer-trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt --snapshot-count=10000 --trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt
 
 ```
 
Screen shot Of Live Pods,Replicasets,Deployments,SVC before reboot
 ```sh
[vagrant@kmaster ~]$ kubectl get po,rs,deployments,svc
NAME                       READY   STATUS    RESTARTS   AGE
pod/blue-b5f45bf75-bl95x   1/1     Running   0          13m
pod/blue-b5f45bf75-h2664   1/1     Running   0          24m
pod/red-cfb6fd69b-9dvsj    1/1     Running   0          31m
pod/red-cfb6fd69b-zqv6h    1/1     Running   0          27m
 
NAME                             DESIRED   CURRENT   READY   AGE
replicaset.apps/blue-b5f45bf75   2         2         2       24m
replicaset.apps/red-cfb6fd69b    2         2         2       31m
 
NAME                   READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/blue   2/2     2            2           24m
deployment.apps/red    2/2     2            2           31m

NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
service/kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   3h19m
```
<<<<>>>>> 
At this point the Sysadmin team completes their maintenance by rebooting Master Server,the cluster is UP but no “Deployments” are seen on K8s Cluster.
Now DevOPS is Paged :) to take a look & restore back deployments from the backup .

<<<<>>>> 

## Procedure to Restore Deployment from the backup
Now that we have the backup stored in a file "/tmp/snapshot-pre-boot.db" , we can restore deployments using "etcdctl snapshot restore" 
```sh
[vagrant@kmaster ~]$ sudo ETCDCTL_API=3 etcdctl snapshot restore --cacert=/etc/kubernetes/pki/etcd/ca.crt --cert=/etc/kubernetes/pki/etcd/server.crt --key=/etc/kubernetes/pki/etcd/server.key --endpoints=127.0.0.1:2379  --data-dir="/var/lib/etcd-from-backup" --initial-cluster="kmaster.mylab.com=https://127.0.0.1:2380" --name="kmaster.mylab.com" --initial-advertise-peer-urls="https://127.0.0.1:2380" --initial-cluster-token="etcd-cluster-1" /tmp/snapshot-pre-boot.db

{"level":"info","ts":1605598777.265926,"caller":"snapshot/v3_snapshot.go:296","msg":"restoring snapshot","path":"/tmp/snapshot-pre-boot.db","wal-dir":"/var/lib/etcd-from-backup/member/wal","data-dir":"/var/lib/etcd-from-backup","snap-dir":"/var/lib/etcd-from-backup/member/snap"}
{"level":"info","ts":1605598777.3072398,"caller":"mvcc/kvstore.go:380","msg":"restored last compact revision","meta-bucket-name":"meta","meta-bucket-name-key":"finishedCompactRev","restored-compact-revision":27342}
{"level":"info","ts":1605598777.3125424,"caller":"membership/cluster.go:392","msg":"added member","cluster-id":"7581d6eb2d25405b","local-member-id":"0","added-peer-id":"e92d66acd89ecf29","added-peer-peer-urls":["https://127.0.0.1:2380"]}
{"level":"info","ts":1605598777.3154528,"caller":"snapshot/v3_snapshot.go:309","msg":"restored snapshot","path":"/tmp/snapshot-pre-boot.db","wal-dir":"/var/lib/etcd-from-backup/member/wal","data-dir":"/var/lib/etcd-from-backup","snap-dir":"/var/lib/etcd-from-backup/member/snap"}


[vagrant@kmaster home]$ sudo ETCDCTL_API=3 etcdctl member list --cacert=/etc/kubernetes/pki/etcd/ca.crt --cert=/etc/kubernetes/pki/etcd/server.crt --key=/etc/kubernetes/pki/etcd/server.key --endpoints=127.0.0.1:2379
e92d66acd89ecf29, started, kmaster.mylab.com, https://127.0.0.1:2380, https://172.42.42.100:2379, false

```
Now make changes in etcd.yaml (edit), for Kubernetes to load new etcd configs (changing default path to new path of restored folder)

```sh
[root@kmaster member]# cd /etc/kubernetes/manifests/
[root@kmaster manifests]# ls -ltr
total 16
-rw-------. 1 root root 3157 Nov 17 03:52 kube-apiserver.yaml
-rw-------. 1 root root 2830 Nov 17 03:52 kube-controller-manager.yaml
-rw-------. 1 root root 1384 Nov 17 03:52 kube-scheduler.yaml
-rw-------. 1 root root 2119 Nov 17 03:52 etcd.yaml

vi etcd.yaml

from
     - --data-dir=/var/lib/etcd
to
     - --data-dir=/var/lib/etcd-from-backup

add below line
     - --initial-cluster-token=etcd-cluster-1

from
     - mountPath: /var/lib/etcd
to
     - mountPath: /var/lib/etcd-from-backup

from
       path: /var/lib/etcd
to
       path: /var/lib/etcd-from-backup
     
```
Save file and check if new etcd docker running
```sh
[root@kmaster manifests]# docker ps | grep etcd
978c681c10c6        0369cf4303ff              "etcd --advertise-cl…"   19 seconds ago      Up 18 seconds                           k8s_etcd_etcd-kmaster.mylab.com_kube-system_5597f23dd355eae5b44179ea36ec0acb_0
5e53c8bad1e9        k8s.gcr.io/pause:3.2      "/pause"                 19 seconds ago      Up 18 seconds                           k8s_POD_etcd-kmaster.mylab.com_kube-system_5597f23dd355eae5b44179ea36ec0acb_0
```
Check member list 
```sh
[vagrant@kmaster home]$ sudo ETCDCTL_API=3 etcdctl member list --cacert=/etc/kubernetes/pki/etcd/ca.crt --cert=/etc/kubernetes/pki/etcd/server.crt --key=/etc/kubernetes/pki/etcd/server.key --endpoints=127.0.0.1:2379
e92d66acd89ecf29, started, kmaster.mylab.com, https://127.0.0.1:2380, https://172.42.42.100:2379, false
```
Now check if Pods,Deployments,Replicaset are restored .
Compare them with screen shot you took prior to reboot , except "AGE" of Pod , everything should remain same.

```sh
[vagrant@kmaster home]$ kubectl get po,rs,deployments,svc
NAME                       READY   STATUS    RESTARTS   AGE
pod/blue-b5f45bf75-bl95x   1/1     Running   0          49m
pod/blue-b5f45bf75-h2664   1/1     Running   0          60m
pod/red-cfb6fd69b-9dvsj    1/1     Running   0          67m
pod/red-cfb6fd69b-zqv6h    1/1     Running   0          63m

NAME                             DESIRED   CURRENT   READY   AGE
replicaset.apps/blue-b5f45bf75   2         2         2       60m
replicaset.apps/red-cfb6fd69b    2         2         2       67m

NAME                   READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/blue   2/2     2            2           60m
deployment.apps/red    2/2     2            2           67m

NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
service/kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   3h55m
```

[etcdctl]: https://chromium.googlesource.com/external/github.com/coreos/etcd/+/HEAD/etcdctl/READMEv2.md

