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
```

## Preparing the Azure Kubernetes environment [pick one variant]
### Variant A - Seperate nodepool
```bash
az group create --name $rgaks --location $region --output table
az aks create -g $rgaks -n $nameaks --enable-cluster-autoscaler  --min-count 1 --max-count 4 --enable-addons monitoring --generate-ssh-keys -s "Standard_B4ms" --enable-addons confcom --enable-sgxquotehelper
az aks nodepool add --cluster-name $nameaks --name $nameconfpool --resource-group $rgaks --node-vm-size "Standard_DC2s_v2"
```

### Variant B - Single nodepool
```bash
az group create --name $rgaks --location $region --output table
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
