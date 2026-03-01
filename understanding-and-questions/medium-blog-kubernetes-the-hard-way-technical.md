# Kubernetes The Hard Way: A Technical Deep Dive from Cluster Bootstrap to Smoke Tests

This article summarizes the core technical concepts behind a Kubernetes The Hard Way setup, with emphasis on control-plane internals, worker-node runtime behavior, networking, and request flow.

## 1) Control Plane Architecture
A minimal control plane in this setup contains:

- `kube-apiserver`
- `kube-controller-manager`
- `kube-scheduler`
- `etcd`

### Component responsibilities
- `kube-apiserver`: API entrypoint, authn/authz, admission, validation, and persistence gateway.
- `etcd`: strongly consistent key-value store for durable cluster state.
- `kube-controller-manager`: reconciliation loops that drive actual state toward desired state.
- `kube-scheduler`: assigns unscheduled pods to suitable worker nodes.

## 2) Source of Truth and State Flow
Kubernetes state is stored in `etcd`; API server is the component that mediates persistence.

Write path (example `kubectl apply`):
1. `kubectl -> kube-apiserver`
2. API server processes request (authn/authz/admission/validation)
3. `kube-apiserver -> etcd` write

Read path (example `kubectl get`):
1. `kubectl -> kube-apiserver`
2. `kube-apiserver -> etcd` read
3. API response returned to client

Watch/reconcile path:
- controllers and kubelets watch API server resources and react to changes.

## 3) Why etcd Uses Client and Peer URLs
`etcd` exposes separate endpoints for different traffic classes:

- Client URL (typically `:2379`): API server, `etcdctl`, and other clients.
- Peer URL (typically `:2380`): etcd member-to-member Raft communication.

Even in single-member setups, peer URL remains part of standard member configuration.

## 4) Raft Membership: Learner vs Voting Member
- Voting member: participates in quorum and leader election.
- Learner: replicates state but does not vote.

Learners are used for safer scaling:
1. add as learner
2. catch up logs/snapshot
3. promote to voting member

## 5) Deployment Scheduling Pipeline
Command example:
```bash
kubectl create deployment nginx --image=nginx:latest
```

Execution flow:
1. Deployment object stored via API server.
2. Deployment controller creates ReplicaSet.
3. ReplicaSet controller creates Pod(s).
4. Scheduler selects node and binds Pod.
5. Kubelet on selected worker starts pod via runtime.
6. Kubelet reports status back through API server.

Note: scheduler runs on control plane, not workers.

## 6) Why API Server and Controller Manager Are Separate
Separation enables:
- independent scaling and tuning
- failure-domain isolation
- cleaner separation between API persistence and reconciliation logic
- improved operational debuggability

## 7) Scheduler Internals (Simplified)
Scheduler loop:
1. watch pending pods
2. queue unscheduled pods
3. filter infeasible nodes
4. score feasible nodes
5. bind pod to best node
6. retry with backoff if unschedulable

Typical filters/scoring use resource availability, taints/tolerations, affinity rules, and topology constraints.

## 8) Worker Node Runtime Stack
Key worker-node components:
- `kubelet`
- `containerd`
- CNI plugins
- kernel networking modules (for example via `modprobe`)

### `containerd` architecture
- containerd daemon
- CRI plugin (kubelet integration)
- `containerd-shim`
- OCI runtime (`runc`)
- snapshotters/content store

Pod start path:
- kubelet CRI calls -> image pull/unpack -> sandbox/container create -> container start -> status reporting.

## 9) Pod Networking with CNI
In this setup, networking uses CNI config files such as:

- `10-bridge.conf`
- `99-loopback.conf`

### Bridge config purpose
- create bridge (`cni0`)
- connect pod interfaces
- assign pod IPs using IPAM
- apply routing/NAT behavior

### Loopback config purpose
- enable `lo` (`127.0.0.1`) inside pod namespace

### IPAM fields
- `ipam.type`: IP allocator type (for example `host-local`)
- `ipam.ranges`: subnet pools used for pod IP allocation

Each node must have non-overlapping pod CIDR ranges.

## 10) Memory Behavior on Worker Nodes
Pods use worker-node RAM via Linux process/cgroup model.

Operational implications:
- memory limits are enforced by cgroups
- under pressure, pods can be evicted
- swap is often disabled for predictable memory and scheduling behavior

## 11) Service and Traffic Exposure Commands
### `kubectl expose deployment`
Creates a Service selecting deployment pods by labels.

Effects:
- stable service identity (ClusterIP/DNS)
- endpoint discovery via EndpointSlice
- node-level forwarding rules through kube-proxy

### `kubectl port-forward`
Creates a temporary local tunnel to pod/service port through API server streaming path.

Use case: debugging and local access, not production ingress.

## 12) Secret Creation and Storage Path
Command:
```bash
kubectl create secret generic kubernetes-the-hard-way --from-literal="mykey=mydata"
```

API call pattern:
- `POST /api/v1/namespaces/default/secrets`

Processing:
- API server request pipeline (authn/authz/admission/validation)
- optional encryption-at-rest processing
- persistence in etcd under resource registry path

## 13) Systemd Operations in Bootstrap
Frequently used service management commands:
- `systemctl daemon-reload`: reload unit definitions/dependencies
- `systemctl enable <service>`: enable at boot
- `systemctl start <service>`: start immediately

`daemon-reload` does not restart services automatically.

## 14) Technical Summary
Kubernetes operation in this tutorial can be reduced to a consistent model:

- API server is the control gateway.
- etcd is durable truth.
- controllers reconcile desired state.
- scheduler performs node placement.
- kubelet + containerd + CNI realize workloads on workers.

Understanding this model makes cluster bootstrap, debugging, and day-2 operations significantly more deterministic.
