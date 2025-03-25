VERSION 0.6
FROM bash:4.4
IMPORT ./templates/nodejs/kubernetes AS nodejs_kubernetes_engine
IMPORT ./templates/nodejs/docker AS nodejs_docker_engine
IMPORT ./templates/php/kubernetes AS php_kubernetes_engine
IMPORT ./templates/php/docker AS php_docker_engine
IMPORT ./templates/python/kubernetes AS python_kubernetes_engine
IMPORT ./templates/python/docker AS python_docker_engine
IMPORT ./templates/java/kubernetes AS java_kubernetes_engine
IMPORT ./templates/java/docker AS java_docker_engine
IMPORT ./templates/bunjs/kubernetes AS bunjs_kubernetes_engine
IMPORT ./templates/bunjs/docker AS bunjs_docker_engine
IMPORT ./templates/rust/kubernetes AS rust_kubernetes_engine
IMPORT ./templates/rust/docker AS rust_docker_engine
WORKDIR /build-arena

install:
	ARG service='sample'
	ARG envs='dev,prod'
	ARG version='0.1'
	ARG docker_registry='drayfocus/earthly-sample'
	ARG apptype='nodejs'

	WORKDIR /setup-arena

	RUN mkdir $service $service/app

	COPY templates $service/templates
	COPY Earthfile $service

	FOR --sep="," env IN "$envs"
		ENV dir="./$service/environments/$env"
		RUN echo "Creating environment $env"

		RUN mkdir -p $dir $dir/extras-$service
		# Add gitkeep file
		RUN touch $dir/extras-$service/gitkeep
	END

	SAVE ARTIFACT $service AS LOCAL ${service}

setup:

	ARG envs='dev'

	ARG apptype='nodejs'

	RUN mkdir app

	WORKDIR app

	RUN apk update

	RUN apk add git

	## Enter repo_url.

	RUN git clone https://{repo_url}.git .

	RUN git checkout ${envs}

	RUN rm -rf package-lock.json composer.lock

	SAVE ARTIFACT * AS LOCAL templates/${apptype}/docker/app/

build:
	ARG version='0.1'
	ARG DOCKER_REGISTRY='drayfocus'
	ARG service='sample'
	ARG envs='dev,prod'
	ARG node_env="production"
	ARG apptype='nodejs'

	IF [ "$apptype" = "bunjs" ]
		BUILD bunjs_docker_engine+bunjs-server --version=$version --docker_registry=$DOCKER_REGISTRY --service=$service --node_env=$node_env
	END

	IF [ "$apptype" = "java" ]
		BUILD java_docker_engine+java-server --version=$version --docker_registry=$DOCKER_REGISTRY --service=$service
	END

	IF [ "$apptype" = "nodejs" ]
		BUILD nodejs_docker_engine+nodejs-server --version=$version --docker_registry=$DOCKER_REGISTRY --service=$service --node_env=$node_env
	END

	IF [ "$apptype" = "php" ]
		BUILD php_docker_engine+fpm-server --version=$version --docker_registry=$DOCKER_REGISTRY --service=$service
		BUILD php_docker_engine+web-server --version=$version --docker_registry=$DOCKER_REGISTRY --service=$service
	END

	IF [ "$apptype" = "python" ]
		BUILD python_docker_engine+python-server --version=$version --docker_registry=$DOCKER_REGISTRY --service=$service
	END

	IF [ "$apptype" = "rust" ]
		BUILD rust_docker_engine+rust-server --version=$version --docker_registry=$DOCKER_REGISTRY --service=$service
	END


deploy:
	FROM mcr.microsoft.com/azure-cli:2.9.0

	# setup kubectl
	ARG envs='dev'
	ARG AZURE_CLIENT_ID
	ARG AZURE_CLIENT_SECRET
	ARG AZURE_TENANT_ID
	ARG AZURE_SUBSCRIPTION_ID
	ARG AZURE_RESOURCE_GROUP
	ARG AZURE_KUBERNETES_CLUSTER_NAME
	ARG CRD_CONTROLLER_NAME
	ARG CRD_KIND
	ARG CRD_GROUP
	ARG apptype='nodejs'
	ARG service='service-name'
	ARG version=""

	RUN mkdir -p ${service}/environments

	# Remove .gitkeep file if it exists
	RUN rm -f ${service}/environments/gitkeep

	# Install necessary tools using apk package manager
	RUN apk update && \
    apk add --no-cache wget bash

	# Step to install kubectl
    RUN wget -O /usr/local/bin/kubectl https://dl.k8s.io/release/$(wget -qO- https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl && \
        chmod +x /usr/local/bin/kubectl && \
        kubectl version --client

	# Step to log in to Azure
	RUN az login --service-principal -u ${AZURE_CLIENT_ID} -p ${AZURE_CLIENT_SECRET} --tenant ${AZURE_TENANT_ID} && \
        az account set --subscription ${AZURE_SUBSCRIPTION_ID}

	# Step to activate a Kubernetes cluster
	RUN az aks get-credentials --resource-group ${AZURE_RESOURCE_GROUP} --name ${AZURE_KUBERNETES_CLUSTER_NAME}

	## Update deployment.yaml, configmap and secrets with latest versions

	IF [ "$apptype" = "bunjs" ]
		DO bunjs_kubernetes_engine+BUNJSAPP  --CRD_KIND=$CRD_KIND --CRD_GROUP=$CRD_GROUP  --service=$service --version=$version --env=$envs --deploymentReplicas=1
		DO bunjs_kubernetes_engine+CONFIGMAP --service=$service  --env=$envs
		DO bunjs_kubernetes_engine+SECRETS --service=$service  --env=$envs
	END

	IF [ "$apptype" = "java" ]
		DO java_kubernetes_engine+JAVAAPP --CRD_KIND=$CRD_KIND --CRD_GROUP=$CRD_GROUP  --service=$service --version=$version --env=$envs --deploymentReplicas=1
		DO java_kubernetes_engine+CONFIGMAP --service=$service  --env=$envs
		DO java_kubernetes_engine+SECRETS --service=$service  --env=$envs
	END

	IF [ "$apptype" = "nodejs" ]
		DO nodejs_kubernetes_engine+NODEJSAPP --CRD_KIND=$CRD_KIND --CRD_GROUP=$CRD_GROUP  --service=$service --version=$version --env=$envs --deploymentReplicas=1
		DO nodejs_kubernetes_engine+CONFIGMAP --service=$service  --env=$envs
		DO nodejs_kubernetes_engine+SECRETS --service=$service  --env=$envs
	END

	IF [ "$apptype" = "php" ]
		DO php_kubernetes_engine+LARAVELAPP --CRD_KIND=$CRD_KIND --CRD_GROUP=$CRD_GROUP  --service=$service --version=$version --env=$envs --deploymentReplicas=1
		DO php_kubernetes_engine+CONFIGMAP --service=$service  --env=$envs
		DO php_kubernetes_engine+SECRETS --service=$service  --env=$envs
	END

	IF [ "$apptype" = "python" ]
		DO python_kubernetes_engine+PYTHONAPP --CRD_KIND=$CRD_KIND --CRD_GROUP=$CRD_GROUP  --service=$service --version=$version --env=$envs --deploymentReplicas=1
		DO python_kubernetes_engine+CONFIGMAP --service=$service  --env=$envs
		DO python_kubernetes_engine+SECRETS --service=$service  --env=$envs
	END

	IF [ "$apptype" = "rust" ]
		DO rust_kubernetes_engine+RUSTAPP --CRD_KIND=$CRD_KIND --CRD_GROUP=$CRD_GROUP  --service=$service --version=$version --env=$envs --deploymentReplicas=1
		DO rust_kubernetes_engine+CONFIGMAP --service=$service  --env=$envs
		DO rust_kubernetes_engine+SECRETS --service=$service --env=$envs
	END


	# deploy resources and perform operations after deployment. Replace CRD_CONTROLLER_NAME with your deployment name
	RUN kubectl cp $service/environments/${envs}/extras-$service $(kubectl get pod -l app=${CRD_CONTROLLER_NAME} -o jsonpath="{.items[0].metadata.name}"):/usr/src/app/configs
	RUN kubectl apply -f $service/environments/${envs}

	# Waiting for pods to build

	RUN sleep 20s

	# Run post-deployment actions below. Example below.
	# RUN kubectl exec -n ${envs}-${service} deploy/${service}-fpm  -- echo "Post deployment actions completed"
