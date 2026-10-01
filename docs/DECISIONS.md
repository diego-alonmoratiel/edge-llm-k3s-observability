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

## Monitoring as plain manifests (not Helm)
Prometheus and Grafana are deployed as plain manifests, pinned to the PC node,
scraping through cluster-internal DNS. Helm's `kube-prometheus-stack` is the
standard for large clusters, but here it would add an operator, CRDs and a
duplicate node-exporter for only two scrape targets. Helm is the right tool for
third-party, complex charts (e.g. `ingress-nginx`), not for two owned workloads.

## Ephemeral storage for the monitoring stack
Prometheus and Grafana use `emptyDir`. Metric history is lost on restart, but the
dashboards and datasource are provisioned from ConfigMaps, so nothing needs
persisting to demonstrate the setup. A PVC with the `local-path` storage class
would be the upgrade path.

## The dashboard and the secret are generated with kubectl, not embedded
The Grafana dashboard is a standalone JSON file (`pc/monitoring/dashboards/`)
turned into a ConfigMap with `kubectl create configmap --from-file`, and the
admin password is a gitignored `.env` turned into a Secret with
`kubectl create secret --from-env-file`. Both use the idempotent
`--dry-run=client -o yaml | kubectl apply -f -` pattern. This keeps the JSON
editable and the secret out of Git without adding kustomize or templating. For a
GitOps setup the next step would be Sealed Secrets or SOPS.

## k3s over Tailscale: node-ip and flannel-iface
The control plane runs inside WSL2, whose default interface IP (`172.23.x`) is
behind NAT and unreachable from the Raspberry Pi. Both nodes are configured with
`node-ip` = their Tailscale IP and `flannel-iface: tailscale0`, so the API
connection and flannel VXLAN (cross-node pod traffic) flow over the tailnet.
Without this, a service on one node is unreachable from the other: for example,
Prometheus (on the PC) could not scrape `llama-server` (on the Pi), because the
response path Pi -> PC had no reachable node IP.

