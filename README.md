# Edge LLM on k3s + Observability

LLM inference on a resource-constrained edge node (Raspberry Pi 5, 4 GB) running
on a two-node **k3s** cluster, with the observability stack offloaded to a PC.
Everything is connected over a **Tailscale** mesh.

- **Phase 1 — LLM serving: working.** `llama-server` (llama.cpp) exposes an
  OpenAI-compatible API from the Pi.
- **Phase 2 — Monitoring: next.** Prometheus + Grafana on the PC, scraping the
  edge node over Tailscale.

## Architecture

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
│  kubectl                  │     over Tailscale   │                               │
│                           │                      │  llama-server   :30080        │
│  Prometheus + Grafana     │◄── scrape :9100 ─────┤  node-exporter  :9100         │
│  (phase 2)                │   and :30080/metrics │                               │
└───────────────────────────┘                      └───────────────────────────────┘
        amd64                                                   arm64
```

The cluster is **mixed-architecture**: the control plane is amd64 and the worker
is arm64. The LLM deployment is pinned to the worker (`nodeSelector` by arch).

## Components

| Component      | Where  | Purpose                                             |
|----------------|--------|-----------------------------------------------------|
| k3s server     | pc     | control plane, `kubectl`                            |
| k3s agent      | rpi5   | runs the workloads                                  |
| llama-server   | rpi5   | LLM inference, OpenAI-compatible API (NodePort 30080) |
| node-exporter  | both   | host metrics on `:9100`                             |
| Tailscale      | both   | private mesh network / remote access                |
| Prometheus     | pc     | metrics scraping (phase 2)                          |
| Grafana        | pc     | dashboards (phase 2)                                |

Model: **Qwen2.5 1.5B Instruct, Q4_K_M** (~1.1 GB).

## Tech stack

Ansible (idempotent provisioning) · k3s · Kubernetes manifests (Deployment,
Service/NodePort, PersistentVolume/PersistentVolumeClaim, DaemonSet, probes) ·
llama.cpp · Tailscale · node-exporter · *(next)* Prometheus + Grafana.

## Repository layout

```
raspberry/
├── ansible/      Ansible playbooks + Kubernetes manifests (phase 1)
└── docs/         Architecture and design decisions
pc/               Monitoring stack (phase 2)
```

See [`raspberry/ansible/README.md`](raspberry/ansible/README.md) to deploy,
[`raspberry/docs/ARCHITECTURE.md`](raspberry/docs/ARCHITECTURE.md) for the
architecture and [`raspberry/docs/DECISIONS.md`](raspberry/docs/DECISIONS.md)
for the rationale.

## Quick start (phase 1)

```bash
cd raspberry/ansible
make install     # provisions the Pi, joins it to the cluster, applies manifests
make verify      # checks the cluster and the LLM API
```

Prerequisites (manual, once): k3s server on the PC, Tailscale on both nodes, SSH
access from the PC to the Pi, and `inventory.yml` filled in.

## Using the API

`llama-server` speaks the OpenAI API. The NodePort is reachable on **any** IP of
the worker (Tailscale or LAN):

```bash
curl http://<pi-ip>:30080/v1/models

curl http://<pi-ip>:30080/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -d '{"model":"qwen2.5-1.5b",
       "messages":[{"role":"user","content":"Hello"}]}'
```

## Metrics (ready for phase 2)

- `node-exporter`: `http://<node-ip>:9100/metrics`
- `llama-server`: `http://<pi-ip>:30080/metrics`

## Roadmap

- [x] Two-node k3s cluster over Tailscale
- [x] `llama-server` on the Raspberry Pi (OpenAI-compatible API)
- [x] `node-exporter` on every node
- [ ] Prometheus + Grafana on the PC (scrape over Tailscale)
- [ ] Dashboards for node and LLM metrics
