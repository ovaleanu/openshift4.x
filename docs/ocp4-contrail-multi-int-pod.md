## Multi-interface pod in OpenShift 4.4 with Contrail 2008

This exercise will demonstrate multi-interface pod in OpenShift 4.4 with Contrail 2008.

Create two virtual networks `neta` and `netb` in Contrail

```
$ cat <<EOF > netab.yaml
apiVersion: "k8s.cni.cncf.io/v1"
kind: NetworkAttachmentDefinition
metadata:
 name: neta
 annotations:
   "opencontrail.org/cidr" : "10.10.10.0/24"
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
 annotations:
   "opencontrail.org/cidr" : "20.20.20.0/24"
spec:
 config: '{
   "cniVersion": "0.3.1",
   "type": "contrail-k8s-cni"
}'
EOF

$ oc apply -f netab.yaml
```

Create a ubuntu pod with connected to both networks `neta` and `netb`

```
$ cat <<EOF > ubuntu-multi-nic.yaml
apiVersion: v1
kind: Pod
metadata:
 name: multi-intf-pod
 annotations:
   k8s.v1.cni.cncf.io/networks: '[
     { "name": "neta" },
     { "name": "netb" }
   ]'
spec:
 containers:
   - name: ubuntuapp
     image: ubuntu-upstart
EOF

$ oc apply -f ubuntu-multi-nic.yaml

$ oc get pods
NAME                              READY   STATUS    RESTARTS   AGE
multi-intf-pod                    1/1     Running   0          20h
```

Connect to the pod check network interface. As you can see the pod has a network interface in each virtual network defined and an interface in default podNetwork.

```
[root@helper ocp4-helpernode]# oc exec -it multi-intf-pod bash
kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl kubectl exec [POD] -- [COMMAND] instead.
root@multi-intf-pod:/# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
78: eth0@if79: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether 02:33:9b:0c:56:02 brd ff:ff:ff:ff:ff:ff
    inet 10.254.255.68/16 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::f877:2eff:feff:1442/64 scope link
       valid_lft forever preferred_lft forever
80: net1@if81: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether 02:33:c9:e1:16:02 brd ff:ff:ff:ff:ff:ff
    inet 10.10.10.252/24 scope global net1
       valid_lft forever preferred_lft forever
    inet6 fe80::44df:33ff:fe9a:6a5a/64 scope link
       valid_lft forever preferred_lft forever
82: net2@if83: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether 02:33:f7:31:52:02 brd ff:ff:ff:ff:ff:ff
    inet 20.20.20.252/24 scope global net2
       valid_lft forever preferred_lft forever
    inet6 fe80::18f1:34ff:fe20:275/64 scope link
       valid_lft forever preferred_lft forever
root@multi-intf-pod:/#
```
