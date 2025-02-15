include ../metadata.mk

PACKAGE_NAME = github.com/projectcalico/calico/cni-plugin

# Add in local static-checks
LOCAL_CHECKS=check-boring-ssl

# Name of the images.
# e.g., <registry>/<name>:<tag>
CNI_PLUGIN_IMAGE ?=cni
BUILD_IMAGES   ?=$(CNI_PLUGIN_IMAGE)

###############################################################################
# Download and include ../lib.Makefile
#   Additions to EXTRA_DOCKER_ARGS need to happen before the include since
#   that variable is evaluated when we declare DOCKER_RUN and siblings.
###############################################################################
include ../lib.Makefile

###############################################################################
BIN=bin/$(ARCH)
SRC_FILES=$(shell find pkg cmd internal -name '*.go')
TEST_SRC_FILES=$(shell find tests -name '*.go')
WINFV_SRCFILES=$(shell find win_tests -name '*.go')
LOCAL_IP_ENV?=$(shell ip route get 8.8.8.8 | head -1 | awk '{print $$7}')

# fail if unable to download
CURL=curl -C - -sSf

CNI_VERSION=v1.0.0

# By default set the CNI_SPEC_VERSION to 0.3.1 for tests.
CNI_SPEC_VERSION?=0.3.1

DEPLOY_CONTAINER_MARKER=cni_deploy_container-$(ARCH).created

# We need CGO to leverage Boring SSL.  However, the cross-compile doesn't support CGO yet.
ifeq ($(ARCH), $(filter $(ARCH),amd64))
CGO_ENABLED=1
CHECK_BORINGSSL=go tool nm $(BIN)/install > $(BIN)/tags.txt && grep '_Cfunc__goboringcrypto_' $(BIN)/tags.txt 1> /dev/null
else
CGO_ENABLED=0
CHECK_BORINGSSL=echo 'Skipping boringSSL check for $(ARCH)'
endif

.PHONY: clean
clean:
	# Clean .created files which indicate images / releases have been built.
	find . -name '.*.created*' -type f -delete
	find . -name '.*.published*' -type f -delete
	rm -rf $(BIN) bin $(DEPLOY_CONTAINER_MARKER) .go-pkg-cache pkg/install/install.test 
	rm -f *.created
	rm -f crds.yaml
	rm -rf config/

###############################################################################
# Building the binary
###############################################################################
BIN=bin/$(ARCH)
build: $(BIN)/install $(BIN)/calico $(BIN)/calico-ipam
ifeq ($(ARCH),amd64)
# Go only supports amd64 for Windows builds.
BIN_WIN=bin/windows
build: $(BIN_WIN)/calico.exe $(BIN_WIN)/calico-ipam.exe
endif
# If ARCH is arm based, find the requested version/variant
ifeq ($(word 1,$(subst v, ,$(ARCH))),arm)
ARM_VERSION := $(word 2,$(subst v, ,$(ARCH)))
endif
# Define go architecture flags to support arm variants
GOARCH_FLAGS :=-e GOARCH=$(ARCH)
ifdef ARM_VERSION
GOARCH_FLAGS :=-e GOARCH=arm -e GOARM=$(ARM_VERSION)
endif

build-all: $(addprefix sub-build-,$(VALIDARCHES))
sub-build-%:
	$(MAKE) build ARCH=$*
	
## Build the Calico network plugin and ipam plugins
$(BIN)/install binary: $(SRC_FILES)
	-mkdir -p .go-pkg-cache
	-mkdir -p $(BIN)
	$(DOCKER_RUN) -e CGO_ENABLED=$(CGO_ENABLED) $(CALICO_BUILD) sh -c '\
		$(GIT_CONFIG_SSH) go build -v -o $(BIN)/install -ldflags "-X main.VERSION=$(GIT_VERSION) -w" $(PACKAGE_NAME)/cmd/calico && \
		$(CHECK_BORINGSSL)'

## Build the Calico network plugin and ipam plugins for Windows
$(BIN_WIN)/calico.exe $(BIN_WIN)/calico-ipam.exe: $(SRC_FILES)
	$(DOCKER_RUN) \
	-e GOOS=windows \
	-e CGO_ENABLED=1 \
	    $(CALICO_BUILD) sh -c '$(GIT_CONFIG_SSH) \
		go build -v -o $(BIN_WIN)/calico.exe -ldflags "-X main.VERSION=$(GIT_VERSION) -s -w" $(PACKAGE_NAME)/cmd/calico && \
		go build -v -o $(BIN_WIN)/calico-ipam.exe -ldflags "-X main.VERSION=$(GIT_VERSION) -s -w" $(PACKAGE_NAME)/cmd/calico'


###############################################################################
# Building the image
###############################################################################
image: $(DEPLOY_CONTAINER_MARKER)
image-all: $(addprefix sub-image-,$(VALIDARCHES))
sub-image-%:
	$(MAKE) image ARCH=$*

$(DEPLOY_CONTAINER_MARKER): Dockerfile.$(ARCH) build fetch-cni-bins
	docker build -t $(CNI_PLUGIN_IMAGE):latest-$(ARCH) --build-arg QEMU_IMAGE=$(CALICO_BUILD) --build-arg UBI_IMAGE=$(UBI_IMAGE) --build-arg GIT_VERSION=$(GIT_VERSION) -f Dockerfile.$(ARCH) .
	$(MAKE) retag-build-images-with-registries VALIDARCHES=$(ARCH) IMAGETAG=latest
	touch $@

.PHONY: fetch-cni-bins
fetch-cni-bins: $(BIN)/flannel $(BIN)/loopback $(BIN)/host-local $(BIN)/portmap $(BIN)/tuning $(BIN)/bandwidth

$(BIN)/loopback $(BIN)/host-local $(BIN)/portmap $(BIN)/tuning $(BIN)/bandwidth:
	mkdir -p $(BIN)
	$(CURL) -L --retry 5 https://github.com/containernetworking/plugins/releases/download/$(CNI_VERSION)/cni-plugins-linux-$(subst v7,,$(ARCH))-$(CNI_VERSION).tgz | tar -xz -C $(BIN) ./loopback ./host-local ./portmap ./tuning ./bandwidth

$(BIN)/flannel:
	# Flannel was removed from v1.0.0 CNI plugins, but doesn't yet have its own release.
	# For now, just pull the older version of the binary hosted at containernetworking/plugins.
	# Eventually, replace with a release from https://github.com/flannel-io/cni-plugin
	mkdir -p $(BIN)
	$(CURL) -L --retry 5 https://github.com/containernetworking/plugins/releases/download/v0.9.1/cni-plugins-linux-$(subst v7,,$(ARCH))-v0.9.1.tgz | tar -xz -C $(BIN) ./flannel

###############################################################################
# Unit Tests
###############################################################################
## Run the unit tests.
ut: run-k8s-controller $(BIN)/install $(BIN)/host-local $(BIN)/calico-ipam $(BIN)/calico
	$(MAKE) ut-datastore DATASTORE_TYPE=etcdv3
	$(MAKE) ut-datastore DATASTORE_TYPE=kubernetes

check-boring-ssl: $(BIN)/install
	$(DOCKER_RUN) -e CGO_ENABLED=$(CGO_ENABLED) $(CALICO_BUILD) $(CHECK_BORINGSSL)
	-rm -f $(BIN)/tags.txt

$(BIN)/calico-ipam $(BIN)/calico: $(BIN)/install
	cp "$<" "$@"

ut-datastore:
	# The tests need to run as root
	docker run --rm -t --privileged --net=host \
	$(EXTRA_DOCKER_ARGS) \
	-e ETCD_IP=$(LOCAL_IP_ENV) \
	-e LOCAL_USER_ID=$(LOCAL_USER_ID) \
	-e RUN_AS_ROOT=true \
	-e ARCH=$(ARCH) \
	-e PLUGIN=calico \
	-e BIN=/go/src/$(PACKAGE_NAME)/$(BIN) \
	-e CNI_SPEC_VERSION=$(CNI_SPEC_VERSION) \
	-e DATASTORE_TYPE=$(DATASTORE_TYPE) \
	-e ETCD_ENDPOINTS=http://$(LOCAL_IP_ENV):2379 \
	-e KUBECONFIG=/home/user/certs/kubeconfig \
	-v $(CURDIR)/../:/go/src/github.com/projectcalico/calico:rw \
	-v $(CURDIR)/tests/certs:/home/user/certs \
	$(CALICO_BUILD) sh -c '$(GIT_CONFIG_SSH) \
			cd  /go/src/$(PACKAGE_NAME) && \
			ginkgo -cover -r -skipPackage pkg/install $(GINKGO_ARGS)'

ut-etcd: run-k8s-controller build $(BIN)/host-local
	$(MAKE) ut-datastore DATASTORE_TYPE=etcdv3
	make stop-etcd
	make stop-k8s-controller

ut-kdd: run-k8s-controller build $(BIN)/host-local
	$(MAKE) ut-datastore DATASTORE_TYPE=kubernetes
	make stop-etcd
	make stop-k8s-controller

## Run the tests in a container (as root) for different CNI spec versions
## to make sure we don't break backwards compatibility.
.PHONY: test-cni-versions
test-cni-versions:
	for cniversion in "0.2.0" "0.3.1" ; do \
		if make ut CNI_SPEC_VERSION=$$cniversion; then \
			echo "CNI version $$cniversion PASSED"; \
		else \
			echo "CNI version $$cniversion FAILED"; \
			exit 1; \
		fi; \
	done

config/crd: mod-download
	mkdir -p config/crd
	$(DOCKER_GO_BUILD) sh -c ' \
		cp -r ../libcalico-go/config/crd/* config/crd; \
		chmod +w config/crd/*'

## Kubernetes apiserver used for tests
run-k8s-apiserver: config/crd stop-k8s-apiserver run-etcd
	docker run --detach --net=host \
	  --name calico-k8s-apiserver \
	  -v $(CURDIR)/config:/config \
	  -v $(CURDIR)/tests/certs:/home/user/certs \
	  -e KUBECONFIG=/home/user/certs/kubeconfig \
	  $(CALICO_BUILD) kube-apiserver \
	    --etcd-servers=http://$(LOCAL_IP_ENV):2379 \
	    --service-cluster-ip-range=10.101.0.0/16 \
            --authorization-mode=RBAC \
            --service-account-key-file=/home/user/certs/service-account.pem \
            --service-account-signing-key-file=/home/user/certs/service-account-key.pem \
            --service-account-issuer=https://localhost:443 \
            --api-audiences=kubernetes.default \
            --client-ca-file=/home/user/certs/ca.pem \
            --tls-cert-file=/home/user/certs/kubernetes.pem \
            --tls-private-key-file=/home/user/certs/kubernetes-key.pem \
            --enable-priority-and-fairness=false \
            --max-mutating-requests-inflight=0 \
            --max-requests-inflight=0

	# Wait until the apiserver is accepting requests.
	while ! docker exec calico-k8s-apiserver kubectl get nodes; do echo "Waiting for apiserver to come up..."; sleep 2; done
	while ! docker exec calico-k8s-apiserver kubectl apply -f /config/crd; do echo "Waiting for CRDs..."; sleep 2; done

## Kubernetes controller manager used for tests
run-k8s-controller: stop-k8s-controller run-k8s-apiserver
	docker run --detach --net=host \
	  --name calico-k8s-controller \
	  -v $(CURDIR)/tests/certs:/home/user/certs \
	  $(CALICO_BUILD) kube-controller-manager \
            --master=https://127.0.0.1:6443 \
            --kubeconfig=/home/user/certs/kube-controller-manager.kubeconfig \
            --min-resync-period=3m \
            --allocate-node-cidrs=true \
            --cluster-cidr=192.168.0.0/16 \
            --v=5 \
            --service-account-private-key-file=/home/user/certs/service-account-key.pem \
            --root-ca-file=/home/user/certs/ca.pem

## Stop Kubernetes apiserver
stop-k8s-apiserver:
	@-docker rm -f calico-k8s-apiserver

## Stop Kubernetes controller manager
stop-k8s-controller:
	@-docker rm -f calico-k8s-controller

## Etcd is used by the tests
run-etcd: stop-etcd
	docker run --detach \
	  -p 2379:2379 \
	  --name calico-etcd $(ETCD_IMAGE) \
	  etcd \
	  --advertise-client-urls "http://$(LOCAL_IP_ENV):2379,http://127.0.0.1:2379,http://$(LOCAL_IP_ENV):4001,http://127.0.0.1:4001" \
	  --listen-client-urls "http://0.0.0.0:2379,http://0.0.0.0:4001"

## Stops calico-etcd containers
stop-etcd:
	@-docker rm -f calico-etcd

###############################################################################
# Install test
###############################################################################
# We pre-build the test binary so that we can run it outside a container and allow it
# to interact with docker.
pkg/install/install.test: pkg/install/*.go
	$(DOCKER_RUN) $(CALICO_BUILD) sh -c '$(GIT_CONFIG_SSH) \
			cd /go/src/$(PACKAGE_NAME) && \
			go test ./pkg/install -c --tags install_test -o ./pkg/install/install.test'

.PHONY: test-install-cni
## Test the install
test-install-cni: image pkg/install/install.test
	cd pkg/install && CONTAINER_NAME=$(CNI_PLUGIN_IMAGE):latest-$(ARCH) ./install.test

###############################################################################
# CI/CD
###############################################################################
.PHONY: ci
ci: clean mod-download build static-checks test-cni-versions image-all test-install-cni

## Deploys images to registry
cd: image-all cd-common

## Build fv binary for Windows
$(BIN_WIN)/win-fv.exe: $(WINFV_SRCFILES)
	$(DOCKER_RUN) -e GOOS=windows $(CALICO_BUILD) sh -c '$(GIT_CONFIG_SSH) go test ./win_tests -c -o $(BIN_WIN)/win-fv.exe'

###############################################################################
# Release
###############################################################################
## Produces a clean build of release artifacts at the specified version.
release-build: .release-$(VERSION).created 
.release-$(VERSION).created:
	$(MAKE) clean image-all RELEASE=true
	$(MAKE) retag-build-images-with-registries RELEASE=true IMAGETAG=$(VERSION)
	$(MAKE) retag-build-images-with-registries RELEASE=true IMAGETAG=latest
	touch $@

## Verifies the release artifacts produces by `make release-build` are correct.
release-verify: release-prereqs
	# Check the reported version is correct for each release artifact.
	docker run --rm $(CNI_PLUGIN_IMAGE):$(VERSION)-$(ARCH) calico -v | grep -x $(VERSION) || ( echo "Reported version:" `docker run --rm $(CNI_PLUGIN_IMAGE):$(VERSION)-$(ARCH) calico -v` "\nExpected version: $(VERSION)" && exit 1 )
	docker run --rm $(CNI_PLUGIN_IMAGE):$(VERSION)-$(ARCH) calico-ipam -v | grep -x $(VERSION) || ( echo "Reported version:" `docker run --rm $(CNI_PLUGIN_IMAGE):$(VERSION)-$(ARCH) calico-ipam -v | grep -x $(VERSION)` "\nExpected version: $(VERSION)" && exit 1 )
	docker run --rm quay.io/calico/cni:$(VERSION)-$(ARCH) calico -v | grep -x $(VERSION) || ( echo "Reported version:" `docker run --rm quay.io/calico/cni:$(VERSION)-$(ARCH) calico -v | grep -x $(VERSION)` "\nExpected version: $(VERSION)" && exit 1 )
	docker run --rm quay.io/calico/cni:$(VERSION)-$(ARCH) calico-ipam -v | grep -x $(VERSION) || ( echo "Reported version:" `docker run --rm quay.io/calico/cni:$(VERSION)-$(ARCH) calico-ipam -v | grep -x $(VERSION)` "\nExpected version: $(VERSION)" && exit 1 )

	# TODO: Some sort of quick validation of the produced binaries.

release-publish: release-prereqs .release-$(VERSION).published
.release-$(VERSION).published:
	$(MAKE) push-images-to-registries push-manifests IMAGETAG=$(VERSION) RELEASE=$(RELEASE) CONFIRM=$(CONFIRM)
	touch $@

# WARNING: Only run this target if this release is the latest stable release. Do NOT
# run this target for alpha / beta / release candidate builds, or patches to earlier Calico versions.
## Pushes `latest` release images. WARNING: Only run this for latest stable releases.
release-publish-latest: release-prereqs
	# Check latest versions match.
	if ! docker run $(CNI_PLUGIN_IMAGE):latest-$(ARCH) calico -v | grep '^$(VERSION)$$'; then echo "Reported version:" `docker run $(CNI_PLUGIN_IMAGE):latest-$(ARCH) calico -v` "\nExpected version: $(VERSION)"; false; else echo "\nVersion check passed\n"; fi
	if ! docker run quay.io/$(CNI_PLUGIN_IMAGE):latest-$(ARCH) calico -v | grep '^$(VERSION)$$'; then echo "Reported version:" `docker run quay.io/$(CNI_PLUGIN_IMAGE):latest-$(ARCH) calico -v` "\nExpected version: $(VERSION)"; false; else echo "\nVersion check passed\n"; fi

	$(MAKE) push-images-to-registries push-manifests RELEASE=true IMAGETAG=latest RELEASE=$(RELEASE) CONFIRM=$(CONFIRM)

###############################################################################
# Developer helper scripts (not used by build or test)
###############################################################################
.PHONY: test-watch
## Run the unit tests, watching for changes.
test-watch: $(BIN)/install run-etcd run-k8s-apiserver
	# The tests need to run as root
	CGO_ENABLED=0 ETCD_IP=127.0.0.1 PLUGIN=calico GOPATH=$(GOPATH) $(shell which ginkgo) watch -skipPackage pkg/install
