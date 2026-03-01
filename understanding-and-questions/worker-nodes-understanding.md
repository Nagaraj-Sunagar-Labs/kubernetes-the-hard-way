# Kubernetes Worker Nodes: Understanding Notes

## 1) Swap in Linux (and Kubernetes)
Swap is disk used as extra virtual memory when RAM is under pressure.

- RAM is fast; swap is much slower.
- If RAM fills up, kernel may move less-used pages to swap.
- In Kubernetes setups, swap is commonly disabled for predictable memory behavior.

Useful commands:
```bash
swapon --show
free -h
sudo swapoff -a
```

## 2) Do Pods use worker node RAM?
Yes. Pods/containers run as Linux processes on worker nodes, and their memory comes from node RAM.

- Kubelet and cgroups enforce memory limits.
- If memory pressure is high, Kubernetes may evict pods.
- Kernel OOM killer can terminate processes when needed.

## 3) CNI in beginner mode
CNI (Container Network Interface) is the mechanism/plugins Kubernetes uses to set up pod networking.

Kubernetes says: run pod.
CNI says: I will wire pod networking.

CNI typically does:
- create pod network interface
- assign pod IP
- set routes and networking rules

Without CNI, pods may start but networking will not work correctly.

## 4) CNI plugins used in this lab

### bridge plugin (`10-bridge.conf`)
- Creates Linux bridge `cni0` on each worker.
- Connects pod interfaces to that bridge.
- Enables pod networking on the node.

Important fields:
- `type: bridge`
- `bridge: cni0`
- `isGateway: true`
- `ipMasq: true`

### loopback plugin (`99-loopback.conf`)
- Enables `lo` interface inside pod namespace (`127.0.0.1`).
- Needed for localhost communication inside the pod.

Important fields:
- `type: loopback`
- `name: lo`

### host-local IPAM (inside `10-bridge.conf`)
- IPAM = IP Address Management.
- `host-local` allocates pod IPs locally on each worker.

Important fields:
- `ipam.type: host-local`
- `ipam.ranges`: CIDR pool for pod IP assignment on that worker

## 5) `ipam.type` and `ipam.ranges` (simple view)
- `ipam.type`: who assigns pod IPs.
- `ipam.ranges`: from which subnet(s) IPs are assigned.

Example:
- node-0 range: `10.200.0.0/24`
- pod A gets `10.200.0.2`
- pod B gets `10.200.0.3`

node-1 should use a different, non-overlapping range (for example `10.200.1.0/24`).

## 6) Simple analogy for bridge/loopback/host-local
- `bridge`: building's local road/switch connecting apartments (pods).
- `loopback`: internal intercom inside one apartment (`localhost`).
- `host-local`: apartment-number allocator (assigns pod IP numbers).

## 7) What is `modprobe`?
`modprobe` loads/unloads Linux kernel modules (with dependencies).

Why used on worker nodes:
- Enable required kernel capabilities for container networking/runtime.

Common examples:
```bash
sudo modprobe br_netfilter
sudo modprobe overlay
```

Check loaded modules:
```bash
lsmod | grep -E 'br_netfilter|overlay'
```
