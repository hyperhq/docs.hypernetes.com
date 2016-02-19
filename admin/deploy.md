# Hypernetes deployment

This document covers the basic depolyment procedure for Hypernetes. If you are looking for detailed depolyment of Kubernetes, refer to [http://kubernetes.io/gettingstarted/](http://kubernetes.io/gettingstarted/).

## Deploy OpenStack with Ceph

Since Hypernetes is working together with OpenStack, a OpenStack cluster must be deployed first. There are a lot of OpenStack deployment solutions, including

* [OpenStack Installation Guide for Ubuntu 14.04](http://docs.openstack.org/kilo/install-guide/install/apt/content/)
* [OpenStack Installation Guide for Red Hat](http://docs.openstack.org/kilo/install-guide/install/yum/content/)
* [RDO](https://www.rdoproject.org/Main_Page)
* [MAAS](http://www.ubuntu.com/download/cloud/install-ubuntu-openstack)
* [Fuel](https://www.mirantis.com/products/mirantis-openstack-software/)
* [devstack](http://docs.openstack.org/developer/devstack/)
* [Puppet](https://github.com/puppetlabs/puppetlabs-openstack)
* And so on

Choose any tool you like to deploy a new OpenStack cluster, or you can just re-use your existing OpenStack environment.

Don't forget to deploy neutron L2 agent and Ceph client for Kubernetes nodes.

## Deploy Kubernetes

Since Hypernetes is Kubernetes based, you can deploy Hypernetes following the same procedure as kubernetes. Here are some distribution-specific links:

* [deploy Kubernetes on centos](http://kubernetes.io/v1.0/docs/getting-started-guides/centos/centos_manual_config.html)
* [deploy Kubernetes on ubuntu](http://kubernetes.io/v1.0/docs/getting-started-guides/ubuntu.html)
* [deploy Kubernetes on fedora](http://kubernetes.io/v1.0/docs/getting-started-guides/fedora/fedora_ansible_config.html)

See more at [Kubernetes getstarted guide](http://kubernetes.io/gettingstarted/)

## Deploy KubeStack

KubeStack is an OpenStack network provider for Kubernetes and it is deployed on all Kubernetes masters and nodes.

```shell
mkdir -p $GOPATH/src/github.com/hyperhq
cd $GOPATH/src/github.com/hyperhq
git clone https://github.com/hyperhq/kubestack.git
cd kubestack
make && make install
```

Configure KubeStack

```shell
# cat /etc/kubestack.conf
[Global]
auth-url = http://keystone-server:5000/v2.0
username = admin
password = admin
tenant-name = admin
region = RegionOne
ext-net-id = <your-external-network-id>

[LoadBalancer]
create-monitor = yes
monitor-delay = 1m
monitor-timeout = 30s
monitor-max-retries = 3

[Plugin]
plugin-name = ovs
```

```shell
# cat /usr/lib/systemd/system/kubestack.service
[Unit]
Description=OpenStack Network Provider for Kubernetes
After=syslog.target network.target openvswitch.service
Requires=openvswitch.service

[Service]
ExecStart=/usr/local/bin/kubestack \
  -logtostderr=false -v=4 \
  -group=kube \
  -log_dir=/var/log/kubestack \
  -conf=/etc/kubestack.conf
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

```shell
systemctl start kubestack.service
```

## Configure Kubernetes

Disable selinux

```shell
sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config
setenforce 0
```

Create the service account key

```shell
mkdir /var/lib/kubernetes/
openssl genrsa -out /var/lib/kubernetes/serviceaccount.key 2048
chown kube:kube /var/lib/kubernetes/serviceaccount.key
```

Create Kubernetes log dir

```shell
mkdir /var/log/kubernetes
chown kube:kube /var/log/kubernetes
mkdir /var/run/kubernetes/
chown kube:kube /var/run/kubernetes/
```

Configure etcd

```shell
cat >> /etc/etcd/etcd.conf <<EOF
ETCD_LISTEN_CLIENT_URLS="http://kube-master:2379"
EOF
```

Common configs for all Kubernetes services

```shell
# cat /etc/kubernetes/config
# logging to stderr means we get it in the systemd journal
KUBE_LOGTOSTDERR="--logtostderr=false --log-dir=/var/log/kubernetes"
# journal message level, 0 is debug
KUBE_LOG_LEVEL="--v=4”
# Should this cluster be allowed to run privileged docker containers
KUBE_ALLOW_PRIV="--allow_privileged=false"
# How the controller-manager, scheduler, and proxy find the apiserver
KUBE_MASTER="--master=http://kube-master:8080"
```

Configure kube-apiserver

```
# cat /etc/kubernetes/apiserver
# The address on the local server to listen to.
KUBE_API_ADDRESS="--insecure-bind-address=127.0.0.1"
# The port on the local server to listen on.
# KUBE_API_PORT="--port=8080"
# Port minions listen on
# KUBELET_PORT="--kubelet_port=10250"
# Comma separated list of nodes in the etcd cluster
KUBE_ETCD_SERVERS="--etcd_servers=http://kube-master:2379"
# Address range to use for services
KUBE_SERVICE_ADDRESSES="--service-cluster-ip-range=10.254.0.0/16"
# default admission control policies
KUBE_ADMISSION_CONTROL="--admission_control=NamespaceLifecycle,NamespaceExists,LimitRanger,SecurityContextDeny,ServiceAccount,ResourceQuota"
# Add your own!
KUBE_API_ARGS="--service-account-key-file=/var/lib/kubernetes/serviceaccount.key"
```

Configure kube-controller-manager

```shell
# cat /etc/kubernetes/controller-manager
###
# The following values are used to configure the Kubernetes controller-manager
# defaults from config and apiserver should be adequate

# Add your own!
KUBE_CONTROLLER_MANAGER_ARGS="--service-account-private-key-file=/var/lib/kubernetes/serviceaccount.key --network-provider=openstack"
```

Configure kube-proxy

```shell
# cat /etc/kubernetes/proxy
# Kubernetes proxy config
# default config should be adequate

# Add your own!
KUBE_PROXY_ARGS="--proxy-mode=haproxy"
```

Configure kubelet

```shell
# cat /etc/kubernetes/kubelet
# The address for the info server to serve on (set to 0.0.0.0 or "" for all interfaces)
KUBELET_ADDRESS="--address=0.0.0.0"
# The port for the info server to serve on
# KUBELET_PORT="--port=10250"
# You may leave this blank to use the actual hostname
KUBELET_HOSTNAME=""
# location of the api-server
KUBELET_API_SERVER="--api_servers=http://kube-master:8080"
# Add your own!
KUBELET_ARGS="--container-runtime=hyper --network-provider=openstack --cinder-config=/etc/kubernetes/cinder.conf"

# cat /etc/kubernetes/cinder.conf
[Global]
auth-url = http://keystone-server:5000/v2.0
username = admin
password = admin
tenant-name = admin
region = RegionOne

[RBD]
keyring = "AQAtqv9V3u4nKRA8Cxfic687DqPW1FV/rly3nw=="
```

## Start services

```shell
systemctl restart etcd
systemctl restart hyperd
systemctl restart kubestack
systemctl restart kube-apiserver.service
systemctl restart kube-scheduler.service
systemctl restart kube-controller-manager.service
systemctl restart kubelet.service
systemctl restart kube-proxy
````
