# Rook Ceph Installation on Minikube for Mac M1 with Docker Driver 
This article will help you to set up a test ceph cluster running on single node minikube cluster. We have used docker as driver to create the minikube cluster.

System Requirements:

2 CPUs or more

2GB of free memory

20GB of free disk space

Internet connection


## Procedure:

### Install docker

```
brew install docker
brew install colima
colima start
```
### Install and start minikube

Minikube is local Kubernetes, focusing on making it easy to learn and develop for Kubernetes.

```
brew install minikube
minikube start --disk-size=20g --driver docker
```
Install kubectl on your host machine 

```
curl -LO "https://dl.k8s.io/release/v1.26.1/bin/darwin/arm64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/
```
As with docker on M1 mac, we can not attach /dev/sd* or /dev/vd* devices as it is ARM based and Ceph does not use /dev/loop devices which are available in minikube with docker driver. That's why we will be using network block device /dev/nbd0
```
minikube ssh
sudo mkdir /mnt/disks

# Create an empty file of size 10GB to mount disk as ceph osd 
sudo dd if=/dev/zero of=/mnt/disks/mydisk.img bs=1M count=10240
sudo apt update
sudo apt upgrade
sudo apt-get install qemu-utils

# To bind nbd device to the file 
sudo qemu-nbd --format raw -c /dev/nbd0 /mnt/disks/mydisk.img
```
Note: Please check there is no necessary data in /dev/nbd0, otherwise please take backup.
```
sudo wipefs -a /dev/nbd0
```
Verify the size of nbd device using lsblk
```
 lsblk | grep nbd0
```
Clone rook repository on your host machine.

```
git clone https://github.com/rook/rook.git
```
Change into rook/deploy/examples/ directory

```
cd rook/deploy/examples/
```
Deploy rook operator
```
kubectl create -f crds.yaml -f common.yaml -f operator.yaml
```
Verify the rook-ceph-operator is in running state before proceeding
```
kubectl get pods -n rook-ceph
```
In cluster-test.yaml, make necessary changes to storage section with the device selection:
```
  storage:
    useAllNodes: false
    useAllDevices: false
    nodes:
    - name: minikube    # node name of minikube node
      devices:
      - name: /dev/nbd0   # device name being used
    allowDeviceClassUpdate: true
    allowOsdCrushWeightUpdate: false
```
Create ceph cluster
```
kubectl create -f cluster-test.yaml
```
Verify the cluster is running by checking pods status in rook-ceph namespace
```
kubectl -n rook-ceph get pod
NAME                                           READY   STATUS      RESTARTS        AGE
csi-cephfsplugin-m5vzn                         2/2     Running     1 (2m12s ago)   2m48s
csi-cephfsplugin-provisioner-d4d7df87-bmj4h    5/5     Running     1 (2m ago)      2m48s
csi-rbdplugin-4ksmc                            2/2     Running     1 (2m13s ago)   2m48s
csi-rbdplugin-provisioner-6d4bbf78d7-b2b6z     5/5     Running     1 (2m6s ago)    2m48s
rook-ceph-exporter-minikube-5dd959bff6-2p8kj   1/1     Running     0               33s
rook-ceph-mgr-a-86c478bd7-jmvcg                1/1     Running     0               67s
rook-ceph-mon-a-664997c65c-tb4xg               1/1     Running     0               91s
rook-ceph-operator-59dcf6d55b-2qvk5            1/1     Running     0               12m
rook-ceph-osd-0-57bb7f4f89-92xbw               1/1     Running     0               35s
rook-ceph-osd-prepare-minikube-m7vxx           0/1     Completed   0               46s
```

If the rook-ceph-mon, rook-ceph-mgr, or rook-ceph-osd pods are not created, please refer to the Ceph common issues for more details and potential solutions.

To verify that the cluster is in a healthy state, connect to the Rook Toolbox.
```
kubectl create -f toolbox.yaml
```
Wait for the toolbox pod to download its container and get to the running state:
```
kubectl -n rook-ceph rollout status deploy/rook-ceph-tools
```
Once the rook-ceph-tools pod is running, you can connect to it with:
```
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- bash
```
Run the ceph status command and check

All mons should be in quorum

A mgr should be active

At least 1 OSDs should be up and in

If the health is not HEALTH_OK, the warnings or errors should be investigated
```
bash-5.1$ ceph -s
  cluster:
    id:     f89dd5e5-e2bb-44e8-8969-659f0fc9dc55
    health: HEALTH_OK

  services:
    mon: 1 daemons, quorum a (age 7m)
    mgr: a(active, since 5m)
    osd: 1 osds: 1 up (since 6m), 1 in (since 6m)

  data:
    pools:   1 pools, 1 pgs
    objects: 2 objects, 449 KiB
    usage:   27 MiB used, 10 GiB / 10 GiB avail
    pgs:     1 active+clean
```
If the cluster is not healthy, please refer to the https://rook.io/docs/rook/latest/Troubleshooting/ceph-common-issues/ for potential solutions.

References:

https://rook.io/docs/rook/latest/Getting-Started/quickstart/
