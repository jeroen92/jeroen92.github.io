---
title: "Homelab"
publishDate: "2021-11-09"
tags:
  - reddit
  - homelab
aliases:
  - "/homelab"
---
This is a crosspost of my [Reddit post](https://www.reddit.com/r/homelab/comments/qtcpx0/how_a_hobby_unfolded_into_running_a_small/) on `r/homelab`. It's about my homelab I built during fall 2021. First off, it basically consists of three parts. First and most important is the OSPF backbone that runs on top of three site-to-site IPsec VPN tunnels. These tunnels are powered by Strongswan running on a VPS, an EdgeRouter and a Juniper SRX300.

{{< zoom-img src="/posts/images/network_diagram.png" >}}

The second layer, if you will, runs all core infrastructure services such as DNS (authoritative and recursors), PXE boot using TFTP and nginx, LDAP multi master and DHCP. Each of these services has an anycast IP assigned. Each site partaking in the PtP IPsec backbone runs a mirror of these services. For two sites, these Docker containers are running on a Raspberry Pi and on the VPS this stuff runs on well, the VPS :-).

The Pis and VPS also run a Bird Docker instance. This container is accompanied with a watchdog to monitor the health of the Docker containers (e.g. the DNS recursor), and injects the container's anycast IP into OSPF as long as that container is healthy. This has been running for a little over a year now and it's by far the best way I've ever got this part of the infrastructure high available. All this stuff (containers + authoritative DNS) is managed using Terraform by the way.

**Last is the actual homelab ;)**

![your image](/posts/images/rack.jpg)

The network specs:
- Juniper SRX300
- Juniper EX2300-24T (2x)
- Cisco 2951 (unused)

At my home I'm running OSPF Area 64 with the SRX being the ABR. The SRX and EX switches are connected by a simple triangle setup using 1Gbit fiber between the SRX and EX switches. The EX switches are connected using two 10G fiber links which are heavily underutilized at the moment due to the way the servers are connected. I've got MSTP enabled, so one of these two 10G links is disabled, until it isn't. The switches also run VRRP to provide high available gateway IPs for my wired and wireless client networks.

**Storage specs:**
- Supermicro A2SDi-H-TF
- 128GB ECC RAM
- 512GB NVMe
- 3x8TB WD Red
- Broadcom 9400-8i HBA
- Supermicro SC826BE1C4-R1K23LPB chassis

This is, unfortunately, my only FreeBSD host. It isn't doing much aside of running my ZFS pool and exporting some datasets over NFS. It has two 10Gbase-T ports which connects this machine to both switches. Bird joins this machine to area 64, advertising its loopback address. The 128GB of RAM is nice for ARC, though still a bit small for my datasets. The chassis has room for U.2 NVMe drives, so I'm contemplating getting myself a nice L2ARC device once I can get my hands on a nice deal.Hypervisor/Kubernetes specs:
- Supermicro X11SSH-LN4F, Xeon E3-1230 v6, 64GB ECC RAM, 512GB NVMe, 3x 2.5" 250GB EVO 850 SSD
- Supermicro X11SSH-LN4F, Xeon E3-1230 v6, 64GB ECC RAM, 512GB NVMe, 3x 2.5" 250GB EVO 850 SSD
- Supermicro X11SSH, Xeon E3-1220 v6, 64GB ECC RAM, 256GB NVMe

These three hypervisors run CentOS 8 and libvirt/KVM/Qemu. Their main purpose is to host VMs running my Kubernetes cluster. These VMs are managed by Terraform using the libvirt provider. Unfortunately this provider doesn't yet support bhyve (or rather, libvirt doesn't expose the required host capabilities when connecting to bhyve). Otherwise I'd definitely run BSD on these boxes.

I've set up Kubernetes the hard way in the past, but nowadays I'm in ez mode using Rancher. Using Calico as CNI maily for its extensive configurability and multi IP pool support. MetalLB peers with Bird running on the hypervisors to publish service IPs outside of the cluster. The first two hypervisors have their three SSDs passed through to the k8s VMs. Longhorn uses these drives to let my workloads consume persistent volumes. On my previous clusters I used Ceph, but I found that required a bit more configuration and resources than Longhorn. Everything that's not core infrastructure (e.g. doesn't run on the Pis) has to run in k8s. I don't have a lot running yet; mainly Plex, Gitlab, Overleaf, some cronjobs for backups, a collector I wrote myself to export PSU metrics from IPMI, and of course Prometheus and a bunch of exporters.

In the near future I'll mainly use the idle resources on this cluster for some personal software projects that require some APIs, queues, databases and batch jobs to run. I'd also like to do a bit of autoscaling to shutdown at least one of the three nodes when the cluster is idling.
