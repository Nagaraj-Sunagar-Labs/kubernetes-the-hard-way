# Docker Runtime and CRI: Understanding Notes

## 1) How to check running containers on a Kubernetes worker node
In this setup, workers use `containerd`, so use `crictl` for runtime inspection.

Commands:
```bash
sudo crictl ps
sudo crictl ps -a
sudo crictl pods
sudo crictl images
```

If endpoint is not auto-detected:
```bash
sudo crictl --runtime-endpoint unix:///var/run/containerd/containerd.sock ps
```

## 2) Difference between `crictl` and `docker`
- `docker` is a full container platform CLI that talks to Docker Engine (`dockerd`).
- `crictl` is a lightweight CLI for CRI-compatible runtimes used by Kubernetes nodes (`containerd`, `CRI-O`).

Practical difference in Kubernetes nodes:
- Use `crictl` to debug what kubelet manages via CRI.
- `docker` may not show the relevant containers in modern Kubernetes setups that do not use dockershim.

Quick mapping:
- `docker ps` <-> `crictl ps`
- `docker images` <-> `crictl images`

Limitation:
- `crictl` is not an image build tool (no `docker build` equivalent).

## 3) Who built `crictl`?
`crictl` is developed and maintained by the Kubernetes ecosystem (SIG Node) in the `cri-tools` project (`kubernetes-sigs/cri-tools`).

## 4) Runtime context in Kubernetes The Hard Way
Worker-node runtime path in this tutorial:
1. kubelet receives assigned Pod
2. kubelet calls CRI runtime (`containerd`)
3. runtime creates sandbox/container and starts process
4. kubelet reports status to API server

So for node-level runtime debugging in this lab, `crictl` is the primary operational tool.
