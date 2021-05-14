## OpenShift Virtualisation (KubeVirt) on OpenShift 4.5 with Contrail cluster


[KubeVirt](https://kubevirt.io/) is the upsream project for OpenShift Virtualisation. OpenShift Virtualisation is an operator that enables developers to create and add virtualised applications to their projects from OperatorHub in the same way they would for a containerised application. The resulting virtual machines will run in parallel on the same Red Hat OpenShift nodes as traditional application containers.

You will go through the steps to install OpenShift Virtualisation 2.4 on OpenShift 4.5 with Contrail.

### Prerequisites
A Red Hat OpenShift 4.5 with Contrail 2008 or later, running on BMS or VMs supporting nested virtualisation. For installation procedure please check [here](https://github.com/ovaleanu/openshift4.x/blob/master/docs/ocp4-contrail-vm-bms.md).

```
$ oc get pods -n contrail
NAME                                          READY   STATUS      RESTARTS   AGE
cassandra1-cassandra-statefulset-0            1/1     Running     0          27h
cassandra1-cassandra-statefulset-1            1/1     Running     0          27h
cassandra1-cassandra-statefulset-2            1/1     Running     0          27h
cnimasternodes-contrailcni-job-gjxjv          0/1     Completed   0          27h
cnimasternodes-contrailcni-job-r9z4v          0/1     Completed   0          27h
cnimasternodes-contrailcni-job-zgvz7          0/1     Completed   0          27h
cniworkernodes-contrailcni-job-mq9rb          0/1     Completed   0          27h
cniworkernodes-contrailcni-job-sr7v8          0/1     Completed   0          27h
config1-config-statefulset-0                  10/10   Running     1          27h
config1-config-statefulset-1                  10/10   Running     0          27h
config1-config-statefulset-2                  10/10   Running     2          27h
contrail-operator-76488c8cb9-7k5lc            1/1     Running     0          27h
contrail-operator-76488c8cb9-fgllk            1/1     Running     1          27h
contrail-operator-76488c8cb9-fmp4c            1/1     Running     0          27h
control1-control-statefulset-0                4/4     Running     0          27h
control1-control-statefulset-1                4/4     Running     0          27h
control1-control-statefulset-2                4/4     Running     0          27h
kubemanager1-kubemanager-statefulset-0        2/2     Running     1          27h
kubemanager1-kubemanager-statefulset-1        2/2     Running     0          27h
kubemanager1-kubemanager-statefulset-2        2/2     Running     1          27h
provmanager1-provisionmanager-statefulset-0   1/1     Running     1          27h
rabbitmq1-rabbitmq-statefulset-0              1/1     Running     0          27h
rabbitmq1-rabbitmq-statefulset-1              1/1     Running     0          27h
rabbitmq1-rabbitmq-statefulset-2              1/1     Running     0          27h
vroutermasternodes-vrouter-daemonset-7kxmx    1/1     Running     0          27h
vroutermasternodes-vrouter-daemonset-9l4wt    1/1     Running     0          27h
vroutermasternodes-vrouter-daemonset-xlzr7    1/1     Running     0          27h
vrouterworkernodes-vrouter-daemonset-qjgrs    1/1     Running     0          27h
vrouterworkernodes-vrouter-daemonset-zm6ml    1/1     Running     0          27h
webui1-webui-statefulset-0                    3/3     Running     0          27h
webui1-webui-statefulset-1                    3/3     Running     0          27h
webui1-webui-statefulset-2                    3/3     Running     0          27h
zookeeper1-zookeeper-statefulset-0            1/1     Running     0          27h
zookeeper1-zookeeper-statefulset-1            1/1     Running     0          27h
zookeeper1-zookeeper-statefulset-2            1/1     Running     0          27h
```

### Installing OpenShift Virtualisation operator

Reference for this instalaltion procedure is OpenShift Virtualisation official [documentation](https://docs.openshift.com/container-platform/4.5/virt/install/installing-virt-cli.html).

Login as a user with `cluster-admin` privileges
Create a yaml file containing the following manifest

```
$ cat <<EOF > cnv.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: openshift-cnv
---
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: kubevirt-hyperconverged-group
  namespace: openshift-cnv
spec:
  targetNamespaces:
    - openshift-cnv
---
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: hco-operatorhub
  namespace: openshift-cnv
spec:
  source: redhat-operators
  sourceNamespace: openshift-marketplace
  name: kubevirt-hyperconverged
  startingCSV: kubevirt-hyperconverged-operator.v2.4.1
  channel: "2.4"
EOF
```
This yaml file will create the required `Namespace`, `OperatorGroup` and `Subscription` for the OpenShift Virtualisation.
```
$ oc apply -f cnv.yaml
```

Now you can deploy the OpenShift Virtualisation operator. Create a yaml file with the following content
```
$ cat <<EOF > kubevirt-hyperconverged.yaml
apiVersion: hco.kubevirt.io/v1alpha1
kind: HyperConverged
metadata:
  name: kubevirt-hyperconverged
  namespace: openshift-cnv
spec:
  BareMetalPlatform: true
EOF
```
```
$ oc apply -f kubevirt-hyperconverged.yaml
```

Check if all the pods are Running in `openshift-cnv` namespace
```
$ oc get pods -n openshift-cnv
NAME                                                  READY   STATUS    RESTARTS   AGE
bridge-marker-5tndk                                   1/1     Running   0          22h
bridge-marker-d2gff                                   1/1     Running   0          22h
bridge-marker-d8cgd                                   1/1     Running   0          22h
bridge-marker-r6glh                                   1/1     Running   0          22h
bridge-marker-rt5lb                                   1/1     Running   0          22h
cdi-apiserver-7c4566c98c-z89qz                        1/1     Running   0          22h
cdi-deployment-79fdcfdccb-xmphs                       1/1     Running   0          22h
cdi-operator-7785b655bb-7q5k6                         1/1     Running   0          22h
cdi-uploadproxy-5d4cc54b4c-g2ztz                      1/1     Running   0          22h
cluster-network-addons-operator-67d7f76cbd-8kl6l      1/1     Running   0          22h
hco-operator-854f5988c8-v2qbm                         1/1     Running   0          22h
hostpath-provisioner-operator-595b955c9d-zxngg        1/1     Running   0          22h
kube-cni-linux-bridge-plugin-5w67f                    1/1     Running   0          22h
kube-cni-linux-bridge-plugin-kjm8b                    1/1     Running   0          22h
kube-cni-linux-bridge-plugin-rgrn8                    1/1     Running   0          22h
kube-cni-linux-bridge-plugin-s6xkz                    1/1     Running   0          22h
kube-cni-linux-bridge-plugin-ssw29                    1/1     Running   0          22h
kubemacpool-mac-controller-manager-6f9c447bbd-phd5n   1/1     Running   0          22h
kubevirt-node-labeller-297nh                          1/1     Running   0          22h
kubevirt-node-labeller-cbjnl                          1/1     Running   0          22h
kubevirt-ssp-operator-75d54556b9-zq2kb                1/1     Running   0          22h
nmstate-handler-9prp8                                 1/1     Running   1          22h
nmstate-handler-dk4ht                                 1/1     Running   0          22h
nmstate-handler-fzjmk                                 1/1     Running   0          22h
nmstate-handler-rqwmq                                 1/1     Running   1          22h
nmstate-handler-spx7w                                 1/1     Running   0          22h
node-maintenance-operator-6486bcbfcd-rhn4l            1/1     Running   0          22h
ovs-cni-amd64-4t9ld                                   1/1     Running   0          22h
ovs-cni-amd64-5mdmq                                   1/1     Running   0          22h
ovs-cni-amd64-bz5d9                                   1/1     Running   0          22h
ovs-cni-amd64-h9j6j                                   1/1     Running   0          22h
ovs-cni-amd64-k8hwf                                   1/1     Running   0          22h
virt-api-7686f978db-ngwn2                             1/1     Running   0          22h
virt-api-7686f978db-nkl4d                             1/1     Running   0          22h
virt-controller-7d567db8c6-bbdjk                      1/1     Running   0          22h
virt-controller-7d567db8c6-n2vgk                      1/1     Running   0          22h
virt-handler-lkpsq                                    1/1     Running   0          5h30m
virt-handler-vfcbd                                    1/1     Running   0          5h30m
virt-operator-7995d994c4-9bxw9                        1/1     Running   0          22h
virt-operator-7995d994c4-q8wnv                        1/1     Running   0          22h
virt-template-validator-5d9bbfbcc7-g2zph              1/1     Running   0          22h
virt-template-validator-5d9bbfbcc7-lhhrw              1/1     Running   0          22h
vm-import-controller-58469cdfcf-kwkgb                 1/1     Running   0          22h
vm-import-operator-9495bd74c-dkw2h                    1/1     Running   0          22h
```
and `PHASE` of the ClusterServiceVersion (CSV) is Succeded.

```
$ oc get csv -n openshift-cnv
NAME                                      DISPLAY                    VERSION   REPLACES   PHASE
kubevirt-hyperconverged-operator.v2.4.1   OpenShift Virtualization   2.4.1                Succeeded
```

If you are running OpenShift Virtualisation in a nested enviroment, kubevirt-config ConfigMap must be updated to support [software emulation](https://github.com/kubevirt/kubevirt/blob/master/docs/software-emulation.md#software-emulation).

Add to kubevirt-config ConfigMap
```
data:
  debug.useEmulation: "true"
```
```
$ oc edit cm kubevirt-config -n openshift-cnv

apiVersion: v1
kind: ConfigMap
metadata:
  name: kubevirt-config
  namespace: openshift-cnv
data:
  debug.useEmulation: "true"
```
Then you need to restart `virt-handler` pods

### Creating Virtual Machines on OpenShift Virtualisation

Create namespace for the demo. I will call it `cnv-demo`

```
$ oc new-project cnv-demo
```

Using Virtual Machine Instance (VMI) custom resources you can create VMs fully integrated in OpenShift.

Create a Virtual Machine with Centos 7 using the following manifest:

```
cat <<EOF > kubevirt-centos.yaml
apiVersion: kubevirt.io/v1alpha3
kind: VirtualMachineInstance
metadata:
  labels:
    special: vmi-centos7
  name: vmi-centos7
  namespace: cnv-demo
spec:
  domain:
    devices:
      disks:
      - disk:
          bus: virtio
        name: containerdisk
      - disk:
          bus: virtio
        name: cloudinitdisk
      interfaces:
      - name: default
        bridge: {}
    resources:
      requests:
        memory: 1024M
  networks:
  - name: default
    pod: {}
  volumes:
  - containerDisk:
      image: ovaleanu/centos:latest
    name: containerdisk
  - cloudInitNoCloud:
      userData: |-
        #cloud-config
        password: centos
        ssh_pwauth: True
        chpasswd: { expire: False }
    name: cloudinitdisk
EOF

$ oc apply -f kubevirt-centos.yaml
virtualmachineinstance.kubevirt.io/vmi-centos7 created
```

Check if the pod and VirtualMachineInstance was created
```
$ oc get pods
NAME                              READY   STATUS    RESTARTS   AGE     IP              NODE                       NOMINATED NODE   READINESS GATES
virt-launcher-vmi-centos7-ttngl   2/2     Running   0          3h57m   10.254.255.90   worker0.ocp4.example.com   <none>           <none>

$ oc get vmi
NAME          AGE    PHASE     IP                 NODENAME
vmi-centos7   4h1m   Running   10.254.255.90/16   worker0.ocp4.example.com
```

### Test VM to pod connectivity

To test VM to pod connectivity, create a small Ubuntu pod

```
cat <<EOF > ubuntu.yaml
apiVersion: v1
kind: Pod
metadata:
  name: ubuntuapp
  labels:
    app: ubuntuapp
spec:
  containers:
    - name: ubuntuapp
      image: ubuntu-upstart
EOF

$ oc create -f ubuntu.yaml

$ oc get pods
NAME                              READY   STATUS    RESTARTS   AGE     IP              NODE                       NOMINATED NODE   READINESS GATES
ubuntuapp                         1/1     Running   0          3h52m   10.254.255.89   worker1.ocp4.example.com   <none>           <none>
virt-launcher-vmi-centos7-ttngl   2/2     Running   0          3h57m   10.254.255.90   worker0.ocp4.example.com   <none>           <none>
```

Create a service for Centos VM to connect with ssh through NodePort using node ip

```
cat <<EOF > kubevirt-centos-svc.yaml
apiVersion: v1
kind: Service
metadata:
  name: vmi-centos-ssh-svc
  namespace: cnv-demo
spec:
  ports:
  - name: centos-ssh-svc
    nodePort: 30000
    port: 27017
    protocol: TCP
    targetPort: 22
  selector:
    special: vmi-centos7
  type: NodePort
EOF

$ oc apply -f kubevirt-centos-svc.yaml

$ oc get svc
NAME                 TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)           AGE
vmi-centos-ssh-svc   NodePort   172.30.115.77    <none>        27017:30000/TCP   4h2m
```

Connect to Centos VM with ssh via service NodePort using worker node IP address

```
$ ssh centos@192.168.7.11 -p 30000
The authenticity of host '[192.168.7.11]:30000 ([192.168.7.11]:30000)' can't be established.
ECDSA key fingerprint is SHA256:kk+9dbMqzpXDoPucnxiYozBgDt75IBSNS8Y4hUcEEmI.
ECDSA key fingerprint is MD5:86:b6:e9:3b:f0:55:ee:e7:fd:56:96:c3:4a:c6:fd:e0.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '[192.168.7.11]:30000' (ECDSA) to the list of known hosts.
centos@192.168.7.11's password:

[centos@vmi-centos7 ~]$ uname -sr
Linux 3.10.0-957.12.2.el7.x86_64
```
Confirm the VM has access to outside world
```
[centos@vmi-centos7 ~]$ ping www.google.com
PING www.google.com (142.250.73.196) 56(84) bytes of data.
64 bytes from iad23s87-in-f4.1e100.net (142.250.73.196): icmp_seq=1 ttl=108 time=13.1 ms
64 bytes from iad23s87-in-f4.1e100.net (142.250.73.196): icmp_seq=2 ttl=108 time=11.9 ms
^C
--- www.google.com ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1003ms
rtt min/avg/max/mdev = 11.990/12.547/13.104/0.557 ms
```

Ping the Ubuntu pod IP

```
[centos@vmi-centos7 ~]$ ping 10.254.255.89
PING 10.254.255.89 (10.254.255.89) 56(84) bytes of data.
64 bytes from 10.254.255.89: icmp_seq=1 ttl=63 time=3.83 ms
64 bytes from 10.254.255.89: icmp_seq=2 ttl=63 time=2.26 ms
^C
--- 10.254.255.89 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1003ms
rtt min/avg/max/mdev = 2.263/3.047/3.831/0.784 ms
```

### Contrail Security policy

Create a network security policy to isolate the VM in its namespace. You allow only ssh for ingress and egress only to podNetwork.

```
cat <<EOF > kubevirt-centos-netpol.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
 name: netpol
 namespace: cnv-demo
spec:
 podSelector:
   matchLabels:
    special: vmi-centos7
 policyTypes:
 - Ingress
 - Egress
 ingress:
 - from:
   ports:
   - port: 22
 egress:
 - to:
   - ipBlock:
       cidr: 10.254.255.0/16
EOF

$ oc apply -f kubevirt-centos-netpol.yaml
networkpolicy.networking.k8s.io/netpol
```

Connect to Centos VM again and ping Ubuntu pod ip. Pinging www.google.com will not work.

```
[root@helper ocp4]# ssh centos@192.168.7.11 -p 30000
centos@192.168.7.11's password:
[centos@vmi-centos7 ~]$ ping 10.254.255.89
PING 10.254.255.89 (10.254.255.89) 56(84) bytes of data.
64 bytes from 10.254.255.89: icmp_seq=1 ttl=63 time=2.58 ms
64 bytes from 10.254.255.89: icmp_seq=2 ttl=63 time=2.39 ms
^C
--- 10.254.255.89 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1003ms
rtt min/avg/max/mdev = 2.394/2.490/2.587/0.108 ms
[centos@vmi-centos7 ~]$ ping www.google.com
^C
[centos@vmi-centos7 ~]$
````

### Creating a Virtual Machine with multiple interfaces

You will create two virtual networks `neta` and `netb` in Contrail

```
$ cat <<EOF > netab.yaml
apiVersion: "k8s.cni.cncf.io/v1"
kind: NetworkAttachmentDefinition
metadata:
 name: neta
 annotations: {
   "opencontrail.org/cidr" : "10.10.10.0/24",
   "opencontrail.org/ip_fabric_snat": "true"
  }
spec:
 config: '{
   "cniVersion": "0.3.1",
   "type": "contrail-k8s-cni"
}'

---
apiVersion: "k8s.cni.cncf.io/v1"
kind: NetworkAttachmentDefinition
metadata:
 name: netb
 annotations: {
   "opencontrail.org/cidr" : "20.20.20.0/24",
   "opencontrail.org/ip_fabric_snat": "true"
  }
spec:
 config: '{
   "cniVersion": "0.3.1",
   "type": "contrail-k8s-cni"
}'
EOF

$ oc apply -f netab.yaml
```

Now you will create a Fedora VM with an interface in `neta` and another one in `netb`.
(by default all the pods with multiple interfaces will have a interface in default podNetwork as well)

Use the following manifest to create the VM

```
apiVersion: kubevirt.io/v1alpha3
kind: VirtualMachineInstance
metadata:
  labels:
    special: vmi-fedora
  name: vmi-fedora
spec:
  domain:
    devices:
      disks:
      - disk:
          bus: virtio
        name: containerdisk
      - disk:
          bus: virtio
        name: cloudinitdisk
      interfaces:
      - name: default
        bridge: {}
      - name: neta
        bridge: {}
      - name: netb
        bridge: {}
    resources:
      requests:
        memory: 1024M
  networks:
  - name: default
    pod: {}
  - name: neta
    multus:
      networkName: neta
  - name: netb
    multus:
      networkName: netb
  volumes:
  - containerDisk:
      image: kubevirt/fedora-cloud-registry-disk-demo
    name: containerdisk
  - cloudInitNoCloud:
      userData: |-
        #cloud-config
        password: fedora
        ssh_pwauth: True
        chpasswd: { expire: False }
    name: cloudinitdisk
EOF

$ oc apply -f kubevirt-fedora.yaml
```

Check if the pod and VirtualMachineInstance was created

```
$ oc get pods
NAME                              READY   STATUS    RESTARTS   AGE
ubuntuapp                         1/1     Running   0          5h11m
virt-launcher-vmi-centos7-ttngl   2/2     Running   0          5h16m
virt-launcher-vmi-fedora-czwhx    2/2     Running   0          102m

$ oc get vmi
NAME          AGE     PHASE     IP                 NODENAME
vmi-centos7   5h17m   Running   10.254.255.90/16   worker0.ocp4.example.com
vmi-fedora    103m    Running   10.254.255.88      worker1.ocp4.example.com
```

Like I did previously with Centos VM, I will create a service to connect with ssh using NodePort

```
cat <<EOF > kubevirt-fedora-svc.yaml
apiVersion: v1
kind: Service
metadata:
  name: vmi-fedora-ssh-svc
  namespace: cnv-demo
spec:
  ports:
  - name: fedora-ssh-svc
    nodePort: 31000
    port: 25025
    protocol: TCP
    targetPort: 22
  selector:
    special: vmi-fedora
  type: NodePort
EOF

$ oc apply -f kubevirt-fedora-svc.yaml
service/vmi-fedora-ssh-svc created

$ oc get svc -n cnv-demo
NAME                 TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)           AGE
vmi-centos-ssh-svc   NodePort   172.30.115.77    <none>        27017:30000/TCP   5h16m
vmi-fedora-ssh-svc   NodePort   172.30.247.145   <none>        25025:31000/TCP   98m
```


Connect to Fedora VM with ssh via service NodePort using worker node IP address, like you did previously with Centos VM. You will need to enable manually the network interfaces in the custom networks, `neta` and `netb`.

```
$ ssh fedora@192.168.7.12 -p 31000
The authenticity of host '[192.168.7.12]:31000 ([192.168.7.12]:31000)' can't be established.
ECDSA key fingerprint is SHA256:JlhysyH0XiHXszLLqu8GmuSHB4msOYWPAJjZhv5j3FM.
ECDSA key fingerprint is MD5:62:ca:0b:b9:21:c9:2b:73:db:b6:09:e2:b0:b4:81:60.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '[192.168.7.12]:31000' (ECDSA) to the list of known hosts.
fedora@192.168.7.12's password:

[fedora@vmi-fedora ~]$ uname -sr
Linux 4.13.9-300.fc27.x86_64

[fedora@vmi-fedora ~]$ cat /etc/sysconfig/network-scripts/ifcfg-eth0
# Created by cloud-init on instance boot automatically, do not edit.
#
BOOTPROTO=dhcp
DEVICE=eth0
HWADDR=02:dd:00:37:08:0d
ONBOOT=yes
TYPE=Ethernet
USERCTL=no

[fedora@vmi-fedora ~]$ cat /etc/sysconfig/network-scripts/ifcfg-eth1
# Created by cloud-init on instance boot automatically, do not edit.
#
BOOTPROTO=dhcp
DEVICE=eth1
HWADDR=02:dd:3a:e6:dc:0d
ONBOOT=yes
TYPE=Ethernet
USERCTL=no

[fedora@vmi-fedora ~]$ cat /etc/sysconfig/network-scripts/ifcfg-eth2
# Created by cloud-init on instance boot automatically, do not edit.
#
BOOTPROTO=dhcp
DEVICE=eth2
HWADDR=02:dd:71:6e:fa:0d
ONBOOT=yes
TYPE=Ethernet
USERCTL=no

$ sudo systemctl restart network

[fedora@vmi-fedora ~]$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 02:dd:00:37:08:0d brd ff:ff:ff:ff:ff:ff
    inet 10.254.255.88/16 brd 10.254.255.255 scope global dynamic eth0
       valid_lft 86307318sec preferred_lft 86307318sec
    inet6 fe80::dd:ff:fe37:80d/64 scope link
       valid_lft forever preferred_lft forever
3: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 02:dd:3a:e6:dc:0d brd ff:ff:ff:ff:ff:ff
    inet 10.10.10.252/24 brd 10.10.10.255 scope global dynamic eth1
       valid_lft 86307327sec preferred_lft 86307327sec
    inet6 fe80::dd:3aff:fee6:dc0d/64 scope link
       valid_lft forever preferred_lft forever
4: eth2: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 02:dd:71:6e:fa:0d brd ff:ff:ff:ff:ff:ff
    inet 20.20.20.252/24 brd 20.20.20.255 scope global dynamic eth2
       valid_lft 86307336sec preferred_lft 86307336sec
    inet6 fe80::dd:71ff:fe6e:fa0d/64 scope link
       valid_lft forever preferred_lft forever
```





```
