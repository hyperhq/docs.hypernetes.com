# Hypernetes

## What is Hypernetes?

Hypernetes is a secure, multi-tenant [Kubernetes](http://kubernetes.io) distro. Simply put,

Hypernetes = Bare-metal + [Hyper](https://hyper.sh) + Kubernetes + [KeyStone](https://wiki.openstack.org/wiki/Keystone) + [Cinder](https://wiki.openstack.org/wiki/Cinder) + [Neutron](https://wiki.openstack.org/wiki/Neutron).

## What does Hypernetes do?

Hypernetes envisions the future of ***"Container-as-a-Service without IaaS"***. The idea is to combine the orchestration power of Kubernetes with [the runtime isolation of Hyper](https://hyper.sh/why-hyper.html) to build the truly secure multi-tenant CaaS platform.

![](https://trello-attachments.s3.amazonaws.com/55545e127c7cbe0ec5b82f2b/1660x705/895000bf0d7e25aee600d3cfaf0fd3f2/upload_10_19_2015_at_3_02_11_PM.png)

Hypernetes also integrates with a number of OpenStack projects:

- Keystone (Authentication)
- Neutron (SDN)
- Cinder/Ceph (Persistent Storage)

## How Hypernetes works?

![Architecture Diagram](architecture.png?raw=true "Architecture overview")

