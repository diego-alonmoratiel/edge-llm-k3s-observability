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
5. Sudo: the playbooks use `become: true`. The `pi` user on Raspberry Pi OS
   normally has passwordless sudo; if it does not, run it once on the Pi:
   ```bash
   echo "pi ALL=(ALL) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/010_pi-nopasswd
   sudo chmod 0440 /etc/sudoers.d/010_pi-nopasswd
   ```
   Never run the playbooks with `sudo`. If the local (`pc`) sudo asks for a
   password, run them as `make install EXTRA="-K"`.

## Layout

```
inventory.yml        hosts and how to connect to them
group_vars/all.yml   all variables
site.yml             installs and deploys everything
verify.yml           checks the cluster and the LLM API
teardown.yml         removes what site.yml created
Makefile             make install / verify / teardown
```

The edge manifests live in `../raspberry/manifests/` and the monitoring stack in
`../pc/monitoring/`; `site.yml` applies both.

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

`llama-server` speaks the OpenAI API. The NodePort is reachable on any IP of the
worker (Tailscale or LAN):

```bash
curl http://<pi-ip>:30080/v1/models

curl http://<pi-ip>:30080/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -d '{"model":"qwen2.5-1.5b",
       "messages":[{"role":"user","content":"Hello"}]}'
```

The built-in web UI is served at `http://<pi-ip>:30080/`.

## Monitoring

Prometheus + Grafana are deployed by the same `site.yml` run, from
`../pc/monitoring/`. They run on the PC (control-plane) node and scrape through
cluster-internal DNS.

- Grafana: `http://localhost:30300` from the PC (or `http://<node-ip>:30300`),
  login `admin` + the password in `../pc/monitoring/grafana-admin.env` (created
  automatically if missing).
- Prometheus: cluster-internal `prometheus.monitoring.svc:9090`.

Raw metrics are also exposed at:

- `node-exporter`: `http://<node-ip>:9100/metrics`
- `llama-server`: `http://<pi-ip>:30080/metrics`
