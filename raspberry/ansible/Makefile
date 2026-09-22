export ANSIBLE_HOST_KEY_CHECKING = False

.DEFAULT_GOAL := help

help:
	@echo "  make install   -> ansible-playbook -i inventory.yml site.yml"
	@echo "  make verify    -> ansible-playbook -i inventory.yml verify.yml"
	@echo "  make teardown  -> ansible-playbook -i inventory.yml teardown.yml"

install:
	ansible-playbook -i inventory.yml site.yml

verify:
	ansible-playbook -i inventory.yml verify.yml

teardown:
	ansible-playbook -i inventory.yml teardown.yml

.PHONY: help install verify teardown
