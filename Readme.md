# Secure Access to Azure Key Vault from AKS Powered by Tanzu

The following repo is a companion to the blog [Deliver Secure Access to Azure Key Vault from AKS Powered by Tanzu](https://dodd-pfeffer.medium.com/deliver-secure-access-to-azure-key-vault-from-aks-powered-by-tanzu-dfdfc98138c5).

## Overview

![High Level Overview](docs/high-level-summary.png)

Setup Azure AKS Cluster and Key Vault leveraging Azure AD Workload Identity. Leverage External Secerts and Tanzu Mission Control to

- Create a Multi Tentant solution using External Secrets [Shared Cluster Store](https://external-secrets.io/v0.7.2/guides/multi-tenancy/#shared-clustersecretstore) model
- Allow individual app teams to provision External Secrets.  This is constrained by the platform operators designation of the namespace begin allowed to create External Secrets and which External Secrets they are allowed to retrieve.
- Leverage Tanzu Mission Control for to power Kubernetes Operations

The primary activity in this lab represents activities that the Azure Cloud Operator and Kubernetes Operator perform.  The final steps demonstrate the k8s user activities.

## References

- Inspiration - https://blog.container-solutions.com/tutorial-external-secrets-with-azure-keyvault - Great blog introducing the integration of AKS and Azure Key Vault through External Secrets and then leveraging OPA Gatekeeper to apply policy for which namespaces can access which secrets in the key vault.
- External-Secret Azure Provider Docs - https://external-secrets.io/v0.7.2/provider/azure-key-vault/#workload-identity - How to leverage Azure AD Workload Identity instead of requiring AD client creds in the cluster
- Azure AD Workload Identity Quick Start - https://azure.github.io/azure-workload-identity/docs/quick-start.html#5-create-a-kubernetes-service-account - Walks through the new standard (via Preview) for establishing a trust between AKS and Azure AD for external service access.  Result is no creds in AKS.
- External-Secret Multi Tenancy Guide - https://external-secrets.io/v0.7.2/guides/multi-tenancy/ - Clearly articulates how you may consider having different teams access different vaults or sections of the same vault    

## Prerequisites

Assumes you have already:
- Logged in to TMC using the `tmc` cli with context set to your organization
- Logged into Azure using the `az` cli
- Have active Tanzu Net credentials
- Have a Tanzu Mission Control subscription
- Running on a Mac (you can adjust the commands otherwise for Windows or Linux)
- Copied `local-config/params-REDACTED.yaml` and updated for your environment details

```bash
brew upgrade az

# Add feature preview flag for Workload Identity
az feature register --namespace "Microsoft.ContainerService" --name "EnableWorkloadIdentityPreview"
az provider register -n Microsoft.ContainerServic
```

## Setup Environment variables
```bash
PARAMS_YAML=local-config/params.yaml

RESOURCE_GROUP=$(yq e .resource_group $PARAMS_YAML)
LOCATION=$(yq e .location $PARAMS_YAML)
CLUSTER_NAME=$(yq e .cluster_name $PARAMS_YAML)
VAULT_NAME=$(yq e .vault_name $PARAMS_YAML)
TMC_CLUSTER_GROUP=$(yq e .tmc_cluster_group $PARAMS_YAML)
AD_APP_NAME=$(yq e .ad_app_name $PARAMS_YAML)
PLATFORM_OPS_NAMESPACE=$(yq e .platform_ops_namespace $PARAMS_YAML)
TMC_PREFIX=$(yq e .tmc_prefix $PARAMS_YAML)
PLATFORM_OPS_WORKSPACE=$TMC_PREFIX-$(yq e .platform_ops_workspace $PARAMS_YAML)
```

![Detail References](docs/detailed-references.png)

## Create Azure Resources (Azure Platform Operator)

```bash

# create resource group
az group create --name $RESOURCE_GROUP --location $LOCATION

# create cluster
az aks create --resource-group $RESOURCE_GROUP --name $CLUSTER_NAME --node-count 2 --enable-oidc-issuer --node-vm-size Standard_D4s_v3

# Rerive the the OIDC issuer URL of the newly created cluster
OIDC_ISSUER_URL=$(az aks show --resource-group $RESOURCE_GROUP --name $CLUSTER_NAME --query "oidcIssuerProfile.issuerUrl" -otsv) && echo $OIDC_ISSUER_URL

# Create Key Vault
az keyvault create --name $VAULT_NAME --resource-group $RESOURCE_GROUP

# Populate Key Vault with secrets
az keyvault secret set --name app-1--postgres972--connectionstring --vault-name $VAULT_NAME --value "app-1-secret-1-value"
az keyvault secret set --name app-2--rabbitmq361--password --vault-name $VAULT_NAME --value "app-2-secret-1-value"
az keyvault secret set --name app-3--gemfire912--password --vault-name $VAULT_NAME --value "app-3-secret-1-value"

# Create an AAD application
az ad sp create-for-rbac --name "$AD_APP_NAME"
AD_APP_CLIENT_ID=$(az ad sp list --display-name "$AD_APP_NAME" --query '[0].appId' -otsv) && echo $AD_APP_CLIENT_ID

# Grant permissions to the client to access the key vault
az keyvault set-policy --name "$VAULT_NAME" \
  --secret-permissions get \
  --spn "$AD_APP_CLIENT_ID"

# Retrieve kubeconfig for AKS Cluster
az aks get-credentials --resource-group $RESOURCE_GROUP --name $CLUSTER_NAME --overwrite-existing 

```

## Perform TMC Activities (Kubernetes Platform Operator)

![TMC Interactions](docs/tmc-interactions.png)

```bash

mkdir -p generated/tmc

# Attach the cluster to TMC
tmc cluster attach \
    --name $CLUSTER_NAME \
    --cluster-group $TMC_CLUSTER_GROUP \
    --output generated/tmc/tmc-$CLUSTER_NAME-cluster.yaml
kubectl apply -f generated/tmc/tmc-$CLUSTER_NAME-cluster.yaml
# Wait for status = TRUE
tmc cluster get $CLUSTER_NAME \
    -p attached -m attached \
    | yq .status.conditions.Agent-READY

# Create platform ops workspace in TMC and namespace in the cluster
tmc workspace create \
    --name $PLATFORM_OPS_WORKSPACE
tmc cluster namespace create \
    --name $PLATFORM_OPS_NAMESPACE \
    --workspace-name $PLATFORM_OPS_WORKSPACE \
    --cluster-name $CLUSTER_NAME -p attached -m attached 

# Add tanzunet registry credentials to the cluster, for image pull secrets
ytt -f $PARAMS_YAML -f tmc/tanzunet-creds-secret.yaml > generated/tmc/tanzunet-creds-secret.yaml
tmc cluster secret create -f generated/tmc/tanzunet-creds-secret.yaml
# Wait for status = TRUE
tmc cluster secret get tanzunet-creds \
    --namespace-name $PLATFORM_OPS_NAMESPACE \
    --cluster-name $CLUSTER_NAME -p attached -m attached \
    | yq .status.conditions.Ready

# Export tanzunet registry crednetials to all namespaces
ytt -f $PARAMS_YAML -f tmc/tanzunet-creds-secret-export.yaml > generated/tmc/tanzunet-creds-secret-export.yaml
tmc cluster secretexport create -f generated/tmc/tanzunet-creds-secret-export.yaml
# Wait for status = TRUE
tmc cluster secretexport get tanzunet-creds \
    --namespace-name $PLATFORM_OPS_NAMESPACE \
    --cluster-name $CLUSTER_NAME -p attached -m attached \
    | yq .status.conditions.Ready

# Add TAP Package Repository to the Cluster
tmc cluster tanzupackage repository create \
    --name tap \
    --imgpkg-bundle-image registry.tanzu.vmware.com/tanzu-application-platform/tap-packages:1.4.0 \
    --cluster-name $CLUSTER_NAME -p attached -m attached 
# Wait for status = TRUE
tmc cluster tanzupackage repository get tap \
    --cluster-name $CLUSTER_NAME -p attached -m attached \
    | yq .status.conditions.Ready

# Install the External Secrets package from the TAP package repository
tmc cluster tanzupackage install create \
    --name external-secrets \
    --package-metadata-name external-secrets.apps.tanzu.vmware.com \
    --version-selection-constraints "0.6.1+tap.2" \
    --namespace-name $PLATFORM_OPS_NAMESPACE \
    --cluster-name $CLUSTER_NAME -p attached -m attached 
# Wait for status = TRUE
tmc cluster tanzupackage install get external-secrets \
    --namespace-name $PLATFORM_OPS_NAMESPACE \
    --cluster-name $CLUSTER_NAME -p attached -m attached \
    | yq .status.conditions.Ready

# Create policy template that enforces esternal secret contraints for azure RBAC
tmc policy templates create -f tmc/es-begins-with-custom-policy-template.yaml

# Create policy on cluster based upon template
ytt -f $PARAMS_YAML -f tmc/es-access-by-label-custom-policy.yaml > generated/tmc/es-access-by-label-custom-policy.yaml
tmc cluster custom-policy create -f generated/tmc/es-access-by-label-custom-policy.yaml

# Create app-1, app-2, and app-3 workspaces and namespaces

# Add external secrets label to the namespace for RBAC to azure key vault
tmc workspace create \
    --name $TMC_PREFIX-app-1
tmc cluster namespace create \
    --name app-1 \
    --workspace-name $TMC_PREFIX-app-1 \
    --labels acme.com/es-regex=app-1 \
    --cluster-name $CLUSTER_NAME -p attached -m attached 

# Add external secrets label to the namespace for RBAC to azure key vault
tmc workspace create \
    --name $TMC_PREFIX-app-2
tmc cluster namespace create \
    --name app-2 \
    --workspace-name $TMC_PREFIX-app-2 \
    --labels acme.com/es-regex=app-2 \
    --cluster-name $CLUSTER_NAME -p attached -m attached 

# App 3 namespace does not get the required label
tmc workspace create \
    --name $TMC_PREFIX-app-3
tmc cluster namespace create \
    --name app-3 \
    --workspace-name $TMC_PREFIX-app-3 \
    --cluster-name $CLUSTER_NAME -p attached -m attached 
```

## Create ClusterSecretStore and Trusted Access (Mix permissions of Azure Platform Operator & Kubernetes Platform Operator)

```bash
# Create a Kubernetes service account for use by ESO
kubectl create serviceaccount workload-identity-sa --namespace $PLATFORM_OPS_NAMESPACE
kubectl label serviceaccount workload-identity-sa --namespace $PLATFORM_OPS_NAMESPACE azure.workload.identity/use=true
kubectl annotate serviceaccount workload-identity-sa --namespace $PLATFORM_OPS_NAMESPACE azure.workload.identity/client-id=$AD_APP_CLIENT_ID
TENANT_ID=$(az account show --query tenantId | tr -d \") && echo $TENANT_ID
kubectl annotate serviceaccount workload-identity-sa --namespace $PLATFORM_OPS_NAMESPACE azure.workload.identity/tenant-id=$TENANT_ID
                  ???
# Establish federated identity credential between the identity and the service account issuer & subject
AD_APP_OBJECT_ID="$(az ad app show --id $AD_APP_CLIENT_ID --query id -otsv)" && echo $AD_APP_OBJECT_ID
ytt -f azure-fed-credential-params.yaml -f $PARAMS_YAML -v oidc_issuer_url=$OIDC_ISSUER_URL -ojson > generated/fed-credential-params.json
az ad app federated-credential create --id $AD_APP_OBJECT_ID --parameters @generated/fed-credential-params.json

# Create the cluster secret store referencing the service account
ytt -f $PARAMS_YAML -f akv-demo-cluster-secret-store.yaml | kubectl apply -f -

# Validate ClusterSecretStore, should be READY, may initially say InvalidProviderConfig.  If doesn't turn ready in 10s, there is an issue
kubectl get clustersecretstore
```

## Test Out Access (Kubernetes User)

```bash
# Test ability for app-1 to create an ExternalSecret
ytt -f $PARAMS_YAML -f app-1-good-secret.yaml | kubectl apply -f -
kubectl get externalsecret,secret -n app-1
kubectl get secret secret/app-1--postgres972--connectionstring -n app-1 -ojsonpath="{.data.value}" | base64 --decode

# Test usecase where app-1 is blocked from accessing app-2 secrets
ytt -f $PARAMS_YAML -f app-1-bad-secret.yaml | kubectl apply -f -
kubectl get externalsecret,secret -n app-1

# Test ability for app-2 to create an ExternalSecret
ytt -f $PARAMS_YAML -f app-2-good-secret.yaml | kubectl apply -f -
kubectl get externalsecret,secret -n app-2
kubectl get secret app-2--rabbitmq361--password -n app-2 -ojsonpath="{.data.value}" | base64 --decode

# Test failure to create external secret when label is not on the namespace
ytt -f $PARAMS_YAML -f app-3-good-secret.yaml | kubectl apply -f -
kubectl get externalsecret,secret -n app-3
```


## Clean-up (Mix permissions of Azure Platform Operator & Kubernetes Platform Operator)

```bash
tmc cluster delete $CLUSTER_NAME -m attached -p attached --force
tmc workspace delete $PLATFORM_OPS_WORKSPACE
tmc workspace delete $TMC_PREFIX-app-1
tmc workspace delete $TMC_PREFIX-app-2
tmc workspace delete $TMC_PREFIX-app-3
tmc policy templates delete es-begins-with

az aks delete -g $RESOURCE_GROUP -n $CLUSTER_NAME -y
az ad app delete --id $AD_APP_CLIENT_ID
az keyvault delete --name $VAULT_NAME
az keyvault purge --name $VAULT_NAME
az group delete --name $RESOURCE_GROUP -y
```
