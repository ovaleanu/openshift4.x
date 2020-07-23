## Contrail with OpenShift 4.x installation on AWS

### Configure DNS

A DNS zone must be created and available in Route 53 for your AWS account before installation. Register a domain for your cluster in [AWS Route 53](https://aws.amazon.com/route53/).
Entries created in the Route 53 zone are expected to be resolvable from the nodes.

### Configure AWS credentials

The installer creates a number of resources in AWS that are necessary to run your cluster, e.g. EC2 instances, VPCs, security groups and IAM roles. To configure credentials, see the [AWS docs](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-configure.html) for details.

### Download the installer and the command-line tools

Check which versions are available

```
$ curl -s https://mirror.openshift.com/pub/openshift-v4/clients/ocp/ | \
  awk '{print $5}'| \
  grep -o '4.[0-9].[0-9]*' | \
  uniq | \
  sort | \
  column
```
Set the version and download the installer and cli tool

```
$ VERSION=4.4.11
$ wget https://mirror.openshift.com/pub/openshift-v4/clients/ocp/$VERSION/openshift-install-linux-$VERSION.tar.gz
$ wget https://mirror.openshift.com/pub/openshift-v4/clients/ocp/$VERSION/openshift-client-mac-$VERSION.tar.gz

$ tar -xvzf openshift-install-mac-4.4.11.tar.gz -C /usr/local/bin
$ tar -xvzf openshift-client-mac-4.4.11.tar.gz -C /usr/local/bin

$ openshift-install version
$ oc version
$ kubectl version
```

### Deploy cluster

Generate an SSH private key and adding it to the agent
```
$ ssh-keygen -b 4096 -t rsa -f ~/.ssh/id_rsa -N ""
```
Create a working folder
```
$ mkdir ~/aws-ocp4 ; cd ~/ocp4
```
Create a installation configuration file
```
$ openshift-install create install-config
```

In created YAML file under specified directory setup all settings of cluster under networking section change `networkType` field to `Contrail` (instead of `OpenShiftSDN`)

_NOTE:_
- Master nodes need larger instances. For example, If you run cluster on AWS, use e.g. m5.2xlarge.
- See `install-config` example for an example cluster configuration

Example `install-config.yaml`
```
apiVersion: v1
baseDomain: ovsandbox.com
compute:
- architecture: amd64
  hyperthreading: Enabled
  name: worker
  platform: {}
  replicas: 2
controlPlane:
  architecture: amd64
  hyperthreading: Enabled
  name: master
  platform:
    aws:
       type: m5.2xlarge
  replicas: 3
metadata:
  creationTimestamp: null
  name: w1
networking:
  clusterNetwork:
  - cidr: 10.128.0.0/14
    hostPrefix: 23
  machineNetwork:
  - cidr: 10.0.0.0/16
  networkType: Contrail
  serviceNetwork:
  - 172.30.0.0/16
platform:
  aws:
    region: eu-west-1
publish: External
pullSecret: '{"auths"...}'
sshKey: |
  ssh-rsa ...
```

Create the installation manifests
```
# openshift-install create manifests
```

Clone contrail operator repository

```
# git clone https://github.com/Juniper/contrail-operator.git
# git checkout R2008
```

Create Contrail operator configuration file

```
# cat <<EOF > config_contrail_operator.yaml
CONTRAIL_VERSION=2008.20
CONTRAIL_REGISTRY=hub.juniper.net/contrail-nightly
DOCKER_CONFIG=eyJhdXRocyI6eyJodWIuanVuaXBlci5uZXQvY29udHJhaWwtbmlnaHRseSI6eyJ1c2VybmFtZSI6IkpOUFItRmllbGRVc2VyMTc5IiwicGFzc3dvcmQiOiJleWZMYkFxS2RFUTdXNG1EYVI2ViIsImVtYWlsIjoib3ZhbGVhbnVAanVuaXBlci5uZXQiLCJhdXRoIjoiU2s1UVVpMUdhV1ZzWkZWelpYSXhOems2WlhsbVRHSkJjVXRrUlZFM1Z6UnRSR0ZTTmxZPSJ9fX0=
EOF
```

Install Contrail manifests

```
# ./contrail-operator/deploy/openshift/install-manifests.sh --dir ./ --config ./config_contrail_operator.yaml
```

Create cluster

```
$ openshift-install create cluster --log-level=debug
```

When AWS resources are created manually add rules to Security Groups using this [CLI tool](https://github.com/Juniper/contrail-operator/tree/master/deploy/openshift/tools/contrail-sc-open)

When service router-default will be created in openshift-ingress namespace patch it with command

```
$ oc -n openshift-ingress patch service router-default --patch '{"spec": {"externalTrafficPolicy": "Cluster"}}'
```

Wait for installation to finish. Properly installed cluster should return information similar to

```
INFO Waiting up to 10m0s for the openshift-console route to be created...
DEBUG Route found in openshift-console namespace: console
DEBUG Route found in openshift-console namespace: downloads
DEBUG OpenShift console route is created
INFO Install complete!
INFO To access the cluster as the system:admin user when using 'oc', run 'export KUBECONFIG=/Users/ovaleanu/aws1-ocp4/auth/kubeconfig'
INFO Access the OpenShift web-console here: https://console-openshift-console.apps.w1.ovsandbox.com
INFO Login to the console with user: kubeadmin, password: XXXxx-XxxXX-xxXXX-XxxxX
```

Access the cluster
```
$ export KUBECONFIG=~/aws-ocp4/auth/kubeconfig
```
