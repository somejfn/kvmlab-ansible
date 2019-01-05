# KVM Lab Ansible Playbook

Build a set of virtual machines on a single KVM host specifically for deploying a K8s lab.  

My goal was to have a virtual infrastructure allowing to integrate a multi-node K8s cluster with:
*  _metallb_ with BGP based routing and ECMP load balancing. This enables routable VIPs for services of type LoadBalancer such as an ingress controller.
* _external-dns_ setup to publish ingress and service names to an authoritative DNS system.

A router VM separates the K8s worker segment from the KVM host and has the bits needed to peer with _metallb_ and also runs a _powerdns_ instance to work with _external-dns_.

All the guest VM images are unmodified *cloud* images from CentOS 7 and are configured via cloud init (by a config drive)

System requirements:

  - A KVM host with at least 8GB of RAM (16BG better)
	- Ansible >=2.7
	- Libvirtd with the virt-install and genisoimage utilities
  - 2 KVM networks defined to simulate a public and private segments (see sample XML files)


# Usage

See the defaults for the VMs defined under `ansible/roles/kvm-infra/defaults/main.yml`.  This builds the dual-homed router VM and 3 more KVM guests to be used as a K8s controller (node1) and 2 worker nodes (nodes[2-3])

Make sure to update the SSH key in there as this is the only way to SSH in after deploying the playbook.

Once the virtual infrastructure is up,  I suggest to deploy your K8s cluster easily with the excellent _kubeadm-ansible_ project.

Then install _helm_/_tiller_ on your cluster, _metallb_ and _external-dns_ (sample provided should work unmodified).

```
helm install stable/metallb -n metallb --namespace=metallb-system -f k8s/metallb/values.yaml
```

See BGP daemon log on router VM has established peering with the 2 worker nodes

```
[root@routerng ~]# docker logs bgp
time="2019-01-03T14:21:14Z" level=info msg="gobgpd started"
time="2019-01-03T14:21:14Z" level=info msg="Finished reading the config file" Topic=Config
time="2019-01-03T14:21:14Z" level=info msg="PeerGroup k8s-workers is added"
time="2019-01-03T14:21:14Z" level=info msg="Add a peer group configuration" Name=k8s-workers Topic=Peer
time="2019-01-03T14:21:14Z" level=info msg="Dynamic Neighbor 192.168.200.0/24 is added to PeerGroup k8s-workers"
time="2019-01-03T17:53:19Z" level=info msg="Peer Up" Key=192.168.200.33 State=BGP_FSM_OPENCONFIRM Topic=Peer
time="2019-01-03T17:53:22Z" level=info msg="Peer Up" Key=192.168.200.32 State=BGP_FSM_OPENCONFIRM Topic=Peer
```

Create service LB type.  Here as an example for the k8s dashboard
```
$ kubectl expose service  kubernetes-dashboard -n kube-system --name lb-dash --type=LoadBalancer --target-port=8443 --port=443
service/lb-dash exposed
```

Router has ECMP routing setup for the service LB IP

```
ip route
...
198.51.100.0 proto zebra
	nexthop via 192.168.200.33 dev eth0 weight 1
	nexthop via 192.168.200.32 dev eth0 weight 1
```

Now you can curl it
