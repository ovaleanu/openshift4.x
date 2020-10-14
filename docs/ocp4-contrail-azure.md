## Contrail with OpenShift 4.x installation on Azure

### Configure DNS

You must have a DNS domain [Azure App Service Domain](https://docs.microsoft.com/en-us/azure/app-service/manage-custom-dns-buy-domain) and create DNS for your Azure account before installation.

My domain is azovsandbox.com

![](https://github.com/ovaleanujnpr/openshift4.x/blob/master/images/azure-domain.png)

### Configure Azure CLI

The installer creates a number of resources in Azure that are necessary to run your cluster, e.g. a resource group and then virtual machines, vnets, network security group, load balancer. To configure Azure CLI, see the [Azure docs](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli-macos?view=azure-cli-latest) for details.

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

### Prepare the Azure setup

Login to Azure
```
$ az login
```
Check the Azure account
```
$ az account show
{
  "environmentName": "AzureCloud",
  "homeTenantId": "..........",
  "id": "..............",
  "isDefault": true,
  "managedByTenants": [],
  "name": "Azure MultiCloud Development Subscription",
  "state": "Enabled",
  "tenantId": ".............",
  "user": {
    "name": "ovaleanu@juniper.net",
    "type": "user"
  }
}
```
The OpenShift installer will use a service principal to do the installation. Because this service principale will create the cluster, I will specify also a RBAC role for this service principal.
```
$ az ad sp create-for-rbac --role Contributor --name openshift-4
Changing "openshift-4" to a valid URI of "http://openshift-4", which is the required format used for service principal names
Creating a role assignment under the scope of "/subscriptions/f39bcbf9-3a71-414d-a17f-794d0fffe65c"
  Retrying role assignment creation: 1/36
  Retrying role assignment creation: 2/36
{
  "appId": ".......",
  "displayName": "openshift-4",
  "name": "http://openshift-4",
  "password": "..........",
  "tenant": "..........."
}
```

Next, you need to assign two roles for this service principal
```
$ az role assignment create --role "User Access Administrator" --assignee "appId_output" --output none
$ az role assignment create --role Contributor --assignee "appId_output" --output none
```
Before starting deploying the cluster, the service principal needs application rw owned by application permission from the Azure Active Directory Graph. You need to run this command
```
$ az ad app permission add --id "appId_output" \
 --api 00000002-0000-0000-c000-000000000000 \
 --api-permissions 824c81eb-e3f8-4ee6-8f6d-de7f50d565b7=Role
Invoking "az ad app permission grant --id appId_output --api 00000002-0000-0000-c000-000000000000" is needed to make the change effective

$ az ad app permission grant --id appId_output --api 00000002-0000-0000-c000-000000000000 --output none
```

### Deploy cluster

Generate an SSH private key and adding it to the agent
```
$ ssh-keygen -b 4096 -t rsa -f ~/.ssh/id_rsa -N ""
```
Create a working folder
```
$ mkdir ~/az-ocp4 ; cd ~/az-ocp4
```
Create a installation configuration file. Check details on [OpenShift docs](https://docs.openshift.com/container-platform/4.5/installing/installing_azure/installing-azure-customizations.html#installation-initializing_installing-azure-customizations)
```
$ openshift-install create install-config
```

In created YAML file under specified directory setup all settings of cluster under networking section change `networkType` field to `Contrail` (instead of `OpenShiftSDN`)

_NOTE:_
- Master nodes need larger instances. For example use Standard_D8s_v3.
- See `install-config` example for an example cluster configuration

Example `install-config.yaml`
```
apiVersion: v1
baseDomain: azovsandbox.com
compute:
- architecture: amd64
  hyperthreading: Enabled
  name: worker
  platform:
    azure:
      osDisk:
        diskSizeGB: 512
      type: Standard_D4s_v3
  replicas: 3
controlPlane:
  architecture: amd64
  hyperthreading: Enabled
  name: master
  platform:
    azure:
      osDisk:
        diskSizeGB: 512
      type: Standard_D8s_v3
  replicas: 3
metadata:
  creationTimestamp: null
  name: az1
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
  azure:
    baseDomainResourceGroupName: openshift-4
    region: uksouth
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
INFO To access the cluster as the system:admin user when using 'oc', run 'export KUBECONFIG=/Users/ovaleanu/az-ocp4/auth/kubeconfig'
INFO Access the OpenShift web-console here: https://console-openshift-console.apps.az1.azovsandbox.com
INFO Login to the console with user: kubeadmin, password: XXXxx-XxxXX-xxXXX-XxxxX
```

Access the cluster
```
$ export KUBECONFIG=~/az-ocp4/auth/kubeconfig
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
$ oc create secret generic htpass-secret --from-file=htpasswd=/Users/ovaleanu/az-ocp4/users.htpasswd -n openshift-config
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
Authentication required for https://api.az1.azovsandbox.com:6443 (openshift)
Username: ovaleanu
Password:
Login successful.
```

Now it is safe to remove kubeadmin user. Details [here](https://docs.openshift.com/container-platform/4.5/authentication/remove-kubeadmin.html).
