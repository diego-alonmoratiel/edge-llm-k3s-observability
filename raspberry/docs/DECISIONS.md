# Design decisions

Short rationales for the main choices — useful to defend the project.

## k3s instead of kubeadm / full Kubernetes
k3s is a single binary, ships containerd, and fits a 4 GB edge node. kubeadm
assumes more RAM and more moving parts than this project needs.

## Control plane on the PC, worker on the Pi
The Pi is the constrained device, so it only runs workloads. Keeping the API
server and etcd on a beefier machine means they do not compete with the LLM for
the Pi's 4 GB of RAM.

## One playbook, no roles
The whole deployment is three goals in one file that reads top to bottom. Roles
add indirection that is not needed at this size; a reader can follow `site.yml`
without jumping between files.

## Plain variables in group_vars/all.yml
All settings live in one file so there is a single place to look. For a project
this small, role defaults would only scatter them.

## Official multi-arch image instead of building llama.cpp
`ghcr.io/ggml-org/llama.cpp:server` already ships an arm64 build and is pulled
by k3s on first run. Building a custom image would add a Dockerfile, a build
step and a registry without changing the result for this use case.

## Model on a hostPath PV/PVC, not inside the image
The GGUF is ~1 GB. Keeping it on the host and mounting it means changing models
never requires rebuilding or re-pushing an image. PV/PVC model the storage as a
Kubernetes object rather than an inline `hostPath` volume.

## nodeSelector by architecture
The cluster mixes amd64 and arm64. Pinning the Deployment to
`kubernetes.io/arch: arm64` guarantees the pod lands on the Pi, where the model
is.

## Tailscale assumed to be installed
Both nodes are expected to already be on the tailnet, so the playbook does not
manage Tailscale. The worker joins the API server over the control plane's
Tailscale IP; this is what makes the setup work behind NAT (the control plane is
on WSL2).

## tls-san on the API server
k3s certificates only include node IPs by default. Because the worker joins over
the control plane's Tailscale IP, that IP must be added to the serving
certificate (`tls-san`), otherwise the agent's TLS verification fails.

## NodePort instead of an Ingress
There is no external load balancer and Traefik is disabled to save RAM. A
NodePort on the tailnet is the simplest way to expose one service securely.

## Observability deferred (phase 2)
Only the collection side (`node-exporter`) is deployed now. Prometheus and
Grafana will run on the PC, scraping the tailnet.
