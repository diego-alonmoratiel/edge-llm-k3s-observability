# Architecture

Two machines on a private tailnet. The PC is the Kubernetes control plane; the
Raspberry Pi is a resource-constrained worker that runs the inference workload.

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
│                           │                      │                               │
│  Prometheus + Grafana     │◄── scrape :9100 ─────┤  pod: llama-server (NodePort 30080)
│                           │   and :30080/metrics │  pod: node-exporter (hostNetwork)
└───────────────────────────┘                      └───────────────────────────────┘
        amd64                                                   arm64
```

## Nodes

| Node   | Role       | Arch  | Runs                                              |
|--------|------------|-------|---------------------------------------------------|
| `pc`   | k3s server | amd64 | control plane, `kubectl`, Prometheus/Grafana      |
| `rpi5` | k3s agent  | arm64 | `llama-server`, `node-exporter`, the model GGUF   |

## What Ansible does (site.yml)

Three plays, no roles:

1. **Read from the control plane** — Tailscale IP, k3s version, cluster token;
   add the Tailscale IP to the API server certificate (`tls-san`) and configure
   k3s to use the tailnet (`node-ip` + `flannel-iface: tailscale0`).
2. **Prepare the worker** — packages, cgroups, swap, install the k3s **agent**
   (also on the tailnet) and join the cluster, download the model.
3. **Apply the manifests** — `k3s kubectl apply` of `raspberry/manifests/` and
   `pc/monitoring/` from the PC.

## Data flow of a request

```
client (tailnet) ──► http://<pi-tailscale-ip>:30080/v1/chat/completions
                                   │
                         Service (NodePort 30080)
                                   │
                         Pod llama-server (on rpi5)
                                   │  reads
                         /models/model.gguf  ← hostPath PV, downloaded by Ansible
```

## Ports

| Port  | Where | Purpose                                        |
|-------|-------|------------------------------------------------|
| 6443  | pc    | k3s API server (joined over Tailscale)         |
| 30080 | rpi5  | llama-server OpenAI-compatible API (NodePort)  |
| 9100  | both  | node-exporter metrics (hostNetwork)            |
| 9090  | pc    | Prometheus UI/API (ClusterIP)                  |
| 30300 | pc    | Grafana UI (NodePort)                          |

## Images

The workload uses the **official multi-architecture** image
`ghcr.io/ggml-org/llama.cpp:server`, so no custom build or registry is needed.
The model is *not* baked into the image; it lives on the host and is mounted.

## Monitoring

Prometheus and Grafana are deployed as manifests and pinned to the PC node
(`nodeSelector: kubernetes.io/arch: amd64`), so they never consume the Pi's
memory. Prometheus discovers its targets through cluster-internal DNS:

- `node-exporter.monitoring.svc.cluster.local:9100`
- `llama-server.llm.svc.cluster.local:8080` (`/metrics`)

Grafana is provisioned with the Prometheus datasource and a starter dashboard
(CPU, memory, load, targets up) and is exposed through NodePort `30300`. The
dashboard JSON (`pc/monitoring/dashboards/`) and the admin password (gitignored
`pc/monitoring/grafana-admin.env`) are turned into a ConfigMap and a Secret with
`kubectl create ... --dry-run=client -o yaml | kubectl apply -f -`.
