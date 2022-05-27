- [Introduction](#introduction)
- [Preconditions](#preconditions)
- [Lab Topology](#lab-topology)
- [Resources](#resources)
- [Steps](#steps)
  - [1. Create Vlan, Self IP and Vxlan Tunnel to Kubernetes Cluster via TMSH;](#1-create-vlan-self-ip-and-vxlan-tunnel-to-kubernetes-cluster-via-tmsh)
  - [2 Get the Mac address of the vxlan tunnel from BIGIP:](#2-get-the-mac-address-of-the-vxlan-tunnel-from-bigip)
  - [2b. Ansible playbook f5_modules for above step1 and step2](#2b-ansible-playbook-f5_modules-for-above-step1-and-step2)
  - [3. Install cilium-cli](#3-install-cilium-cli)
  - [4. Install Cilium](#4-install-cilium)
  - [5. Install BIGIP Ingress Controller;](#5-install-bigip-ingress-controller)
- [Feedback](#feedback)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Introduction

Cilium 1.12 has a new feature vtep mode, with which we can integrate Cilium with BIG-IP Ingress Controller(CIS) [CIS config-options](https://clouddocs.f5.com/containers/latest/userguide/config-options.html#config-options). This document provides a demo configuration for this integration.

For how Cilium or ebpf helps to improve Kubernetes performance, please refer:
- [Cilium](https://cilium.io/)
- [ebpf](https://ebpf.io/)

## Preconditions

- [AS3](https://clouddocs.f5.com/products/extensions/f5-appsvcs-extension/latest/) is installed on BIG-IP;
- A Kubernetes cluster is deployment with '--skip-phases=addon/kube-proxy' option;
- [Cilium-cli](https://docs.cilium.io/en/stable/gettingstarted/k8s-install-default/#install-the-cilium-cli) and Kubectl are installed on your laptop;

## Lab Topology

- BIG-IP has 2 Vlans:
  - Public(preconfigured): receives Client Requests(10.170.155.136/24);
  - VlanKube: connects to Kubernetes Cluster(10.80.255.202/24);
- Kubernetes Cluster has 3 nodes(10.80.255.101, 10.80.255.102, 10.80.255.103);
- BIG-IP vtep CIDR: 172.18.202.0/24
- Kubernetes pod CIDR: 10.0.0.0/16

## Resources
```bash
$ tree
.
├── ansible
│   ├── cis-tunnel.yaml
│   ├── inventory
│   │   └── dut.yaml
│   └── vars
│       ├── bip-cis.yaml
│       ├── bip_vars.yaml
│       ├── cis-node.ini
│       └── pass_vars.yaml
├── README.md
└── yamls
    ├── 01-cis-sa.yaml
    ├── 02-cis-secret.yaml
    ├── 03-cis-rbac.yaml
    ├── 04-cis-ingrssclass.yaml
    ├── 05-cis-ctrl-deployment.yaml
    ├── 06-cis-ingress.yaml
    └── svc
        └── 10-svc-cafe.yaml
```

## Steps

### 1. Create Vlan, Self IP and Vxlan Tunnel to Kubernetes Cluster via TMSH;
```bash 
# Create Vlan and Self IP to Kubernetes Cluster
tmsh create net vlan VlanKube { interfaces add { 1.2 } tag 4094 }
tmsh create net self SelfKube { address 10.80.255.202/24 allow-service default traffic-group traffic-group-local-only vlan VlanKube }

# Create Vxlan profile and Tunnel
tmsh create net tunnels vxlan vxlan_profi port 8472 flooding-type multipoint
tmsh create net tunnels tunnel vxlan_cilium key 2 profile vxlan_profi local-address 10.20.255.202
tmsh create net self SelfVxlan address 172.18.202.3/24 allow-service none vlan vxlan_cilium

# Create route to Kubernetes Cluster
tmsh net route toKube { interface /Common/vxlan_cilium network 10.0.0.0/16 }

# Save configuration
tmsh save sys config
```

### 2 Get the Mac address of the vxlan tunnel from BIGIP:
```bash
tmsh show net tunnels tunnel vxlan_cilium all-properties | grep -o -E '([[:xdigit:]]{1,2}:){5}[[:xdigit:]]{1,2}'
```

### 2b. Ansible playbook [f5_modules](https://clouddocs.f5.com/products/orchestration/ansible/devel/f5_modules/getting_started.html) for above step1 and step2

```bash
# This step requires F5 Ansile mod
$ ansible-playbook -i inventory/dut.yaml -e '{"ply_host": "DUT1"}' cis-tunnel.yaml
```

### 3. Install [cilium-cli](https://docs.cilium.io/en/stable/gettingstarted/k8s-install-default/#install-the-cilium-cli)

### 4. Install Cilium
```bash
$ cilium install --version=v1.12.0-rc2 --kube-proxy-replacement strict --config  enable-bpf-masquerade=true,k8sServiceHost=10.80.255.101,k8sServicePort=6443,l7Proxy=true,enable-vtep=true,vtep-endpoint="10.80.255.202",vtep-cidr="172.18.202.0/24",vtep-mac="00:50:56:bd:f4:7a",vtep-mask="255.255.255.0"

# k8sServiceHost, k8sServicePort: the KubeAPI IP and Port
# k8sServiceHost, k8sServicePort: the KubeAPI IP and Port
# vtep-endpoint: BIGIP Self IP on the Vlan to Kubernetes Cluster
# vtep-cidr, vtep-mask: BIGIP Self IP CIDR on the Vxlan Tunnel
# vtep-mac: Mac address of Vxlan Tunnel which get from Step2
```

### 5. Install BIGIP Ingress Controller;
```bash
# Create serviceaccount, secret, namespace and label namespace
$ kubectl create secret generic bigip-login -n kube-system --from-literal=username=admin --from-literal=password=admin
$ kubectl create serviceaccount bigip-ctlr -n kube-system
$ kubectl create namespace mel
$ kubectl label ns mel type=ingressbip

# Create RBAC for CIS(BIG-IP Ingress Controller)
$ kubectl create -f 03-cis-rbac.yaml

# Create IngressClass
$ kubectl create -f 04-cis-ingrssclass.yaml

# Create the CIS(BIG-IP Ingress Controller) deployment 
$ kubectl create -f 05-cis-ctrl-deployment.yaml

# Create the demo Ingress
$ kubectl create -f 06-cis-ingress.yaml -n mel

# Create the demo Service 
$ kubectl create -f svc/10-svc-cafe.yaml -n mel
```

## Feedback
