# pc — monitoring host

Observability stack that runs on the PC (the k3s control-plane node). The
workloads are pinned to this node, so they never consume the Raspberry Pi's
memory.

```
monitoring/
├── 50-prometheus.yaml          Prometheus (scrapes node-exporter + llama-server)
├── 60-grafana.yaml             Grafana (NodePort 30300)
├── dashboards/edge.json        Grafana dashboard (kept out of the manifest)
└── grafana-admin.env.example   example for the local, gitignored secret
```

Two files are **not** embedded in the manifests:

- The dashboard JSON (`dashboards/edge.json`) becomes the `grafana-dashboard-edge`
  ConfigMap with `kubectl create configmap --from-file`.
- The Grafana admin password lives in `grafana-admin.env`, which is **gitignored**,
  and becomes the `grafana-admin` Secret with `kubectl create secret --from-env-file`.
  Ansible creates the file with a random value if it is missing; you can also
  create it yourself:
  ```bash
  cp grafana-admin.env.example grafana-admin.env
  # edit it and set a strong admin-password
  ```

The Ansible playbook does all of this. By hand:

```bash
k3s kubectl -n monitoring create configmap grafana-dashboard-edge \
  --from-file=edge.json=dashboards/edge.json --dry-run=client -o yaml | k3s kubectl apply -f -
k3s kubectl -n monitoring create secret generic grafana-admin \
  --from-env-file=grafana-admin.env --dry-run=client -o yaml | k3s kubectl apply -f -
k3s kubectl apply -f 50-prometheus.yaml -f 60-grafana.yaml
```

Grafana: `http://localhost:30300` from the PC (or `http://<node-ip>:30300`),
login `admin` with the password from `grafana-admin.env`.
