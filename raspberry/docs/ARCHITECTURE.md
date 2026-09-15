# Architecture

Two machines on a private tailnet. The PC is the Kubernetes control plane and
the operational entry point; the Raspberry Pi is a resource-constrained worker
that runs the inference workload.

```
                          Tailnet (Tailscale, WireGuard)
                                     │
        ┌────────────────────────────┴────────────────────────────┐
        │                                                          │
┌───────┴───────────────────┐                      ┌───────────────┴───────────────┐
│  pc  (control plane)      │                      │  rpi5  (worker, arm64)        │
│  Windows 11 + WSL2 Ubuntu │                      │  Raspberry Pi 5, 4 GB         │
│                           │                      │                               │
│  k3s server  :6443        │◄── k3s agent joins ──┤  k3s-agent                    │
│  kubectl / manifests      │     over Tailscale   │                               │
│  TAILSCALE                │                      │  TAILSCALE                    │
│                           │                      │                               │
│  (phase 2)                │                      │  pod: llama-server            │
│  Prometheus + Grafana     │◄── scrape :9100 ─────┤  pod: node-exporter (hostNet) │
└───────────────────────────┘      and :30080      │                               │
        amd64                │                      │  docker build → k3s ctr       │
                             │                      │  model on /opt/llama/models   │
                             └──────────────────────┴───────────────────────────────┘
```

## Nodes

| Node   | Role          | Arch  | Runs                                                        |
|--------|---------------|-------|-------------------------------------------------------------|
| `pc`   | k3s server    | amd64 | control plane, `kubectl`, later Prometheus/Grafana          |
| `rpi5` | k3s agent     | arm64 | `llama-server`, `node-exporter`, the model GGUF             |

## Request / data flow

1. `ansible-playbook playbooks/site.yml` runs from the **PC**.
2. `tailscale` role puts both nodes on the tailnet.
3. `k3s` role on the PC reads the node token and adds the PC's Tailscale IP to
   the API server certificate (`tls-san`).
4. `k3s` role on the Pi installs the **agent** and joins
   `https://<pc-tailscale-ip>:6443`.
5. `llm` role on the Pi: installs Docker, **builds llama-server natively**
   (`-DGGML_NATIVE=ON`), **imports the image into k3s' containerd**, downloads
   the GGUF model.
6. `deploy` play runs on the PC: `k3s kubectl apply -f manifests/`.
7. A client on the tailnet calls `http://<pi-tailscale-ip>:30080/v1/...`.

## Why the workload lands on the Pi

The cluster is **mixed-architecture** (amd64 control plane + arm64 worker). The
Deployment sets `nodeSelector: kubernetes.io/arch: arm64`, so the pod only
schedules where the ARM64 image and the model actually exist.

## Ports

| Port | Where | Purpose                                   |
|------|-------|-------------------------------------------|
| 6443 | pc    | k3s API server (joined over Tailscale)    |
| 30080| rpi5  | llama-server OpenAI-compatible API (NodePort) |
| 9100 | both  | node-exporter metrics (hostNetwork)       |

## What is intentionally out of scope (phase 2)

Prometheus and Grafana are not deployed yet. `node-exporter` is already running
on every node, so the observability stack only needs to be added and pointed at
`:9100/metrics` and llama-server's `:30080/metrics`.
