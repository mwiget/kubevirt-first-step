# kubevirt on xeon

## Install kubeadm and kubectl

Overall instructions followed from
https://kubernetes.io/docs/setup/independent/install-kubeadm/

In my setup I'm using Ubuntu 18.04.2 LTS:

```
$ lsb_release -a
No LSB modules are available.
Distributor ID:	Ubuntu
Description:	Ubuntu 18.04.2 LTS
Release:	18.04
Codename:	bionic
```

### Installing kubelet, kubeadm and kubectl:

```
sudo apt-get update && sudo apt-get install -y apt-transport-https curl
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

### Disable swap

kubernetes refuses to operate on a node with swap enabled. I disabled this by commenting out swap in /etc/fstab.

## Create kube cluster with kubeadm

First, a few pre-requisites must be satisfied, like disable swap, add socat and 

```
mwiget@xeon:~$ sudo kubeadm init --pod-network-cidr=10.244.0.0/16
[init] Using Kubernetes version: v1.13.4
[preflight] Running pre-flight checks
	[WARNING SystemVerification]: this Docker version is not on the list of validated versions: 18.09.3. Latest validated version: 18.06
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Activating the kubelet service
[certs] Using certificateDir folder "/etc/kubernetes/pki"
[certs] Generating "front-proxy-ca" certificate and key
[certs] Generating "front-proxy-client" certificate and key
[certs] Generating "etcd/ca" certificate and key
[certs] Generating "etcd/peer" certificate and key
[certs] etcd/peer serving cert is signed for DNS names [xeon localhost] and IPs [192.168.0.39 127.0.0.1 ::1]
[certs] Generating "etcd/healthcheck-client" certificate and key
[certs] Generating "apiserver-etcd-client" certificate and key
[certs] Generating "etcd/server" certificate and key
[certs] etcd/server serving cert is signed for DNS names [xeon localhost] and IPs [192.168.0.39 127.0.0.1 ::1]
[certs] Generating "ca" certificate and key
[certs] Generating "apiserver-kubelet-client" certificate and key
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [xeon kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] and IPs [10.96.0.1 192.168.0.39]
[certs] Generating "sa" key and public key
[kubeconfig] Using kubeconfig folder "/etc/kubernetes"
[kubeconfig] Writing "admin.conf" kubeconfig file
[kubeconfig] Writing "kubelet.conf" kubeconfig file
[kubeconfig] Writing "controller-manager.conf" kubeconfig file
[kubeconfig] Writing "scheduler.conf" kubeconfig file
[control-plane] Using manifest folder "/etc/kubernetes/manifests"
[control-plane] Creating static Pod manifest for "kube-apiserver"
[control-plane] Creating static Pod manifest for "kube-controller-manager"
[control-plane] Creating static Pod manifest for "kube-scheduler"
[etcd] Creating static Pod manifest for local etcd in "/etc/kubernetes/manifests"
[wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests". This can take up to 4m0s
[apiclient] All control plane components are healthy after 19.502838 seconds
[uploadconfig] storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config-1.13" in namespace kube-system with the configuration for the kubelets in the cluster
[patchnode] Uploading the CRI Socket information "/var/run/dockershim.sock" to the Node API object "xeon" as an annotation
[mark-control-plane] Marking the node xeon as control-plane by adding the label "node-role.kubernetes.io/master=''"
[mark-control-plane] Marking the node xeon as control-plane by adding the taints [node-role.kubernetes.io/master:NoSchedule]

[bootstrap-token] Using token: 8xr9w1.egsp4idctq3uikow
[bootstrap-token] Configuring bootstrap tokens, cluster-info ConfigMap, RBAC Roles
[bootstraptoken] configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstraptoken] configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstraptoken] configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[bootstraptoken] creating the "cluster-info" ConfigMap in the "kube-public" namespace
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

Your Kubernetes master has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of machines by running the following on each node
as root:

  kubeadm join 192.168.0.39:6443 --token 8xr9w1.egsp4idctq3uikow --discovery-token-ca-cert-hash sha256:d271bcb2bcc25a7f87648ae505c71adc99260cdbe55e1f3fe942041c83cf9ec9
```

Create/update ~/.kube in order to use kubectl against the created cluster according to the instruction given by 'kubeadm init' above. I've added a step to save a previous config.

```
mkdir -p $HOME/.kube
mv $HOME/.kube/config $HOME/.kube/config.old
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Checking the status of the cluster, it reports NotReady, because we haven't installed yet any pod network add-on:

```
mwiget@xeon:~$ kubectl get nodes
NAME   STATUS     ROLES    AGE    VERSION
xeon   NotReady   master   3m5s   v1.13.4
mwiget@xeon:~$
```

## Install pod networking flannel

Start by enabling bridge-nf-call-iptables:

```
sudo sysctl net.bridge.bridge-nf-call-iptables=1
```

Make it permanent by adding it to /etc/sysctl.conf

```
mwiget@xeon:~$ tail -2 /etc/sysctl.conf

net.bridge.bridge-nf-call-iptables=1
```

### Launching flannel:

```
$ kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/a70459be0084506e4ec919aa1c114638878db11b/Documentation/kube-flannel.yml
clusterrole.rbac.authorization.k8s.io/flannel created
clusterrolebinding.rbac.authorization.k8s.io/flannel created
serviceaccount/flannel created
configmap/kube-flannel-cfg created
daemonset.extensions/kube-flannel-ds-amd64 created
daemonset.extensions/kube-flannel-ds-arm64 created
daemonset.extensions/kube-flannel-ds-arm created
daemonset.extensions/kube-flannel-ds-ppc64le created
daemonset.extensions/kube-flannel-ds-s390x created
```

### Checking the cluster again

```
$ kubectl get nodes
NAME   STATUS   ROLES    AGE     VERSION
xeon   Ready    master   8m20s   v1.13.4
```

Checking all info's up to this point:

```
$ kubectl get all --all-namespaces
NAMESPACE     NAME                               READY   STATUS    RESTARTS   AGE
kube-system   pod/coredns-86c58d9df4-k6n5t       1/1     Running   0          18m
kube-system   pod/coredns-86c58d9df4-ws4qs       1/1     Running   0          18m
kube-system   pod/etcd-xeon                      1/1     Running   0          17m
kube-system   pod/kube-apiserver-xeon            1/1     Running   0          17m
kube-system   pod/kube-controller-manager-xeon   1/1     Running   0          17m
kube-system   pod/kube-flannel-ds-amd64-v9j6w    1/1     Running   0          11m
kube-system   pod/kube-proxy-sg44j               1/1     Running   0          18m
kube-system   pod/kube-scheduler-xeon            1/1     Running   0          17m

NAMESPACE     NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)         AGE
default       service/kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP         18m
kube-system   service/kube-dns     ClusterIP   10.96.0.10   <none>        53/UDP,53/TCP   18m

NAMESPACE     NAME                                     DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR                     AGE
kube-system   daemonset.apps/kube-flannel-ds-amd64     1         1         1       1            1           beta.kubernetes.io/arch=amd64     11m
kube-system   daemonset.apps/kube-flannel-ds-arm       0         0         0       0            0           beta.kubernetes.io/arch=arm       11m
kube-system   daemonset.apps/kube-flannel-ds-arm64     0         0         0       0            0           beta.kubernetes.io/arch=arm64     11m
kube-system   daemonset.apps/kube-flannel-ds-ppc64le   0         0         0       0            0           beta.kubernetes.io/arch=ppc64le   11m
kube-system   daemonset.apps/kube-flannel-ds-s390x     0         0         0       0            0           beta.kubernetes.io/arch=s390x     11m
kube-system   daemonset.apps/kube-proxy                1         1         1       1            1           <none>                            18m

NAMESPACE     NAME                      READY   UP-TO-DATE   AVAILABLE   AGE
kube-system   deployment.apps/coredns   2/2     2            2           18m

NAMESPACE     NAME                                 DESIRED   CURRENT   READY   AGE
kube-system   replicaset.apps/coredns-86c58d9df4   2         2         2       18m
```

## Allow pod scheduling on master node

I have currently just a single master node, hence need to allow pod scheduling on the master node too. This is done via

```
$ kubectl taint nodes --all node-role.kubernetes.io/master-
node/xeon untainted
```

## Installing kubevirt

Instructions taken from https://kubevirt.io/user-guide/docs/latest/administration/intro.html

Installing libvirt-clients (to run virt-host-validate):

```
$ sudo apt install libvirt-clients
```

```
mwiget@xeon:~$ virt-host-validate qemu
  QEMU: Checking for hardware virtualization                                 : PASS
  QEMU: Checking if device /dev/kvm exists                                   : PASS
  QEMU: Checking if device /dev/kvm is accessible                            : FAIL (Check /dev/kvm is world writable or you are in a group that is allowed to access it)
  QEMU: Checking if device /dev/vhost-net exists                             : PASS
  QEMU: Checking if device /dev/net/tun exists                               : PASS
  QEMU: Checking for cgroup 'memory' controller support                      : PASS
  QEMU: Checking for cgroup 'memory' controller mount-point                  : PASS
  QEMU: Checking for cgroup 'cpu' controller support                         : PASS
  QEMU: Checking for cgroup 'cpu' controller mount-point                     : PASS
  QEMU: Checking for cgroup 'cpuacct' controller support                     : PASS
  QEMU: Checking for cgroup 'cpuacct' controller mount-point                 : PASS
  QEMU: Checking for cgroup 'cpuset' controller support                      : PASS
  QEMU: Checking for cgroup 'cpuset' controller mount-point                  : PASS
  QEMU: Checking for cgroup 'devices' controller support                     : PASS
  QEMU: Checking for cgroup 'devices' controller mount-point                 : PASS
  QEMU: Checking for cgroup 'blkio' controller support                       : PASS
  QEMU: Checking for cgroup 'blkio' controller mount-point                   : PASS
  QEMU: Checking for device assignment IOMMU support                         : PASS
  QEMU: Checking if IOMMU is enabled by kernel                               : WARN (IOMMU appears to be disabled in kernel. Add intel_iommu=on to kernel cmdline arguments)
```

Install kubeVirt v0.14.0:

```
$ kubectl apply -f https://github.com/kubevirt/kubevirt/releases/download/v0.14.0/kubevirt.yaml
namespace/kubevirt created
service/kubevirt-prometheus-metrics created
clusterrole.rbac.authorization.k8s.io/kubevirt.io:default created
clusterrolebinding.rbac.authorization.k8s.io/kubevirt.io:default created
clusterrole.rbac.authorization.k8s.io/kubevirt.io:admin created
clusterrole.rbac.authorization.k8s.io/kubevirt.io:edit created
clusterrole.rbac.authorization.k8s.io/kubevirt.io:view created
serviceaccount/kubevirt-privileged created
clusterrolebinding.rbac.authorization.k8s.io/kubevirt-privileged-cluster-admin created
serviceaccount/kubevirt-apiserver created
clusterrole.rbac.authorization.k8s.io/kubevirt-apiserver created
clusterrolebinding.rbac.authorization.k8s.io/kubevirt-apiserver created
clusterrolebinding.rbac.authorization.k8s.io/kubevirt-apiserver-auth-delegator created
role.rbac.authorization.k8s.io/kubevirt-apiserver created
rolebinding.rbac.authorization.k8s.io/kubevirt-apiserver created
serviceaccount/kubevirt-controller created
clusterrole.rbac.authorization.k8s.io/kubevirt-controller created
clusterrolebinding.rbac.authorization.k8s.io/kubevirt-controller created
service/virt-api created
deployment.apps/virt-api created
deployment.apps/virt-controller created
daemonset.apps/virt-handler created
customresourcedefinition.apiextensions.k8s.io/virtualmachineinstances.kubevirt.io created
customresourcedefinition.apiextensions.k8s.io/virtualmachineinstancereplicasets.kubevirt.io created
customresourcedefinition.apiextensions.k8s.io/virtualmachineinstancepresets.kubevirt.io created
customresourcedefinition.apiextensions.k8s.io/virtualmachines.kubevirt.io created
customresourcedefinition.apiextensions.k8s.io/virtualmachineinstancemigrations.kubevirt.io created
```

Checking again

```
$ kubectl get all --all-namespaces
NAMESPACE     NAME                                  READY   STATUS    RESTARTS   AGE
kube-system   pod/coredns-86c58d9df4-k6n5t          1/1     Running   0          25m
kube-system   pod/coredns-86c58d9df4-ws4qs          1/1     Running   0          25m
kube-system   pod/etcd-xeon                         1/1     Running   0          24m
kube-system   pod/kube-apiserver-xeon               1/1     Running   0          24m
kube-system   pod/kube-controller-manager-xeon      1/1     Running   0          24m
kube-system   pod/kube-flannel-ds-amd64-v9j6w       1/1     Running   0          17m
kube-system   pod/kube-proxy-sg44j                  1/1     Running   0          25m
kube-system   pod/kube-scheduler-xeon               1/1     Running   0          24m
kubevirt      pod/virt-api-6d6f8846d6-9jl5b         1/1     Running   0          38s
kubevirt      pod/virt-api-6d6f8846d6-gbqb7         1/1     Running   0          38s
kubevirt      pod/virt-controller-5cfc5b66c-8fxj5   0/1     Running   0          38s
kubevirt      pod/virt-controller-5cfc5b66c-hwg9b   1/1     Running   0          38s
kubevirt      pod/virt-handler-qs9qj                1/1     Running   0          38s

NAMESPACE     NAME                                  TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)         AGE
default       service/kubernetes                    ClusterIP   10.96.0.1       <none>        443/TCP         25m
kube-system   service/kube-dns                      ClusterIP   10.96.0.10      <none>        53/UDP,53/TCP   25m
kubevirt      service/kubevirt-prometheus-metrics   ClusterIP   10.104.65.192   <none>        443/TCP         38s
kubevirt      service/virt-api                      ClusterIP   10.102.242.33   <none>        443/TCP         38s

NAMESPACE     NAME                                     DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR                     AGE
kube-system   daemonset.apps/kube-flannel-ds-amd64     1         1         1       1            1           beta.kubernetes.io/arch=amd64     17m
kube-system   daemonset.apps/kube-flannel-ds-arm       0         0         0       0            0           beta.kubernetes.io/arch=arm       17m
kube-system   daemonset.apps/kube-flannel-ds-arm64     0         0         0       0            0           beta.kubernetes.io/arch=arm64     17m
kube-system   daemonset.apps/kube-flannel-ds-ppc64le   0         0         0       0            0           beta.kubernetes.io/arch=ppc64le   17m
kube-system   daemonset.apps/kube-flannel-ds-s390x     0         0         0       0            0           beta.kubernetes.io/arch=s390x     17m
kube-system   daemonset.apps/kube-proxy                1         1         1       1            1           <none>                            25m
kubevirt      daemonset.apps/virt-handler              1         1         1       1            1           <none>                            38s

NAMESPACE     NAME                              READY   UP-TO-DATE   AVAILABLE   AGE
kube-system   deployment.apps/coredns           2/2     2            2           25m
kubevirt      deployment.apps/virt-api          2/2     2            2           38s
kubevirt      deployment.apps/virt-controller   1/2     2            1           38s

NAMESPACE     NAME                                        DESIRED   CURRENT   READY   AGE
kube-system   replicaset.apps/coredns-86c58d9df4          2         2         2       25m
kubevirt      replicaset.apps/virt-api-6d6f8846d6         2         2         2       38s
kubevirt      replicaset.apps/virt-controller-5cfc5b66c   2         2         1       38s
```

Or just kubevirt:

```
$ kubectl get all -n kubevirt
NAME                                  READY   STATUS    RESTARTS   AGE
pod/virt-api-6d6f8846d6-9jl5b         1/1     Running   0          73s
pod/virt-api-6d6f8846d6-gbqb7         1/1     Running   0          73s
pod/virt-controller-5cfc5b66c-8fxj5   1/1     Running   0          73s
pod/virt-controller-5cfc5b66c-hwg9b   1/1     Running   0          73s
pod/virt-handler-qs9qj                1/1     Running   0          73s

NAME                                  TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
service/kubevirt-prometheus-metrics   ClusterIP   10.104.65.192   <none>        443/TCP   73s
service/virt-api                      ClusterIP   10.102.242.33   <none>        443/TCP   73s

NAME                          DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
daemonset.apps/virt-handler   1         1         1       1            1           <none>          73s

NAME                              READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/virt-api          2/2     2            2           73s
deployment.apps/virt-controller   2/2     2            2           73s

NAME                                        DESIRED   CURRENT   READY   AGE
replicaset.apps/virt-api-6d6f8846d6         2         2         2       73s
replicaset.apps/virt-controller-5cfc5b66c   2         2         2       73s
```

### Install virtctl

```
$ sudo wget -O /usr/local/bin/virtctl https://github.com/kubevirt/kubevirt/releases/download/v0.14.0/virtctl-v0.14.0-linux-amd64
$ sudo chmod +rx /usr/local/bin/virtctl
```

Done!! Lets launch now a container and a VM.

## Launch alpine container

```
$ cat alpine.yml
apiVersion: v1
kind: Pod
metadata:
  name: samplepod
spec:
  containers:
  - name: samplepod
    command: ["/bin/ash", "-c", "sleep 2000000000000"]
    image: alpine
```

$ kubectl apply -f alpine.yml
pod/samplepod created

$ kubectl get pods
NAME        READY   STATUS    RESTARTS   AGE
samplepod   1/1     Running   0          17s
```

Check its IP address 

```
$ kubectl exec -ti samplepod ifconfig eth0
eth0      Link encap:Ethernet  HWaddr 0A:58:0A:F4:00:09
          inet addr:10.244.0.9  Bcast:0.0.0.0  Mask:255.255.255.0
          UP BROADCAST RUNNING MULTICAST  MTU:1450  Metric:1
          RX packets:18 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:1160 (1.1 KiB)  TX bytes:0 (0.0 B)

```

Or launch a shell in the samplepod alpine container for interactive use:

```
 kubectl exec -ti samplepod sh
/ # ifconfig
eth0      Link encap:Ethernet  HWaddr 0A:58:0A:F4:00:09
          inet addr:10.244.0.9  Bcast:0.0.0.0  Mask:255.255.255.0
          UP BROADCAST RUNNING MULTICAST  MTU:1450  Metric:1
          RX packets:16 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:1020 (1020.0 B)  TX bytes:0 (0.0 B)

lo        Link encap:Local Loopback
          inet addr:127.0.0.1  Mask:255.0.0.0
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)
/ # exit
command terminated with exit code 127
$
```

## Launch a VM

```
$ cat fedora-vm.yml
apiVersion: kubevirt.io/v1alpha3
kind: VirtualMachineInstance
metadata:
  name: testvmi-nocloud
spec:
  terminationGracePeriodSeconds: 30
  domain:
    resources:
      requests:
        memory: 1024M
    devices:
      disks:
      - name: containerdisk
        disk:
          bus: virtio
      - name: emptydisk
        disk:
          bus: virtio
      - disk:
          bus: virtio
        name: cloudinitdisk
  volumes:
  - name: containerdisk
    containerDisk:
      image: kubevirt/fedora-cloud-container-disk-demo:latest
  - name: emptydisk
    emptyDisk:
      capacity: "2Gi"
  - name: cloudinitdisk
    cloudInitNoCloud:
      userData: |-
        #cloud-config
        password: fedora
        chpasswd: { expire: False }
```

$ kubectl apply -f fedora-vm.yml
virtualmachineinstance.kubevirt.io/testvmi-nocloud created
```

The very first time, this will take some time, even minutes. It first pulls the fedora demo disk, then pulls the virt-launcher. You can check progress using 'kubectl describe pods':

```
$ kubectl get pods
NAME                                  READY   STATUS    RESTARTS   AGE
samplepod                             1/1     Running   0          3m35s
virt-launcher-testvmi-nocloud-7l8w4   0/2     Running   0          64s
```

```
$ kubectl describe pods virt-launcher-testvmi-nocloud-7l8w4
Name:               virt-launcher-testvmi-nocloud-7l8w4
Namespace:          default
Priority:           0
PriorityClassName:  <none>
Node:               xeon/192.168.0.39
Start Time:         Mon, 04 Mar 2019 10:21:24 +0000
Labels:             kubevirt.io=virt-launcher
                    kubevirt.io/created-by=43953d5c-3e67-11e9-8bea-901b0ea56da4
Annotations:        kubevirt.io/domain: testvmi-nocloud
Status:             Running
IP:                 10.244.0.10
Controlled By:      VirtualMachineInstance/testvmi-nocloud
Containers:
  volumecontainerdisk:
    Container ID:  docker://3e85ad38eac00f6eb58b6e762185933ae338656a003fdf2cb1cd1e5653ef33e3
    Image:         kubevirt/fedora-cloud-container-disk-demo:latest
    Image ID:      docker-pullable://kubevirt/fedora-cloud-container-disk-demo@sha256:d13c849705bb4e6776631fb6471d7b92b06870fea984011432613b00b4f03063
    Port:          <none>
    Host Port:     <none>
    Command:
      /entry-point.sh
    State:          Running
      Started:      Mon, 04 Mar 2019 10:21:54 +0000
    Ready:          True
    Restart Count:  0
    Readiness:      exec [cat /tmp/healthy] delay=2s timeout=5s period=5s #success=2 #failure=5
    Environment:
      COPY_PATH:  /var/run/kubevirt-ephemeral-disks/container-disk-data/default/testvmi-nocloud/disk_containerdisk/disk-image
    Mounts:
      /var/run/kubevirt-ephemeral-disks from ephemeral-disks (rw)
  compute:
    Container ID:  docker://53042df6b1a727fba7bdc1d4f3a7104d2788be4c4cec7e3815aabb0c16809510
    Image:         docker.io/kubevirt/virt-launcher:v0.14.0
    Image ID:      docker-pullable://kubevirt/virt-launcher@sha256:4a30833b31d0e21236722512fba35ef064a8d779dc9762948fef57610c933953
    Port:          <none>
    Host Port:     <none>
    Command:
      /usr/bin/virt-launcher
      --qemu-timeout
      5m
      --name
      testvmi-nocloud
      --uid
      43953d5c-3e67-11e9-8bea-901b0ea56da4
      --namespace
      default
      --kubevirt-share-dir
      /var/run/kubevirt
      --ephemeral-disk-dir
      /var/run/kubevirt-ephemeral-disks
      --readiness-file
      /var/run/kubevirt-infra/healthy
      --grace-period-seconds
      45
      --hook-sidecars
      0
      --less-pvc-space-toleration
      10
    State:          Running
      Started:      Mon, 04 Mar 2019 10:22:26 +0000
    Ready:          True
    Restart Count:  0
    Limits:
      devices.kubevirt.io/kvm:        1
      devices.kubevirt.io/tun:        1
      devices.kubevirt.io/vhost-net:  1
    Requests:
      devices.kubevirt.io/kvm:        1
      devices.kubevirt.io/tun:        1
      devices.kubevirt.io/vhost-net:  1
      memory:                         1123554432
    Readiness:                        exec [cat /var/run/kubevirt-infra/healthy] delay=2s timeout=5s period=2s #success=1 #failure=5
    Environment:                      <none>
    Mounts:
      /var/run/kubevirt from virt-share-dir (rw)
      /var/run/kubevirt-ephemeral-disks from ephemeral-disks (rw)
      /var/run/kubevirt-infra from infra-ready-mount (rw)
      /var/run/libvirt from libvirt-runtime (rw)
Conditions:
  Type              Status
  Initialized       True
  Ready             True
  ContainersReady   True
  PodScheduled      True
Volumes:
  infra-ready-mount:
    Type:    EmptyDir (a temporary directory that shares a pod's lifetime)
    Medium:
  virt-share-dir:
    Type:          HostPath (bare host directory volume)
    Path:          /var/run/kubevirt
    HostPathType:
  libvirt-runtime:
    Type:    EmptyDir (a temporary directory that shares a pod's lifetime)
    Medium:
  ephemeral-disks:
    Type:        EmptyDir (a temporary directory that shares a pod's lifetime)
    Medium:
QoS Class:       Burstable
Node-Selectors:  kubevirt.io/schedulable=true
Tolerations:     node.kubernetes.io/not-ready:NoExecute for 300s
                 node.kubernetes.io/unreachable:NoExecute for 300s
Events:
  Type     Reason     Age                From               Message
  ----     ------     ----               ----               -------
  Normal   Scheduled  2m25s              default-scheduler  Successfully assigned default/virt-launcher-testvmi-nocloud-7l8w4 to xeon
  Normal   Pulling    2m23s              kubelet, xeon      pulling image "kubevirt/fedora-cloud-container-disk-demo:latest"
  Normal   Pulled     116s               kubelet, xeon      Successfully pulled image "kubevirt/fedora-cloud-container-disk-demo:latest"
  Normal   Created    115s               kubelet, xeon      Created container
  Normal   Started    115s               kubelet, xeon      Started container
  Normal   Pulling    115s               kubelet, xeon      pulling image "docker.io/kubevirt/virt-launcher:v0.14.0"
  Normal   Pulled     84s                kubelet, xeon      Successfully pulled image "docker.io/kubevirt/virt-launcher:v0.14.0"
  Normal   Created    83s                kubelet, xeon      Created container
  Normal   Started    83s                kubelet, xeon      Started container
  Warning  Unhealthy  73s (x4 over 79s)  kubelet, xeon      Readiness probe failed: cat: /var/run/kubevirt-infra/healthy: No such file or directory
```

Listing virtual machines

```
$ kubectl get vmis
$ kubectl get vmis
NAME              AGE   PHASE     IP            NODENAME
testvmi-nocloud   4m    Running   10.244.0.10   xeon
```

Serial console access (username password for this example is fedora/fedora). Test connectivity to the alpine container:

```
$ virtctl console testvmi-nocloud
Successfully connected to testvmi-nocloud console. The escape sequence is ^]

Fedora 29 (Cloud Edition)
Kernel 4.18.16-300.fc29.x86_64 on an x86_64 (ttyS0)

testvmi-nocloud login:
Fedora 29 (Cloud Edition)
Kernel 4.18.16-300.fc29.x86_64 on an x86_64 (ttyS0)

testvmi-nocloud login: fedora
Password:
[fedora@testvmi-nocloud ~]$
[fedora@testvmi-nocloud ~]$ ping 10.244.0.9
PING 10.244.0.9 (10.244.0.9) 56(84) bytes of data.
64 bytes from 10.244.0.9: icmp_seq=1 ttl=64 time=0.397 ms
64 bytes from 10.244.0.9: icmp_seq=2 ttl=64 time=0.218 ms

--- 10.244.0.9 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 2ms
rtt min/avg/max/mdev = 0.218/0.307/0.397/0.091 ms
[fedora@testvmi-nocloud ~]$
```


