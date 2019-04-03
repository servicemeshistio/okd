# Origin Community Distribution - okd.io

## Prerequisites

* A decent Windows 7 / 10 laptop with minimum 16 GB RAM and Intel Core i7 processor and a preferable minimum 512 GB SSD
* VMWare Workstation 14.1.2/15.0.4 - Even though it is a licensed software, it is worth buying it. The VMware Workstation is a rock solid software.
* You can use `free` VMware player but you can run only one VM per host, which is OK for this environment.
* Build your VM. My personal choice is to use `CentOS` but you can use any Linux distribution of your choice.
* In VMware Workstation, `Edit` ⇨ `Virtual Network Editor` to set the `vmnet8` to subnet address `192.168.142.0` in NAT mode.
* Attach 100 GB thin provisioned disk (vmdk) for storing Docker images and containers

## Download Base VM without OpenShift Installed

You can download the base VM from [here](#) for the purpose of following through the book. The VM was built using CentOS 7.6 Linux distribution on VMware Workstation 15.0.4. If you prefer your own choice of Linux distribution, build your VM and meet the following prerequisites:

* The root password is `password` in the base VM and it automatically logs in user `db2psc` for which the password is `password`.

If you want to skip the following OKD Installation procedure, you can download the VM with CentOS Origin on CentOS from this [link](#).

## Resource for VM

You need minimum Intel Core i7 processor on your laptop. Assign 16 GB of RAM and 8 cores to the VM.

The installation requires minimum 8 cores for the Master VM. So, with a Intel Core i7 having only 4 cores, how do we get the extra cores.

If using Intel Core i7 processor in a laptop, it gives 4 cores and 2 threads per core. It is OK to allocate 8 CPU to the VM. The VMware is pretty good in dynamic CPU scheduling.

## Internet Access

Make sure that you can run `ping -c4 google.com` from the VM to make sure that Internet access is available.  

If ping does not succeed, it is quite possible that the network address of VMnet8 adapter needs to be fixed. Make sure that you use 192.168.142.0 subnet for the `nmnet8` adapter. Refer to [this](vnet.md) link for instructions to fix `vmnet8` subnet address in VMware. 

## Firewall

Either disable firewall or allow access to ssh

```bash
systemctl is-enabled firewalld
```
If enabled, then run the following:
```bash
firewall-cmd --add-service=ssh --permanent 
firewall-cmd --reload
```

## Generate SSH Key-pair

We are going to use just one VM for the installation of OpenShift. We still need to make SSH key=pair for password less SSH to the same VM. The root password is password.

```bash
yum -y install sshpass
ssh-keygen -f $HOME/.ssh/id_rsa -t rsa -N '' -q
ssh-keyscan 192.168.142.102 osc02.servicemesh.local osc02 > $HOME/.ssh/known_hosts
sshpass -p password ssh-copy-id -i $HOME/.ssh/id_rsa.pub root@localhost


ls -l .ssh
total 16
-rw------- 1 root root  402 Mar 28 17:33 authorized_keys
-rw------- 1 root root 1675 Mar 28 17:31 id_rsa
-rw-r--r-- 1 root root  402 Mar 28 17:31 id_rsa.pub
-rw-r--r-- 1 root root  171 Mar 28 17:33 known_hosts
```

### Test ssh

```bash
ssh localhost date
ssh osc02 date
ssh osc02.servicemesh.local date
ssh 192.168.142.102 date
```

## Set selinux

SELinux requirements

Check if `selinux` is enabled or not.

```bash
getenforce
Enforcing
```

If above shows `disabled`, then enable `selinux`.

Security-Enhanced Linux (SELinux) must be enabled on all of the servers before installing OpenShift Container Platform or the installer will fail. Also, configure `SELINUX=enforcing` and `SELINUXTYPE=targeted` in the `/etc/selinux/config` file and then reboot the machine.

After reboot, confirm its status is Enforcing:

```
# getenforce
Enforcing
```

## DNS Requirements

A very nice [article on DNS for OpenShift](https://developers.redhat.com/blog/2015/11/19/dns-your-openshift-v3-cluster/).

OpenShift Container Platform requires a fully functional DNS server in the environment. This is ideally a separate host running DNS server and can provide name resolution to hosts and containers running on the platform.

But, we will configure DNS using `dnsmasq` locally. 

Edit `dnsmasq.conf` to set the options.

```bash
# cat /etc/dnsmasq.conf 
log-queries
domain-needed
bogus-priv
no-resolv
# Forward cluster.local queries to SkyDNS
server=/cluster.local/127.0.0.1#8053
server=192.168.142.2
#server=8.8.8.8
#server=8.8.4.4
no-hosts
addn-hosts=/etc/dnsmasq.hosts
conf-dir=/etc/dnsmasq.d
# Reverse DNS record for master
host-record=osc02.servicemesh.local,192.168.142.102
# Wildcard DNS for OpenShift Applications - Points to Router
address=/.apps.servicemesh.local/192.168.142.102
# Forward reverse queries for service network to SkyDNS.
# This is for default OpenShift SDN - change as needed.
server=/17.30.172.in-addr.arpa/127.0.0.1#8053
```

Reload dnsmasq

```bash
systemctl restart dnsmasq
```

Reverse queries are for the OpenShift SDN address range. The SDN CIDR range can be identified by looking `serviceNetworkCIDR` in `/etc/origin/master/master-config.yaml` file.

Local hosts file `/etc/dnsmasq.hosts`

Edit your `/etc/dnsmasq.hosts` as per entries below.

```bash
# cat /etc/dnsmasq.hosts
192.168.142.102 apps.servicemesh.local apps
192.168.142.102 osc02.servicemesh.local osc02
```

The sub-domain `apps.servicemesh.local` will be used for deploying applications. If a new application `cloud` needs to be deployed, the `nslookup` must be able to translate name `cloud.apps.servicemesh.local` to the IP address of the host identified by `openshift_public_hostname` in the inventory file. This is accomplished by an entry `address=/.apps.servicemesh.local/192.168.142.102` in the `/etc/dnsmasq.conf`.

Example:

```bash
# nslookup -debug cloud.apps.servicemesh.local
Server:		192.168.142.102
Address:	192.168.142.102#53

------------
    QUESTIONS:
	cloud.apps.servicemesh.local, type = A, class = IN
    ANSWERS:
    ->  cloud.apps.servicemesh.local
	internet address = 192.168.142.102
	ttl = 0
    AUTHORITY RECORDS:
    ADDITIONAL RECORDS:
------------
Name:	cloud.apps.servicemesh.local
Address: 192.168.142.102
```

Final `/etc/resolv.conf` file

Note: Do not put search `servicemesh.local` or `cluster.local` in this file.
The `nameserver` should point to its own dnsmasq running on the same host at 53.

```bash
cat /etc/resolv.conf
# Generated by NetworkManager
nameserver 192.168.142.102
```

## Network Interface Configuration

The documentation says that we should keep NM_CONTROLLED to YES. This is done so that to automate DNS configuration. This is not the case with us where we want to build a single VM and wild card name resolution for apps should happen inside the VM only. 

Edit `ifcfg-eth0` for `NM_CONTROLLED="no"`. The `peerdns` is set to `yes` and DNS is set to itself IP address as we are using `dnsmasq` on port 53. Openshift uses `skydns` at port `8053`, so our `dnsmasq` should forward `cluster.local` domain queries to `skydns`.

```bash
# cat /etc/sysconfig/network-scripts/ifcfg-eth0
TYPE="Ethernet"
NM_CONTROLLED="no"
PROXY_METHOD="none"
BROWSER_ONLY="no"
BOOTPROTO="none"
DEFROUTE="yes"
IPV4_FAILURE_FATAL="yes"
IPV6INIT="no"
IPV6_AUTOCONF="yes"
IPV6_DEFROUTE="yes"
IPV6_FAILURE_FATAL="no"
IPV6_ADDR_GEN_MODE="stable-privacy"
NAME="eth0"
DEVICE="eth0"
ONBOOT="yes"
IPADDR="192.168.142.102"
PREFIX="24"
GATEWAY="192.168.142.2"
PEERDNS="yes"
DNS="192.168.142.102"
IPV6_PRIVACY="no"
```

After making changes, restart `network`.
```
systemctl restart network
```



### Check interfaces are in control of NM

```bash
# nmcli device
DEVICE   TYPE      STATE      CONNECTION 
eth0     ethernet  unmanaged  --          
lo       loopback  unmanaged  --  
```

## Meet prerequisites - Mostly development related

```bash
yum install -y  wget git zile nano net-tools docker-1.13.1 bind-utils iptables-services bridge-utils bash-completion kexec-tools sos psacct openssl-devel httpd-tools NetworkManager python-cryptography python2-pip python-devel  python-passlib  java-1.8.0-openjdk-headless "@Development Tools"
```

## Install Ansible-2.6.5 

We need to install `ansible-2.6.5` and without using version, it will install 2.7 which does not work with OpenShift 3.11.

```bash
yum -y install centos-release-ansible26
yum provides ansible
yum -y install ansible-2.6.5-1.el7.noarch
```

Uncomment the timer plugin and add `profile_tasks, timer` to show the elapsed time in the output.

```
sed -i 's/#callback_whitelist.*/callback_whitelist = profile_tasks, timer /g' /etc/ansible/ansible.cfg
```

## Configure local registry for insecure connection

Edit `/etc/containers/registries.conf` and change the following: 

from

```
[registries.insecure]
registries = []
```

to

```
[registries.insecure]
registries = [172.30.0.0/16]
```

Also Edit `/etc/docker/daemon.json` and add the entry as shown.

```json
{
    "insecure-registries" : [ "172.30.0.0/16" ]
}
```

Restart Docker

```
systemctl daemon-reload
systemctl restart docker
```

The address `172.30.0.0` is the `serviceSubnet` defined in `/etc/origin/master/master-config.yaml`.

Install OpenShift Origin, which is the Open Source implementation of Red Hat OpenShift.

The project name [OpenShift Origin] is changed to [OKD] starting version 3.10.

The latest version is 3.11 at the time of writing.

To install OpenShift in single VM, multiple options are available.

Please choose method - 3 for install. You can practice method - 2.

## Method - 1 : Install Using MiniShift

[Minishift Install](https://docs.okd.io/latest/minishift/getting-started/preparing-to-install.html) - You can explore this but we are not showing this as an example. This option is very good if you are a developer and need a small foot print.

## Method - 2 : oc cluster up

This is a simple approach to build a quick Kubernetes environment in a single VM.

```
yum -y install origin-clients-3.11.0
```

```bash
# oc cluster up
Getting a Docker client ...
... OMITTED ...
Login to server ...
Creating initial project "myproject" ...
Server Information ...
OpenShift server started.

The server is accessible via web console at:
    https://127.0.0.1:8443

You are logged in as:
    User:     developer
    Password: <any value>

To login as administrator:
    oc login -u system:admin

[root@osc02 docker]# oc login -u system:admin
Logged into "https://127.0.0.1:8443" as "system:admin" using existing credentials.

You have access to the following projects and can switch between them with 'oc project <projectname>':

    default
    kube-dns
    kube-proxy
    kube-public
    kube-system
  * myproject
    openshift
    openshift-apiserver
    openshift-controller-manager
    openshift-core-operators
    openshift-infra
    openshift-node
    openshift-service-cert-signer
    openshift-web-console

Using project "myproject".

[root@osc02 docker]# oc get pods -n kube-system
NAME                                READY     STATUS    RESTARTS   AGE
kube-controller-manager-localhost   1/1       Running   0          6m
kube-scheduler-localhost            1/1       Running   0          6m
master-api-localhost                1/1       Running   0          6m
master-etcd-localhost               1/1       Running   0          6m
```

Caveat: You can not login remote to the OpnShift Web Console. It is available through http://localhost:8443 unless you do proxy or port forwarding.

This is a very good way for the development environment.

### Bring down OKD cluster 

```
# oc cluster down 

```

## Method - 3 : Steps to build Origin using an inventory file and Ansible playbooks

Even though, we are using single VM but this method will also be applicable to do a multi-node install. 

### Install from Github repo

Using Github, we are choosing a specific fix `openshift-ansible-3.11.88-1` in 3.11.

```bash
git clone https://github.com/openshift/openshift-ansible.git
cd openshift-ansible
git branch -a # List all branches
git tag -l | grep 3.11. # List all tags
git checkout tags/openshift-ansible-3.11.88-1
git describe
openshift-ansible-3.11.88-1
```

## Configuring inventory file for Install

```
yum -y install epel-release
# Disable the EPEL repository globally so that is not accidentally used during later steps of the installation
yum-config-manager --disable epel
yum -y --enablerepo=epel install pyOpenSSL
```

Reference purpose only: You can skip to Final `inventory.ini` and just copy the file. You do not have to download the inventory.ini.

> Consult this [link](https://github.com/gshipley/installcentos) for the inventory file for a single VM and using CentOS.

```bash
cd
curl -o inventory.download https://raw.githubusercontent.com/gshipley/installcentos/master/inventory.ini
```

> Define variable so that they can be replaced in the inventory file.

```bash
export IP=192.168.142.102
export VERSION=3.11
export METRICS=true
export LOGGING=false
export DOMAIN=servicemesh.local
export API_PORT=8443
envsubst < inventory.download > inventory.ini
```

After taking above inventory file, this is final inventory file that we used to install a single VM OpenShift cluster without having to use an external DNS server.

We disabled catalog service feature by setting `openshift_enable_service_catalog=False` to `false` in the inventory file. We will install it after basic OKD install.

Final - Inventory file after environment substitution and by making some changes. Copy this file.

```
mkdir -p /etc/origin/master/
touch /etc/origin/master/htpasswd
```

```bash
# cat inventory.ini 
[OSEv3:children]
masters
nodes
etcd

[masters]
osc02.servicemesh.local openshift_ip=192.168.142.102 openshift_schedulable=true 

[etcd]
osc02.servicemesh.local openshift_ip=192.168.142.102

[nodes]
osc02.servicemesh.local openshift_ip=192.168.142.102 openshift_schedulable=true openshift_node_group_name="node-config-all-in-one"

[OSEv3:vars]
#openshift_additional_repos=[{'id': 'centos-paas', 'name': 'centos-paas', 'baseurl' :'https://buildlogs.centos.org/centos/7/paas/x86_64/openshift-origin311', 'gpgcheck' :'0', 'enabled' :'1'}]

ansible_ssh_user=root
enable_excluders=False
enable_docker_excluder=False
ansible_service_broker_install=False
openshift_enable_service_catalog=False

containerized=True
os_sdn_network_plugin_name='redhat/openshift-ovs-multitenant'
openshift_disable_check=disk_availability,docker_storage,memory_availability,docker_image_availability

deployment_type=origin
openshift_deployment_type=origin

template_service_broker_selector={"region":"infra"}
openshift_metrics_image_version="v3.11"
openshift_logging_image_version="v3.11"
openshift_logging_elasticsearch_proxy_image_version="v1.0.0"
openshift_logging_es_nodeselector={"node-role.kubernetes.io/infra":"true"}
logging_elasticsearch_rollout_override=false
osm_use_cockpit=true

openshift_metrics_install_metrics=true
openshift_logging_install_logging=false

openshift_master_identity_providers=[{'name': 'htpasswd_auth', 'login': 'true', 'challenge': 'true', 'kind': 'HTPasswdPasswordIdentityProvider'}]
openshift_master_htpasswd_file='/etc/origin/master/htpasswd'

openshift_public_hostname=osc02.servicemesh.local
openshift_master_default_subdomain=apps.servicemesh.local
openshift_master_api_port=8443
openshift_master_console_port=8443
```

## Check prerequisites playbook

```bash
ansible-playbook -i inventory.ini openshift-ansible/playbooks/prerequisites.yml
```

## Copy `/etc/resolv.conf` since we have NM_CONTROLLED=no in our case

Make sure that your `/etc/resolve.conf` is as follows.

```bash
# cat /etc/resolv.conf
nameserver 192.168.142.102
```

And, then copy `/etc/resolv.conf` to `/etc/origin/node/`

```
mkdir -p /etc/origin/node
cp /etc/resolv.conf /etc/origin/node/resolv.conf
```

Documentation link for  - [OpneShift 3.11 install](https://docs.openshift.com/container-platform/3.11/install/running_install.html#advanced-retrying-installation)

Install can be done in a single step or by taking individual steps as above link. 

## Run installer playbook without service-catalog
```bash
ansible-playbook -i inventory.ini openshift-ansible/playbooks/deploy_cluster.yml 
```
### Check pods status

```
oc get pods --all-namespaces
Unable to connect to the server: x509: certificate is valid for 172.30.0.1, 192.168.142.102, not 127.0.0.1
```

If you get the above error message, run the following commands to fix the kube `config` file.

``` bash
 cd 
 cd .kube/
 mv config config_old    #take a backup of the config
 cp /etc/origin/master/admin.kubeconfig ~/.kube/config
 oc status --config=/etc/origin/master/admin.kubeconfig
```
`oc get pods --all-namespace` should work now

## Add user id and password

```bash
htpasswd -b /etc/origin/master/htpasswd admin admin

# Grant cluster admin role to user admin 
oc adm policy add-cluster-role-to-user cluster-admin admin
oc login -u admin -p admin https://osc02.servicemesh.local:8443
```

## Install Service Catalog

```
ansible-playbook -i inventory.ini openshift-ansible/playbooks/openshift-service-catalog/config.yml
```

Now, the above will fail to bring the `apiserver` due to `etcd` readiness failure. This is due to hostname of etcd server getting messed up due to things that I do not know. 

But, when I change the `etcd-servers` in `oc -n kube-service-catalog edit ds apiserver` from FQDN to IP address, it worked fine.

Changed this
```
        - --etcd-servers
        - https://oce02.servicemesh.local:2379

```
to
```
        - --etcd-servers
        - https://192.168.142.102:2379
=
```

## Install Bookinfo and Bookapps applications

### Create istio-lab project

```
oc new-project istio-lab
```

### Download and install Istio bookinfo application

```
curl -L https://raw.githubusercontent.com/istio/istio/master/samples/bookinfo/platform/kube/bookinfo.yaml -o bookinfo.yaml

oc adm policy add-scc-to-user anyuid system:serviceaccount:istio-lab:default
oc adm policy add-scc-to-user privileged -z default -n istio-lab
oc apply -f bookinfo.yaml 
```
Output:
```
service/details created
deployment.extensions/details-v1 created
service/ratings created
deployment.extensions/ratings-v1 created
service/reviews created
deployment.extensions/reviews-v1 created
deployment.extensions/reviews-v2 created
deployment.extensions/reviews-v3 created
service/productpage created
deployment.extensions/productpage-v1 created

[root@osc02 istio]# oc get pods
NAME                              READY     STATUS    RESTARTS   AGE
details-v1-68868454f5-kqkqz       1/1       Running   0          19h
productpage-v1-5cb458d74f-pl2dc   1/1       Running   0          44s
ratings-v1-76f4c9765f-c574q       1/1       Running   0          19h
reviews-v1-56f6855586-l77fw       1/1       Running   0          13s
reviews-v2-65c9df47f8-nkrdv       1/1       Running   0          13s
reviews-v3-6cf47594fd-dfcdj       1/1       Running   0          13s
```

### Download and install Linkerd bookinfo application

```
curl -L https://run.linkerd.io/booksapp.yml -o booksapp.yaml
oc new-project linkerd-lab
oc apply -f booksapp.yaml 
```
Output:
```
service/webapp created
deployment.extensions/webapp created
service/authors created
deployment.extensions/authors created
service/books created
deployment.extensions/books created
deployment.extensions/traffic created
```

You are done now. The following commands are for reference purpose only.

## Uninstall

If Origin (OKD) needs to be uninstalled, run the command.

```bash
# cd /etc/ansible
# ansible-playbook -i hosts /usr/share/ansible/openshift-ansible/playbooks/adhoc/uninstall.yml -e openshift_uninstall_images=False
```

The switch `-e openshift_uninstall_images=False` allows to retain the docker images downloaded. Without the switch, the docker images are also deleted.



## Miscellaneous Commands

Get logs from dead (stopped) containers.

```
docker ps -a | grep -v CONTAINER| awk '{system("docker logs "$1)}'
```

### What is my IP

```bash
curl -s ipinfo.io/ip
```

### What is my local IP address

```
ip route get 8.8.8.8 | awk '{print $NF; exit}'
hostname -i
```

### Enable / Disable repo commands - example

```bash
# yum-config-manager --enable docker-ce-stable
# yum-config-manager --disable docker-ce-stable
```

### Network Manager
NetworkManager, a program for providing detection and configuration for systems to automatically connect to the network, is required on the nodes in order to populate dnsmasq with the DNS IP addresses.

NM_CONTROLLED is set to yes by default. If NM_CONTROLLED is set to no, then the NetworkManager dispatch script does not create the relevant origin-upstream-dns.conf dnsmasq file, and you would need to configure dnsmasq manually.

In almost all cases, when referencing VMs you must use host names, and the host names that you use must match the output of the hostname -f command on each node.
	
In your /etc/resolv.conf file on each node host, ensure that the DNS server that has the wildcard entry is not listed as a nameserver or that the wildcard domain is not listed in the search list. Otherwise, containers managed by OKD may fail to resolve host names properly.

### etcd health check

```
etcdctl --cert-file /etc/etcd/peer.crt --key-file /etc/etcd/peer.key --ca-file /etc/etcd/ca.crt --endpoints https://192.168.142.102:2379 cluster-health
```

### catalog api server

```
/usr/bin/service-catalog apiserver --storage-type etcd --secure-port 6443 --etcd-servers       https://osc02.servicemesh.local:2379 --etcd-cafile /etc/origin/master/master.etcd-ca.crt       --etcd-certfile /etc/origin/master/master.etcd-client.crt --etcd-keyfile /etc/origin/master/master.etcd-client.key -v 3 --cors-allowed-origins localhost       --enable-admission-plugins NamespaceLifecycle,DefaultServicePlan,ServiceBindingsLifecycle,ServicePlanChangeValidator,BrokerAuthSarCheck --feature-gates OriginatingIdentity=true       --feature-gates NamespacedServiceBroker=true
```
### oc cluster without service catalog 
oc cluster up --service-catalog

### Customizing Service Catalog Options
The service catalog is enabled by default during installation. Enabling the service broker allows you to register service brokers with the catalog. When the service catalog is enabled, the OpenShift Ansible broker and template service broker are both installed as well; see Configuring the OpenShift Ansible Broker and Configuring the Template Service Broker for more information. If you disable the service catalog, the OpenShift Ansible broker and template service broker are not installed.
```
openshift_enable_service_catalog=false
```

### When NM_CONTROLLED is set to NO

CRD not creating - router problem
when NM_CONTROLLED is set to no for ifcfg-eth0

```
start_network.go:106] could not start DNS, unable to read config file: open /etc/origin/node/resolv.conf: no such file or directory

cp /etc/resolv.conf /etc/origin/node/resolv.conf
oc -n openshift-sdn delete sdn-skbqt
```

### If need to insatll from the CentOS repo

```bash
yum -y install centos-release-openshift-origin311
yum -y install origin-clients
```

### etcd diagnostics

```
etcdctl --cert-file /etc/origin/master/master.etcd-client.crt --key-file /etc/origin/master/master.etcd-client.key --ca-file /etc/origin/master/master.etcd-ca.crt --endpoints https://osc02.servicemesh.local:2379 cluster-health

member 68588e2e5fbc1a86 is healthy: got healthy result from https://192.168.142.102:2379
cluster is healthy

etcdctl --cert-file /etc/origin/master/master.etcd-client.crt --key-file /etc/origin/master/master.etcd-client.key --ca-file /etc/origin/master/master.etcd-ca.crt --endpoints https://osc02.servicemesh.local:2379 member list

68588e2e5fbc1a86: name=osc02.servicemesh.local peerURLs=https://192.168.142.102:2380 clientURLs=https://192.168.142.102:2379 isLeader=true

# oc -n kube-system exec -it master-etcd-osc02.servicemesh.local sh
/var/lib/etcd # source /etc/etcd/etcd.conf
/var/lib/etcd # ETCDCTL_API=3 etcdctl --cert=$ETCD_PEER_CERT_FILE --key=$ETCD_PEER_KEY_FILE --cacert=$ETCD_TRUSTED_CA_FILE --endpoints=$ETCD_LISTEN_CLIENT_URLS endpoint he
alth
https://192.168.142.102:2379 is healthy: successfully committed proposal: took = 1.270261ms
```
