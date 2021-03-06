include $(dir $(word $(words $(MAKEFILE_LIST)),$(MAKEFILE_LIST)))/../../Makefile

TGIR := tgir-s01e08
DKE_CLUSTER := $(TGIR)
DKE_NODES_COUNT ?= 3
# make dke-regions
# Because Frankfurt is closer to @GSantomaggio than either London or Amsterdam
DKE_REGION ?= fra1
# make dke-sizes
DKE_NODES_TYPE ?= s-2vcpu-4gb
# make dke-versions
DKE_VERSION ?= 1.19.3-do.2
# Keeping this in sync with DKE version
KUBECTL_VERSION = 1.19.3

KUBECONFIG_DIR := $(XDG_CONFIG_HOME)/kubectl
KUBECONFIG := $(KUBECONFIG_DIR)/config
export KUBECONFIG

$(KUBECONFIG_DIR):
	@mkdir -p $(@)

.PHONY: k9s
k9s: | $(K9S) $(KUBECONFIG) ## Interact with our K8S cluster via a terminal UI
	$(K9S) --all-namespaces

define ENV
export PATH=$(LOCAL_BIN):$$PATH
export XDG_CONFIG_HOME="$(XDG_CONFIG_HOME)"
export KUBECONFIG="$(KUBECONFIG)"
unalias k 2>/dev/null; alias k=kubectl
unalias m 2>/dev/null; alias m=make
endef
export ENV
.PHONY: env
env:: | $(KUBECONFIG_DIR) ## Configure shell env - eval "$(make env)" OR source .env
	@echo "$$ENV"

DOCTL_RELEASES := https://github.com/digitalocean/doctl/releases
DOCTL_VERSION := 1.54.0
DOCTL_BIN_DIR := doctl-$(DOCTL_VERSION)-$(platform)-amd64
DOCTL_URL := https://github.com/digitalocean/doctl/releases/download/v$(DOCTL_VERSION)/doctl-$(DOCTL_VERSION)-$(platform)-amd64.tar.gz
DOCTL := $(LOCAL_BIN)/$(DOCTL_BIN_DIR)/doctl
$(DOCTL): | $(CURL) $(LOCAL_BIN)
	$(CURL) --progress-bar --fail --location --output $(LOCAL_BIN)/$(DOCTL_BIN_DIR).tar.gz "$(DOCTL_URL)"
	mkdir -p $(LOCAL_BIN)/$(DOCTL_BIN_DIR) && tar zxf $(LOCAL_BIN)/$(DOCTL_BIN_DIR).tar.gz -C $(LOCAL_BIN)/$(DOCTL_BIN_DIR)
	touch $(DOCTL)
	chmod +x $(DOCTL)
	$(DOCTL) version | grep $(DOCTL_VERSION)
	ln -sf $(DOCTL) $(LOCAL_BIN)/doctl
.PHONY: doctl
doctl: $(DOCTL)

DOCTL_CONFIG_DIR := $(XDG_CONFIG_HOME)/doctl
$(DOCTL_CONFIG_DIR):
	mkdir -p $(@)
DOCTL_CONFIG := $(DOCTL_CONFIG_DIR)/config.yaml
DOCTL_WITH_CONFIG = $(DOCTL) --config $(DOCTL_CONFIG)
$(DOCTL_CONFIG): | $(DOCTL_CONFIG_DIR) $(DOCTL)
	$(DOCTL_WITH_CONFIG) auth init

.PHONY: all
all:: ## Setup all resources

.PHONY: clean
clean:: ## Remove all installed resources


.PHONY: dke
dke: | $(DOCTL_CONFIG) $(KUBECONFIG_DIR) ## Create Digital Ocean Kubernetes (DKE) cluster
	$(DOCTL_WITH_CONFIG) kubernetes cluster get $(DKE_CLUSTER) \
	|| time $(DOCTL_WITH_CONFIG) kubernetes cluster create $(DKE_CLUSTER) \
	   --region $(DKE_REGION) \
	   --size $(DKE_NODES_TYPE) \
	   --version $(DKE_VERSION) \
	   --count $(DKE_NODES_COUNT) \
	   --auto-upgrade \
	   --maintenance-window saturday=20:00

$(KUBECONFIG): | $(KUBECONFIG_DIR) $(KUBECTL) $(DOCTL)
	$(DOCTL_WITH_CONFIG) kubernetes cluster kubeconfig save $(DKE_CLUSTER)
.PHONY: kubeconfig
kubeconfig: $(KUBECONFIG)

.PHONY: dke-ls
dke-ls: | $(DOCTL_CONFIG) ## List DKE clusters
	$(DOCTL_WITH_CONFIG) kubernetes cluster list

.PHONY: dke-rm
dke-rm: | $(DOCTL_CONFIG) ## Delete DKE cluster
	$(DOCTL_WITH_CONFIG) kubernetes cluster delete $(DKE_CLUSTER)

.PHONY: dke-regions
dke-regions: | $(DOCTL_CONFIG) ## Show all regions where DKE can be deployed
	$(DOCTL_WITH_CONFIG) kubernetes options regions

.PHONY: dke-sizes
dke-sizes: | $(DOCTL_CONFIG) ## Show all size options for DKE nodes
	$(DOCTL_WITH_CONFIG) kubernetes options sizes

.PHONY: dke-versions
dke-versions: | $(DOCTL_CONFIG) ## Show all size options for DKE nodes
	$(DOCTL_WITH_CONFIG) kubernetes options versions

# helm search repo --versions traefik/traefik
TRAEFIK_VERSION ?= 9.11.0
.PHONY: traefik
traefik: | $(HELM) $(KUBECONFIG) ## 5. Install Traefik
	$(HELM) repo add traefik https://helm.traefik.io/traefik
	$(HELM) upgrade traefik traefik/traefik \
	  --install \
	  --version $(TRAEFIK_VERSION) \
	  --values yaml/values.yaml
all:: traefik

.PHONY: traefik-rm
traefik-rm: | $(HELM) kubeconfig ## Remove Traefik
	$(HELM) uninstall traefik \
	; true
clean:: traefik-rm

# https://github.com/jetstack/cert-manager/releases
CERT_MANAGER_VERSION ?= 1.1.0
CERT_MANAGER_MANIFEST = https://github.com/jetstack/cert-manager/releases/download/v$(CERT_MANAGER_VERSION)/cert-manager.yaml
CERT_MANAGER_NAMESPACE = cert-manager
.PHONY: cert-manager
cert-manager: | kubeconfig ## 3. Deploy cert-manager for Let's Encrypt TLS certs
	$(KUBECTL) apply \
	  --filename $(CERT_MANAGER_MANIFEST)
all:: cert-manager

.PHONY: cert-manager-rm
cert-manager-rm: | kubeconfig
	$(KUBECTL) delete \
	  --filename $(CERT_MANAGER_MANIFEST) \
	  --ignore-not-found
clean:: cert-manager-rm

.PHONY: cert-manager-cloudflare
cert-manager-cloudflare: | cloudflare-secret kubeconfig ## 4. Enable rabbitmq.com TLS certs verification
	$(KUBECTL) apply \
	  --filename yaml/cert-manager-cloudflare-issuer.yaml
all:: cert-manager-cloudflare

.PHONY: cert-manager-cloudflare-rm
cert-manager-cloudflare-rm: | cloudflare-secret-rm kubeconfig
	$(KUBECTL) delete \
	  --filename yaml/cert-manager-cloudflare-issuer.yaml \
	  --ignore-not-found
	$(KUBECTL) delete \
	    secret/letsencrypt-issuer-account-key \
	    --namespace $(CERT_MANAGER_NAMESPACE) \
	    --ignore-not-found
clean:: cert-manager-cloudflare-rm

CLOUDFLARE_GLOBAL_API_KEY_PATH = tmp/tgir-s01e08-cert-manager-cloudflare-global-api-key
CLOUDFLARE_GLOBAL_API_KEY = "$$(cat $(CLOUDFLARE_GLOBAL_API_KEY_PATH))"
.PHONY: cloudflare-secret
cloudflare-secret: | kubeconfig
ifeq ($(wildcard $(CLOUDFLARE_GLOBAL_API_KEY_PATH)),)
	@printf "Create a new Clouflare Global API key for cert-manager $(BOLD)https://dash.cloudflare.com/profile/api-tokens$(NORMAL)\n"
	@printf "And then write it to $(BOLD)$(CLOUDFLARE_GLOBAL_API_KEY_PATH)$(NORMAL)\n"
	@exit 1
endif
	@$(KUBECTL) get secret cloudflare --namespace $(CERT_MANAGER_NAMESPACE) \
	|| ( \
	    $(KUBECTL) --dry-run=client --output=yaml \
	      create secret generic cloudflare \
	      --from-literal=global_api_key=$(CLOUDFLARE_GLOBAL_API_KEY) \
	    | $(KUBECTL) apply \
		--namespace $(CERT_MANAGER_NAMESPACE) \
		--filename - \
	  )

.PHONY: cloudflare-secret-rm
cloudflare-secret-rm: | kubeconfig
	$(KUBECTL) delete \
	  secret/cloudflare \
	  --namespace $(CERT_MANAGER_NAMESPACE) \
	  --ignore-not-found

# https://github.com/rabbitmq/cluster-operator/releases
CLUSTER_OPERATOR_VERSION ?= 1.3.0
CLUSTER_OPERATOR_MANIFEST = https://github.com/rabbitmq/cluster-operator/releases/download/v$(CLUSTER_OPERATOR_VERSION)/cluster-operator.yml
.PHONY: cluster-operator
cluster-operator: | kubeconfig ## 1. Install RabbitMQ Cluster Operator for easy & correct RabbitMQ clusters on K8S
	$(KUBECTL) apply --filename $(CLUSTER_OPERATOR_MANIFEST)
all:: cluster-operator

.PHONY: rabbitmq-%-insecure-rm
rabbitmq-%-insecure-rm: | kubeconfig ## Delete RabbitMQ cluster & public IP
	kubectl delete --filename yaml/rabbitmq-$(*)-insecure.yaml --ignore-not-found
clean:: rabbitmq-bugs-insecure-rm
clean:: rabbitmq-bunny-insecure-rm

.PHONY: rabbitmq-%-insecure
rabbitmq-%-insecure: | kubeconfig ## 2. Deploy RabbitMQ cluster with a public IP
	kubectl apply --filename yaml/rabbitmq-$(*)-insecure.yaml

.PHONY: rabbitmq-%-secure-rm
rabbitmq-%-secure-rm: | kubeconfig ## Delete RabbitMQ cluster
	$(KUBECTL) delete --filename yaml/rabbitmq-$(*)-secure.yaml --ignore-not-found
	$(KUBECTL) delete --filename yaml/rabbitmq-$(*)-cert.yaml --ignore-not-found
	$(KUBECTL) delete secret/rabbitmq-$(*)-tls --ignore-not-found
clean:: rabbitmq-bugs-secure-rm
clean:: rabbitmq-bunny-secure-rm

.PHONY: cluster-operator-rm
cluster-operator-rm:
	$(KUBECTL) delete --filename $(CLUSTER_OPERATOR_MANIFEST) --ignore-not-found
clean:: cluster-operator-rm

.PHONY: rabbitmq-%-secure-perftest-rm
rabbitmq-%-secure-perftest-rm: | kubeconfig ## Delete PerfTest that runs against RabbitMQ cluster secure public IP
	kubectl delete --filename yaml/rabbitmq-$(*)-secure-perftest.yaml --ignore-not-found
clean:: rabbitmq-bugs-secure-perftest-rm
clean:: rabbitmq-bunny-secure-perftest-rm

.PHONY: rabbitmq-%-secure-perftest
rabbitmq-%-secure-perftest: | kubeconfig ## 8. Run PerfTest against RabbitMQ cluster secure public IP
	kubectl apply --filename yaml/rabbitmq-$(*)-secure-perftest.yaml
all:: rabbitmq-bugs-secure-perftest
all:: rabbitmq-bunny-secure-perftest

.PHONY: rabbitmq-%-secure
rabbitmq-%-secure: | kubeconfig ## 6. Deploy a RabbitMQ cluster with secure public IP
	$(KUBECTL) apply --filename yaml/rabbitmq-$(*)-cert.yaml
	$(KUBECTL) apply --filename yaml/rabbitmq-$(*)-secure.yaml
all:: rabbitmq-bugs-secure
all:: rabbitmq-bunny-secure

.PHONY: traefik-%-ingress-rm
traefik-%-ingress-rm: | kubeconfig ## Deploy HTTPS and AMQPS Traefik ingress
	$(KUBECTL) delete --filename yaml/rabbitmq-$(*)-traefik-ingress-amqps.yaml --ignore-not-found
	$(KUBECTL) delete --filename yaml/rabbitmq-$(*)-traefik-ingress-https.yaml --ignore-not-found
clean:: traefik-bugs-ingress-rm
clean:: traefik-bunny-ingress-rm

.PHONY: traefik-%-ingress
traefik-%-ingress: | kubeconfig ## 7. Deploy HTTPS and AMQPS Traefik ingress
	$(KUBECTL) apply --filename yaml/rabbitmq-$(*)-traefik-ingress-amqps.yaml
	$(KUBECTL) apply --filename yaml/rabbitmq-$(*)-traefik-ingress-https.yaml
all:: traefik-bugs-ingress
all:: traefik-bunny-ingress
