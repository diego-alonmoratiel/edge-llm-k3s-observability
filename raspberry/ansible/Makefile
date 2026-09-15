SHELL := /bin/bash

# Force the use of this ansible.cfg even when the directory is world-writable
# (e.g. when working from WSL under /mnt/c), where Ansible would ignore it.
ANSIBLE_DIR := $(dir $(realpath $(lastword $(MAKEFILE_LIST))))
export ANSIBLE_CONFIG := $(ANSIBLE_DIR)ansible.cfg

# Use vault/vault-password automatically when it exists.
ifneq (,$(wildcard $(ANSIBLE_DIR)vault/vault-password))
export ANSIBLE_VAULT_PASSWORD_FILE := $(ANSIBLE_DIR)vault/vault-password
endif

INVENTORY  ?= inventory/hosts.yml
LIMIT      ?=
VAULT_FILE ?= inventory/group_vars/all/vault.yml
EXTRA      ?=

PLAYBOOK := ansible-playbook -i $(INVENTORY) $(if $(LIMIT),--limit $(LIMIT),) $(EXTRA)

.DEFAULT_GOAL := help
.PHONY: help deps ping site deploy verify teardown check lint \
        vault-create vault-edit tailscale-up clean

help: ## Show this help
	@grep -hE '^[a-zA-Z_-]+:.*?## ' $(MAKEFILE_LIST) \
		| sort \
		| awk 'BEGIN {FS = ":.*?## "}; {printf "  \033[36m%-16s\033[0m %s\n", $$1, $$2}'

deps: ## Install the required Ansible collections
	ansible-galaxy collection install -r requirements.yml

ping: ## Check connectivity with both nodes
	ansible -i $(INVENTORY) all -m ping

site: ## Full install (control plane + worker + workloads)
	$(PLAYBOOK) playbooks/site.yml

deploy: ## Re-apply only the Kubernetes manifests from the control plane
	$(PLAYBOOK) playbooks/deploy.yml

verify: ## Smoke test the cluster and the inference endpoint
	$(PLAYBOOK) playbooks/verify.yml

teardown: ## Remove workloads, the worker node agent and Docker (asks for confirmation)
	$(PLAYBOOK) playbooks/teardown.yml

check: ## Syntax-check every playbook
	$(PLAYBOOK) --syntax-check playbooks/site.yml playbooks/deploy.yml \
		playbooks/verify.yml playbooks/teardown.yml

lint: ## Run ansible-lint (if installed)
	@command -v ansible-lint >/dev/null 2>&1 || { \
		echo "ansible-lint is not installed (pip install ansible-lint)"; exit 1; }
	ansible-lint playbooks roles

vault-create: ## Encrypt the Tailscale auth key into the inventory
	mkdir -p $(dir $(VAULT_FILE))
	cp vault/secrets.example.yml $(VAULT_FILE)
	ansible-vault encrypt $(VAULT_FILE)

vault-edit: ## Edit the encrypted secrets
	ansible-vault edit $(VAULT_FILE)

tailscale-up: ## Interactive Tailscale login on both nodes (when no auth key is used)
	ansible -i $(INVENTORY) all -m ansible.builtin.command -a "tailscale up --ssh" --become

clean: ## Remove Ansible temporary files
	rm -rf .ansible tmp *.retry
