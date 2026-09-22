# Atajos. Ejecuta `make` (sin argumentos) para ver la ayuda.
export ANSIBLE_HOST_KEY_CHECKING = False

.DEFAULT_GOAL := help
.PHONY: help install verify teardown

help: ## Muestra esta ayuda
	@grep -hE '^[a-z]+:.*?## ' $(MAKEFILE_LIST) \
		| awk 'BEGIN {FS = ":.*?## "}; {printf "  %-10s %s\n", $$1, $$2}'

install: ## Despliega todo (instala y configura la Pi + aplica manifiestos)
	ansible-playbook -i inventory.yml site.yml

verify: ## Comprueba el cluster y la API del LLM
	ansible-playbook -i inventory.yml verify.yml

teardown: ## Borra los workloads y desinstala el agente k3s de la Pi
	ansible-playbook -i inventory.yml teardown.yml
