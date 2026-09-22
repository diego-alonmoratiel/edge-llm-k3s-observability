# Edge LLM on k3s + llama.cpp + Tailscale

A two-node k3s cluster that serves a small LLM through the llama.cpp
OpenAI-compatible API, reachable over Tailscale.

- `pc` (WSL2): k3s control plane (already installed).
- `rpi5` (Raspberry Pi 5, 4 GB): worker running `llama-server`.

## Prerequisites (manual, once)

1. `pc`: k3s server installed and running.
2. `pc` and `rpi5`: Tailscale installed and logged in.
3. Access from `pc` to `rpi5` over SSH:
   ```bash
   ssh-keygen -t ed25519
   ssh-copy-id pi@192.168.1.50
   ```
4. Edit `inventory.yml` with your Pi's IP and user.

## Layout

```
inventory.yml        hosts and how to connect to them
group_vars/all.yml   all variables
site.yml             installs and deploys everything
verify.yml           checks the cluster and the LLM API
teardown.yml         removes what site.yml created
Makefile             make install / verify / teardown
manifests/           Kubernetes YAML
```

See also `../docs/ARCHITECTURE.md` and `../docs/DECISIONS.md`.

## Usage

```bash
make install    # ansible-playbook -i inventory.yml site.yml
make verify
make teardown
```

`site.yml` runs three plays: read data from the control plane, prepare the
worker and join it to the cluster, then apply the manifests.

## API

```bash
curl http://rpi5-edge:30080/v1/models

curl http://rpi5-edge:30080/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -d '{"model":"qwen2.5-1.5b",
       "messages":[{"role":"user","content":"Hello"}]}'
```

## Metrics

`node-exporter` is exposed on every node at `http://<node-ip>:9100/metrics`
and llama-server at `http://<pi-ip>:30080/metrics` (phase 2 will scrape them).
