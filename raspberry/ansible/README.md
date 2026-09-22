# Edge LLM con k3s + llama.cpp + Tailscale (versión simple)

Este directorio despliega un cluster k3s de 2 nodos y sirve un LLM pequeño con
la API de llama.cpp, todo accesible por Tailscale:

- **pc** (WSL2) → **control plane** (k3s server ya instalado).
- **rpi5** (Raspberry Pi 5, 4 GB) → **worker** que ejecuta `llama-server`.

Es intencionadamente pequeño: **sin roles, sin Docker, sin vault**. Un playbook,
un fichero de variables y unos manifiestos de Kubernetes.

## Requisitos previos (a mano, una sola vez)

Esto NO lo hace Ansible (para mantenerlo simple):

1. **PC**: k3s server instalado y funcionando (`k3s kubectl get nodes` responde).
2. **PC y Pi**: Tailscale instalado y logueado (`tailscale status`). La Pi se une
   al cluster por la IP de Tailscale del PC.
3. **PC → Pi**: acceso SSH funcionando. Lo más fácil:
   ```bash
   ssh-keygen -t ed25519          # si no tienes clave
   ssh-copy-id pi@192.168.1.50    # te pide la contraseña una vez
   ```
4. Edita `inventory.yml` con la IP y el usuario de tu Raspberry.

## Ficheros

```
ansible/
├── inventory.yml        # las 2 máquinas (y cómo conectarse)
├── group_vars/all.yml   # TODAS las variables (zona horaria, modelo, swap...)
├── site.yml             # el playbook: instala y despliega todo
├── verify.yml           # comprueba que funciona
├── teardown.yml         # deshace lo instalado
├── Makefile             # atajos: make install / verify / teardown
└── manifests/           # YAML de Kubernetes (namespaces, PV/PVC, Deployment...)
```

## Cómo se despliega

```bash
make install
# equivale a:  ansible-playbook -i inventory.yml site.yml
```

`site.yml` tiene 3 bloques ("plays"):

1. **Lee del PC** la IP de Tailscale, la versión de k3s y el *token* del cluster,
   y añade esa IP a `tls-san` del API server (si no, la Pi no puede unirse).
2. **Prepara la Pi**: paquetes, zona horaria, hostname, cgroups (reinicia si
   hace falta), swap, instala el **agente** de k3s y descarga el modelo.
3. **Aplica los manifiestos** con `k3s kubectl apply` desde el PC.

## Comprobar y borrar

```bash
make verify     # estado del cluster + /health y /v1/models del LLM
make teardown   # borra workloads y desinstala el agente de la Pi
```

## Usar la API

```bash
# desde el PC (o cualquier equipo en la tailnet)
curl http://rpi5-edge:30080/v1/models

curl http://rpi5-edge:30080/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -d '{"model":"qwen2.5-1.5b",
       "messages":[{"role":"user","content":"Hola, ¿qué eres?"}]}'
```

## Cosas que debes poder explicar

- **Un solo playbook**: se lee de arriba a abajo; 3 plays, cada uno con sus tareas.
- **`inventory.yml`**: dos grupos, `control_plane` y `raspberry`. El PC usa
  `ansible_connection: local` (no SSH).
- **`group_vars/all.yml`**: por qué las variables están separadas de las tareas.
- **`become: true`**: Ansible usa `sudo` (para escribir en `/etc`, leer el token…).
- **`set_fact` + `hostvars`**: cómo se pasa el token del play 1 al play 2.
- **`tls-san`**: por qué la IP de Tailscale del PC tiene que estar en el certificado.
- **k3s agent vs server**: el PC es el servidor; la Pi solo ejecuta workloads.
- **`nodeSelector: kubernetes.io/arch=arm64`**: el cluster es mixto; hay que
  fijar el pod a la Pi.
- **PV/PVC**: el modelo vive en el host (`/opt/llama/models`) y se monta en el
  contenedor en `/models`.
- **`notify` + handlers + `flush_handlers`**: por qué el reinicio se fuerza antes
  de instalar k3s.
- **Idempotencia**: `creates:`, `changed_when`, `when`… re-ejecutar `make install`
  no rompe nada ni reinstala.

## Problemas típicos

- **La Pi no se une**: comprueba que ves el PC por Tailscale (`tailscale ping pc`)
  y que `tls-san` tiene la IP (`sudo cat /etc/rancher/k3s/config.yaml` en el PC).
- **Pod en `Pending`**: el selector `arm64` no encuentra nodo; revisa
  `k3s kubectl get nodes`.
- **`ImagePullBackOff`**: la Pi necesita internet la primera vez para bajar la
  imagen oficial.
- **Se queda sin memoria**: baja `--ctx-size` en `manifests/20-llama-server.yaml`
  o el `limits.memory`.
