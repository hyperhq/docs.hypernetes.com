# Hypernetes user guide

## Client Installation

**Install OpenStack Client**

```shell
sudo pip install python-openstackclient
```

**Setup OpenStack auth environment variables, such as**

```shell
unset OS_SERVICE_TOKEN
export OS_USERNAME=username
export OS_PASSWORD=password
export OS_AUTH_URL=http://auth-server:5000/v2.0
export OS_TENANT_NAME=tenant-name
export OS_REGION_NAME=RegionOne
```

**Install kubectl**

Download kubernetes-client at <https://github.com/hyperhq/hypernetes/releases> and extract `kubectl` to your system PATH.

**Setup kubectl options**

```shell
kubectl config set-cluster default --server=http://kubernetes-master:8080 --insecure-skip-tls-verify=true
kubectl config set-context default --cluster=default
kubectl config use-context default
```

## Manage network

Create network spec `network.yaml` with subnet `192.168.0.0/24`

```yaml
apiVersion: v1
kind: Network
metadata:
  name: net1
spec:
  tenantID: 065f210a2ca9442aad898ab129426350
  subnets:
    subnet1:
      cidr: 192.168.0.0/24
      gateway: 192.168.0.1
```

```
# kubectl create -f ./network.yaml
network "net1" created
# kubectl get network
NAME      SUBNETS          PROVIDERNETWORKID    LABELS    STATUS
net1      192.168.0.0/24                        <none>    Active
```

This operation will create a new Neutron network with a default router and a subnet `192.168.0.0/24` automatically.

## Manage namespace

You can create a namespace with or without network. A namespace without network is only suggested for administration because there is no network isolation. For users, a network is required to create a namespace.

Create namespace spec `namespace.yaml` and set its network to `net1`

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: ns1
spec:
  network: net1
```

```
# kubectl create -f ./namespace.yaml
namespace "ns1" created
# kubectl get namespace
NAME      LABELS    STATUS    AGE
default   <none>    Active    30d
ns1       <none>    Active    6m
```

## Manage Pod

Create Pod spec `pod-ns1.yaml` and set its namespace to `ns1`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  namespace: ns1
  labels:
    app: nginx
spec:
  containers:
  - name: nginx
    image: nginx
    ports:
    - containerPort: 80
```

```
# kubectl create -f ./pod-ns1.yaml
pod "nginx" created
# kubectl --namespace=ns1 get pod
NAME      READY     STATUS    RESTARTS   AGE
nginx     1/1       Running   0          2m
```

## Manage Pod with Cinder volume

Hypernetes [Volumes](https://kubernetes.io/docs/concepts/storage/volumes/) support native Cinder volumes (only rbd backend is supported now), which means that you can simply create a Pod with Cinder volume. A sample Pod with Cinder volume `651b2a7b-683e-47e1-bdd6-e3c62e8f91c0` is

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  containers:
  - name: nginx
    image: nginx
    ports:
    - containerPort: 80
    volumeMounts:
    - name: nginx-persistent-storage
      mountPath: /var/lib/nginx
  volumes:
  - name: nginx-persistent-storage
    cinder:
      volumeID: 651b2a7b-683e-47e1-bdd6-e3c62e8f91c0
      fsType: ext4
```

## Manage service

Create a service spec `nginx-ns1.yaml` with namespace `ns1` and type `NetworkProvider` (Service must be type `NetworkProvider` if its namespace is with a network):

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
  namespace: ns1
spec:
  type: NetworkProvider
  ports:
  - port: 8078
    name: http
    targetPort: 80
    protocol: TCP
  selector:
    app: nginx
```

```
# kubectl create -f ./nginx-ns1.yaml
service "nginx" created
# kubectl --namespace=ns1 get svc nginx
NAME      CLUSTER_IP       EXTERNAL_IP   PORT(S)    SELECTOR    AGE
nginx     10.254.223.206                 8078/TCP   app=nginx   10m
```

Cluster service `10.254.223.206:8078` can be only visited on Pods in namespace `ns1`.

Now let's create another service `nginx2-ns1.yaml`. It's the same configuration as `nginx` service, but with externalIP `23.23.0.30`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx2
  namespace: ns1
spec:
  type: NetworkProvider
  externalIPs:
  - 23.23.0.30
  ports:
  - port: 8078
    name: http
    targetPort: 80
    protocol: TCP
  selector:
    app: nginx
```

```
# kubectl create -f ./nginx2-ns1.yaml
service "nginx2" created
# kubectl --namespace=ns1 get svc nginx2
NAME      CLUSTER_IP      EXTERNAL_IP              PORT(S)    SELECTOR    AGE
nginx2    10.254.154.51   192.168.0.4,23.23.0.30   8078/TCP   app=nginx   1m
```

Notes about service `nginx2`

* Cluster service `10.254.154.51:8078` can be only visited on Pods in namespace `ns1`
* External ip `192.168.0.4` can be visited on all Pods in the same network `net1`, since it is the vip internal load balancer's network
* External ip `23.23.0.30` can be visited on public since it is a public IP

## clean up

```sh
kubectl delete namespace ns1
kubectl delete network net1
```

For a more detailed user guide, see [Kubernetes user guide](https://kubernetes.io/docs/tutorials/)
