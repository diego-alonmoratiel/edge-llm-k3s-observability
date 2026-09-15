# Edge LLM on a Raspberry Pi 5 with k3s + llama.cpp + Tailscale

Ansible automation for a small edge cluster: a **PC (WSL2) as the k3s control
plane** and a **Raspberry Pi 5 (4 GB) as a worker** that serves a lightweight
LLM with **llama.cpp** over an OpenAI-compatible API, reachable through
**Tailscale**. The image is built natively on the Pi with **Docker** and
imported into k3s' containerd, and **node-exporter** is ready for the
observability phase.

See [`../docs/ARCHITECTURE.md`](../docs/ARCHITECTURE.md) and
[`../docs/DECISIONS.md`](../docs/DECISIONS.md) for the why.

## Requirements

- **PC (control plane):** Windows 11 + WSL2 Ubuntu with k3s already installed
  and running (`k3s` v1.x), plus `sudo`. Ansible runs here.
- **Raspberry Pi 5:** Raspberry Pi OS Lite 64-bit (Bookworm), SSH enabled,
  passwordless `sudo` (or use `--ask-become-pass`).
- Ansible >= 2.15.
- Internet on the Pi (k3s, Docker, Tailscale, model).
- A Tailscale account. *(Optional but recommended: USB SSD for k3s/Docker I/O.)*

## Layout

```
ansible/
├── Makefile
├── requirements.yml
├── inventory/
│   ├── hosts.yml                       # control_plane (pc) + raspberry (rpi5)
│   ├── group_vars/{all,control_plane,raspberry}/main.yml
│   ├── group_vars/all/vault.yml        # encrypted auth key (gitignored)
│   └── host_vars/{pc,rpi5}.yml
├── manifests/                          # plain Kubernetes YAML (kubectl apply)
├── playbooks/{site,deploy,verify,teardown}.yml
└── roles/{base,tailscale,k3s,llm}
```

## Getting started

### 1. Dependencies
```bash
make deps
```

### 2. Fill in the worker connection details
`inventory/host_vars/rpi5.yml` → `ansible_host`, `ansible_user`.

The control plane is managed locally (`ansible_connection: local` in
`inventory/host_vars/pc.yml`).

### 3. Tailscale auth key (optional now)
```bash
make vault-create     # copies the example into the inventory and encrypts it
make vault-edit       # paste your tskey-auth-...
```
Without a key, the role installs Tailscale but leaves the login pending. When
you have it, run `make tailscale-up`. See the section below.

### 4. Full install
```bash
make ping             # check connectivity with both nodes
make site             # control plane -> worker -> manifests
```
`site` will, on the Pi, add the cgroup kernel options and **reboot if needed**,
install the k3s agent, build the ARM64 image, and download the model.

### 5. Verify
```bash
make verify
```

### 6. Use the API
```bash
curl http://rpi5-edge:30080/v1/models

curl http://rpi5-edge:30080/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -d '{"model":"qwen2.5-1.5b-instruct-q4_k_m.gguf",
       "messages":[{"role":"user","content":"Hello, what are you?"}]}'
```

## Tailscale key: where does it come from?

The key is generated in the **Tailscale admin console** (web), not on either
machine, and stored encrypted on the PC in the inventory. Ansible then injects
it into each node over the tailnet/SSH.

- **Recommended (automatable):** admin console → Settings → Keys → Generate
  auth key (tick *Reusable*), then `make vault-create && make vault-edit`, then
  `make site`.
- **Interactive:** run `make tailscale-up`, or on a node
  `sudo tailscale up --ssh` and open the printed URL. No key stored.

After the first login the node keeps its own key; the auth key is no longer
needed.

## Day-to-day operations

| Command         | What it does                                             |
|-----------------|----------------------------------------------------------|
| `make deploy`   | Re-apply only the manifests (runs on the control plane)  |
| `make verify`   | Smoke test the cluster and the inference endpoint        |
| `make check`    | Syntax-check the playbooks                               |
| `make lint`     | `ansible-lint` (if installed)                            |
| `make teardown` | Delete workloads, the worker agent and Docker            |

Rebuild the image after changing the Dockerfile:
```bash
make site EXTRA="-e llm_rebuild=true --tags never"   # or simply: -e llm_rebuild=true
```

## Things you must be able to explain

- **cgroups** (`cgroup_memory=1 cgroup_enable=memory`): required for the
  container runtime; Raspberry Pi OS ships them off.
- **k3s server vs agent:** the PC runs the API server/etcd; the Pi only runs a
  kubelet-like agent that talks back to `:6443`.
- **tls-san:** why the control plane's Tailscale IP must be in the API
  certificate for the worker to join.
- **containerd import:** why `docker save | k3s ctr images import -` and
  `imagePullPolicy: Never` let an edge node run an image without a registry.
- **nodeSelector by arch:** a mixed amd64/arm64 cluster must pin the workload.
- **NodePort vs Ingress / LoadBalancer:** why a NodePort is enough here.
- **PV/PVC over hostPath:** storage as a first-class object on one node.
- **Probes:** readiness vs liveness on `/health` while the model loads.
- **Idempotency:** handlers, `creates:`, `changed_when`, and why re-running
  `make site` is safe.

## Troubleshooting

- **Agent never joins:** check the worker can reach the control plane over
  Tailscale (`tailscale ping <pc>`), and that `tls-san` includes the PC's
  Tailscale IP (see `roles/k3s/tasks/server.yml`).
- **Image missing on the Pi:** `sudo k3s ctr images ls | grep llama-server`;
  rebuild with `-e llm_rebuild=true`.
- **OOM or restart loops:** lower `--ctx-size` (manifest) and/or
  `llm_memory_limit`; the base role configures 2 GB of swap.
- **Docker build fails on RAM:** lower `BUILD_JOBS` in
  `roles/llm/files/Dockerfile` (default 2).
- **WSL Tailscale needs userspace networking:** set
  `tailscale_extra_args: "--tun=userspace-networking"` in
  `inventory/host_vars/pc.yml`.

## Phase 2 (roadmap)

Deploy Prometheus + Grafana on the control plane (Docker Compose or manifests),
scrape `:9100/metrics` (all nodes) and llama-server's `:30080/metrics`, and
import the standard dashboards.
