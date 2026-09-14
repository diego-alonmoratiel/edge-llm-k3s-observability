# 🦙 Edge LLM K3s Stack

> Distributed, resource-optimized LLM inference on Raspberry Pi (4GB RAM) using K3s, llama.cpp, and offloaded observability via Tailscale.

## 📌 Architecture Overview

- **Edge Node (Raspberry Pi 4GB):** K3s Control Plane, `llama.cpp` (Qwen2.5 1.5B Q4_K_M), Node Exporter.
- **Monitoring Host (PC):** Prometheus & Grafana stack scraping metrics over Tailscale mesh.
- **Networking:** Secure peer-to-peer overlay network using Tailscale.
