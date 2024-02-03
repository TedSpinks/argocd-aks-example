# aks-argocd-example

This is an example Argo CD deployment for AKS. It is configured with Entra ID SAML for SSO and with dual stack networking (IPv4 + IPv6). It includes Ansible AWX as its initial application to be deployed. The Setup steps below will walk you through how to customize this deployment for your specific environment.

This example does not include:
- Automating the DNS records - you can do this with [Terraform](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/dns_aaaa_record) or [external-dns](https://github.com/kubernetes-sigs/external-dns/blob/master/docs/tutorials/azure.md).
- Deploying/syncing secrets from a vault, such as Azure Key Vault - you can do this with the [Secrets Store CSI Driver](https://learn.microsoft.com/en-us/azure/aks/csi-secrets-store-driver) or an open source tool such as the [External Secrets Operator](https://github.com/external-secrets/external-secrets).

## Design Decisions

AWX
- We are using an external PostgreSQL 13 service in Azure, to simplify maintenance of the K8s-based components (keep them stateless)
- We are using an `Ingress` resource for TLS termination (AWX documentation does not have other TLS options)

IPv6
- For IPv6 compatibility, we are using NGINX ingress controller, which we can front-end with our own IPv6 `LoadBalancer` service
  - Note: Azure App Gateway support for IPv6 is in Preview as of 11/2023, and not yet supported by App Gateway Ingress Controller
  - For internal private IP support, we are using the *community* NGINX ingress controller, as opposed to AKS' managed "Application Router Add-on"

Argo CD
- We are using Entra ID SAML for SSO
- Since we are terminating TLS at our ingress controller (via a wildcard cert), we are using [TLS passthrough](https://argo-cd.readthedocs.io/en/stable/operator-manual/ingress/#option-1-ssl-passthrough) for the Argo CD ingress.
- Argo CD does not come with CPU or Memory [requests/limits](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/), which are essential for proper Kubernetes resource management.
  - [VPA + Goldilocks](https://goldilocks.docs.fairwinds.com/) are installed to provide recommended requests/limits for Argo CD after 1 week of regular use. This is a nice, lightweight introduction to K8s resource management.
  - Initial requests/limits are provided in the `prod` overlay. These are from the VPA in my lab (very low usage), so these should  be replaced with recommendations from *your* Goldilocks.
  - **IMPORTANT**: 
    - Goldilocks' recommendations for the **argocd-repo-server** Deployment are very undersized, which causes sync errors and should not be used.
    - Goldilocks' recommendations for the **argocd-server** Deployment are also undersized, which might cause 404 errors during SAML login.

## Supporting Documentation

AWX Installation References

- [Basic Installation](https://github.com/ansible/awx-operator/blob/devel/docs/installation/basic-install.md)
- [Database Configuration](https://github.com/ansible/awx-operator/blob/devel/docs/user-guide/database-configuration.md)
- [Network and TLS Configuration](https://github.com/ansible/awx-operator/blob/devel/docs/user-guide/network-and-tls-configuration.md)

NGINX Ingress Controller Installation References

- [AKS Ingress](https://learn.microsoft.com/en-us/azure/aks/ingress-basic?tabs=azure-cli)
- [AKS IPv6 LoadBalancer](https://learn.microsoft.com/en-us/azure/aks/configure-kubenet-dual-stack?tabs=azure-cli%2Cyaml#expose-the-workload-via-a-loadbalancer-type-service)

Argo CD Installation

- [Installation](https://argo-cd.readthedocs.io/en/stable/operator-manual/installation/)
- [Ingress](https://argo-cd.readthedocs.io/en/stable/operator-manual/ingress/)
- [Azure SAML](https://argo-cd.readthedocs.io/en/stable/operator-manual/user-management/microsoft/#azure-ad-saml-enterprise-app-auth-using-dex)
- [Goldilocks](https://goldilocks.docs.fairwinds.com/) and its [sizing issue with argocd-repo-server](https://github.com/argoproj/argo-cd/issues/9701)


## Setup

#### 1. Verify AKS' Requirements

Installing AKS with a System Managed Identity should grant it Network Contributor access to the VNet where its node subnet lives. But sometimes it doesn't work!

This is required for assigning the static IPs for our ingress. Manually add the permission via the Azure Portal or the CLI:

```bash
CLUSTER_NAME=MY-CLUSTER
RESOURCE_GROUP=MY-RG
VNET_ID="/subscriptions/MY-SUBSCRIPTION/resourceGroups/MY-RG/providers/Microsoft.Network/virtualNetworks/MY-VNET"

MANAGED_ID=$(az aks show -g $RESOURCE_GROUP -n $CLUSTER_NAME --query "identity.principalId" --output tsv)
az role assignment create --assignee $MANAGED_ID --role "Network Contributor" --scope $VNET_ID
```

Make sure the AKS cluster has the Vertical Pod Autoscaler (VPA) installed. We will be using this to size the Argo CD pods.

```bash
az aks update -g $RESOURCE_GROUP -n $CLUSTER_NAME --enable-vpa
```

#### 2. Manually create the `awx` database in your external PostgreSQL 13 server.

#### 3. Create DNS records

Assign a static IPv4 and IPv4 address to be used for ingress URLs. Create the following DNS records, which point to that address:

- AWX
  - ansible.example.com
- Argo CD
  - argocd.example.com
- Goldilocks
  - goldilocks.example.com

#### 4. Setup SAML in Azure

Follow the [documentation](https://argo-cd.readthedocs.io/en/stable/operator-manual/user-management/microsoft/#azure-ad-saml-enterprise-app-auth-using-dex) to setup SAML on the Azure side.

#### 5. Configure the environment in Git. 

Update the following files with the values for your environment. 

*To create an additional environment, first copy the `prod` directory and name it after the new environment, and then update the copied files.*

AWX
- [awx/prod/kustomization.yaml](./awx/prod/kustomization.yaml) - AWX namespace, AWX instance name and AWX FQDN

Ingress
- [ingress-nginx/prod/kustomization.yaml](./ingress-nginx/prod/kustomization.yaml) - Static IPv4 and IPv6 addresses for ingress

Argo CD
- Ingress
  - [argocd/prod/kustomization.yaml](./argocd/prod/kustomization.yaml) - FQDN for UI and API
- Apps
  - [argocd/prod/app-of-apps-root.yaml](./argocd/prod/app-of-apps-root.yaml) - GitOps repo URL
  - [argocd/prod/apps/argocd.yaml](./argocd/prod/apps/argocd.yaml) - GitOps repo URL
  - [argocd/prod/apps/awx.yaml](./argocd/prod/apps/awx.yaml) - GitOps repo URL
  - [argocd/prod/apps/goldilocks.yaml](./argocd/prod/apps/goldilocks.yaml) - GitOps repo URL
  - [argocd/prod/apps/ingress-nginx.yaml](./argocd/prod/apps/ingress-nginx.yaml) - GitOps repo URL
- SAML
  - [argocd/prod/patches/argocd-cm.yaml](./argocd/prod/patches/argocd-cm.yaml) - SAML config
  - [argocd/prod/patches/argocd-rbac-cm.yaml](./argocd/prod/patches/argocd-rbac-cm.yaml) - Azure group object IDs

Goldilocks
- [goldilocks/prod/kustomization.yaml](goldilocks/prod/kustomization.yaml) - FQDN for UI

Verify that all 4 Kustomize directories still build successfully with your new values.

```bash
kustomize build argocd/prod
kustomize build awx/prod
kustomize build --enable-helm goldilocks/prod
kustomize build --enable-helm ingress-nginx/prod
```

#### 6. Precreate the secret for AWX to authenticate to Postgres. 

You can use the example commands below, but fill in the correct `host` FQDN for your postgres server, and the correct `username` and `password`.
```bash
kubectl create ns awx-prod

kubectl create secret generic postgres-configuration -n awx-prod \
  --from-literal=host=postgres-0.postgres.postgres.svc.cluster.local \
  --from-literal=port=5432 \
  --from-literal=database=awx \
  --from-literal=username=postgres  \
  --from-literal=password=Letmein123 \
  --from-literal=sslmode=prefer  \
  --from-literal=type=unmanaged
```

#### 7. Precreate your default TLS secret for ingress and Argo CD

```bash
# Default ingress TLS cert (wildcard)
kubectl create ns ingress-nginx
kubectl create secret tls default-tls -n ingress-nginx --key wildcard-tls-cert.key --cert wildcard-tls-cert.crt

# TLS cert for Argo CD (can be same as default ingress)
kubectl create ns argocd
kubectl create secret tls argocd-server-tls -n argocd --key wildcard-tls-cert.key --cert wildcard-tls-cert.crt
```

#### 8. Bootstrap the Argo CD installation

```bash
kubectl apply -k argocd/prod
```

#### 9. Add your Git authentication secret to Argo CD

Install the [Argo CD CLI](https://argo-cd.readthedocs.io/en/stable/cli_installation/).

Run the commands below to add a Git secret to Argo CD. Be sure to replace the Git URL, username, and password

```bash
# Get the initial admin password
kubectl get -n argocd secret argocd-initial-admin-secret -o jsonpath={.data.password} | base64 -d

# Login to Argo CD
argo cd login grpc-argocd.example.com

# Add a token for authenticating to your copy of this Git repo
argocd repo add https://github.com/TedSpinks/aks-argocd-example --username MyServiceAccount --password Letmein123
```

#### 10. Connect to the Argo CD UI and watch it deploy the other apps
