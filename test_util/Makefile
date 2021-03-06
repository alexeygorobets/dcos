SHELL := /bin/bash -euo pipefail

TERRAFORM_VERSION := 0.11.14
OS ?= $(shell uname|tr A-Z a-z)

# Path to Terraform binary.
TERRAFORM ?= ./terraform

export AWS_REGION ?= us-west-2
export TF_VAR_custom_dcos_download_path ?= https://downloads.dcos.io/dcos/testing/master/dcos_generate_config.sh
export TF_VAR_custom_dcos_download_path_win ?= https://downloads.dcos.io/dcos/testing/master/windows/dcos_generate_config_win.sh
export TF_VAR_variant ?= open

SSH_KEY ?= ./tf-dcos-rsa.pem

ifndef DCOS_LICENSE_CONTENTS 
$(error DCOS_LICENSE_CONTENTS is not set)
endif

################################### TARGETS ################################### 

.DEFAULT_GOAL := test

license.txt:
	@echo ${DCOS_LICENSE_CONTENTS} > $@

.PHONY: terraform-init
terraform-init: license.txt $(TERRAFORM)
	$(TERRAFORM) init --upgrade

$(SSH_KEY):
	ssh-keygen -t rsa -b 2048 -f $@ -q -N ""; \

.PHONY: cluster
cluster: terraform-init $(SSH_KEY)
	if [ -z "$$SSH_AUTH_SOCK" ]; then eval $$(ssh-agent -s); fi; \
	ssh-add "$(SSH_KEY)"; \
	$(TERRAFORM) apply -auto-approve -var ssh_public_key_file="$(SSH_KEY).pub" -var windowsagent_num=1

cluster.json: cluster
	$(TERRAFORM) output -json > $@ 

.PHONY: test
test: cluster.json
	ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null -i $(SSH_KEY) centos@$(shell jq '.masters_public_ip.value' $<) -- \
		"source /opt/mesosphere/environment.export && cd /opt/mesosphere/active/dcos-integration-test && \
		MASTER_PUBLIC_IP=$(shell jq '.masters_public_ip.value' $<) \
		MASTERS_PRIVATE_IPS=$(shell jq '.masters_private_ip.value' $<) \
		MASTER_HOSTS=$(shell jq '.masters_private_ip.value' $<) \
		PRIVATE_AGENTS_PRIVATE_IPS=$(shell jq '.private_agent_ips.value' $<) \
		SLAVE_HOSTS=$(shell jq '.private_agent_ips.value' $<) \
		PUBLIC_AGENTS_PRIVATE_IPS=$(shell jq '.public_agent_ips.value' $<) \
		PUBLIC_SLAVE_HOSTS=$(shell jq '.public_agent_ips.value' $<) \
		pytest --full-trace --log-level=DEBUG --windows-only"

.PHONY: ssh 
ssh: cluster.json
	ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null -i $(SSH_KEY) centos@$(shell jq '.masters_public_ip.value' $<)

.PHONY: destroy
destroy:
	$(TERRAFORM) destroy -auto-approve;
	rm cluster.json || true;

.PHONY: clean
clean:
	rm -rf ./.terraform/
	rm -rf ./inventory
	rm -rf ./terraform.tfstate
	rm -rf ./terraform.tfstate.backup
	rm -rf ./terraform_*.zip
	rm -rf ./*.pem
	rm -rf ./*.pub
	rm -rf ./license.txt

################################### HELPERS ################################### 

# Provision Terraform if not available.

terraform_$(TERRAFORM_VERSION)_$(OS)_amd64.zip:
	wget -nv https://releases.hashicorp.com/terraform/$(TERRAFORM_VERSION)/$@

$(TERRAFORM): terraform_$(TERRAFORM_VERSION)_$(OS)_amd64.zip
	unzip -n $<;
	chmod +x terraform;

