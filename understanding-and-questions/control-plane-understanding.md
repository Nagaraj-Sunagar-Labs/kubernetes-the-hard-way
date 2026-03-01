# Kubernetes Control Plane Understanding Notes

## Control Plane Data Flow

```mermaid
flowchart LR
    U[kubectl / clients] --> A[kube-apiserver]
    K[kubelet on nodes] --> A
    C[kube-controller-manager] -->|watch/read| A
    S[kube-scheduler] -->|watch/read| A

    A -->|persist state| E[(etcd)]
    E -->|read/watch state| A

    C -->|create/update objects\n(deployments->replicasets->pods,\nnode/serviceaccount/endpoints, etc.)| A
    S -->|bind Pod to Node| A
    A -->|assigned PodSpec| K
    K -->|Pod/Node status| A
```

## Why these binaries are copied on control-plane machine
In this step, the control-plane host needs these components:
- `kube-apiserver`: cluster API entrypoint.
- `kube-controller-manager`: reconciliation controllers.
- `kube-scheduler`: pod placement engine.

These three (plus `etcd`) form the core control plane.

## kube-controller-manager functionality
`kube-controller-manager` runs many controllers in one process.

Main job:
- Continuously reconcile desired state with actual state.

Examples:
- Deployment/ReplicaSet controller: ensures expected pod count.
- Node controller: tracks node health.
- ServiceAccount controller: creates default service accounts.
- Endpoints controller: updates service endpoints.
- Job controller: manages job pod completion.

## kube-scheduler functionality
`kube-scheduler` places unscheduled Pods onto nodes.

How it works (simplified):
1. Watch for Pods without `nodeName`.
2. Filter nodes that cannot run the Pod.
3. Score remaining nodes.
4. Choose best node and write binding via API server.

## Why it is named `kube-apiserver` (not `kube-apiservice`)
- It is the server process that exposes Kubernetes API endpoints.
- In Kubernetes, `APIService` is already a specific resource kind used for API aggregation.
- So `apiserver` is the correct and unambiguous name.
