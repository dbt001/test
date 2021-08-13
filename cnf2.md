```text
SPDX-License-Identifier: Apache-2.0     
Copyright (c) 2020 Intel Corporation  
```
<!-- omit in toc -->
# Converged Edge Reference Architecture for SD-WAN(vpp based)
- [Introduction](#introduction)
- [SD-WAN Implementation](#sd-wan-implementation)
  - [SD-WAN CNF](#sd-wan-cnf)
  - [SD-WAN CRD Controller](#sd-wan-crd-controller)
  - [Custom Resources (CRs)](#custom-resources-crs)
- [CNF Configuration via CRs](#cnf-configuration-via-openwrt-packages)
  - [IPSec](#ipsec)
- [SD-WAN CNF Packet Flow](#sd-wan-cnf-packet-flow)
- [OpenNESS Integration](#openness-integration)
  - [Goals](#goals)
  - [Networking Implementation](#networking-implementation)
  - [Converged Edge Reference Architectures (CERA)](#converged-edge-reference-architectures-cera)
    - [SD-WAN Edge Reference Architecture](#sd-wan-edge-reference-architecture)
    - [SD-WAN Hub Reference Architecture](#sd-wan-hub-reference-architecture)
- [Deployment And Configuration](#deployment-and-configuration)
  - [E2E Scenarios](#e2e-scenarios)
    - [Hardware Specification](#hardware-specification)
    - [Software Specification](#software-specification)
    - [Scenario 1 : Using Flavor to deploy on Hub Node](#scenario-1-using-flavor-to-deploy-on-hub-node)
    - [Scenario 2 : Using Flavor to deploy on Edge Node](#scenario-2-using-flavor-to-deploy-on-edge-node)
    - [Scenario 3 : Verify Data Flow](#scenario-3-verify-data-flow)
    - [Scenario 4 : Verify Data Flow with 3 Single Cluster Manually](#scenario-4-verify-data-flow-with-3-single-cluster-manually)    
- [Resource Consumption](#resource-consumption)
  - [Methodology](#methodology)
  - [Results](#results)
- [References](#references)
- [Acronyms](#acronyms)
- [Terminology](#terminology)

## Introduction
Software-defined WAN (SD-WAN) introduces a new way to operate a WAN. Firstly,  since it is defined by software, its management can be decoupled from the underlying networking hardware (e.g., routers) and managed in a centralized manner, making it more scalable. Secondly, SD-WAN network functions can now be hosted on Universal Customer Premises Equipment (uCPE), which also host software versions of traditional customer premises equipment. Finally, an SD-WAN can be complemented by edge computing solutions, allowing, for example, latency-sensitive traffic to be steered to edge nodes for local processing, and to allow uCPE functions to be hosted in edge nodes. 

There co-exist two SDEWAN (Software-defined edge WAN) solutions in OpenNESS:
  - one is OpenWrt based solution that was added with OpenNESS 20.12 release [here](https://github.com/otcshare/ido-specs/blob/master/doc/reference-architectures/cera_sdwan.md)
  - the other is VPP/DPDK based solution with high-performance featured leveraging user space network stack.

This document targets the latter only and here is the high-level topo.            
![Figure-1 ](sdwan-images/sdewan-topo.png)

According to Figure-1, in Enterprise Network scenario, the SDEWAN of hub node is defined as a gateway of all branch networks, so high-performance VPP is needed at hub node. For example, an OpenWrt based solution is deployed on edge nodes, while high-performance VPP based solution works on traffic hub nodes that work as bridge among edge nodes. 
 
The SDEWAN CNF supports deployment at Hub node and Edge node. It has the same features either in Hub or Edge node. 


## SD-WAN Implementation
The CERA SD-WAN is based on FD.io(vpp), an Open Source Terabit Software Dataplane. FD.io is a networking technology that can used to build a galaxy of Network Functions. Today it is being used by a several major communication network providers and equipment manufactures to build world-class network functions. More details about FD.io can be found [here](https://fd.io/).

The Fast Data Project (FD.io) Universal Dataplane is a collaborative open source project that aims to significantly establish a high-performance IO services framework for dynamic computing environments,so a number of packages of use in implementing SD-WAN functional elements, which are written as FD.io plugins. These include:

  - ABF (for Multiple WAN link support) [mwan](https://fd.io/vppproject/vppfeatures/#acl-based-forwarding)

  - ACL (for firewall) [fw](https://fd.io/vppproject/vppfeatures/#acls-for-security-groups)

  - NAT (for SNAT,DNAT) [nat](https://fd.io/vppproject/vppfeatures/#network-address-translation)

  - IKEv2 (for IKE) [ike](https://fd.io/vppproject/vppfeatures/#ikev2-plugin)
  
  - IPSEC (for IPSec) [ipsec](https://fd.io/vppproject/vppfeatures/#ipsec-crypto-engine-provided-by-intel-ipsecmb-library)

These packages support the following functionality:

  - IPSec tunnels across K8s clusters;  
     
  - Support of multiple types of K8s clusters: 
  
    - K8s clusters having static public IP address, 
  
    - K8s clusters having dynamic public IP address with static FQDN, and 
  
    - K8s clusters with no public IP;

  - Stateful inspection firewall (for inbound and outbound connections);

  - Source NAT and Destination NAT for  K8s clusters whose POD and ClusterIP subnets are overlapping;

  - Multiple WAN links.


The  SD-WAN implementation uses the following three primary components:

  - SD-WAN Cloud-Native Network Function (CNF) based on vpp packages;
  
  - Custom Resource Definition (CRD) Controller;
  
  - Custom Resource Definitions (CRD).

The CNF contains the vpp services that perform SD-WAN operations. The CRD Controller and CRDs allow Custom Resources (i.e., extensions to Kubernetes APIs) to be created. Together these components allow information to be sent and received, and commands performed, from the Kubernetes Controller to the SD-WAN.

This behavior is described in the following subsections.

### SD-WAN CNF
The SD-WAN CNF is deployed as a pod with external network connections. The CNF runs the Multiple WAN, Firewall, IPSec, and NAT applications, as described in the previous section. The configuration parameters for the CNF include:

  - LAN interface configuration – to create and connect virtual, local networks within the edge cluster (local branch) to the CNF.

  - WAN interface configuration – to initialize interfaces that connect the CNF and connected LANs to the external Internet - WAN and to initialize the traffic rules (e.g., policy, rules) for the interfaces. The external WAN is also referred to in this document as a provider network.

SD-WAN traffic rules and WAN interfaces are configured at runtime via a RESTful API. The CNF implements the RestAPI plugin to provide this API. The API calls are initiated and passed to the CNF by a CRD Controller described in the next paragraph. The API provides the capability to list available SD-WAN services (e.g., Multiple WAN, Firewall, and IPSec), get service status, and execute service operations for adding, viewing, and deleting settings for these services.

### SD-WAN CRD Controller
The CRD Controller (also referred to in the implementation as a Config Agent), interacts with the SD-WAN CNF via RESTful API calls. It monitors CRs applied through K8s APIs and translates them into  API calls that carry the CNF configuration to the CNF instance.

the CRD Controller includes several functions:

  - MWAN Controller, to monitor the Multiple WAN CR;

  - FW Controller, to monitor the Firewall and NAT CR;

  - IPSec Controller, to monitor the IPSec and IKE CRs.


### Custom Resources (CRs)

As explained above, the behavior of the SD-WAN is governed by rules established in the CNF services.
In order to set these rules externally, CRs are defined to allow rules to be transmitted from the Kubernetes API. The CRs are created from the CRDs that are part of the SD-WAN implementation.

The types of rules supported by the CRs are: 

  - Multipule WAN class, with 2 subclasses, mwan3_policy and mwan3_rule. 

  - The firewall class has 5 kinds of rules: firewall_zone, firewall_snat, firewall_dnat, firewall_forwarding, firewall_rule.  

  - IPSec class.
  
  The rules are defined by the OpenWrt services and compatibled with it, and can be found in the OpenWrt documentation, e.g., [here](https://openwrt.org/docs/guide-user/network/wan/multiwan/mwan3).
  
  Each kind of SD-WAN rule corresponds to a CRD, which are used to instantiate the CRs.

In a Kubernetes namespace, with more than one CNF deployment and many SD-WAN rule CRDs, labels are used to correlate a CNF with SD-WAN rule CRDs.

## CNF Configuration via CRs

As explained earlier, the SD-WAN CNF contains a collection of services, implemented by vpp plugins. In this section, the services are described in greater detail.

### IPSec
The SD-WAN leverages IPSec functionality to setup secure tunnels for  Edge-to-WAN and Edge-WAN-Edge (i.e., to interconnect two edges) communication. The SD-WAN uses the vpp IPSec crypto plugin implementation of IPSec. IPsec rules are integrated with the firewall, which enables custom firewall rules. vpp uses the default firewall mechanism to update the firewall rules and injects all the additionally required settings, according to the IPsec configuration from CRD Controller . 

The SD-WAN configures the IPSec site-to-site tunnels to connect edge networks through a hub located in the external network. The hub is a server that acts as a proxy between pairs of edges.  The hub also runs SD-WAN CRD Controller and CNF configured such that it knows how to access SD-WAN CNFs deployed on both edges.  In that case, to create the IPsec tunnel, the WAN interface on the edge is treated as one side of the tunnel, and the connected  WAN interface on the hub is configured as the "responder". Both edges are configured as "initiator".

## SD-WAN CNF Packet Flow

Packets that arrives at the edge come through a WAN link that connects the edge to an external provoder network. This WAN interface should be already configured with traffic rules. If there is an IPSec tunnel created on the WAN interface, the packet enters the IPSec tunnel and is forwarded according to IPSec and Firewall/NAT rules. The packet eventually leaves the CNF via a LAN link connecting the OVN network on the edge. 

The following figure shows the typical packet flow through the SD-WAN CNF for Rx (WAN to LAN) when a packet sent from external network enters the edge cluster:

![SD-WAN Rx packet flow ](sdwan-images/packet-flow-rx2.png)

Packets that attempt to leave the edge come into the CNF through a LAN link attached to the OVN network on the edge cluster. This packet is then marked by the Multiple WAN application. This mark is used by the firewall to apply rules on the packet, and steer it to the proper WAN link used by the IPSec tunnel connecting the CNF to the WAN. The packet enters the IPSec tunnel and leaves the edge through the WAN interface.

The following figure shows the typical packet flow through the SD-WAN CNF for Tx (LAN to WAN), when a packet leaves from the edge cluster to the external network:

![SD-WAN Tx packet flow ](sdwan-images/packet-flow-tx2.png)

## OpenNESS Integration
The previous sections of this document describe the operation of an SD-WAN implemention built from vpp and its various plugins. We now turn to the subject of how the SD-WAN is integrated with OpenNESS.

### Goals
OpenNESS leverages the SD-WAN project to offer SD-WAN service within an on-premise edge, to enable secure and optimized inter-edge data transfer. This functionality is sought by global corporations with  branch offices distributed across many geographical locations, as it creates an optimized WAN between edge locations implemented on top of a public network.

At least one SD-WAN CNF is expected to run on each OpenNESS cluster (as shown in a previous figure), and act as a proxy for edge applications traffic entering and exiting the cluster. The primary task for the CNF is to provide software-defined routes connecting the edge LANs with the (public network) WAN.

Currently, the OpenNESS SD-WAN is intended only for single node clusters, accommodating only one instance of a CNF and a CRD Controller.


### Networking Implementation
OpenNESS deployment featuring SD-WAN implements networking within the cluster with three CNIs: 

  - calico CNI, that acts as the primary CNI. 
  - ovn4nfv k8s plugin CNI  that acts as the secondary CNI. 
  - Multus CNI, that allows for attaching multiple network interfaces to pods, required by the CNF pod. Without Multus, Kubernetes pods could support only one network interface.

The [Calico](https://docs.projectcalico.org/about/about-calico) CNI is used to configure the default network overlay for the OpenNESS cluster. It provides the commuication between the pods of the cluster and acts as the management interface. Calico is considered a lighter solution than Kube-OVN, which currently is the preferable CNI plugin for the primary network in OpenNESS clusters.

The [ovn4nfv-k8s-plugin](https://github.com/opnfv/ovn4nfv-k8s-plugin) is a CNI plugin based on OVN and OpenVSwitch (OVS). It works with the Multus CNI to add multiple interfaces to the pod. If Multus is used, the net1 interface is by convention the OVN default interface that connects to Multus. The other interfaces are added by ovn4nfv-k8s-plugin according to the pod annotation. With ovn4nfv-k8s-plugin, virtual networks can be created at runtime. The CNI plugin also utilises physical interfaces to connect a pod to an external network (provider network). This is particularly important for the SD-WAN CNF. ovn4nfv also enables Service Function Chaining ([SFC](https://github.com/opnfv/ovn4nfv-k8s-plugin/blob/master/demo/sfc-setup/README.md)).

In order for the SD-WAN CNF to act as a proxy between the virtual LANs in the cluster and the WAN, it needs to have  two types of network interfaces configured:

 -  A virtual LAN network on one of the CNF's virtual interfaces. This connects  application pods belonging to the same OVN network in the cluster. The ovn4nfv plugin allows for simplified creation of a virtual OVN network based on the provided configuration. The network is then attached on one of the CNF's interfaces.
 -  A Enterprise network, to connect the CNF pod to an Enterprise LAN network. The Enterprise network is attached to the physical network infrastructure which is takeovered by vpp.
 -  A provider network, to connect the CNF pod to an external network (WAN). The provider network is also attached to the physical network infrastructure which is takeovered by vpp.

### Converged Edge Reference Architectures (CERA)
CERA is a business program that creates and maintains validated reference architectures of edge networks, including both hardware and software elements. The reference architectures are used by ISVs, system integrators, and others to accelerate the development of production edge computing systems.

The OpenNESS project has created a CERA reference architecture for SD-WAN edge and SD-WAN hub. They are used, with OpenNESS, to create a uCPE platform for an SD-WAN CNF on edge and hub accordingly. Even though there is only one implementation of CNF, it can be used for two different purposes, as described below.

#### SD-WAN Edge Reference Architecture
The SD-WAN Edge CERA reference implementation is used to deploy SD-WAN CNF on a single-node edge cluster that will also accomodate enterprize edge applications. The major goal of SD-WAN Edge is to support the creation of a Kubernetes-based platform that boosts the performance of deployed edge applications and reduces resource usage by the Kubernetes system. To accomplish this, the underlying platform must be optimized and made ready to use IA accelerators. OpenNESS provides support for the deployment of OpenVINO™ applications and workloads acceleration with the Intel® Movidius™ VPU HDDL-R add-in card.  SD-WAN Edge also enables the Node Feature Discovery (NFD) building block on the cluster to provide awareness of the nodes’ features to  edge applications. Finally, SD-WAN Edge implements Istio Service Mesh (SM) in the default namespace to connect the edge applications. SM acts as a middleware between  edge applications/services and the OpenNESS platform, and provides abstractions for traffic management, observability, and security of the building blocks in the platform. Istio is a cloud-native service mesh that provides capabilities such as Traffic Management, Security, and Observability uniformly across a network of services. OpenNESS integrates with Istio to reduce the complexity of large scale edge applications, services, and network functions. More information on SM in OpenNESS can be found on the OpenNESS [website](https://openness.org/developers/).


To minimalize resource consumption by the cluster, SD-WAN Edge disables services such as EAA, Edge DNS, and Kafka. Telemetry service stays active for all the Kubernetes deployments.

The following figure shows the system architecture of the SD-WAN Edge Reference Architecture.

![OpenNESS SD-WAN Edge Architecture ](sdwan-images/sdwan-edge-arch2.png)


#### SD-WAN Hub Reference Architecture
The SD-WAN Hub reference architecture prepares an OpenNESS platform for a single-node cluster that functions primarily as an SD-WAN hub. That cluster will also deploy a SD-WAN CRD Controller and a CNF, but no other corporate applications are expected to run on it. That is why the node does not enable support for an HDDL card or for Network Feature Discovery and Service Mesh.

The Hub is another OpenNESS single-node cluster that acts as a proxy between different edge clusters. The Hub is essential to connect  edges through a WAN when applications within the edge clusters have no public IP addresses, which requires additional routing rules to provide access. These rules can be configured globally on a device acting as a hub for the edge locations. 

The Hub node has two expected use-cases:

- If the edge application wants to access the internet, or an external application wants to access service running in the edge node, the Hub node can act as a gateway with a security policy in force.

- For communication between a pair of edge nodes located at different locations (and in different clusters), if both edge nodes have public IP addresses, then an IP Tunnel can be configured directly between the edge clusters, otherwise the Hub node is required to act as a proxy to enable the communication.

The following figure shows the system architecture of the SD-WAN Hub Reference Architecture.

![OpenNESS SD-WAN Hub Architecture ](sdwan-images/sdwan-hub-arch2.png)

## Deployment And Configuration 
### E2E Scenarios
Four end-to-end scenarios have been validated to verify deployment of an SD-WAN on OpenNESS. The four scenarios are described in the following sections of this document. 

#### Hardware Specification
- Three physical servers for two single-node sdewan edge clusters and one single-node sdewan hub cluster.
- Xeon (D-2145NT): 1 Core 2 HT, 2G DDR (1G DDR as stretch goal) 
- CPU:
  - _Intel(R) Xeon(R) D-2145NT CPU @ 1.90GHz @8 cores 16 threads_ 
- Memory:
  - _Total 64G DDR4 (16Gx4)_
- NICs:
  - _Intel Corporation Ethernet Connection X722 for 10GbE SFP+_
  - _Intel Corporation Ethernet Controller X710 for 10GbE SFP+_
  - _Intel Corporation I350 Gigabit Network Connection_

#### Software Specification

- Host Operating System
  - _ubuntu 20.04_
- Host Linux Kernel
  - _5.4.0-77-generic_
- BIOS VT-d Configuration
  - _Enable_
- Boot settings
  - _iommu=pt intel_iommu=on_
	
#### Test Suite Topology Overview

![Topology Overview](sdwan-images/itp_ned_46_01.png)

**Notes:** 

If there is no WAN Switch/Router equipment in the lab environment, DHCP server can be setting for adjusting the environment *as optional*, dhcp configuration file `cat /etc/dhcp/dhcpd.conf`, like as:

  ```
  ddns-update-style interim;
  ignore client-updates;
  allow booting;
  allow bootp;
  subnet 192.168.123.0 netmask 255.255.255.0 {
  option routers 192.168.123.1;
  option subnet-mask 255.255.255.0;
  option broadcast-address 192.168.123.255;
  range dynamic-bootp 192.168.123.10 192.168.123.100;
  default-lease-time 1200;
  max-lease-time 7200;
  ```

#### Scenario 1 : Using Flavor to deploy on Hub Node

场景1用于验证使用Flavor的方式成功在Hub Node部署SDEWAN CNF.  

在这个场景中，OpenNESS Edge采用单节点集成方式运行，SDEWAN CNF将以POD的方式运行在OpenNESS Hub，同时将对SDEWAN CNF配置WAN接口，验证SDEWAN POD成功运行，并获取到IP地址.

![OpenNESS SD-WAN Scenario 1 ](sdwan-images/sdewan-vpp-scenario1.png)
Support flavor to deploy SDEWAN VPP CNF automatically on Hub Node. Hub Single Cluster contains two wan ports, and one management port

A more detailed description of this E2E test is provided under the link in the OpenNESS documentation for this SD-WAN [scenario](https://github.com/otcshare/x-test/blob/br_tieto_pr_test_plan/test_plans/ned/integration/ts46-sdwan-vpp.md).

##### Prerequisites

1. `otcshare/ido-converged-edge-experience-kits` checked out branch *br_tieto_ido-ewo* on hub server.
2. Use command `git submodule update --init`  checkout `ceek` folder.
3. SSH key generated & copied for the remote server with `ssh-keygen` and `ssh-copy-id nodename@ip`
4. Configure `ansible_host` and `ansible_user` for `controller_group` and `edgenode_group` in `inventory.yml` file under `/ido-converged-edge-experience-kits/`.
5. Set `git_repo_token` in _ceek/inventory/default/group_vars/all/10-default.yml_
6. Current time on machine must be set.
7. NTP/time role enabled in _group_vars/all/10-default.yml_ in _Network Time Protocol (NTP)_ section.
8. NTP server has to be specified in _ceek/inventory/default/group_vars/all/10-default.yml_ in list _ntp_servers e.g. ntp_servers: [" 0.centos.pool.ntp.org"]_.
9. Set proper proxy if need.

##### Test Steps
1. Deploy sdewan vpp cnf and crd controller on hub node using **sdwan** flavor. 

2. Modify _roles/applications/sdewan/vpp/controlplane/defaults/main.yml_ for pre-defined configuration, according to hardware requirements to modify, as below:

   2.1) *devs*:  **devs' pci number** can be got with `lspci | grep "I350 Gigabit Network"`, like as:

      ```
      66:00.1 Ethernet controller: Intel Corporation I350 Gigabit Network Connection 
      66:00.2 Ethernet controller: Intel Corporation I350 Gigabit Network Connection
      66:00.3 Ethernet controller: Intel Corporation I350 Gigabit Network Connection 
      ```
   
   2.2) *wan*: two wan informations need to set.
   
   ```yaml
   _sdewan_cnfvpp_cfg:
     devs:
       - "0000:66:00.1"
       - "0000:66:00.2"
     qats:
       - "0000:b7:01.0"
     wan:
       - name: GigabitEthernet66/0/1
         static_ip_address: 157.1.2.3
         enable_default_gateway: true
         preference: 0
         weight: 1
         enable_dhcp: true
         enable_nat: true
       - name: GigabitEthernet66/0/2
         static_ip_address: 157.1.2.3
         enable_default_gateway: true
         preference: 10
         weight: 11
         enable_dhcp: true
         enable_nat: true
     dhcp:
       server:
         ippool_start: 172.30.10.100
         ippool_end: 172.30.10.200
         ippool_gateway: 172.30.10.1
         ippool_mask: 255.255.255.0
         ippool_leasetime: 28800
         ippool_dns: 8.8.8.8
         ippool_address: 172.30.10.4/24
         ippool_subnet: 172.30.10.0/24
     gateway_tap:
       vpp_ip_address: 169.254.10.1
       host_ip_address: 169.254.10.5/24
     metadata:
       name: sdewan-cnfvpp-pod
       namespace: cnfvpp
       labels: sdewan-cnfvpp-pod
       cert: cnf-default-cert
       auth:
         name: cnf-default-auth
         user: admin
         pwd: admin
     nodeSelectorLabelName: "sdewan-cnfvpp-node"
     nodeSelectorLabelValue: "controller"
     resources:
       limits:
         hugepages: 200Mi
         memory: 500Mi
       requests:
         cpu: 150m
     cpu:
       maincore: 0
       corelist_workers:
     image:
       sdewan_cnfvpp_image: sdewan-cnfvpp-service
     ipsec_rules:
       - name: ipsectunnel1
         remote: 10.10.10.35
         pre_shared_key: "0123456789012345"
         local_identifier: cnfvpp
         remote_identifier: cnfvpp
         local_subnet: 192.168.5.1/24
         remote_sourceip: "192.168.6.1-192.168.6.254"
         remote_subnet: 192.168.1.1/24
         local_sourceip: "192.168.6.1-192.168.6.254"
     features:
       enable_ipsec: false
       enable_qat: true
   ```
   
3. Modify _flavors/sdwan/all.yml_  for sdewan vpp cnf(vpp) hub deployment：

   ```yaml
   ## sde-wan edge enable
   sdewan_edge_enable: False
   
   ## sde-wan hub enable
   sdewan_hub_enable: False
   
   ## sde-wan vpp enable
   sdewan_controller_enable: True
   sdewan_edge_vpp_enable: False
   sdewan_hub_vpp_enable: True
   
   ## fixme: input your own git repo token here
   git_repo_token: "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"
   ```

4. Modify _inventory.yml_, flavor is **sdwan** and single_node_deployment is **true**:

   ```yaml
   all:
     vars:
     cluster_name: cluster_test    # NOTE: Use `_` instead of spaces.
     flavor: sdwan                 # NOTE: Flavors can be found in `flavors` directory.
     single_node_deployment: true  # Request single node deployment (true/false).
     limit:                        # Limit ansible deployment to certain inventory group or hosts
   controller_group:
     hosts:
       controller:
         ansible_host: 192.168.0.165
         ansible_user: openness
   edgenode_group:
     hosts:
       node01:
         ansible_host: 192.168.0.165
         ansible_user: openness
   edgenode_vca_group:
     hosts:
   ptp_master:
     hosts:
   ptp_slave_group:
     hosts:
   ```

5. Run _`./deploy.py`_ and wait till it ends successfully.

6. Check the node result:

   - Execute:

    ```shell
   openness@controller:~$ # kubectl get nodes
    ```

   - Example output:

    ```
   NAME         STATUS   ROLES                  AGE   VERSION
   controller   Ready    control-plane,master   18h   v1.20.0
    ```

7. Check the pods(such as sdewan-cnfvp, sdewan-controller,harbor,calico,cert-manager) status:

   - Execute:

    ```shell
   openness@controller:~$ # kubectl get po -n cnfvpp
    ```

   - Example output:

    ```
   NAMESPACE       NAME                                               READY   STATUS      RESTARTS   AGE
   cert-manager    cert-manager-5597cff495-5rmn7                      1/1     Running     1          17h
   cert-manager    cert-manager-cainjector-bd5f9c764-t9zcq            1/1     Running     3          17h
   cert-manager    cert-manager-webhook-5f57f59fbc-7vwtk              1/1     Running     0          17h
   cnfvpp          sdewan-cnfvpp-pod-6b67dcd5d5-7wxlk                 2/2     Running     0          17h
   harbor          harbor-app-harbor-chartmuseum-f56cb984b-w77zw      1/1     Running     0          18h
   harbor          harbor-app-harbor-clair-779df4555b-n9lhz           2/2     Running     11         18h
   harbor          harbor-app-harbor-core-86f6575644-77mr5            1/1     Running     4          18h
   harbor          harbor-app-harbor-database-0                       1/1     Running     3          18h
   harbor          harbor-app-harbor-jobservice-7df4c9968c-2xb9v      1/1     Running     2          18h
   harbor          harbor-app-harbor-nginx-678c759dc4-bd7bq           1/1     Running     0          18h
   harbor          harbor-app-harbor-notary-server-67f644d8bc-cbspx   1/1     Running     3          18h
   harbor          harbor-app-harbor-notary-signer-7bf556c585-tjbnl   1/1     Running     0          18h
   harbor          harbor-app-harbor-portal-fd5ff4bc9-h7cq4           1/1     Running     0          18h
   harbor          harbor-app-harbor-redis-0                          1/1     Running     0          18h
   harbor          harbor-app-harbor-registry-5d8c9879c4-72zb4        2/2     Running     0          18h
   harbor          harbor-app-harbor-trivy-0                          1/1     Running     0          18h
   kube-system     calico-kube-controllers-7f4f5bf95d-4jwsx           1/1     Running     2          18h
   kube-system     calico-node-h98ss                                  1/1     Running     2          18h
   kube-system     coredns-74ff55c5b-6nbzb                            1/1     Running     0          18h
   kube-system     coredns-74ff55c5b-v6sf8                            1/1     Running     0          18h
   kube-system     descheduler-cronjob-1628316600-rqzlr               0/1     Completed   0          4m32s
   kube-system     descheduler-cronjob-1628316720-bztxx               0/1     Completed   0          2m32s
   kube-system     descheduler-cronjob-1628316840-tp6l2               0/1     Completed   0          31s
   kube-system     etcd-controller                                    1/1     Running     1          18h
   kube-system     kube-apiserver-controller                          1/1     Running     1          18h
   kube-system     kube-controller-manager-controller                 1/1     Running     4          18h
   kube-system     kube-proxy-9j88v                                   1/1     Running     0          18h
   kube-system     kube-scheduler-controller                          1/1     Running     6          18h
   sdewan-system   sdewan-controller-5b98cc5dfd-wzlmm                 2/2     Running     24         17h
   telemetry       collectd-mcrfc                                     2/2     Running     0          18h
   telemetry       custom-metrics-apiserver-5677d4ff98-z965s          1/1     Running     0          18h
   telemetry       grafana-598f94f88f-qxthd                           2/2     Running     0          18h
   telemetry       otel-collector-649fb7bf69-rpb97                    2/2     Running     0          18h
   telemetry       prometheus-node-exporter-g8s62                     1/1     Running     0          18h
   telemetry       prometheus-server-548ddd7959-87vkf                 3/3     Running     0          18h
   telemetry       telemetry-aware-scheduling-7697dfb9c6-p4mst        2/2     Running     2          18h
   telemetry       telemetry-collector-certs-hh2dl                    0/1     Completed   0          18h
   telemetry       telemetry-node-certs-6lqrt                         1/1     Running     0          18h
    ```

8. Check the reference volumes special "cnf-default-cert" and “cnf-default-auth” have been mounted into cnf pod:

   - Execute:

    ```shell
   openness@controller:~$ #kubectl exec -it sdewan-cnfvpp-pod-6b67dcd5d5-7wxlk -n cnfvpp -c cnfvpp -- ls -l /etc/crd-ctrlr-adpt/
    ```
   
   - Example output:
   
    ```
   drwxrwxrwt 3 root root 120 Aug  2 08:13 auth
   drwxrwxrwt 3 root root 140 Aug  2 08:13 cert
    ```
   
9. Check the interface address result inside cnf pod:

   - Execute and output:

    ```shell
   openness@controller:~$ #kubectl exec -it sdewan-cnfvpp-pod-6b67dcd5d5-7wxlk -n cnfvpp -c cnfvpp -- vppctl sh int addr
   sh int addr
   GigabitEthernet66/0/1 (up):
     L3 192.168.0.210/24 ip4 table-id 1 fib-idx 1
   GigabitEthernet66/0/2 (up):
     L3 192.168.1.191/24 ip4 table-id 1 fib-idx 1
   local0 (dn):
   loop0 (up):
     L2 bridge bd-id 1 idx 1 shg 0 bvi
     L3 172.30.20.1/24
   tap0 (up):
     L2 bridge bd-id 1 idx 1 shg 0
   tap1 (up):
     L3 192.168.5.1/24 ip4 table-id 1 fib-idx 1
   openness@controller:~$ #kubectl exec -it sdewan-cnfvpp-pod-68d99d5d4c-chq5r -n cnfvpp -c cnfvpp -- vppctl sh nat44 interfaces
   NAT44 interfaces:
    loop0 in
    GigabitEthernet66/0/1 out
    GigabitEthernet66/0/2 out
    ```

#### Scenario 2 : Using Flavor to deploy on Edge Node
场景2用于验证使用Flavor的方式成功在Edge Node部署SDEWAN CNF.  

在这个场景中，OpenNESS Edge采用单节点集成方式运行，SDEWAN CNF将以POD的方式运行在OpenNESS Edge，同时将对SDEWAN CNF配置WAN接口和LAN接口，验证SDEWAN POD成功运行，并获取到IP地址.

![OpenNESS SD-WAN Scenario 2 ](sdwan-images/sdewan-vpp-scenario2.png)
Support flavor to deploy SDEWAN VPP CNF automatically on Edge Nodes. One LAN port and one WAN port are dpdk bound.

A more detailed description of this E2E test is provided under the link in the OpenNESS documentation for this SD-WAN [scenario](https://github.com/otcshare/x-test/blob/br_tieto_pr_test_plan/test_plans/ned/integration/ts46-sdwan-vpp.md).

##### Prerequisites

1. `otcshare/ido-converged-edge-experience-kits` checked out branch *br_tieto_ido-ewo* on all edges.
2. Use command `git submodule update --init`  checkout `ceek` folder.
3. SSH key generated & copied for the remote server with `ssh-keygen` and `ssh-copy-id nodename@ip`
4. Configure `ansible_host` and `ansible_user` for `controller_group` and `edgenode_group` in `inventory.yml` file under `/ido-converged-edge-experience-kits/`.
5. Set `git_repo_token` in _ceek/inventory/default/group_vars/all/10-default.yml_
6. Current time on machine must be set.
7. NTP/time role enabled in _group_vars/all/10-default.yml_ in _Network Time Protocol (NTP)_ section.
8. NTP server has to be specified in _ceek/inventory/default/group_vars/all/10-default.yml_ in list _ntp_servers e.g. ntp_servers: [" 0.centos.pool.ntp.org"]_.
9. Set proper proxy if need.

##### Test Steps

1. Edge A: Deploy sdewan vpp cnf on edge nodes A using flavor sdwan. 

   1.1) Modify _roles/applications/sdewan/vpp/common/defaults/main.yml_ for pre-defined configuration, according to hardware requirements to modify:

      - *devs*: devs' pci number can be got with `lspci | grep "I350 Gigabit Network"`, like as: 

        ```
        66:00.1 Ethernet controller: Intel Corporation I350 Gigabit Network Connection 
        66:00.2 Ethernet controller: Intel Corporation I350 Gigabit Network Connection
        ```

      -  *wan*: one wan informations need to set.

      - *lan* : one lan informations need to set.

   ```yaml
   _sdewan_cnfvpp_cfg:
     devs:
       - "0000:66:00.1"
       - "0000:66:00.2"
     qats:
       - "0000:b7:01.0"
     wan:
       - name: GigabitEthernet66/0/1
         static_ip_address: 157.1.2.3
         enable_default_gateway: true
         preference: 0
         weight: 1
         enable_dhcp: true
         enable_nat: true
     lan:
       - name: GigabitEthernet66/0/2
     dhcp:
       server:
         ippool_start: 172.30.10.100
         ippool_end: 172.30.10.200
         ippool_gateway: 172.30.10.1
         ippool_mask: 255.255.255.0
         ippool_leasetime: 28800
         ippool_dns: 8.8.8.8
         ippool_address: 172.30.10.4/24
         ippool_subnet: 172.30.10.0/24
     gateway_tap:
       vpp_ip_address: 169.254.10.1
       host_ip_address: 169.254.10.5/24
     metadata:
       name: sdewan-cnfvpp-pod
       namespace: cnfvpp
       labels: sdewan-cnfvpp-pod
       cert: cnf-default-cert
       auth:
         name: cnf-default-auth
         user: admin
         pwd: admin
     nodeSelectorLabelName: "sdewan-cnfvpp-node"
     nodeSelectorLabelValue: "controller"
     resources:
       limits:
         hugepages: 200Mi
         memory: 500Mi
       requests:
         cpu: 150m
     cpu:
       maincore: 0
       corelist_workers:
     image:
       sdewan_cnfvpp_image: sdewan-cnfvpp-service
     ipsec_rules:
       - name: ipsectunnel1
         remote: 10.10.10.35
         pre_shared_key: "0123456789012345"
         local_identifier: cnfvpp
         remote_identifier: cnfvpp
         local_subnet: 192.168.5.1/24
         remote_sourceip: "192.168.6.1-192.168.6.254"
         remote_subnet: 192.168.1.1/24
         local_sourceip: "192.168.6.1-192.168.6.254"
     features:
       enable_ipsec: false
       enable_qat: true
   ```
   
   1.2) Modify _flavors/sdwan/all.yml_  for sdewan vpp cnf(vpp) edge node deployment：
   
   ```yaml
   ## sde-wan edge enable
    sdewan_edge_enable: False
   
    ## sde-wan hub enable
    sdewan_hub_enable: False
   
    ## sde-wan vpp enable
    sdewan_controller_enable: True
    sdewan_edge_vpp_enable: True
    sdewan_hub_vpp_enable: False
    
    ## fixme: input your own git repo token here
    git_repo_token: "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"
   ```
   
   1.3) Modify _inventory.yml_, flavor is **sdwan** and single_node_deployment is **true**:
   
   ```yaml
    all:
      vars:
        cluster_name: cluster_test    # NOTE: Use `_` instead of spaces.
        flavor: sdwan                 # NOTE: Flavors can be found in `flavors` directory.
        single_node_deployment: true  # Request single node deployment (true/false).
        limit:                        # Limit ansible deployment to certain inventory group or hosts
    controller_group:
      hosts:
        controller:
          ansible_host: 192.168.0.164
          ansible_user: openness
    edgenode_group:
      hosts:
        node01:
          ansible_host: 192.168.0.164
          ansible_user: openness
    edgenode_vca_group:
      hosts:
    ptp_master:
      hosts:
    ptp_slave_group:
      hosts:
   ```
   
   1.4) Run _`./deploy.py`_ and wait till it ends successfully.
   
   1.5) Check the node result:
   
   - Execute:
   
   ```shell
    openness@controller:~$ # kubectl get nodes
   ```
   
   - Example output:

   ```
    NAME	     STATUS	ROLES				   AGE	  VERSION
    controller   Ready	control-plane,master   3d1h	  v1.20.0
   ```
   
   1.6) Check the pods(sdewan-cnfvpp-pod,sdewan-controller,harbor,calico,cert-manager ) status:
   
   - Execute:
   
   ```shell
    openness@controller:~$ # kubectl get po -n cnfvpp
   ```
   
   - Example output:
   
   ```
    NAMESPACE       NAME                                               READY   STATUS    RESTARTS   AGE
     cert-manager    cert-manager-5597cff495-dcmvm                      1/1     Running   0          24h
     cert-manager    cert-manager-cainjector-bd5f9c764-8rdms            1/1     Running   0          24h
     cert-manager    cert-manager-webhook-5f57f59fbc-wgnsr              1/1     Running   0          24h
     cnfvpp          sdewan-cnfvpp-pod-68d99d5d4c-chq5r                 2/2     Running   0          20h
     harbor          harbor-app-harbor-chartmuseum-685569858b-w8wc9     1/1     Running   0          3d1h
     harbor          harbor-app-harbor-clair-779df4555b-ftfjc           2/2     Running   440        3d1h
     harbor          harbor-app-harbor-core-57fdf4d4-pc28q              1/1     Running   0          3d1h
     harbor          harbor-app-harbor-database-0                       1/1     Running   0          3d1h
     harbor          harbor-app-harbor-jobservice-994696fc8-bkspf       1/1     Running   2          3d1h
     harbor          harbor-app-harbor-nginx-7ff49cf9c4-f6cfn           1/1     Running   0          3d1h
     harbor          harbor-app-harbor-notary-server-b4bb8f78b-xcwmv    1/1     Running   3          3d1h
     harbor          harbor-app-harbor-notary-signer-8485f97c8c-rkwx6   1/1     Running   4          3d1h
     harbor          harbor-app-harbor-portal-fd5ff4bc9-tq55c           1/1     Running   0          3d1h
     harbor          harbor-app-harbor-redis-0                          1/1     Running   0          3d1h
     harbor          harbor-app-harbor-registry-656b744d74-qppxx        2/2     Running   0          3d1h
     harbor          harbor-app-harbor-trivy-0                          1/1     Running   0          3d1h
     kube-system     calico-kube-controllers-856cdfb67c-67hdt           1/1     Running   0          3d1h
     kube-system     calico-node-59jxf                                  1/1     Running   0          3d1h
     kube-system     coredns-74ff55c5b-4zkcm                            1/1     Running   0          3d1h
     kube-system     coredns-74ff55c5b-fp4l8                            1/1     Running   0          3d1h
     kube-system     etcd-openness                                      1/1     Running   0          3d1h
     kube-system     kube-apiserver-openness                            1/1     Running   0          3d1h
     kube-system     kube-controller-manager-openness                   1/1     Running   0          3d1h
     kube-system     kube-proxy-nfw5x                                   1/1     Running   0          3d1h
     kube-system     kube-scheduler-openness                            1/1     Running   0          3d1h
     sdewan-system   sdewan-controller-5bbcfc6875-jqg4m                 2/2     Running   0          21h
   ```
   
   1.7) Check the reference volumes special "cnf-default-cert" and “cnf-default-auth” have been mounted into cnf pod:
   
   - Execute:
   
   ```shell
     openness@controller:~$ # kubectl get pod -n cnfvpp
     NAME                                 READY   STATUS    RESTARTS   AGE
     sdewan-cnfvpp-pod-68d99d5d4c-chq5r   2/2     Running   0          20h
     openness@controller:~$ #kubectl exec -it sdewan-cnfvpp-pod-68d99d5d4c-chq5r -n cnfvpp -c cnfvpp -- ls -l /etc/crd-ctrlr-adpt/
   ```
   
   - Example output:

   ```
   drwxrwxrwt 3 root root 120 Aug  2 09:13 auth
   drwxrwxrwt 3 root root 140 Aug  2 09:13 cert
   ```
   
   1.8) Check the interface address result inside cnf pod:
   
   - Execute:
   
   ```shell
     openness@controller:~$ # kubectl get pod -n cnfvpp
     NAME                                READY   STATUS    RESTARTS   AGE
     sdewan-cnfvpp-pod-68d99d5d4c-chq5r   2/2     Running   0          20h
     openness@controller:~$ #kubectl exec -it sdewan-cnfvpp-pod-68d99d5d4c-chq5r -n cnfvpp -c cnfvpp -- vppctl sh int addr
   ```
   
   - Example output:
   
   ```
     GigabitEthernet66/0/1 (up):
       L2 bridge bd-id 1 idx 1 shg 0
     GigabitEthernet66/0/2 (up):
       L3 192.168.1.194/24 ip4 table-id 1 fib-idx 1
     local0 (dn):
     loop0 (up):
       L2 bridge bd-id 1 idx 1 shg 0 bvi
       L3 172.30.10.1/24
     tap0 (up):
       L2 bridge bd-id 1 idx 1 shg 0  
     tap1 (up):
       L3 192.168.6.1/24 ip4 table-id 1 fib-idx 1
   ```
   
   1.9) Check the nat interface result inside cnf pod:
   
   - Execute:
   
    ```shell
   openness@controller:~$ #kubectl exec -it sdewan-cnfvpp-pod-68d99d5d4c-chq5r -n cnfvpp -c cnfvpp -- vppctl sh nat44 interfaces
    ```
   
   - Example output:
   
    ```
    NAT44 interfaces:
     loop0 in
     GigabitEthernet66/0/2 out
    ```
   
2. Edge B:   

   2.1) Configure Edge B `ansible_host` and `ansible_user` for `controller_group` and `edgenode_group` in `inventory.yml` file under `/ido-converged-edge-experience-kits/`
   2.2) Repeat the steps of Edge A to deploy sdewan vpp cnf on edge node B using flavor sdwan.


#### Scenario 3 : Verify Data Flow
场景3在场景1，场景2的基础上进一步验证Overlay network的配置，需要同时在Hub cluster和Edge cluster中部署SDEWAN CNF，并使能IKEv2和IPsec功能。

![OpenNESS SD-WAN Scenario 3 ](sdwan-images/sdewan-vpp-scenario3.png)
Verify data flow base on 3 single cluster topology. Ipsec CRs should be configurated on Overlay controller.

##### Prerequisites

1. Deploy sdewan vpp cnf using flavor on 3 single Edge and Hub nodes according to scenario A and B.
2. Independent of Edge and Hub cluster, Overlay Controller has been deployed sucessfuly on Kubernetes platform.
3. Checkout files `ngfw.yaml`  , `ipsec_hub.yaml`,`ipsec_edgea.yaml` and `ipsec_edgeb.yaml` are store in `ts_resources/ts_resources/ts46_files/`

##### Test Steps

1. Configing IKEv2 IPSec for 3 single Edge and Hub nodes：

   - Hub node:

   ```yaml
    kubectl apply -f ipsec_hub.yaml
   ```

    `cat ipsec_hub.yaml` like as below: 

   ```
   ---
   
   apiVersion: batch.sdewan.akraino.org/v1alpha1
   kind: IpsecSite
   metadata:
     name: ipsectunnela
     namespace: cnfvpp
     labels:
       sdewanPurpose: sdewan-cnfvpp-pod
   spec:
     name: ipsectunnela
     remote: "%any"
     pre_shared_key: "0123456789012345"
     authentication_method: psk
     local_identifier: cnfvppa
     remote_identifier: cnfvppa
     crypto_proposal:
       - ipsecproposal
     force_crypto_proposal: "0"
     connections:
     - name: connA
       conn_type: tunnel
       mode: start
       local_subnet: 192.168.5.1/24
       remote_sourceip: 192.168.6.1-192.168.6.254
       crypto_proposal:
         - ipsecproposal
         
   ---
   
   apiVersion: batch.sdewan.akraino.org/v1alpha1
   kind: IpsecSite
   metadata:
     name: ipsectunnelb
     namespace: cnfvpp
     labels:
       sdewanPurpose: sdewan-cnfvpp-pod
   spec:
     name: ipsectunnelb
     remote: "%any"
     pre_shared_key: "0123456789012345"
     authentication_method: psk
     local_identifier: cnfvppb
     remote_identifier: cnfvppb
     crypto_proposal:
       - ipsecproposal
     force_crypto_proposal: "0"
     connections:
     - name: connA
       conn_type: tunnel
       mode: start
       local_subnet: 192.168.5.1/24
       remote_sourceip: 192.168.7.1-192.168.7.254
       crypto_proposal:
         - ipsecproposal
   ```

   - Edge nodeA:

   ```yaml
    kubectl apply -f ipsec_edgea.yaml
   ```

   `cat ipsec_edgea.yaml` like as below: 

   ```
   ---
   
   apiVersion: batch.sdewan.akraino.org/v1alpha1
   kind: IpsecHost
   metadata:
     name: ipsectunnela
     namespace: cnfvpp
     labels:
       sdewanPurpose: sdewan-cnfvpp-pod
   spec:
     name: ipsectunnela
     remote: 192.168.0.191
     pre_shared_key: "0123456789012345"
     authentication_method: psk
     local_identifier: cnfvppa
     remote_identifier: cnfvppa
     crypto_proposal:
       - ipsecproposal
     force_crypto_proposal: "0"
     connections:
     - name: connA
       conn_type: tunnel
       mode: start
       remote_subnet: 192.168.5.1/24
       local_sourceip: 192.168.6.1-192.168.6.254
       crypto_proposal:
         - ipsecproposal
   
   ```

   - Edge  nodeB:

   ```yaml
    kubectl apply -f ipsec_edgeb.yaml
   ```

   `cat ipsec_edgeb.yaml` like as below: 

   ```
   ---
   
   apiVersion: batch.sdewan.akraino.org/v1alpha1
   kind: IpsecHost
   metadata:
     name: ipsectunnelb
     namespace: cnfvpp
     labels:
       sdewanPurpose: sdewan-cnfvpp-pod
   spec:
     name: ipsectunnelb
     remote: 192.168.0.190
     pre_shared_key: "0123456789012345"
     authentication_method: psk
     local_identifier: cnfvppb
     remote_identifier: cnfvppb
     crypto_proposal:
       - ipsecproposal
     force_crypto_proposal: "0"
     connections:
     - name: connA
       conn_type: tunnel
       mode: start
       remote_subnet: 192.168.5.1/24
       local_sourceip: 192.168.7.1-192.168.7.254
       crypto_proposal:
         - ipsecproposal
   
   ```

2. On Overlay Controller

   - [ ] Route the traffic originated from SCC to CNF, apply these on the host, like as：

   ```
   ip rule add to <overlay ip of the edge's CNF> lookup 40
   ip route add default via <cnf-ip-address> dev <calico-interface-for-cnf> table 40
   ```

   - [ ] Change the mark of second configuration **Mark: 50**，  like as：

   ```
   $ kubectl describe ipsechost -n cnfvpp
   Name: ipsectunnelb
   Namespace: cnfvpp
   Kind: IpsecHost
   Metadata:
   Spec:
    authentication_method: pubkey
    Connections:
    conn_type: tunnel
    crypto_proposal:
    proposal1
    proposal2
    local_updown: /etc/updown
    Mark: 50
    Mode: start
    Name: Conndevice2
    remote_sourceip: 192.168.0.180
    crypto_proposal:
    proposal1
    proposal2
    force_crypto_proposal: 0
    local_identifier: CN=sdewan-controller-base
    local_private_cert: …
    local_public_cert: …
    Remote: %any
    remote_identifier: CN=device-device-2-cert
    shared_ca: …
    Type: VTI-base
   ```

3. Check the interface address result inside cnf pod of 3 nodes:

   - Execute:

    ```shell
   openness@controller:~$ # kubectl get pod -n cfvpp
   NAME                                 READY   STATUS    RESTARTS   AGE
   sdewan-cnfvpp-pod-68d99d5d4c-chq5r   2/2     Running   0          20h
   openness@controller:~$ #kubectl exec -it sdewan-cnfvpp-pod-68d99d5d4c-chq5r -n cnfvpp -c cnfvpp -- vppctl sh int addr
   openness@controller:~$ #kubectl exec -it sdewan-cnfvpp-pod-68d99d5d4c-chq5r -n cnfvpp -c cnfvpp -- vppctl sh ikev2 profile
    ```

   - Example output:

     - Hub：ikev2 profiles contains two named *ipsecsite_cnfvpp_ipsec_tunnela_connA* and *ipsecsite_cnfvpp_ipsec_tunnela_connB*

       ```
       vpp# sh ikev2 profile
       profile ipsecsite_cnfvpp_ipsec_tunnela_connA
         auth-method shared-key-mic auth data 0123456789012345
         local id-type fqdn data cnfvppa
         remote id-type fqdn data cnfvppa
         local traffic-selector addr 192.168.5.1 -192.168.5.254 port 0 - 65535 protocol 0
         remote traffic-selector addr 192.168.6.1 - 192.168.6.254 port 0 - 65535 protocol 0
         lifetime 0 jitter 0 handover 0 maxdata 0
       profile ipsecsite_cnfvpp_ipsec_tunnelb_connA
         auth-method shared-key-mic auth data 0123456789012345
         local id-type fqdn data cnfvppb
         remote id-type fqdn data cnfvppb
         local traffic-selector addr 192.168.5.1 -192.168.5.254 port 0 - 65535 protocol 0
         remote traffic-selector addr 192.168.7.1 - 192.168.7.254 port 0 - 65535 protocol 0
         lifetime 0 jitter 0 handover 0 maxdata 0
       
       vpp# sh int addr
       GigabitEthernet66/0/1 (up):
         L3 192.168.0.190/24 ip4 table-id 1 fib-idx 1
       GigabitEthernet66/0/2 (up):
         L3 192.168.0.191/24 ip4 table-id 1 fib-idx 1
       ipip0 (up):
       ipip1 (up):
       ```
       
     - EdgeA：
     
       ```
       vpp# sh ikev2 profile
       profile ipsecsite_cnfvpp_ipsec_tunnela_connA
         auth-method shared-key-mic auth data 0123456789012345
         local id-type fqdn data cnfvppa
         remote id-type fqdn data cnfvppa
         local traffic-selector addr 192.168.6.1 - 192.168.6.254 port 0 - 65535 protocol 0
         remote traffic-selector addr 192.168.5.1 -192.168.5.254 port 0 - 65535 protocol 0
         responder GigabitEthernetb6/0/0 192.168.0.191
         ike-crypto-alg aes-cbc 256 ike-integ-alg sha1-96 ike-dh modp-2048
         esp-crypto-alg aes-cbc 256 esp-integ-alg sha1-96
         lifetime 0 jitter 0 handover 0 maxdata 0
      
       vpp# sh int addr
       GigabitEthernetb6/0/0 (up):
         L3 192.168.0.77/24 ip4 table-id 1 fib-idx 1
       GigabitEthernetb6/0/1 (up):
         L2 bridge bd-id 1 idx 1 shg 0
       local0 (dn):
       loop0 (up):
         L2 bridge bd-id 1 idx 1 shg 0 bvi
         L3 172.30.10.1/24
       tap0 (up):
         L2 bridge bd-id 1 idx 1 shg 0
       ipip0 (up):
       ```
     
     - EdgeB：
     
       ```
       vpp# sh ikev2 profile
       profile ipsecsite_cnfvpp_ipsec_tunnelb_connB
         auth-method shared-key-mic auth data 0123456789012345
         local id-type fqdn data cnfvppb
         remote id-type fqdn data cnfvppb
         local traffic-selector addr 192.168.7.1 -192.168.7.254 port 0 - 65535 protocol 0
         remote traffic-selector addr 192.168.5.1 -192.168.5.254 port 0 - 65535 protocol 0
         responder GigabitEthernetb6/0/0 192.168.0.190
         ike-crypto-alg aes-cbc 256 ike-integ-alg sha1-96 ike-dh modp-2048
         esp-crypto-alg aes-cbc 256 esp-integ-alg sha1-96
         lifetime 0 jitter 0 handover 0 maxdata 0
       
       vpp# sh int addr
       GigabitEthernetb6/0/0 (up):
         L3 192.168.0.76/24 ip4 table-id 1 fib-idx 1
       GigabitEthernetb6/0/1 (up):
         L2 bridge bd-id 1 idx 1 shg 0
       local0 (dn):
       loop0 (up):
         L2 bridge bd-id 1 idx 1 shg 0 bvi
         L3 172.30.30.1/24
       tap0 (up):
         L2 bridge bd-id 1 idx 1 shg 0
       ipip0 (up):
       ```
4. Ping app pod ip(any one ip) of Edge B inside app pod of Edge A:

   - Execute:

      ```shell
     on Edge node B:
      openness@controller:~$ #  kubectl get po -n harbor -o wide | grep clair
      harbor-app-harbor-clair-779df4555b-h52cr  2/2  Running   138  23h   10.245.49.6 
     
     on Edge node A:
      openness@controller:~$ #  kubectl get po -n harbor -o wide | grep clair
      harbor-app-harbor-clair-779df4555b-j6fd7  2/2  Running   138  23h   10.245.49.13 
     
      openness@controller:~$ # kubectl exec -it harbor-app-harbor-clair-779df4555b-j6fd7 -n harbor -- ping 10.245.49.6
     ```

   - Example output:

     ```
     ping 10.245.49.13 56(84) bytes of data.
      64 bytes from 10.245.49.13: icmp_seq=1 ttl=115 time=137 ms
      64 bytes from 10.245.49.13: icmp_seq=2 ttl=115 time=351 ms
     ```


4. Check the er result inside cnf pod of 3 nodes:

   - Execute:

     ```
     openness@controller:~$ #kubectl exec -it sdewan-cnfvpp-pod-68d99d5d4c-chq5r -n cnfvpp -c cnfvpp -- vppctl sh er
     ```

   - Example output:  contains *ikev2, esp4-decrypt-tun* and *esp4-encrypt-tun*

     ```
         Count                    Node                  Reason
              2                  ikev2                 IKEv2 packets processed
              1                  ikev2                 IKE request ignore (old msgid)
             21               dpdk-input               no error
              2                arp-reply               ARP replies sent
             11                arp-reply               ARP request IP4 source address learned
              2             ip4-udp-lookup             No error
              1             ip4-udp-lookup             No listener for dst port
             12            esp4-decrypt-tun            ESP pkts received
             12            esp4-encrypt-tun            ESP pkts received
             12            ipsec4-tun-input            good packets received
              8                ip4-input               Multicast RPF check failed
              1             ip4-icmp-error             destination unreachable response sent
     ```

#### Scenario 4 : Verify Data Flow with 3 Single Cluster Manually
场景4在场景3的基础上进一步验证Overlay network的数据流量通路，采用Hub-Spoke组网模型，需要同时在Hub cluster和两个Edge cluster中部署SDEWAN CNF，并使能IKEv2和IPsec功能，验证流量从EdgeA cluster app1 pod，经Hub cluster，到达EdgeB cluster app2 pod的通路可达性。

![OpenNESS SD-WAN Scenario 3 ](sdwan-images/sdewan-vpp-scenario4.png)
Data flow from EdgeA -> Hub -> EdgeB through IPSec Tunnel with 3 single cluster test topology. Ipsec CRs should be configurated based on ido-ceek deployment.

##### Prerequisites

1. Independent of Edge and Hub cluster, Overlay Controller has been deployed sucessfuly on Kubernetes platform.
2. `otcshare/ido-converged-edge-experience-kits` checked out branch *br_tieto_ido-ewo* on all edges and Hub.
3. Use command `git submodule update --init`  checkout `ceek` folder.
4. SSH key generated & copied for the remote server with `ssh-keygen` and `ssh-copy-id nodename@ip`
5. Configure `ansible_host` and `ansible_user` for `controller_group` and `edgenode_group` in `inventory.yml` file under `/ido-converged-edge-experience-kits/`.
6. Set `git_repo_token` in _ceek/inventory/default/group_vars/all/10-default.yml_
7. Current time on machine must be set.
8. NTP/time role enabled in _group_vars/all/10-default.yml_ in _Network Time Protocol (NTP)_ section.
9. NTP server has to be specified in _ceek/inventory/default/group_vars/all/10-default.yml_ in list _ntp_servers e.g. ntp_servers: [" 0.centos.pool.ntp.org"]_.
10. Set proper proxy if need.
11. Checkout files `ngfw.yaml` is store in `ts_resources/ts_resources/ts46_files/`

##### Test Steps

1. Hub：

   - Using flavor sdwan to deploy Hub NodeC,  references **ITP/NED/46/01**, different configurations like as:

     - Config two *ipsec_rules* *profiles* as responder
     - enable_dhcp: false 
     - Config *static_ip_address*, that is the same ip network segment with Wan GW(such as 192.168.0.xx)
     - remote: %any

     ```yaml
     _sdewan_cnfvpp_cfg:
       devs:
         - "0000:66:00.1"
         - "0000:66:00.2"
       wan:
         - name: GigabitEthernet66/0/1
           static_ip_address: 192.168.0.10
           enable_default_gateway: true
           preference: 0
           weight: 1
           enable_dhcp: true
           enable_nat: true
         - name: GigabitEthernet66/0/2
           static_ip_address: 192.168.0.20
           enable_default_gateway: true
           preference: 10
           weight: 10
           enable_dhcp: true
           enable_nat: true
       gateway_tap:
         vpp_ip_address: 169.254.20.1
         host_ip_address: 169.254.20.5/24
       metadata:
         name: sdewan-cnfvpp-pod
         namespace: cnfvpp
         labels: sdewan-cnfvpp-pod
         cert: cnf-default-cert
         auth:
           name: cnf-default-auth
           user: admin
           pwd: admin
       nodeSelectorLabelName: "sdewan-cnfvpp-node"
       nodeSelectorLabelValue: "controller"
       resources:
         limits:
           hugepages: 200Mi
           memory: 500Mi
         requests:
           cpu: 150m
       cpu:
         maincore: 0
         corelist_workers: 
       image:
         sdewan_cnfvpp_image: sdewan-cnfvpp-service
       enable_ipsec: true
       ipsec_rules:
         - name: cnfvpp_ipsec_tunnelA
           remote: 192.168.1.64
           pre_shared_key: "0123456789012345"
           local_identifier: cnfvppA
           remote_identifier: cnfvppA
           local_subnet: 169.254.20.1/24
           remote_sourceip: "169.254.21.1-169.254.21.254"
           remote_subnet: 169.254.21.1/24
           local_sourceip: "169.254.20.1-169.254.20.254"
         - name: cnfvpp_ipsec_tunnelB
           remote: 192.168.0.64
           pre_shared_key: "0123456789012345"
           local_identifier: cnfvppB
           remote_identifier: cnfvppB
           local_subnet: 169.254.20.1/24
           remote_sourceip: "169.254.22.1-169.254.22.254"
           remote_subnet: 169.254.22.1/24
           local_sourceip: "169.254.20.1-169.254.20.254"
     ```
     
   - Run `./deploy.py` and wait till it ends successfully.
   
2. EdgeA:

   - Using flavor sdwan to deploy EdgeA,  references **ITP/NED/46/02**, Config one *ipsec_rules* *profiles* as initiate， different configurations like as:

     - *enable_dhcp*: true
     - *static_ip_address**: no need to set*
     - *remote*: set Hub WAN interface ip for Edge A， such as 192.168.0.10
     - *vpp_ip_address*:  as IPSec local subnet ip segment

     ```yaml
     _sdewan_cnfvpp_cfg:
       devs:
         - "0000:66:00.1"
         - "0000:66:00.2"
       wan:
         - name: GigabitEthernet66/0/1
           static_ip_address: 
           enable_default_gateway: true
           preference: 0
           weight: 1
           enable_dhcp: true
           enable_nat: true
       lan:
         - name: GigabitEthernet66/0/2
       dhcp:
         server:
           ippool_start: 172.30.10.100
           ippool_end: 172.30.10.200
           ippool_gateway: 172.30.10.1
           ippool_mask: 255.255.255.0
           ippool_leasetime: 28800
           ippool_dns: 8.8.8.8
           ippool_address: 172.30.10.4/24
           ippool_subnet: 172.30.10.0/24
       gateway_tap:
         vpp_ip_address: 169.254.21.1
         host_ip_address: 169.254.21.5/24
       metadata:
         name: sdewan-cnfvpp-pod
         namespace: cnfvpp
         labels: sdewan-cnfvpp-pod
         cert: cnf-default-cert
         auth:
           name: cnf-default-auth
           user: admin
           pwd: admin
       nodeSelectorLabelName: "sdewan-cnfvpp-node"
       nodeSelectorLabelValue: "controller"
       resources:
         limits:
           hugepages: 200Mi
           memory: 500Mi
         requests:
           cpu: 150m
       cpu:
         maincore: 0
         corelist_workers:
       image:
         sdewan_cnfvpp_image: sdewan-cnfvpp-service
       enable_ipsec: true
       ipsec_rules:
         - name: cnfvpp_ipsec_tunnelA
           remote: 192.168.0.10
           pre_shared_key: "0123456789012345"
           local_identifier: cnfvppA
           remote_identifier: cnfvppA
           local_subnet: 169.254.21.1/24
           remote_sourceip: "169.254.20.1-169.254.20.254"
           remote_subnet: 169.254.20.1/24
           local_sourceip: "169.254.21.1-169.254.21.254"
     
     ```

   - Run `./deploy.py` and wait till it ends successfully.

3. EdgeB:
   - Using flavor sdwan to deploy EdgeA,  references **ITP/NED/46/02**, Config one *ipsec_rules* *profiles* as initiate， different configurations like as:

     - *enable_dhcp*: true
     - *static_ip_address**: no need to set***
     - remote*: set Hub WAN interface ip for Edge B， such as 192.168.0.20*
     - vpp_ip_address*:  as IPSec local subnet ip segment

     ```yaml
     _sdewan_cnfvpp_cfg:
       devs:
         - "0000:66:00.1"
         - "0000:66:00.2"
       wan:
         - name: GigabitEthernet66/0/1
           static_ip_address: 
           enable_default_gateway: true
           preference: 0
           weight: 1
           enable_dhcp: true
           enable_nat: true
       lan:
         - name: GigabitEthernet66/0/2
       dhcp:
         server:
           ippool_start: 172.30.10.100
           ippool_end: 172.30.10.200
           ippool_gateway: 172.30.10.1
           ippool_mask: 255.255.255.0
           ippool_leasetime: 28800
           ippool_dns: 8.8.8.8
           ippool_address: 172.30.10.4/24
           ippool_subnet: 172.30.10.0/24
       gateway_tap:
         vpp_ip_address: 169.254.22.1
         host_ip_address: 192.254.22.5/24
       metadata:
         name: sdewan-cnfvpp-pod
         namespace: cnfvpp
         labels: sdewan-cnfvpp-pod
         cert: cnf-default-cert
         auth:
           name: cnf-default-auth
           user: admin
           pwd: admin
       nodeSelectorLabelName: "sdewan-cnfvpp-node"
       nodeSelectorLabelValue: "controller"
       resources:
         limits:
           hugepages: 200Mi
           memory: 500Mi
         requests:
           cpu: 150m
       cpu:
         maincore: 0
         corelist_workers: 
       image:
         sdewan_cnfvpp_image: sdewan-cnfvpp-service
       enable_ipsec: true
       ipsec_rules:
         - name: cnfvpp_ipsec_tunnelB
           remote: 192.168.0.20
           pre_shared_key: "0123456789012345"
           local_identifier: cnfvppB
           remote_identifier: cnfvppB
           local_subnet: 169.254.22.1/24
           remote_sourceip: "169.254.20.1-169.254.20.254"
           remote_subnet: 169.254.20.1/24
           local_sourceip: "169.254.22.1-169.254.22.254"
     
     ```

   - Run `./deploy.py` and wait till it ends successfully.

4. Deploy app pod (such as ngfw pod) on the EdgeA and EdgeB，with  `kubectl apply -f ngfw.yaml`,  and check pod with `kubectl get po -n default -o wide`,  the result should be output: 

   On the EdgeA:

   ```
   openness@controller:~$  kubectl get po -n default -o wide
   NAME                    READY   STATUS    RESTARTS   AGE    IP             NODE         NOMINATED NODE   READINESS GATES
   ngfw-759d4d7d4d-2q88v   1/1     Running   0          3d2h   10.245.49.35   controller   <none>           <none>
   ```

   On the EdgeB:

   ```
   openness@controller:~$  kubectl get po -n default -o wide
   NAME                    READY   STATUS    RESTARTS   AGE    IP             NODE         NOMINATED NODE   READINESS GATES
   ngfw-759d4d7d4d-2q56v   1/1     Running   0          3d2h   10.245.49.48   controller   <none>           <none>
   ```

5. Check result on hub, EdgeA and Edge B separatly.

   Run the following commands on both Edge nodes and Hub node:

   ```
   openness@controller:~$ kubectl get po -n cnfvpp
   NAME                                 READY   STATUS    RESTARTS   AGE
   sdewan-cnfvpp-pod-68d99d5d4c-xjxtn   2/2     Running   0          10m
   kubectl get pod -n default -o wide
   NAME                    READY   STATUS    RESTARTS   AGE   IP       
   ngfw-759d4d7d4d-2q88v   1/1     Running   0          47m   10.245.49.48      
   openness@controller:~$ kubectl get po -n sdewan-system
   NAME                                 READY   STATUS    RESTARTS   AGE
   sdewan-controller-8467568d79-7vcqf   2/2     Running   0          51m
   ```
   Run the following command on Edge nodes:
   ```
   openness@controller:~$ kubectl get pod -n default -o wide
   NAME                    READY   STATUS    RESTARTS   AGE   IP
   ngfw-759d4d7d4d-2q88v   1/1     Running   0          47m   10.245.49.50
   ```

6. Ping test,  as below *10.245.49.48* is ngfw pod ip of EdgeB  : 

   - Execute on Edge node A:

   ```shell
   openness@controller:~$ kubectl exec -it ngfw-759d4d7d4d-2q88v  -- ping 10.245.49.48 -c 4
   ```

   - Example output:

   ```
   ping 10.245.49.48 56(84) bytes of data.
    64 bytes from 10.245.49.48: icmp_seq=1 ttl=115 time=137 ms
    64 bytes from 10.245.49.48: icmp_seq=2 ttl=115 time=351 ms
    64 bytes from 10.245.49.48: icmp_seq=1 ttl=115 time=139 ms
    64 bytes from 10.245.49.48: icmp_seq=2 ttl=115 time=301 ms
   ```

## References
- [ICN SDEWAN documentation](https://wiki.akraino.org/display/AK/ICN+-+SDEWAN)
- [ovn4nfv k8s plugin documentation](https://github.com/opnfv/ovn4nfv-k8s-plugin)
- [Service Function Chaining (SFC) Setup](https://github.com/opnfv/ovn4nfv-k8s-plugin/blob/master/demo/sfc-setup/README.md)
- [Utilizing a Service Mesh for Edge Services in OpenNESS](https://github.com/otcshare/ido-specs/blob/master/doc/applications/openness_service_mesh.md)
- [Using Intel® Movidius™ Myriad™ X High Density Deep Learning (HDDL) solution in OpenNESS](https://github.com/otcshare/ido-specs/blob/master/doc/building-blocks/enhanced-platform-awareness/openness_hddl.md)
- [Node Feature Discovery support in OpenNESS](https://github.com/otcshare/ido-specs/blob/master/doc/building-blocks/enhanced-platform-awareness/openness-node-feature-discovery.md)
- [OpenVINO™ Sample Application in OpenNESS](https://github.com/otcshare/ido-specs/blob/78d7797cbe0a21ade2fdc61625c2416d8430df23/doc/applications/openness_openvino.md)

## Acronyms

|             |                                                               |
|-------------|---------------------------------------------------------------|
| API         | Application Programming Interface                             |
| CERA        | Converged Edge Reference Architectures
| CR          | Custom Resource                                               |
| CRD         | Custom Resource Definition                                    |
| CNF         | Cloud-native Network Function                                 |
| DNAT        | Destination Network Address Translation                       |
| HDDL        | High Density Deep Learning                                    |
| IP          | Internet Protocol                                             |
| NAT         | Network Address Translation                                   |
| NFD         | Network Feature Discovery                                     |
| SM          | Service Mesh                                                  |
| SD-WAN      | Software-Defined Wide Area Network                            |
| SNAT        | Source Network Address Translation                            |
| TCP         | Transmission Control Protocol                                 |
| uCPE        | Universal Customer Premise Equipment                          |

## Terminology

| Term | Description |
|:-----: | ----- |
| EWO | <p> Edge WAN Overlay</p>|
| Overlay controller | <p> is a Central Controller provides central control of SDEWAN overlay networks by automatically configuring the SDEWAN CNFs through SDEWAN CRD controller located in edge location clusters and hub clusters</p>|
| EWO Controller | <p>To represent central overlay controller</p>|
| EWO Operator | <p>To represent CRD controller</p>|
| EWO CNF | <p>To represent OpenWRT based CNF. </p>|
| SDEWAN CRD Controller | <p>is implemented as k8s CRD Controller, it manages CRDs (e.g. Firewall related CRDs, Mwan3 related CRDs and IPsec related CRDs etc.) and internally calls SDEWAN Restful API to do CNF configuration. And a remote client (e.g. SDEWAN Central Controller) can manage SDEWAN CNF configuration through creating/updating/deleting SDEWAN CRs. </p>|
| OpenWRT based CNF | <p>The CNF is implemented based on OpenWRT, it enhances OpenWRT Luci web interface with SDEWAN controllers to provide Restful API for network functions configuration and control.</p>|
