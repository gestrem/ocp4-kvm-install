Install OpenShift 4 on Hetzner Server.md
Today
12:03 PM

Nicolas Masse uploaded an item
Text
Install OpenShift 4 on Hetzner Server.md
# Install OpenShift 4 on Hetzner Server
## Install the host system
-> see [[Install RHEL8 on Hetzner Server]].

## Create the DNS entries
Install the Gandi CLI

```sh
yum install python3-pip
pip3 install gandi.cli
```

* Setup gandi-cli

```
# gandi setup 
Welcome to GandiCLI, let's configure a few things before we start.

Api key (xmlrpc) [...]: 
Environnment [production]/ote (ote, production): 
SSH keyfile [~/.ssh/id_rsa.pub]: 
Api key (REST) [...]: 123...456

Setup completed. You can now:
* use 'gandi' to see all command.
* use 'gandi vm create' to create and access a Virtual Machine.
* use 'gandi paas create' to create and access a SimpleHosting instance.
```

* Create the API record

```sh
gandi dns create --ttl 3600 itix.fr api.ocp4 A 136.243.69.83
```

* Create the *.apps record

```sh
gandi dns create --ttl 3600 itix.fr '*.apps.ocp4' A 136.243.69.83
```

## Prepare the host system
* Enable the relevant repos

```sh
subscription-manager repos --disable='*'
subscription-manager repos \
    --enable=rhel-8-for-x86_64-baseos-rpms \
    --enable=rhel-8-for-x86_64-appstream-rpms \
    --enable=rhel-8-for-x86_64-highavailability-rpms \
    --enable=ansible-2.8-for-rhel-8-x86_64-rpms \
    --enable=openstack-15-for-rhel-8-x86_64-rpms
```

* Install ansible (min version 2.8) and git

```sh
yum install -y ansible git
```

* Clone the project

```sh
git clone https://github.com/RedHat-EMEA-SSA-Team/hetzner-ocp4.git
cd hetzner-ocp4
```

## Retrieve valid public certificates
* Create a directory named **certificate** (as expected by the playbook) to hold the certificates:

```sh
mkdir certificate
```

* Install lego

```sh
mkdir -p /usr/local/bin
curl -L -o /tmp/lego.tgz https://github.com/go-acme/lego/releases/download/v3.2.0/lego_v3.2.0_linux_amd64.tar.gz
tar -C /usr/local/bin -zxvf /tmp/lego.tgz lego
chown root:root /usr/local/bin/lego
chmod 755 /usr/local/bin/lego
```

* Get a certificate

```sh
GANDIV5_API_KEY=123...456 lego -m nmasse@redhat.com -d 'api.ocp4.itix.fr' -d '*.apps.ocp4.itix.fr' -a --dns gandiv5 --path ./certificate run --no-bundle
```

**Note:** for AWS, the command is a bit different.

```
AWS_REGION=eu-west-3 AWS_ACCESS_KEY_ID=bla AWS_SECRET_ACCESS_KEY=blabla lego -m nmasse@redhat.com -d 'api.ocp4.itix.fr' -d '*.apps.ocp4.itix.fr' -a --dns route53 --path ./certificate run --no-bundle
```

* Arrange a little bit the certificate hierarchy

```sh
mkdir certificate/ocp4.itix.fr
cd certificate/ocp4.itix.fr
ln -s ../certificates/api.ocp4.itix.fr.crt fullchain.crt
ln -s ../certificates/api.ocp4.itix.fr.key cert.key
cd ../..
```

## Configure the installation
* Create the configuration file "cluster.yml":

```yaml
cluster_name: ocp4
public_domain: itix.fr
dns_provider: custom
storage_nfs: true
auth_redhatsso:
  client_id: "<CLIENT_ID>.apps.googleusercontent.com"
  client_secret: "<CLIENT_SECRET>"
cluster_role_bindings:
  - cluster_role: sudoers
    name: nmasse@redhat.com
  - cluster_role: cluster-admin
    name: nmasse@redhat.com
compute_count: 3
compute_vcpu: 4
compute_memory_size: 65536
image_pull_secret: |-
  {"auths":{"cloud.openshift.com":{"auth":"[REDACTED]","email":"nmasse@redhat.com"},"quay.io":{"auth":"[REDACTED]","email":"nmasse@redhat.com"},"registry.connect.redhat.com":{"auth":"[REDACTED]","email":"nmasse@redhat.com"},"registry.redhat.io":{"auth":"[REDACTED]","email":"nmasse@redhat.com"}}}
```

**Note:**:
	* the pull secret can be fetched from [Red Hat OpenShift Cluster Manager](https://cloud.redhat.com/openshift/install/metal/user-provisioned).
	* `client_id` and `client_secret` is given when you follow this procedure: [Login to OpenShift with your Google Account](https://github.com/nmasse-itix/OpenShift-Examples/blob/master/Login-to-OpenShift-with-your-Google-Account/README.md)

## Run the installation
* Prepare the host

```sh
./ansible/01-prepare-host.yml
```

* Provision cluster

```sh
./ansible/02-create-cluster.yml --skip-tags public_dns,letsencrypt
```

## Useful commands
* Start the cluster

```sh
./ansible/04-start-cluster.yml
```

* Stop the cluster

```sh
./ansible/03-stop-cluster.yml
```

* Complete post-installation (in case it went interrupted)

```sh
./ansible/02-create-cluster.yml -t post-install
```

* Discover node IPs

```
# for vm in $(virsh list --name); do echo $vm; echo; virsh domifaddr $vm; done

ocp4-master-0

 Name       MAC address          Protocol     Address
-------------------------------------------------------------------------------
 vnet1      52:54:00:a8:32:0a    ipv4         192.168.50.10/24

ocp4-master-1

 Name       MAC address          Protocol     Address
-------------------------------------------------------------------------------
 vnet2      52:54:00:a8:32:0b    ipv4         192.168.50.11/24

ocp4-master-2

 Name       MAC address          Protocol     Address
-------------------------------------------------------------------------------
 vnet3      52:54:00:a8:32:0c    ipv4         192.168.50.12/24

ocp4-compute-0

 Name       MAC address          Protocol     Address
-------------------------------------------------------------------------------
 vnet4      52:54:00:a8:32:0d    ipv4         192.168.50.13/24

ocp4-compute-1

 Name       MAC address          Protocol     Address
-------------------------------------------------------------------------------
 vnet5      52:54:00:a8:32:0e    ipv4         192.168.50.14/24

ocp4-compute-2

 Name       MAC address          Protocol     Address
-------------------------------------------------------------------------------
 vnet6      52:54:00:a8:32:0f    ipv4         192.168.50.15/24

```

## Post-install
* Remove the kubeadmin account

```
oc delete secret kubeadmin -n kube-system
```


## Cleanup
* Remove the cluster

```sh
./ansible/99-destroy-cluster.yml --skip-tags public_dns,letsencrypt
```

## Common errors
### Still waiting for the Kubernetes API: x509: certificate signed by unknown authority

**Error message**
```
# /opt/openshift-install-4.3/openshift-install wait-for bootstrap-complete --dir ocp4 --log-level debug
DEBUG OpenShift Installer v4.3.5                   
DEBUG Built from commit 82f9a63c06956b3700a69475fbd14521e139aa1e 
INFO Waiting up to 30m0s for the Kubernetes API at https://api.ocp4.itix.fr:6443... 
DEBUG Still waiting for the Kubernetes API: Get https://api.ocp4.itix.fr:6443/version?timeout=32s: x509: certificate signed by unknown authority 
```

**Cause**: You tried to re-install the cluster over a previous install

**Resolution:** Run `99-destroy-cluster.yml` and then `02-create-cluster.yml`

## How to troubleshoot
[Gathering installation logs | Installing | OpenShift Container Platform 4.2](https://docs.openshift.com/container-platform/4.2/installing/installing-gather-logs.html)


#tips #products/openshift