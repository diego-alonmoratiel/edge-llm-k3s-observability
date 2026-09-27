export ANSIBLE_HOST_KEY_CHECKING = False

INVENTORY ?= inventory.yml
EXTRA     ?=

.DEFAULT_GOAL := help

help:
	@echo "  make install   -> ansible-playbook -i $(INVENTORY) site.yml"
	@echo "  make verify    -> ansible-playbook -i $(INVENTORY) verify.yml"
	@echo "  make teardown  -> ansible-playbook -i $(INVENTORY) teardown.yml"
	@echo "  Extra: add EXTRA=\"-K\" if the local sudo asks for a password"

install:
	ansible-playbook -i $(INVENTORY) $(EXTRA) site.yml

verify:
	ansible-playbook -i $(INVENTORY) $(EXTRA) verify.yml

teardown:
	ansible-playbook -i $(INVENTORY) $(EXTRA) teardown.yml

.PHONY: help install verify teardown
