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
$ VERSION=4.4.20
$ wget https://mirror.openshift.com/pub/openshift-v4/clients/ocp/$VERSION/openshift-install-mac-$VERSION.tar.gz
$ wget https://mirror.openshift.com/pub/openshift-v4/clients/ocp/$VERSION/openshift-client-mac-$VERSION.tar.gz

$ tar -xvzf openshift-install-mac-4.4.20.tar.gz -C /usr/local/bin
$ tar -xvzf openshift-client-mac-4.4.20.tar.gz -C /usr/local/bin

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
$ mkdir ~/aws-ocp4 ; cd ~/aws-ocp4
```
Create a installation configuration file. Check details on [OpenShift docs](https://docs.openshift.com/container-platform/4.5/installing/installing_aws/installing-aws-customizations.html#installation-initializing_installing-aws-customizations)
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
  platform:
    aws:
      rootVolume:
        iops: 2000
        size: 500
        type: io1
      type: m5.4xlarge
  replicas: 3
controlPlane:
  architecture: amd64
  hyperthreading: Enabled
  name: master
  platform:
    aws:
      rootVolume:
        iops: 4000
        size: 500
        type: io1
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
$ openshift-install create manifests
```

Clone contrail operator repository

```
$ git clone https://github.com/Juniper/contrail-operator.git
$ git checkout R2008
```

Create Contrail operator configuration file

```
$ cat <<EOF > config_contrail_operator.yaml
CONTRAIL_VERSION=2008.109
CONTRAIL_REGISTRY=hub.juniper.net/contrail-nightly
DOCKER_CONFIG=<this_needs_to_be_generated>
EOF
```
`DOCKER_CONFIG` is configuration for registry secret to closed container registry (if registry is wide open then no credentials are required) Set `DOCKER_CONFIG` to registry secret with proper data in base64.

_NOTE: You may create base64 encoded value for config with script provided [here](https://github.com/Juniper/contrail-operator/tree/master/deploy/openshift/tools/docker-config-generate). Copy output of the script and paste into config used to install-manifests script._

Install Contrail manifests

```
$ ./contrail-operator/deploy/openshift/install-manifests.sh --dir ./ --config ./config_contrail_operator.yaml
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
$ oc create secret generic htpass-secret --from-file=htpasswd=/Users/ovaleanu/aws-ocp4/users.htpasswd -n openshift-config
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

Add the user to `cluster-amdin` role
```
$ oc adm policy add-cluster-role-to-user cluster-admin ovaleanu
```

Login using the user
```
oc login -u ovaleanu
Authentication required for https://api.w1.ovsandbox.com:6443 (openshift)
Username: ovaleanu
Password:
Login successful.
```

Now it is safe to remove kubeadmin user. Details [here](https://docs.openshift.com/container-platform/4.5/authentication/remove-kubeadmin.html).
