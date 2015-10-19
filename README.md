# Hypernetes architecture

Hypernetes is a secure, multi-tenant CaaS powered by Hyper, Kubernetes and OpenStack. Simply put:

Hypernetes = KeyStone + Cinder/Neutron + Hyper + Kubernetes

Hypernetes ensures:

- multi-tenancy (together with Keystone)
- network isolation (by Neutron)
- persistent storage management (by Cinder)
- container orchestration (by Kubernetes)

![Architecture Diagram](architecture.png?raw=true "Architecture overview")

### About Hyper

Hyper is a hypervisor-agnostic Docker runtime. It allows running Docker images with any hypervisor (KVM, Xen, Vbox, ESX). Hyper is different from the minimalist Linux distros like CoreOS by the fact that Hyper runs on the physical box and loads the Docker images from the metal into the VM instance, in which no guest OS is present. Instead of virtualizing a complete operating system, Hyper boots a minimalist kernel in the VM to host the Docker images (Pod).

With this approach, Hyper is able to bring some encouraging merits:

- 300ms to boot a new HyperVM instance with a Pod of Docker images
- 20MB for min Mem footprint of a HyperVM instance
- Immutable HyperVM, only kernel+images, serving as atomic unit (Pod) for scheduling
- Immune from the shared kernel problem in LXC – i.e. isolated by VM
- Work seamlessly with OpenStack components, e.g. Neutron, Cinder, due to the nature of a hypervisor
- BYOK, bring-your-own-kernel is somewhat mandatory for a public cloud platform
