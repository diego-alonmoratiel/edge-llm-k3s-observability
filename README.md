# Edge LLM on k3s + Observability

LLM inference on a resource-constrained edge node (Raspberry Pi 5, 4 GB) running
on a two-node **k3s** cluster, with the observability stack offloaded to a PC.
Everything is connected over a **Tailscale** mesh.

- **Phase 1 — LLM serving: working.** `llama-server` (llama.cpp) exposes an
  OpenAI-compatible API from the Pi.
- **Phase 2 — Monitoring: working.** Prometheus + Grafana run on the PC node
  and scrape node-exporter and llama-server.

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
│                           │   and :30080/metrics │                               │
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
| Prometheus     | pc     | scrapes node-exporter and llama-server              |
| Grafana        | pc     | dashboards (NodePort 30300)                         |

Model: **Qwen2.5 1.5B Instruct, Q4_K_M** (~1.1 GB).

## Tech stack

Ansible (idempotent provisioning) · k3s · Kubernetes manifests (Deployment,
Service/NodePort, PersistentVolume/PersistentVolumeClaim, DaemonSet, probes,
ConfigMap/Secret, emptyDir) · llama.cpp · Tailscale · node-exporter ·
Prometheus · Grafana.

## Repository layout

```
ansible/             cluster provisioning (Ansible: pc + rpi5)
raspberry/manifests/ edge workloads (LLM)
pc/monitoring/       observability workloads (Prometheus + Grafana)
docs/                architecture and design decisions
```

See [`ansible/README.md`](ansible/README.md) to deploy,
[`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md) for the architecture and
[`docs/DECISIONS.md`](docs/DECISIONS.md) for the rationale.

## Quick start (phase 1)

```bash
cd ansible
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

## Monitoring

Prometheus and Grafana run on the PC (control-plane node):

- **Prometheus** scrapes `node-exporter` and `llama-server` through
  cluster-internal DNS (no hardcoded IPs).
- **Grafana**: `http://localhost:30300` from the PC (or
  `http://<node-ip>:30300`), login `admin` / `admin`.

Raw metrics are also exposed at:

- `node-exporter`: `http://<node-ip>:9100/metrics`
- `llama-server`: `http://<pi-ip>:30080/metrics`

## Roadmap

- [x] Two-node k3s cluster over Tailscale
- [x] `llama-server` on the Raspberry Pi (OpenAI-compatible API)
- [x] `node-exporter` on every node
- [x] Prometheus + Grafana on the PC node
- [x] Starter dashboard (CPU, memory, load, targets up)
