TEMPO_OPERATOR_PATH := ~/redhat/dev/tempo-operator
NAMESPACE_PREFIX := $(USER)

build:
	$(TEMPO_OPERATOR_PATH)/bin/manager generate \
		--config $(TEMPO_OPERATOR_PATH)/config/manager/controller_manager_config.yaml \
		--cr cr.yaml \
		--storage.endpoint http://minio:9000 \
		--output tempo.yaml

	sed 's/mem-ballast-size-mbs=.*/mem-ballast-size-mbs=0/g'    tempo.yaml > tempo-ballast-0gb.yaml
	sed 's/mem-ballast-size-mbs=.*/mem-ballast-size-mbs=1024/g' tempo.yaml > tempo-ballast-1gb.yaml
	sed 's/mem-ballast-size-mbs=.*/mem-ballast-size-mbs=2048/g' tempo.yaml > tempo-ballast-2gb.yaml

	sed 's/value: "9223372036854775807"/value: "2147483648"/g' tempo.yaml > tempo-memlimit-2gb.yaml
	sed 's/value: "9223372036854775807"/value: "4294967296"/g' tempo.yaml > tempo-memlimit-4gb.yaml

	echo "---" > sep
	for f in tempo-*.yaml; do cat sep minio.yaml sep monitoring.yaml sep jaegerui.yaml sep loadgenerator.yaml >> $$f; done
	rm sep

apply-ballast:
	for f in tempo-ballast-*.yaml; do git add $$f; kubectl apply -n $(NAMESPACE_PREFIX)-$$(basename $$f .yaml) -f $$f; done
	-git commit -m "Applied on $$(date)"
