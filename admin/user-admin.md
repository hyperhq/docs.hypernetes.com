# Hypernetes tenant and user management

This document aims to describe how to manage tenants and users in Hypernetes.

### Tenant management

Hypernetes manages tenants and users using OpenStack Keystone. To create a tenant, you must create it in Keystone first, then expose it to Kubernetes:

Create OpenStack `demo` tenant

```sh
openstack project create demo
```

Create a spec file of demo tenant `demo-tenant.yaml`

```yaml
apiVersion: v1
kind: Tenant
metadata:
  name: demo
```

Create demo tenant

```sh
# kubectl create -f ./demo-tenant.yaml
# kubectl get tenant
NAME      LABELS    STATUS    AGE
default   <none>    Active    23h
demo      <none>    Active    57m
```

### User management

You can simply add users to any existing tenant using an OpenStack client. For exmaple, these commands add a `demouser` to tenant `demo`:

```sh
openstack user create --password demo demouser
openstack role add --project demo --user demouser _member_
```
