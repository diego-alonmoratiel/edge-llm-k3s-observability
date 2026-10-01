# raspberry — edge node (worker)

> Part of the [Edge LLM on k3s + Observability](../README.md) project.

Artifacts for the Raspberry Pi 5, which joins the k3s cluster as a worker and
runs the LLM workload.

```
manifests/
├── 00-namespace.yaml       Namespace llm
├── 10-model-storage.yaml   PersistentVolume + PersistentVolumeClaim for the model
└── 20-llama-server.yaml    llama-server Deployment + NodePort Service
```

These manifests are applied by the cluster Ansible (`../ansible/site.yml`),
together with the monitoring stack in `../pc/monitoring/`.

The Pi's OS (cgroups, swap, k3s agent, model download) is configured by the
Ansible playbook, not by manifests in this directory.
