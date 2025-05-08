# Infrastructure Documentation

This document outlines the details of our infrastructure and the deployment process. The infrastructure is based on a Kubernetes architecture and is designed to support two primary environments:

## Architecture Overview

_Current infrastructure architecture diagram:_  

![Screenshot 2025-05-08 at 02 34 53](https://github.com/user-attachments/assets/ae112608-5a59-4588-8e45-1a355ce1de11)

Lucidchart diagram: https://drive.google.com/file/d/1BC0HOE9-reLP78bTedsVr8nvBNinrcl3/view?usp=sharing
## Environments

We maintain two isolated environments:

- **Staging**
- **Production**

Each environment is hosted in a separate Azure Kubernetes Cluster and uses a dedicated database.

---

## Repositories

| Repository       | Description                                                                                                                                |
| ---------------- | ------------------------------------------------------------------------------------------------------------------------------------------ |
| `ani_app_gitops` | Contains HELM templates for each service per environment, used by ArgoCD.                                                                  |
| `ani-iacc`       | Includes Kubernetes initialization scripts for infrastructure setup, and a `trgrnt` folder with Terragrunt configurations per environment. |
| `ani-api`        | Backend service repository.                                                                                                                |
| `ani-app`        | Frontend service repository.                                                                                                               |
| `ani-worker`     | Worker service repository.                                                                                                                 |

---

## Infrastructure Services in Kubernetes

Several infrastructure services are deployed in Kubernetes:

- **cert-manager**: Issues self-signed SSL certificates for Ingresses.  
  _To create a new Ingress with a certificate: TODO._

- **external-dns**: Automatically creates DNS records in Cloudflare for defined Ingress hosts.  
  _To configure a new host: TODO._

- **Dex**: An identity service integrating with external authentication systems (e.g., Google, Azure, Email, etc.).

#### Monitoring services

- **Grafana**: Monitoring platform.  
  Access: [https://grafana.{staging}.ani.tech](https://grafana.{staging}.ani.tech)

- **ELK (Elasticsearch, Logstash, Kibana)**: Centralized logging and visualization system.  
  Access: [https://logs.{staging}.ani.tech](https://logs.{staging}.ani.tech)


#### To access the Kubernetes clusters, follow these steps:

---

#### 1. Ensure Required Permissions

Make sure you have been granted the necessary access in **Entra ID** (formerly Azure Active Directory).

To use Azure CLI, you should install it and then run 

```bash
az login
```


---

#### 2. Retrieve kubeconfig Files

Use the Azure CLI to retrieve credentials and configure access to the clusters.

**Production Cluster:**
```bash
az aks get-credentials --resource-group ani-prod --name aks-ani-prod
```

**Staging Cluster:**
```bash
az aks get-credentials --resource-group ani-staging --name aks-ani-staging
```

This command updates your local ~/.kube/config file.

Also, worh to login in Azure Container Registry, in order to be able to deploy images locally

```bash
az acr login -n acranishared
```
---

## CI/CD Process

Deployments are managed via **GitHub Actions** and **ArgoCD** (GitOps model).

### CI/CD Flow

1. **CI Files** are located in `.github/workflows`:

   - `staging.yaml` for Staging
   - `prod.yaml` for Production

2. **Steps in CI/CD Pipeline**:

   - Authenticate with Azure Container Registry (`acranishared`).
   - Build Docker images for services (`ani-api`, `ani-app`, `ani-websocket`, `ani-worker`).
   - Push Docker image to the container registry.

3. **Image Update and Deployment**:
   - `argocd-image-updater` monitors image versions in the registry.
   - When a new image is detected, it updates the relevant image tag in the `ani_app_gitops` repository.
     - **Production tag format**: `${{ github.ref_name }}-${{ github.run_number }}-stable`
     - **Staging tag format**: `${{ secrets.ANI_ACR_LOGIN_SERVER }}/ani-app:${{ github.ref_name }}-${{ github.run_number }}`
   - ArgoCD detects the change and automatically triggers a new deployment.

---

## Developer Guide

### How to Deploy an Existing Service

To deploy a service (e.g., `ani-app`), follow these steps:

- Merge changes into the `staging` or `main` branch.
- This triggers the appropriate GitHub Actions CI pipeline for the selected environment.

### Locating Environment Variables

Environment variables for each service are located in the `ani_app_gitops` repository:

- Path: `{staging|prod}/{service_name}/values.yaml`
- Look under the `envs` key for non-sensitive variables.
- Sensitive variables are found under the `sealedSecrets` key.

> We use **sealedSecrets** to ensure sensitive data is encrypted and not visible in plain text within Git repositories.

### How to Create a Sealed Secret

To create a new sealed secret:

1. [TODO: Add exact CLI or tool steps here.]
2. Commit the updated `values.yaml` file to the appropriate location in the `ani_app_gitops` repository.
3. This will trigger a redeploy, and the updated secret will be applied to the service.

---

### How to configure deployment for a new service 

In order to configure deployment you should

- Create new repository, push code, configure Dockerfile 
- Copy out existing CI workflow from any other project, create secrets for Actions in repository (Taka ani-app as example, you should add secrets for ACR and Azure access)
- Add Helm chart for your project to ani_app_gitops repository, as example you could use existing charts and just tune values per your needs
- Create staging branch first (it would trigger CI), then after checking status of service on staging, do the same on Production

### How to scale service?

- If you need to scale repository temporary, you can use the following function of Lens IDE for Kubernetes. Go to Deployments, then choose your needed deployment, click on it and use the following button. Then you only need to type needed amount of pods
<img width="1019" alt="Screenshot 2025-05-08 at 02 46 31" src="https://github.com/user-attachments/assets/0801ab79-640a-460e-a3ee-6c4049e8d355" />
- If you need to scale but persistently, then you should go to ani_app_gitops, into {env}/{service_name}/values.yaml and changa value of key replicaCount, then ArgoCD would automatically redeploy it
![Screenshot 2025-05-08 at 02 48 50](https://github.com/user-attachments/assets/58971fa0-a78f-46b2-b637-02e07c08f637)

### How to rollback service to previous version

- You should go to ArgoCD, either staging or production (https://argocd.staging.ani.tech/ or https://argocd.ani.tech/). Login and choose your appropriate service, then there you should choose History & Rollback,
  ![Screenshot 2025-05-08 at 02 51 37](https://github.com/user-attachments/assets/772b90a9-7b48-4b4f-9690-d381780fc14f)
- Then scroll down, click on your version to view env variables)to find your needed version release, use 3 buttons near needed version and select Rollback
  ![Screenshot 2025-05-08 at 02 52 33](https://github.com/user-attachments/assets/e053c663-78bb-46c5-aa79-6c5483577cc5)

### ArgoCD has failed to deploy a new version of Helm Release, how to check what has happened?

- You should go to ArgoCD, either staging or production (https://argocd.staging.ani.tech/ or https://argocd.ani.tech/). Login and choose your appropriate service. Then you can view full grapgh of Kubernetes resource from which your serivce are consisted, once that would be red would suggest that something went wrong of deploying Kubernetes manifest for that service, you could click on that resource, scrow bellow and view needed error information. Also you could view the same problem using Info button
![Screenshot 2025-05-08 at 02 55 29](https://github.com/user-attachments/assets/e08db6ba-4531-487d-96af-c6da709505cf)

### ArgoCD failed to view changes for your service in ani_app_gitops?

- In that case, you should try to sync Helm template manually using Sync buttono on service page in ArgoCD, here is example:
![Screenshot 2025-05-08 at 02 55 29](https://github.com/user-attachments/assets/6e6aee39-b64d-4e35-a80d-a981be8dea18)

### How to use deploy/update Infrastructure service that is ArgoCD depends on

- You could use ani-iacc repository and by the following that path https://github.com/byobeta-team/ani-iaac/tree/ani-iaac/k8s-init and you would find k8s-init script, you could clone that locally, then you should create .env file in the same direcotry with k8s-init.sh script and populate .env script with required values from .env.example (pay attention that .env for staging and production would be different). Then beforehand you should create locally needed ssh keys that are used by argocd-image-updater and provide path to them (ssh_infra_private_key_path and ssh_app_private_key_path). Beforehand of running k8s-init you should have access to Kubernetes cluster and choose appropriate kube context to deploy to right cluster changes. Additionally, in order to not apply changes to all service at the end of k8s-init script, you could comment out seperate instance of function apply_helm_chart and leave only needed service to apply changes

### 
