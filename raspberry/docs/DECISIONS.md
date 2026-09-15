# Design decisions

Short, interview-friendly rationales for the main choices.

## k3s instead of kubeadm / full Kubernetes
k3s is a single binary, ships containerd and a local-path provisioner, and
fits a 4 GB edge node. kubeadm assumes more RAM and more moving parts than this
project needs.

## Control plane on the PC, worker on the Pi
The Pi is the constrained device, so it only runs workloads. Keeping the
control plane on a beefier machine means the API server, scheduler and etcd
are not competing with the LLM for the Pi's 4 GB of RAM.

## Tailscale at the host level, not the Kubernetes operator
Both nodes run `tailscaled` and can reach each other directly. The worker joins
the API server over the control plane's Tailscale IP, and clients reach the API
over the tailnet. This avoids running (and debugging) an in-cluster operator on
an edge box, and reuses the same private network for SSH and metrics.

## tls-san on the API server
k3s certificates include node IPs by default. Because the worker joins over the
control plane's Tailscale IP, that IP must be in the serving certificate
(`tls-san`), otherwise the agent's TLS verification fails.

## Docker is only used to build the image
k3s already uses containerd, so Docker is not the cluster runtime. It exists to
compile `llama-server` with `-DGGML_NATIVE=ON` for the Pi's CPU and to produce
a tar that is imported into containerd (`docker save | k3s ctr images import`).
That keeps the image on-node, with no registry to run or secure.

## kubectl apply instead of the kubernetes.core collection
The manifests are plain, portable YAML. Applying them with `kubectl` keeps them
readable and reusable by hand, and removes the Kubernetes Python client
dependency from the worker. Ansible orchestrates; the manifests stay
Kubernetes-native.

## PersistentVolume + PersistentVolumeClaim over an inline hostPath
Using a PV/PVC documents the storage as a first-class object and keeps the pod
spec clean. The PV is backed by `hostPath` because this is a single-node edge
deployment; a multi-node cluster would use a dynamic provisioner instead.

## nodeSelector by architecture
The cluster mixes amd64 and arm64. Pinning the Deployment to
`kubernetes.io/arch: arm64` guarantees it lands on the Pi, where the built
image and the model file live.

## NodePort instead of an Ingress
There is no external load balancer and Traefik is disabled to save RAM. A
NodePort on the tailnet is the simplest way to expose one service securely.

## Secrets in the inventory vault
The Tailscale auth key is stored with `ansible-vault` in
`inventory/group_vars/all/vault.yml`, so it is auto-loaded when present and
never committed in clear text. If it is absent, the login is simply left manual.

## Observability deferred (phase 2)
The repository name promises observability, but only the collection side
(`node-exporter`) is deployed now. Prometheus and Grafana will run on the PC,
scraping the tailnet. This keeps the first milestone honest and small.
