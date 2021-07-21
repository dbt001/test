```text
SPDX-License-Identifier: Apache-2.0
Copyright © 2021 Intel Corporation
```
- [ITP/NED/46: OpenNESS SDEWAN-CNF-VPP Deployment](#itpned46-openness-sdewan_openwrt)
  - [ITP/NED/46/01:Scenario A:Using Flavor to deploy on Edge Node](#itpned4601-scenario-a)
	- [Test Summary](#test-summary)
	- [Prerequisites](#prerequisites)
	- [Test Steps](#test-steps)
  - [ITP/NED/46/02:Scenario B:Using Flavor to deploy on Hub Node](#itpned4602-scenario-b)
	- [Test Summary](#test-summary-2)
	- [Prerequisites](#prerequisites-2)
	- [Test Steps](#test-steps-2)
  - [ITP/NED/46/03:Scenario C:IKEV2 IPSec](#itpned4603-scenario-c)
	- [Test Summary](#test-summary-3)
	- [Prerequisites](#prerequisites-3)
	- [Test Steps](#test-steps-3)
  - [ITP/NED/46/04:Scenario D:Verify Data Flow](#itpned4604-scenario-d)
	- [Test Summary](#test-summary-4)
	- [Prerequisites](#prerequisites-4)
	- [Test Steps](#test-steps-4)

# ITP/NED/46: OpenNESS SDEWAN-CNF-VPP Deployment
- This test suite includes all tests related to the SDEWAN CNF(Containerized Network Function) based on VPP/DPDK network stack.

**Test suite hardware requirements**:
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

**Test suite software requirements**:

- Host Operating System
  - _ubuntu 20.04_
- Host Linux Kernel
  - _5.4.0-77-generic_
- BIOS VT-d Configuration
  - _Enable_
- Boot settings
  - _iommu=pt intel_iommu=on_
	
**Test Suite Topology Overview**:

![Topology Overview](ts_resources/images/itp_ned_46_01.png)

## ITP/NED/46/01: Scenario A:Using Flavor to deploy on Edge Node

### Test Summary 

> support flavor to deploy SDEWAN VPP CNF automatically on Edge Nodes: Edge nodeA and Edge nodeB.

### Prerequisites

1. Ubuntu OS is installed sucessfully.
2. `otcshare/ido-converged-edge-experience-kits` checked out on all edge and hub servers.
3. use command `git submodule init` and `git submodule update` checkout `ceek` folder.
4. SSH key generated & copied for the remote server.
5. configure `ansible_host` and `ansible_user` for `controller_group` and `edgenode_group` in `inventory.yml` file under `/ido-converged-edge-experience-kits/`.
6. set proper proxy for all nodes.

### Test Steps

1. deploy SDEWAN CNF on edge nodes A&B using flavor. 
2. Modify Pre-defined configuration File：
	```yaml
	to add configuration details here
	``` 
3. Modify _flavors/sdwan/all.yml_  for	sdewan cnf(vpp) edge deployment：
	```yaml
	to add configuration details here
	``` 
4. Modify _inventory.yml (falvor: sdwan)_
	```yaml
	to add configuration details here
	```		
5. run _`./deploy.py`_ and wait till it ends successfully.
6. Check the node result:
   - Execute:
	 ```shell
	 root@ceekvpp:~# kubectl get nodes
	 ```
   - Example output:
	 ```
	 NAME	   STATUS	ROLES				   AGE	  VERSION
	 ceekvpp   Ready	control-plane,master   3d1h	  v1.20.0
	 ```
7. Check the pods(sdewan,sdewan-crd-controller,harbor,calico...) status:
   - Execute:
	 ```shell
	 root@ceekvpp:~# kubectl get po -A
	 ```
   - Example output:
	 ```
	 NAMESPACE       NAME                                               READY   STATUS    RESTARTS   AGE
     cert-manager    cert-manager-5597cff495-dcmvm                      1/1     Running   0          24h
     cert-manager    cert-manager-cainjector-bd5f9c764-8rdms            1/1     Running   0          24h
     cert-manager    cert-manager-webhook-5f57f59fbc-wgnsr              1/1     Running   0          24h
     default         sdewan-9cd4c5c8c-8c886                             2/2     Running   0          20h
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
     kube-system     etcd-ceekvpp                                       1/1     Running   0          3d1h
     kube-system     kube-apiserver-ceekvpp                             1/1     Running   0          3d1h
     kube-system     kube-controller-manager-ceekvpp                    1/1     Running   0          3d1h
     kube-system     kube-proxy-nfw5x                                   1/1     Running   0          3d1h
     kube-system     kube-scheduler-ceekvpp                             1/1     Running   0          3d1h
     sdewan-system   sdewan-crd-controller-5bbcfc6875-jqg4m             2/2     Running   0          21h
	 ```
8. Check the reference volumes special "cnf-default-cert" and “cnf-default-auth” have been mounted into cnf pod:
   - Execute:
	 ```shell
     root@ceekvpp:~# kubectl get pod
     NAME                     READY   STATUS    RESTARTS   AGE
     sdewan-9cd4c5c8c-8c886   2/2     Running   0          20h
     root@ceekvpp:~#kubectl exec -it sdewan-9cd4c5c8c-8c886 -c sdewan -- ll /etc/crd-ctrlr-adpt/cert
     root@ceekvpp:~#kubectl exec -it sdewan-9cd4c5c8c-8c886 -c sdewan -- ll /etc/crd-ctrlr-adpt/auth
	 ```
   - Example output:
	 ```
     The directories cert and auth are existing.
	 ```
	 
9. Check the interface address result inside cnf pod:
   - Execute:
	 ```shell
     root@ceekvpp:~# kubectl get pod
     NAME                     READY   STATUS    RESTARTS   AGE
     sdewan-9cd4c5c8c-8c886   2/2     Running   0          20h
     root@ceekvpp:~#kubectl exec -it sdewan-9cd4c5c8c-8c886 -c sdewan -- vppctl sh int addr
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
	 ```
10. Check the nat interface result inside cnf pod:
   - Execute:
	 ```shell
	 root@ceekvpp:~#kubectl exec -it sdewan-9cd4c5c8c-8c886 -c sdewan -- vppctl sh nat44 interfaces
	 ```
   - Example output:
	 ```
     NAT44 interfaces:
      loop0 in
      GigabitEthernet66/0/2 out
	 ```
	 
## ITP/NED/46/02: Scenario B:Using Flavor to deploy on Hub Node

### Test Summary 

> support flavor to deploy SDEWAN VPP CNF automatically on Hub Node.

### Prerequisites

1. Ubuntu OS is installed sucessfully.
2. `otcshare/ido-converged-edge-experience-kits` checked out on all edge and hub servers.
3. use command `git submodule init` and `git submodule update` checkout `ceek` folder.
4. SSH key generated & copied for the remote server.
5. configure `ansible_host` and `ansible_user` for `controller_group` and `edgenode_group` in `inventory.yml` file under `/ido-converged-edge-experience-kits/`.
6. set proper proxy for all nodes.

### Test Steps

1. deploy SDEWAN CNF on hub node using flavor. 
2. Modify Pre-defined configuration File：
	```yaml
	to add configuration details here
	``` 
3. Modify _flavors/sdwan/all.yml_  for	sdewan cnf(vpp) hub deployment：
	```yaml
	to add configuration details here
	``` 
4. Modify _inventory.yml (falvor: sdwan)_
	```yaml
	to add configuration details here
	```		
5. run _`./deploy.py`_ and wait till it ends successfully.
6. Check the node result:
   - Execute:
	 ```shell
	 root@ceekvpp:~# kubectl get nodes
	 ```
   - Example output:
	 ```
	 NAME	   STATUS	ROLES				   AGE	  VERSION
	 ceekvpp   Ready	control-plane,master   3d1h	  v1.20.0
	 ```
7. Check the pods(sdewan,sdewan-crd-controller,harbor,calico...) status:
   - Execute:
	 ```shell
	 root@ceekvpp:~# kubectl get po -A
	 ```
   - Example output:
	 ```
	 NAMESPACE       NAME                                               READY   STATUS    RESTARTS   AGE
     cert-manager    cert-manager-5597cff495-dcmvm                      1/1     Running   0          24h
     cert-manager    cert-manager-cainjector-bd5f9c764-8rdms            1/1     Running   0          24h
     cert-manager    cert-manager-webhook-5f57f59fbc-wgnsr              1/1     Running   0          24h
     default         sdewan-9cd4c5c8c-8c886                             2/2     Running   0          20h
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
     kube-system     etcd-ceekvpp                                       1/1     Running   0          3d1h
     kube-system     kube-apiserver-ceekvpp                             1/1     Running   0          3d1h
     kube-system     kube-controller-manager-ceekvpp                    1/1     Running   0          3d1h
     kube-system     kube-proxy-nfw5x                                   1/1     Running   0          3d1h
     kube-system     kube-scheduler-ceekvpp                             1/1     Running   0          3d1h
     sdewan-system   sdewan-crd-controller-5bbcfc6875-jqg4m             2/2     Running   0          21h
	 ```
8. Check the reference volumes special "cnf-default-cert" and “cnf-default-auth” have been mounted into cnf pod:
   - Execute:
	 ```shell
     root@ceekvpp:~# kubectl get pod
     NAME                     READY   STATUS    RESTARTS   AGE
     sdewan-9cd4c5c8c-8c886   2/2     Running   0          20h
     root@ceekvpp:~#kubectl exec -it sdewan-9cd4c5c8c-8c886 -c sdewan -- ll /etc/crd-ctrlr-adpt/cert
     root@ceekvpp:~#kubectl exec -it sdewan-9cd4c5c8c-8c886 -c sdewan -- ll /etc/crd-ctrlr-adpt/auth
	 ```
   - Example output:
	 ```
     The directories cert and auth are existing.
	 ``` 
9. Check the interface address result inside cnf pod:
   - Execute:
	 ```shell
     root@ceekvpp:~# kubectl get pod
     NAME                     READY   STATUS    RESTARTS   AGE
     sdewan-9cd4c5c8c-8c886   2/2     Running   0          20h
	 root@ceekvpp:~#kubectl exec -it sdewan-9cd4c5c8c-8c886 -c sdewan -- vppctl sh int addr
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
	 ```

## ITP/NED/46/03: Scenario C:IKEV2 IPSec

### Test Summary 

> Configing IPSec on Overlay Controller->CRD Controller->SDEWAN CNF

### Prerequisites

1. scenario A and B are successful.
2. Independent of Edge and Hub cluster, Overlay Controller has been deployed sucessfuly on Kubernetes platform.

### Test Steps

1. Configing IKEv2 IPSec for sdewan cnf(Edge nodeA and Hub node) on overlay controller：
	- Edge nodeA:
	```yaml
	to add configuration details here
	``` 
	- Hub node:
	```yaml
	to add configuration details here
	``` 
2. Check the interface address result inside cnf pod:
   - Execute:
	 ```shell
     root@ceekvpp:~# kubectl get pod
     NAME                     READY   STATUS    RESTARTS   AGE
     sdewan-9cd4c5c8c-8c886   2/2     Running   0          20h
	 root@ceekvpp:~#kubectl exec -it sdewan-9cd4c5c8c-8c886 -c sdewan -- vppctl sh int addr
	 ```
   - Example output:
      ```
      ‘ipip0’ interface is existing.
	  ```
3. Check the ikev2 profile inside cnf pod:
   - Execute:
	 ```shell
	 root@ceekvpp:~#kubectl exec -it sdewan-9cd4c5c8c-8c886 -c sdewan -- vppctl sh ikev2 profile
	 ```
   - Example output:
     ```
     ’profile‘ informations is existing.
     ```
4. Check the ikev2 sa inside cnf pod:
   - Execute:
	 ```shell
	 root@ceekvpp:~#kubectl exec -it sdewan-9cd4c5c8c-8c886 -c sdewan -- vppctl sh ikev2 sa
	 ```
   - Example output:
     ```
     ’profile‘ informations is existing.
     ```
5. Check the ipsec inside cnf pod:
   - Execute:
	 ```shell
	 root@ceekvpp:~#kubectl exec -it sdewan-9cd4c5c8c-8c886 -c sdewan -- vppctl sh ipsec all
	 ```
   - Example output:
     ```
	 Expected content here
	 ``` 

## ITP/NED/46/04:Scenario D:Verify Data Flow

### Test Summary 

> Verify data flow base on 3 single cluster topology.

### Prerequisites

1. Deploy sdewan cnf using flavor on 3 single Edge and Hub nodes according to scenario A and B.
2. Independent of Edge and Hub cluster, Overlay Controller has been deployed sucessfuly on Kubernetes platform.

### Test Steps

1. Configing IKEv2 IPSec for 3 single Edge and Hub nodes on Overlay Controller：
	- Edge nodeA:
	```yaml
	to add configuration details here
	```
	- Edge nodeB:
	```yaml
	to add configuration details here
	```
	- Hub node:
	```yaml
	to add configuration details here
	``` 
2. Ping app pod ip(any one ip) of Edge B inside app pod of Edge A:
   - Execute:
	 ```shell
     on Edge node B:
	 root@ceekvpp:~# kubectl get po -A -o wide|grep harbo
     harbor          harbor-app-harbor-chartmuseum-575f7c84bd-rhnr8     1/1     Running   2          39h   10.245.95.247   ceekvpp   <none>           <none>
     harbor          harbor-app-harbor-clair-779df4555b-4z7w4           2/2     Running   244        39h   10.245.95.253   ceekvpp   <none>           <none>
     harbor          harbor-app-harbor-core-865d69fc9f-jlt9m            1/1     Running   2          39h   10.245.95.244   ceekvpp   <none>           <none>
     harbor          harbor-app-harbor-database-0                       1/1     Running   2          39h   10.245.95.238   ceekvpp   <none>           <none>
     harbor          harbor-app-harbor-jobservice-79996b68f6-bm2gg      1/1     Running   4          39h   10.245.95.246   ceekvpp   <none>           <none>
     harbor          harbor-app-harbor-nginx-7c5748b8b7-lb4xn           1/1     Running   3          39h   10.245.95.251   ceekvpp   <none>           <none>
     harbor          harbor-app-harbor-notary-server-6fff5467cb-s6nxx   1/1     Running   2          39h   10.245.95.235   ceekvpp   <none>           <none>
	 on Edge node A:
     root@ceekvpp:~# kubectl get pod -n harbor
     NAME                                               READY   STATUS    RESTARTS   AGE
     harbor-app-harbor-chartmuseum-575f7c84bd-rhnr8     1/1     Running   2          39h
     harbor-app-harbor-clair-779df4555b-4z7w4           2/2     Running   244        39h
     harbor-app-harbor-core-865d69fc9f-jlt9m            1/1     Running   2          39h
     root@ceekvpp:~# kubectl exec -it harbor-app-harbor-chartmuseum-575f7c84bd-rhnr8 -c sdewan -- ping 10.245.95.247
	 ```
   - Example output:
     ```
     ping 10.245.95.247 56(84) bytes of data.
	 64 bytes from 10.245.95.247: icmp_seq=1 ttl=115 time=137 ms
     64 bytes from 10.245.95.247: icmp_seq=2 ttl=115 time=351 ms
	 ```
3. Check the interface address result inside cnf pod of 3 nodes:
   - Execute:
	 ```shell
     root@ceekvpp:~# kubectl get pod
     NAME                     READY   STATUS    RESTARTS   AGE
     sdewan-9cd4c5c8c-8c886   2/2     Running   0          20h
	 root@ceekvpp:~#kubectl exec -it sdewan-9cd4c5c8c-8c886 -c sdewan -- vppctl sh int addr
	 ```
   - Example output:
      ```
      ‘ipip0’ interface is existing.
	  ```
4. Check the er result inside cnf pod of 3 nodes:
   - Execute:
	 ```shell
	 root@ceekvpp:~#kubectl exec -it sdewan-9cd4c5c8c-8c886 -c sdewan -- vppctl sh er
	 ```
   - Example output:
      ```
      contains ‘esp4-decrypt-tun’, ‘esp4-encrypt-tun’
	  ```






