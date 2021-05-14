## TungstenFabric with OpenShift 4.6 installation on VMs running on KVM

The following procedure works also if bare metal servers are used. If there are existing DNS, DHCP, HTTP, PXE servers, update services following examples [here](https://github.com/ovaleanu/openshift4.x/tree/master/bare-metal#bare-metal-prerequisites) and jump to [Create Ignition Configs](https://github.com/ovaleanu/openshift4.x/blob/master/docs/ocp4-contrail-vm-bms.md#create-ignition-configs).

The procedure follows [helper node installation guide line](https://github.com/RedHatOfficial/ocp4-helpernode/blob/master/docs/quickstart.md). Some modifications occurs becasue I am using disk type `/dev/sda/` instead of `/dev/vda` and also when applying Contrail manifests.

On the hypervisor host create a working directory

```
# mkdir ~/ocp4-workingdir
# cd ~/ocp4-workingdir
```

### Create a virtual network

Download the virtual network configuration file, virt-net.xml
```
# wget https://raw.githubusercontent.com/RedHatOfficial/ocp4-helpernode/master/docs/examples/virt-net.xml
```
Create a virtual network using this file file provided in this repo (modify if you need to).

```
# virsh net-define --file virt-net.xml
```
Set it to autostart on boot

```
# virsh net-autostart openshift4
# virsh net-start openshift4
```

### Create a CentOS 7/8 VM

Download the Kickstart file for either EL 7 or EL 8 for the helper node.

**EL7**
```
# wget https://github.com/ovaleanu/openshift4.x/blob/master/docs/helper-ks.cfg -o helper-ks.cfg
```

**EL 8**
```
# wget https://github.com/ovaleanu/openshift4.x/blob/master/docs/helper-ks8.cfg -o helper-ks.cfg
```
Edit `helper-ks.cfg` for your environment and use it to install the helper. The following command installs it "unattended".

**EL7**
```
# virt-install --name="ocp4-aHelper" --vcpus=2 --ram=4096 \
--disk path=/var/lib/libvirt/images/ocp4-aHelper.qcow2,bus=scsi,size=30 \
--os-variant centos7.0 --network network=openshift4,model=virtio \
--boot hd,menu=on --location /var/lib/libvirt/iso/CentOS-7-x86_64-Minimal-2009.iso \
--initrd-inject helper-ks.cfg --extra-args "inst.ks=file:/helper-ks.cfg" --noautoconsole
```

**EL 8**
```
# virt-install --name="ocp4-aHelper" --vcpus=2 --ram=4096 \
--disk path=/var/lib/libvirt/images/ocp4-aHelper.qcow2,bus=scsi,size=50 \
--os-variant centos8 --network network=openshift4,model=virtio \
--boot hd,menu=on --location /var/lib/libvirt/iso/CentOS-8.2.2004-x86_64-dvd1.iso \
--initrd-inject helper-ks.cfg --extra-args "inst.ks=file:/helper-ks.cfg" --noautoconsole
```

The provided Kickstart file installs the helper with the following settings (which is based on the virt-net.xml file that was used before).

- IP - 192.168.7.77
- NetMask - 255.255.255.0
- Default Gateway - 192.168.7.1
- DNS Server - 8.8.8.8

You can watch the progress by lauching the viewer
```
# virt-viewer --domain-name ocp4-aHelper
```

Once it's done, it'll shut off...turn it on with the following command
```
# virsh start ocp4-aHelper
```

### Prepare the Helper Node

After the helper node is installed; login to it
```
# ssh -l root 192.168.7.77
```

Install EPEL and update. If kernel is updated, reboot.
```
# yum -y install https://dl.fedoraproject.org/pub/epel/epel-release-latest-$(rpm -E %rhel).noarch.rpm
# yum -y update
# reboot
```

Install `ansible` and `git` and clone this helpernode repo
```
# yum -y install ansible git
# git clone https://github.com/RedHatOfficial/ocp4-helpernode
# cd ocp4-helpernode
```

Copy vars.yaml file. Edit and change `disk` as `sda` and domain name, mac addresses, ipam in case you used a different subnet.
```
# cp docs/examples/vars.yaml .
```

To modify the OpenShift version modify `vars/main.yml` file

Run the playbook to setup your helper node
```
# ansible-playbook -e @vars.yaml tasks/main.yml
```

After it is done run the following to get info about your environment and some install help
```
# /usr/local/bin/helpernodecheck services
```

### Create Ignition Configs

_Note: make sure NTP server is configured on Hypervisor and Helper node, otherwise installation will fail with error `X509: certificate has expired or is not yet valid`_

Create a place to store your pull-secret
```
# mkdir -p ~/.openshift
```

Visit [try.openshift.com](https://cloud.redhat.com/openshift/install) and select "Bare Metal". Download your pull secret and save it under ~/.openshift/pull-secret. Edit the pull-secret to include credentials for Juniepr docker repository.

```
# ls -l ~/.openshift/pull-secret
/root/.openshift/pull-secret
```

This playbook creates an sshkey for you; it's under `~/.ssh/helper_rsa`. You can use this key or create/user another one if you wish.

```
# ls -l ~/.ssh/helper_rsa
/root/.ssh/helper_rsa
```

Create an install directory

```
# mkdir ~/ocp4
# cd ~/ocp4
```

Next, create an `install-config.yaml` file.

```
# cat <<EOF > install-config.yaml
apiVersion: v1
baseDomain: example.com
compute:
- hyperthreading: Enabled
  name: worker
  replicas: 0
controlPlane:
  hyperthreading: Enabled
  name: master
  replicas: 3
metadata:
  name: ocp4
networking:
  clusterNetworks:
  - cidr: 10.128.0.0/14
    hostPrefix: 23
  networkType: TungstenFabric
  serviceNetwork:
  - 172.30.0.0/16
platform:
  none: {}
pullSecret: '$(< ~/.openshift/pull-secret)'
sshKey: '$(< ~/.ssh/helper_rsa.pub)'
EOF
```

Create the OpenShift installation manifests
```
# openshift-install create manifests
```

Edit the `manifests/cluster-scheduler-02-config.yml` Kubernetes manifest file to prevent Pods from being scheduled on the control plane machines by setting `mastersSchedulable` to `false`.
```
# sed -i 's/mastersSchedulable: true/mastersSchedulable: false/g' manifests/cluster-scheduler-02-config.yml
```
It should look something like this after you edit it

```
# cat manifests/cluster-scheduler-02-config.yml
apiVersion: config.openshift.io/v1
kind: Scheduler
metadata:
  creationTimestamp: null
  name: cluster
spec:
  mastersSchedulable: false
  policy:
    name: ""
status: {}
```

Download TF Openshift manifests add add them to the installation
```
# export INSTALL_DIR=/root/ocp4
# git clone https://github.com/tungstenfabric/tf-openshift.git
# ./tf-openshift/scripts/apply_install_manifests.sh $INSTALL_DIR
```

Download TF Operator manifests add add them to the installation
```
# git clone https://github.com/tungstenfabric/tf-operator.git
# cd tf-operator
# git checkout R2011 && cd ..
# export CONTRAIL_DEPLOYER_CONTAINER_TAG="R2011.L1.229"
# export DEPLOYER_CONTAINER_REGISTRY="hub.juniper.net/contrail-nightly"
# export CONTRAIL_CONTAINER_TAG="R2011.L1.229"
# export CONTAINER_REGISTRY="hub.juniper.net/contrail-nightly"
# export KUBECONFIG=$INSTALL_DIR/auth/kubeconfig
# export DEPLOYER="openshift"
# export KUBERNETES_CLUSTER_NAME="ocp4"
# export KUBERNETES_CLUSTER_DOMAIN="example.com"
# export CONTRAIL_REPLICAS=3
# ./tf-operator/contrib/render_manifests.sh

# for i in $(ls ./tf-operator/deploy/crds/) ; do
  cp ./tf-operator/deploy/crds/$i $INSTALL_DIR/manifests/01_$i
done

# for i in namespace service-account role cluster-role role-binding cluster-role-binding ; do
  cp ./tf-operator/deploy/kustomize/base/operator/$i.yaml $INSTALL_DIR/manifests/02-tf-operator-$i.yaml
done

# oc kustomize ./tf-operator/deploy/kustomize/operator/templates/ | sed -n 'H; /---/h; ${g;p;}' > $INSTALL_DIR/manifests/02-tf-operator.yaml
# oc kustomize ./tf-operator/deploy/kustomize/contrail/templates/ > $INSTALL_DIR/manifests/03-tf.yaml
```

Configure a custom NTP server

If you are using a local NTP server you need to create a machineconfig for this. Edit IP address for the NTP server

Create base64 env variable of my NTP server

```
export CHRONY_CONF_BASE64=$(cat << EOF | base64 -w 0
server <IP_NTP_SERVER> iburst
driftfile /var/lib/chrony/drift
makestep 1.0 3
rtcsync
logdir /var/log/chrony
EOF
)
```

Create config for control-plane (master) nodes

```
cat << EOF > ./openshift/99_masters-chrony-configuration.yaml
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfig
metadata:
  labels:
    machineconfiguration.openshift.io/role: master
  name: masters-chrony-configuration
spec:
  config:
    ignition:
      config: {}
      security:
        tls: {}
      timeouts: {}
      version: 2.2.0
    networkd: {}
    passwd: {}
    storage:
      files:
      - contents:
          source: data:text/plain;charset=utf-8;base64,$CHRONY_CONF_BASE64
          verification: {}
        filesystem: root
        mode: 420
        path: /etc/chrony.conf
EOF
```

Create config for worker nodes

```
cat << EOF > ./openshift/99_workers-chrony-configuration.yaml
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfig
metadata:
  labels:
    machineconfiguration.openshift.io/role: worker
  name: workers-chrony-configuration
spec:
  config:
    ignition:
      config: {}
      security:
        tls: {}
      timeouts: {}
      version: 2.2.0
    networkd: {}
    passwd: {}
    storage:
      files:
      - contents:
          source: data:text/plain;charset=utf-8;base64,$CHRONY_CONF_BASE64
          verification: {}
        filesystem: root
        mode: 420
        path: /etc/chrony.conf
EOF
```

Generate the ignition configs

```
# openshift-install create ignition-configs
```

Copy the ignition files in the `ignition` directory for the websever

```
# cp ~/ocp4/*.ign /var/www/html/ignition/
# restorecon -vR /var/www/html/
# restorecon -vR /var/lib/tftpboot/
# chmod o+r /var/www/html/ignition/*.ign
```

### Install VMs

From the hypervisor launch VMs using PXE booting. For BMS, boot the servers using PXE booting.

Launch Bootstrap VM
```
# virt-install --pxe --network bridge=openshift4 --mac=52:54:00:60:72:67 --name ocp4-bootstrap --ram=8192 --vcpus=4 --os-variant rhel8.0 --disk path=/var/lib/libvirt/images/ocp4-bootstrap.qcow2,bus=scsi,size=120 --vnc
```
This command will create a bootstrap node VM, will connect to PXE server (our Helper), assign the IP address from DHCP and download the RHCOS image from the HTTP server. At the end of the installation it will embed the ignition file. After the node is installed and rebooted we can connect to it from the Helper.



Launch the Masters VMs

```
# virt-install --pxe --network bridge=openshift4 --mac=52:54:00:e7:9d:67 --name ocp4-master0 --ram=40960 --vcpus=10 --os-variant rhel8.0 --disk path=/var/lib/libvirt/images/ocp4-master0.qcow2,bus=scsi,size=150 --vnc
# virt-install --pxe --network bridge=openshift4 --mac=52:54:00:80:16:23 --name ocp4-master1 --ram=40960 --vcpus=10 --os-variant rhel8.0 --disk path=/var/lib/libvirt/images/ocp4-master1.qcow2,bus=scsi,size=150 --vnc
# virt-install --pxe --network bridge=openshift4 --mac=52:54:00:d5:1c:39 --name ocp4-master2 --ram=40960 --vcpus=10 --os-variant rhel8.0 --disk path=/var/lib/libvirt/images/ocp4-master2.qcow2,bus=scsi,size=150 --vnc
```

You can login to the Master from Helper Node

```
# ssh -i ~/.ssh/helper_rsa core@192.168.7.21
# ssh -i ~/.ssh/helper_rsa core@192.168.7.22
# ssh -i ~/.ssh/helper_rsa core@192.168.7.23
```

You can monitor pods creation with `sudo crictl ps` command

### Wait for install

The boostrap VM actually does the install for you; you can track it with the following command from the Helper. Make sure you are in `~/ocp4` directory

```
# openshift-install wait-for bootstrap-complete --log-level debug
```

Once you see this message below...
```
INFO Waiting up to 30m0s for the Kubernetes API at https://api.ocp4.example.com:6443...
INFO API v1.13.4+838b4fa up
INFO Waiting up to 30m0s for bootstrapping to complete...
DEBUG Bootstrap status: complete
INFO It is now safe to remove the bootstrap resources
```

You can delete the bootstrap VM and luanch the Worker nodes from the Hypervisor

```
# virt-install --pxe --network bridge=openshift4 --mac=52:54:00:f4:26:a1 --name ocp4-worker0 --ram=16384 --vcpus=6 --os-variant rhel8.0 --disk path=/var/lib/libvirt/images/ocp4-worker0.qcow2,bus=scsi,size=120 --vnc
# virt-install --pxe --network bridge=openshift4 --mac=52:54:00:82:90:00 --name ocp4-worker1 --ram=16384 --vcpus=6 --os-variant rhel8.0 --disk path=/var/lib/libvirt/images/ocp4-worker1.qcow2,bus=scsi,size=120 --vnc
```

### Finish Install

Login to your cluster

```
# export KUBECONFIG=/root/ocp4/auth/kubeconfig
```

Your install may be waiting for worker nodes to get approved. Normally the machineconfig node approval operator takes care of this for you. However, sometimes this needs to be done manually. Check pending CSRs with the following command

```
# oc get csr
```

You can approve all pending CSRs in "one shot" with the following

```
# oc get csr -o go-template='{{range .items}}{{if not .status}}{{.metadata.name}}{{"\n"}}{{end}}{{end}}' | xargs oc adm certificate approve
```

You may have to run this multiple times depending on how many workers you have and in what order they come in. Keep a watch on these CSRs

```
# watch -n5 oc get csr
```

In order to setup your registry, you first have to set the `managementState` to `Managed` for your cluster
```
# oc patch configs.imageregistry.operator.openshift.io cluster --type merge --patch '{"spec":{"managementState":"Managed"}}'
```

For PoCs, using emptyDir is okay (to use PVs follow [this](https://docs.openshift.com/container-platform/latest/installing/installing_bare_metal/installing-bare-metal.html#registry-configuring-storage-baremetal_installing-bare-metal) doc)

```
# oc patch configs.imageregistry.operator.openshift.io cluster --type merge --patch '{"spec":{"storage":{"emptyDir":{}}}}'
```

If you need to expose the registry, run this command
```
# oc patch configs.imageregistry.operator.openshift.io/cluster --type merge -p '{"spec":{"defaultRoute":true}}'
```

To finish the install process, run the following (make sure you are in `~/ocp4` directory)

```
openshift-install wait-for install-complete
```

When the follwoing message is displayed the installation has finished
```
# openshift-install wait-for install-complete
INFO Waiting up to 30m0s for the cluster at https://api.ocp4.example.com:6443 to initialize...
INFO Waiting up to 10m0s for the openshift-console route to be created...
INFO Install complete!
INFO To access the cluster as the system:admin user when using 'oc', run 'export KUBECONFIG=/root/ocp4/auth/kubeconfig'
INFO Access the OpenShift web-console here: https://console-openshift-console.apps.ocp4.example.com
INFO Login to the console with user: kubeadmin, password: XXX-XXXX-XXXX-XXXX
```

### Adding a user

By default, OpenShift4 ships with a single kubeadmin user, that could be used during initial cluster configuration. You will create a Custom Resource (CR) to define a HTTPasswd identity provider.

To use the HTPasswd identity provider, you must generate a flat file that contains the user names and passwords for your cluster by using [htpasswd](https://httpd.apache.org/docs/2.4/programs/htpasswd.html).
```
$ htpasswd -c -B -b users.htpasswd ovaleanu MyPassword
```

You should get a file like this
```
$ cat users.htpasswd
ovaleanu:$2y$05$M7lvBvh7X1ElpYBGO2ZObOfLE8z2NHNUq2yzhC./AbXOxXVFKNMK6
```

Next we need to define a secret that contains the HTPasswd user file
```
$ oc create secret generic htpass-secret --from-file=htpasswd=/root/ocp4/users.htpasswd -n openshift-config
```

This Custom Resource shows the parameters and acceptable values for an HTPasswd identity provider.

```
$ cat htpasswdCR.yaml
apiVersion: config.openshift.io/v1
kind: OAuth
metadata:
  name: cluster
spec:
  identityProviders:
  - name: ovaleanu
    mappingMethod: claim
    type: HTPasswd
    htpasswd:
      fileData:
        name: htpass-secret
```

Apply the defined CR
```
$ oc create -f htpasswdCR.yaml
```

**If you have OpenShift4.5 the OAuth CR already exists and you just need to append it with spec properties**

```
$ oc edit oauth/cluster
```

Append under `spec` with this and save

```
spec:
  identityProviders:
  - name: ovaleanu
    mappingMethod: claim
    type: HTPasswd
    htpasswd:
      fileData:
        name: htpass-secret
```

Add the user to `cluster-amdin` role
```
$ oc adm policy add-cluster-role-to-user cluster-admin ovaleanu
```

Login using the user
```
oc login -u ovaleanu
Authentication required for https://api.ocp4.example.com:6443 (openshift)
Username: ovaleanu
Password:
Login successful.
```

Now it is safe to remove kubeadmin user. Details [here](https://docs.openshift.com/container-platform/4.5/authentication/remove-kubeadmin.html).
