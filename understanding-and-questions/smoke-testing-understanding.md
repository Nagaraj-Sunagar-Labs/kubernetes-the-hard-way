# Smoke Testing Kubernetes Cluster: Understanding Notes

## 1) Secret creation flow (`kubectl create secret generic`)
Command:
```bash
kubectl create secret generic kubernetes-the-hard-way --from-literal="mykey=mydata"
```

End-to-end flow:
1. `kubectl` builds Secret object and sends HTTPS request to `kube-apiserver`.
2. API server runs authentication, authorization (RBAC), admission, and validation.
3. API server converts object to internal storage form.
4. If encryption-at-rest is configured, Secret data is encrypted before persistence.
5. API server writes to `etcd`.
6. `etcd` commits write and API server returns success response.

Storage path conceptually:
- `/registry/secrets/<namespace>/kubernetes-the-hard-way`

## 2) API request structure for Secret create
Typical call:
- `POST /api/v1/namespaces/default/secrets`

Body (simplified):
```json
{
  "apiVersion": "v1",
  "kind": "Secret",
  "metadata": {
    "name": "kubernetes-the-hard-way",
    "namespace": "default"
  },
  "type": "Opaque",
  "data": {
    "mykey": "bXlkYXRh"
  }
}
```

Note:
- `mydata` becomes base64 `bXlkYXRh` in `data`.

## 3) Beginner hierarchy: cluster to container
1. Cluster
2. Namespace
3. Deployment / StatefulSet
4. ReplicaSet (for Deployment)
5. Pod
6. Container

Quick meanings:
- Deployment: manages stateless app rollout/replicas.
- ReplicaSet: keeps N pod replicas running.
- StatefulSet: stable identity/storage for stateful apps.

## 4) `kubectl create deployment nginx --image=nginx:latest`
What it triggers:
1. Deployment object created.
2. Deployment controller creates ReplicaSet.
3. ReplicaSet controller creates Pod(s).
4. Scheduler assigns Pod to a node.
5. Kubelet starts container via container runtime.

Default behavior:
- 1 replica unless specified.

## 5) Scheduler location clarification
- `kube-scheduler` runs on control plane.
- It does **not** run on worker nodes.

## 6) End-to-end control-plane call chain for app scheduling
1. `kubectl -> kube-apiserver` create Deployment.
2. `kube-apiserver -> etcd` persist Deployment.
3. `kube-controller-manager -> kube-apiserver` (watch/list + create ReplicaSet/Pod).
4. `kube-apiserver -> etcd` persist ReplicaSet/Pod.
5. `kube-scheduler -> kube-apiserver` watch unscheduled Pod, then bind chosen node.
6. `kube-apiserver -> etcd` persist Pod with `spec.nodeName`.
7. `kubelet (chosen worker) -> kube-apiserver` watch assigned Pod.
8. Kubelet uses container runtime + CNI to run Pod.
9. `kubelet -> kube-apiserver` send status updates.

Important rule:
- Components use API server; they do not directly write to etcd.

## 7) Why controller-manager exists separately from API server
Separation of concerns:
- API server: API front door + validation + persistence.
- Controller-manager: reconciliation control loops.

Benefits:
- Better reliability and fault isolation.
- Independent scaling/tuning.
- Cleaner modular architecture.

## 8) kube-scheduler internal operation (simplified)
1. Watch unscheduled Pods.
2. Put Pods into scheduling queue.
3. Filter nodes (resource/constraints/taints/affinity).
4. Score feasible nodes.
5. Pick best node.
6. Bind Pod via API server.
7. Retry with backoff if unschedulable.

## 9) containerd architecture (kube worker runtime)
Core parts:
- `containerd` daemon
- CRI plugin (for kubelet integration)
- `containerd-shim`
- OCI runtime (`runc`)
- snapshotters (filesystem layers)
- content/metadata stores

Pod start path:
- kubelet calls CRI -> containerd pulls/unpacks image -> creates sandbox/container -> starts via shim+runc -> kubelet gets status/logs.

## 10) `kubectl port-forward` in detail
Purpose:
- Temporary local tunnel to Pod/Service port.

Example:
```bash
kubectl port-forward service/nginx 8080:80
```

Traffic path:
- local machine -> kubectl -> API server stream -> kubelet on target node -> pod port.

Use case:
- debugging/admin access, not production ingress.

## 11) `kubectl expose deployment` in detail
Example:
```bash
kubectl expose deployment nginx --port=80 --target-port=80 --type=ClusterIP
```

What happens:
- Creates Service selecting deployment pods by labels.
- Service gets stable virtual IP/DNS.
- EndpointSlices track matching pods.
- kube-proxy programs node networking rules for load-balancing to pods.

## 12) Does a Service last forever?
A Service persists until deleted or cluster state is destroyed.

It is removed when:
- manually deleted,
- namespace deleted,
- cluster/etcd reset,
- replaced by automation changes.

Pod restarts do not delete the Service.
