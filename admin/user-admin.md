# Hypernetes tenant and user management

This document aims to describe how to manage tenants and users in Hypernetes.

### Tenant management

Hypernetes manages tenants and users using OpenStack Keystone. To create a tenant, you must create it in Keystone first, then expose it to Kubernetes:

Create OpenStack `demo` tenant

```sh
openstack project create demo
```

### User management

You can simply add users to any existing tenant using an OpenStack client. For exmaple, these commands add a `demouser` to tenant `demo`:

```sh
openstack user create --password demo demouser
openstack role add --project demo --user demouser _member_
```
