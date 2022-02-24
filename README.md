# Azure Confidential Compute meets Azure Kubernetes Service
Demo on how to leverage Azure Confidential Compute with Azure Kubernetes Service

# Howto use this repository
* [Optional] Clone this repo and launch a GitHub CodeSpaces on it
* Change the content of the variables
* Execute the commands
* [Mandatory] Enjoy!

# Prerequisites
* [Azure CLI](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli) installed
* Azure Subscription
* Helm installed

# Resources
This demo has been constructed after going through the following docs ; 
* https://docs.microsoft.com/en-us/cli/azure/aks?view=azure-cli-latest
* https://docs.microsoft.com/en-us/azure/confidential-computing/confidential-enclave-nodes-aks-get-started 
  (Please note : by default a quota of max 8 cores is available in your subscription)
* https://github.com/openenclave/openenclave/tree/master/samples/helloworld

# Set up your Confidential Compute powered AKS

## Prepare your Azure Subscription
### Login to your Subscription
```bash
az login [--use-device-code]
az account set --subscription "your-subscription-name/guid"
```

### Set Variables
```bash
rgaks="demo-test-aks"
nameaks="demoakscluster"
nameconfpool="confidential"
region="westeurope"
acrname="confidentialacr"
```

## Preparing the Azure Kubernetes environment [pick one variant]
### Create Resource Group
```bash
az group create --name $rgaks --location $region --output table
```

### Variant A - Seperate nodepool
```bash
az aks create -g $rgaks -n $nameaks --enable-cluster-autoscaler  --min-count 1 --max-count 4 --enable-addons monitoring --generate-ssh-keys -s "Standard_B4ms" --enable-addons confcom --enable-sgxquotehelper
az aks nodepool add --cluster-name $nameaks --name $nameconfpool --resource-group $rgaks --node-vm-size "Standard_DC2s_v2"
```

### Variant B - Single nodepool
```bash
az aks create -g $rgaks -n $nameaks --enable-cluster-autoscaler  --min-count 1 --max-count 4 --enable-addons monitoring --generate-ssh-keys -s "Standard_DC2s_v2" --enable-addons confcom --enable-sgxquotehelper
```

## Get Kubectl Credentials
```bash
az aks get-credentials --name $nameaks --resource-group $rgaks
```

## Validate SGX running
```bash
kubectl get pods --all-namespaces | grep -i sgx
```

## Test an SGX Hello World
```bash
kubectl apply -f hello-world-enclave.yaml
kubectl get jobs -l app=sgx-test
kubectl get pods -l app=sgx-test
kubectl logs -l app=sgx-test
```


# Running existing container images on AKS powered by Confidential Compute [WORK IN PROGRESS - NOT STABLE EXPERIENCE YET]
## Using Fortanix

### Resources
* https://docs.microsoft.com/en-us/azure/confidential-computing/confidential-containers
* https://support.fortanix.com/hc/en-us/articles/360054022392-Azure-Kubernetes-Service-with-Fortanix-Confidential-Computing-Manager
* https://support.fortanix.com/hc/en-us/articles/360042965552-Using-Fortanix-Confidential-Computing-Manager-to-Build-an-Enclave-OS-Application-from-Scratch
* https://hub.docker.com/_/wordpress?tab=tags
* https://bitnami.com/stack/wordpress/helm
* https://github.com/bitnami/charts/tree/master/bitnami/wordpress
* https://docs.bitnami.com/kubernetes/faq/administration/understand-helm-chart/
* https://docs.microsoft.com/en-us/azure/aks/cluster-container-registry-integration?tabs=azure-cli

### Create the Container Registry
```bash
az acr create --resource-group $rgaks --name $acrname --sku Standard
az aks update -n $nameaks -g $rgaks --attach-acr $acrname
az acr update --name $acrname --anonymous-pull-enabled
```

### Enroll AKS nodes in Fortanix CCM
```bash
kubectl create secret generic em-token --from-literal=token=<token>
kubectl create -f fortanix-ccm.yaml
kubectl get pods --all-namespaces
```

### Build Image in Fortanix
* Add you registry to Fortanix CCM
* Go through the following documentation ; https://support.fortanix.com/hc/en-us/articles/360042965552-Using-Fortanix-Confidential-Computing-Manager-to-Build-an-Enclave-OS-Application-from-Scratch
* Add "wordpress" as the source for the image and "[ACRNAME].azurecr.io/wordpress" (replacing the [ACRNAME] with the actual name of your Azure Container Registry) as output
* Give an existing tag on the wordpress part, and the same for the output
* Approve the build

### Launch Modified Wordpress
```bash
helm repo add azure-marketplace https://marketplace.azurecr.io/helm/v1/repo
helm install my-confential-release azure-marketplace/wordpress --set=global.imageRegistry=confidentialacr.azurecr.io --debug --dry-run
helm install my-confential-release azure-marketplace/wordpress --set=global.imageRegistry=confidentialacr.azurecr.io
```

### Cleanup Cluster
helm ls --all --short | xargs -L1 helm delete