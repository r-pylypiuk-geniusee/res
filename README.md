# Infrastructure Documentation

This document outlines the details of our infrastructure and the deployment process. The infrastructure is based on a Kubernetes architecture and is designed to support two primary environments.

## Architecture Overview

_Current infrastructure architecture diagram:_  

![Screenshot 2025-05-08 at 02 34 53](https://github.com/user-attachments/assets/ae112608-5a59-4588-8e45-1a355ce1de11)


Lucidchart diagram: https://drive.google.com/file/d/1BC0HOE9-reLP78bTedsVr8nvBNinrcl3/view?usp=sharing
---

## Environments

We maintain two isolated environments, each hosted in a separate Azure Kubernetes Cluster and using a dedicated database:

- Staging
- Production

---

## Repositories

The following repositories support the infrastructure:

| Repository       | Description                                                                                                                                |
| ---------------- | ------------------------------------------------------------------------------------------------------------------------------------------ |
| `ani_app_gitops` | Contains Helm templates for each service per environment, managed by ArgoCD.                                                                |
| `ani-iacc`       | Includes Kubernetes initialization scripts and Terragrunt configurations for environment setup.                                                |
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

---

## Node Architecture of Kubernetes

![Screenshot 2025-05-08 at 03 18 01](https://github.com/user-attachments/assets/776b907f-91e7-4464-a663-b7c1378a3986)

- **Staging Cluster (aks-ani-staging)** consists of two NodePools:
  - `default`: Statically configured with a predefined number of nodes.
  - `etljobs`: NodePool that automatically scales from 0 to 5 nodes based on memory/CPU requests.
  
- **Production Cluster (aks-ani-prod)** contains a single static NodeGroup (`aks-ani-prod`) where all services reside.

---

## Scaling Kubernetes Cluster

To scale the Kubernetes cluster:

1. Navigate to **Settings** → **NodePools** in the desired cluster.
2. Select the **default NodePool** and adjust the node count as needed, as shown in the screenshot.

![Screenshot 2025-05-08 at 02 55 29](https://github.com/user-attachments/assets/c9ee28a7-cf43-445e-aced-fc83252941a0)

Scaling can take a few minutes to complete, as new nodes need to be automatically configured and added to the cluster. The `etljobs` NodePool scales automatically, so no manual scaling is needed for it.

---

## Monitoring Services

We use the following monitoring services:

- **Grafana**: Monitoring platform.  
  Access: [https://grafana.{staging}.ani.tech](https://grafana.{staging}.ani.tech)

- **ELK (Elasticsearch, Logstash, Kibana)**: Centralized logging and visualization system.  
  Access: [https://logs.{staging}.ani.tech](https://logs.{staging}.ani.tech)

---

## Accessing Azure Portal
You could use the following link to access Azure - http://portal.azure.com/



## Accessing the Kubernetes Clusters

To access the Kubernetes clusters, follow these steps:

### 1. Ensure Required Permissions

Make sure you have been granted the necessary access in **Entra ID** (formerly Azure Active Directory).

To authenticate with Azure CLI:

`az login`

---

### 2. Retrieve kubeconfig Files

Use the Azure CLI to retrieve credentials and configure access to the clusters.

**Production Cluster:**

`az aks get-credentials --resource-group ani-prod --name aks-ani-prod`

**Staging Cluster:**

`az aks get-credentials --resource-group ani-staging --name aks-ani-staging`

This command updates your local `~/.kube/config` file.

Additionally, log in to the Azure Container Registry to deploy images locally:

`az acr login -n acranishared`

---

## CI/CD Process

Deployments are managed via **GitHub Actions** and **ArgoCD** (GitOps model).

### CI/CD Flow

1. **CI Files**: These are located in the `.github/workflows` directory.
   - `staging.yaml` for Staging
   - `prod.yaml` for Production

2. **Steps in the CI/CD Pipeline**:
   - Authenticate with Azure Container Registry (`acranishared`).
   - Build Docker images for services (`ani-api`, `ani-app`, `ani-websocket`, `ani-worker`).
   - Push Docker images to the container registry.

3. **Image Update and Deployment**:
   - `argocd-image-updater` monitors image versions in the registry.
   - When a new image is detected, it updates the image tag in the `ani_app_gitops` repository.
     - **Production tag format**: `${{ github.ref_name }}-${{ github.run_number }}-stable`
     - **Staging tag format**: `${{ secrets.ANI_ACR_LOGIN_SERVER }}/ani-app:${{ github.ref_name }}-${{ github.run_number }}`
   - ArgoCD detects the change and automatically triggers a new deployment.

---

## Developer Guide

### Deploying an Existing Service

To deploy a service (e.g., `ani-app`), follow these steps:

1. Merge changes into the `staging` or `main` branch.
2. This will trigger the corresponding GitHub Actions CI pipeline for the selected environment.

### Locating Environment Variables

Environment variables for each service are stored in the `ani_app_gitops` repository:

- Path: `{staging|prod}/{service_name}/values.yaml`
- Non-sensitive variables are listed under the `envs` key.
- Sensitive variables are stored under the `sealedSecrets` key.

> **Sealed Secrets** ensure sensitive data is encrypted and not visible in plain text within the repository.

### Creating a Sealed Secret

To create a new sealed secret:

1. [TODO: Add exact CLI or tool steps here.]
2. Commit the updated `values.yaml` file to the appropriate location in the `ani_app_gitops` repository.
3. This will trigger a redeploy, and the updated secret will be applied to the service.

---

### Configuring Deployment for a New Service

To configure deployment for a new service:

1. Create a new repository, push your code, and configure the `Dockerfile`.
2. Copy the existing CI workflow from any other project, then configure the required secrets for GitHub Actions (e.g., ACR and Azure access).
3. Add the Helm chart for your project to the `ani_app_gitops` repository. You can copy an existing chart and modify the values as needed.
4. Create a staging branch, which will trigger the CI pipeline. After verifying the service on staging, repeat the process for production.

---

### Scaling a Service

To scale a service temporarily:

1. Use **Lens IDE** for Kubernetes: 
   - Go to **Deployments**, select the deployment, and adjust the pod count using the provided buttons.
  
  <img width="1019" alt="Screenshot 2025-05-08 at 02 46 31" src="https://github.com/user-attachments/assets/0801ab79-640a-460e-a3ee-6c4049e8d355" />

For persistent scaling, update the `replicaCount` in the `values.yaml` file under the appropriate environment and service path in `ani_app_gitops`. ArgoCD will automatically redeploy the service.

![Screenshot 2025-05-08 at 02 48 50](https://github.com/user-attachments/assets/58971fa0-a78f-46b2-b637-02e07c08f637)

---

### Rolling Back a Service to a Previous Version

To roll back to a previous version:

1. Navigate to ArgoCD (either staging or production) at:
   - [https://argocd.staging.ani.tech/](https://argocd.staging.ani.tech/) or [https://argocd.ani.tech/](https://argocd.ani.tech/).
2. Select the appropriate service and go to **History & Rollback**.
 ![Screenshot 2025-05-08 at 02 51 37](https://github.com/user-attachments/assets/772b90a9-7b48-4b4f-9690-d381780fc14f)
4. Select the version you wish to roll back to, then click **Rollback**

  ![Screenshot 2025-05-08 at 02 52 33](https://github.com/user-attachments/assets/e053c663-78bb-46c5-aa79-6c5483577cc5)

---

### Troubleshooting ArgoCD Deployments

If ArgoCD fails to deploy a new version:

1. Go to ArgoCD, select your service, and view the full Kubernetes resource graph.
2. If the status is red, it indicates an issue with deploying the Kubernetes manifest. Click the resource to view the error details.
3. You can also check the problem using the **Info** button.

![Screenshot 2025-05-08 at 02 55 29](https://github.com/user-attachments/assets/e08db6ba-4531-487d-96af-c6da709505cf)

---

### Manual Syncing in ArgoCD

If ArgoCD fails to detect changes in your service:

1. Sync the Helm template manually by clicking the **Sync** button on the service page in ArgoCD.

![Screenshot 2025-05-08 at 02 55 29](https://github.com/user-attachments/assets/6e6aee39-b64d-4e35-a80d-a981be8dea18)

---

### How to view logs from containers

For sure you could do it via Lens IDE and click on pods to view logs in Pod section, however if you need to view old logs for example logs that were received a week ago, you can go to ELK [https://logs.{staging}.ani.tech](https://logs.{staging}.ani.tech). There you could login using elastic user and open up Analyze tab
![Screenshot 2025-05-08 at 12 05 45](https://github.com/user-attachments/assets/2b9eea4e-f0f6-47ee-bc49-4f526ae9cf5a)

Then you could type out query to filter out pods by service-name, for example on a screenshoot you see query of ani-api - kubernetes.labels.app_kubernetes_io/name : "ani-api", it filters out and shows only that specific service pods, there you also could filter out date of logs to be shown
![Screenshot 2025-05-08 at 12 09 21](https://github.com/user-attachments/assets/31e3c7b2-d2fc-4ecf-a1be-f0f925259e1c)

From fields names on a left tab, you could choose what fields to view (on a screenshoot you could see - kubernetes.node.name, kubernetes.pod.name, message). Here is exact link to query on a screenshoot - https://logs.staging.ani.tech/app/r/s/3ZTkc 

In order to view logs specifically for a staging or production, you could go to TODO

---

### How to monitor Kubernetes cluster resources, service resources on Kubernetes clusters
For that, there are two Grafana instances deployed that export resource metrics from Kubernetes cluster, one per environemnt, for staging - grafana.staging.ani.tech, and for production - grafana.ani.tech. Feel free to login via Dex button.
There you could view all available dashboards via Dashboards button
![Screenshot 2025-05-08 at 12 20 10](https://github.com/user-attachments/assets/719636ba-9836-4bd7-bfcf-3616ee2f8096)

Useful dashboards here, are 
- Kubernetes / Compute Resources / Namespace (Pods) - to view reosurce utilization (memory, cpu) per namespace
- Kubernetes / Compute Resources / Node (Pods) - to view reosurce utilization (memory, cpu) per Node
- Kubernetes / Compute Resources / Pod - to view resources utilization (memory, cpu per, network metrics Pod)
There are also similar dashboards specifically for Network metircs (such as Kubernetes / Networking / Namespace (Pods))

Talking about other dashboards
- Kubernetes / Compute Resources / Namespace (Pods) - worth to mention, it allows to view (HTTP response statuses per service or per specific pod, amount of requests, latency)

You could use that buttons for filtering out ingress/namespaces/pod metrics show in dashboard (appliable for all dashboards mentioned above)
 
---

### Updating Infrastructure Services

If you need to update an infrastructure service that ArgoCD depends on:

1. Clone the `ani-iacc` repository and navigate to `k8s-init`.
2. Create an `.env` file with the necessary values from `.env.example` (ensure staging and production files are different).
3. Generate SSH keys for `argocd-image-updater` and provide their paths in the `.env` file.
4. Run the `k8s-init` script and apply the Helm chart for the desired service.

For services not managed by ArgoCD, you can use `ani_gitops` to apply changes to the `values.yaml` file or add new charts.

---

### Managing Dex Users

To add static users or change values for existing infrastructure services like Dex:

1. Modify the `dex-extra.yaml` file in the `extra-manifests` directory.
2. Use the `staticPasswords` key to add users (passwords should be hashed using a cost factor of 10).
3. Apply the changes and ensure Dex is deployed before ArgoCD.
