# Infrastructure Documentation

This document outlines the details of our infrastructure and the deployment process. The infrastructure is based on a Kubernetes architecture and is designed to support two primary environments.

---

## Architecture Overview

_Current infrastructure architecture diagram:_  
![Screenshot 2025-05-08 at 02 34 53](https://github.com/user-attachments/assets/ae112608-5a59-4588-8e45-1a355ce1de11)

Lucidchart diagram: [https://drive.google.com/file/d/1BC0HOE9-reLP78bTedsVr8nvBNinrcl3/view?usp=sharing](https://drive.google.com/file/d/1BC0HOE9-reLP78bTedsVr8nvBNinrcl3/view?usp=sharing)

---

## Environments

We maintain two isolated environments, each hosted in a separate Azure Kubernetes Cluster and using a dedicated database:

- **Staging**
- **Production**

---

## Repositories

The following repositories support the infrastructure:

| Repository       | Description                                                                                      |
| ---------------- | ------------------------------------------------------------------------------------------------ |
| `ani_app_gitops` | Contains Helm templates for each service per environment, managed by ArgoCD.                    |
| `ani_app_gitops` | Contains Helm templates for each infrastructure service per environment, managed by ArgoCD.      |
| `ani-iacc`       | Includes Kubernetes initialization scripts and Terragrunt configurations for environment setup. |
| `ani-api`        | Backend service repository.                                                                      |
| `ani-app`        | Frontend service repository.                                                                     |
| `ani-worker`     | Worker service repository.                                                                       |

---

## Infrastructure Services in Kubernetes

Several key infrastructure services are deployed within the Kubernetes cluster:

---

### cert-manager

`cert-manager` is responsible for issuing SSL/TLS certificates, including self-signed and Let's Encrypt certificates for your Kubernetes Ingress resources.

To issue a certificate for a new Ingress, include the following:

```yaml
metadata:
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  tls:
    - hosts:
        - <your.domain.com>
      secretName: <your-domain-com>-tls  # Replace dots with dashes
```

Once applied, `cert-manager` will automatically create a valid certificate for your specified DNS.


  
### external-dns

`external-dns` automates the creation and management of DNS records by syncing Kubernetes Ingress resources with external DNS providers, such as **Cloudflare**.

To enable `external-dns` for your Ingress resource, configure it as follows:

```yaml
metadata:
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod  # Optional: Enables TLS via cert-manager
spec:
  ingressClassName: external-nginx
  rules:
    - host: <your.domain.com>
```
This setup will instruct external-dns to:

- Automatically create the specified DNS record (<your.domain.com>) in the connected Cloudflare account.
- Ensure that DNS changes reflect updates in your Kubernetes Ingress objects.



### dex
- An identity service integrating with external authentication systems (e.g., Google, Azure, Email, etc.).
For common actions done on that infrastructure resource, you can face the situation when you need to create new static User for clients in order to test Production Environment

1. Modify `dex-extra.yaml` under `extra-manifests`.
2. Use `staticPasswords` (bcrypt hash with cost 10).
3. Apply and redeploy Dex.


---

## Node Architecture of Kubernetes

![Screenshot 2025-05-08 at 03 18 01](https://github.com/user-attachments/assets/776b907f-91e7-4464-a663-b7c1378a3986)

- **Staging Cluster (`aks-ani-staging`)** consists of two NodePools:
  - `default`: Statically configured with a predefined number of nodes.
  - `etljobs`: Scales automatically from 0 to 5 nodes based on memory/CPU requests.
- **Production Cluster (`aks-ani-prod`)** contains a single static NodeGroup (`aks-ani-prod`) where all services reside.

---

## Scaling Kubernetes Cluster

To scale the Kubernetes cluster:

1. Navigate to **Settings** → **NodePools** in the desired cluster.
2. Select the **default NodePool** and adjust the node count as needed.

![Screenshot 2025-05-08 at 02 55 29](https://github.com/user-attachments/assets/c9ee28a7-cf43-445e-aced-fc83252941a0)

Scaling may take a few minutes as new nodes are configured and added automatically. The `etljobs` NodePool scales automatically.

---

## Monitoring Services

We use the following monitoring services:

- **Grafana**: Monitoring platform.  
  Access: [https://grafana.{staging}.ani.tech](https://grafana.{staging}.ani.tech)
- **ELK (Elasticsearch, Logstash, Kibana)**: Centralized logging and visualization system.  
  Access: [https://logs.{staging}.ani.tech](https://logs.{staging}.ani.tech)

---

## Accessing Azure Portal

You can access Azure using the following link:  
[http://portal.azure.com](http://portal.azure.com)

---

## Accessing the Kubernetes Clusters

### 1. Ensure Required Permissions

Make sure you have the necessary access in **Entra ID** (formerly Azure Active Directory).  
Authenticate with Azure CLI:

```bash
az login
```

### 2. Retrieve kubeconfig Files

Use the Azure CLI to retrieve credentials and configure access:

**Production Cluster:**

```bash
az aks get-credentials --resource-group ani-prod --name aks-ani-prod
```

**Staging Cluster:**

```bash
az aks get-credentials --resource-group ani-staging --name aks-ani-staging
```

This updates your local `~/.kube/config` file.  
To deploy images locally, also log in to the Azure Container Registry:

```bash
az acr login -n acranishared
```

---

## CI/CD Process

Deployments are managed via **GitHub Actions** and **ArgoCD** (GitOps model).

### CI/CD Flow

1. **CI Files** (located in `.github/workflows`):
   - `staging.yaml` for Staging
   - `prod.yaml` for Production

2. **Pipeline Steps**:
   - Authenticate with Azure Container Registry (`acranishared`).
   - Build Docker images for services: `ani-api`, `ani-app`, `ani-websocket`, `ani-worker`.
   - Push images to the registry.

3. **Image Update and Deployment**:
   - `argocd-image-updater` monitors the registry and updates image tags in `ani_app_gitops`.
     - **Production tag format**: `${{ github.ref_name }}-${{ github.run_number }}-stable`
     - **Staging tag format**: `${{ secrets.ANI_ACR_LOGIN_SERVER }}/ani-app:${{ github.ref_name }}-${{ github.run_number }}`
   - ArgoCD detects changes and triggers a deployment.

---

## Developer Guide

### Deploying an Existing Service

1. Merge changes into the `staging` or `main` branch.
2. This triggers the corresponding GitHub Actions pipeline.

### Locating Environment Variables

- Stored in `ani_app_gitops` under: `{staging|prod}/{service_name}/values.yaml`
- Non-sensitive variables: `envs` key.
- Sensitive variables: `sealedSecrets` key (encrypted using Sealed Secrets).

### Creating a Sealed Secret

1. Install Kubeseal: [https://github.com/bitnami-labs/sealed-secrets#homebrew](https://github.com/bitnami-labs/sealed-secrets#homebrew)
2. Ensure access to Kubernetes and a valid context is configured.
3. Example:

```bash
echo -n bar | kubectl create secret generic mysecret --dry-run=client --from-file=foo=/dev/stdin -o json > mysecret.json
```

4. Commit `values.yaml` in the appropriate location in `ani_app_gitops`.
5. A redeploy is triggered and the new secret is applied.

### Viewing a Sealed Secret in Plain Format

1. Ensure correct Kubernetes context.
2. List sealed secrets:

   ![Screenshot](https://github.com/user-attachments/assets/f23adbfc-9fd5-49a8-98d0-603781c0dc92)

3. Download a secret:

```bash
kubectl get sealedsecret ani-api -n ani-api -o yaml > sealed-secret-v2.yaml
```

4. Retrieve the private key:

```bash
kubectl get secret -n sealed-secrets -l sealedsecrets.bitnami.com/sealed-secrets-key -o yaml > sealed-secrets-key.yaml
```

5. Decrypt:

```bash
kubeseal --controller-name=sealed-secrets --controller-namespace=sealed-secrets < sealed-secret-v2.yaml --recovery-unseal --recovery-private-key sealed-secrets-key.yaml -o yaml
```

6. Decode:

```bash
echo '{secret}' | base64 --decode
```

---

### Configuring Deployment for a New Service

1. Create and push a new repository.
2. Configure `Dockerfile` and GitHub Actions secrets.
3. Add a Helm chart to `ani_app_gitops`.
4. Create a `staging` branch to trigger deployment. Repeat for production.

---

### Scaling a Service

- **Temporary**: Use **Lens IDE** → **Deployments** → Adjust pod count.

  ![Screenshot](https://github.com/user-attachments/assets/0801ab79-640a-460e-a3ee-6c4049e8d355)

- **Persistent**: Update `replicaCount` in `values.yaml`. ArgoCD redeploys automatically.

  ![Screenshot](https://github.com/user-attachments/assets/58971fa0-a78f-46b2-b637-02e07c08f637)

---

### Rolling Back a Service

1. Open ArgoCD:
   - [Staging](https://argocd.staging.ani.tech/)
   - [Production](https://argocd.ani.tech/)
2. Go to **History & Rollback** on the service.
3. Select a version and click **Rollback**.

  ![Screenshot](https://github.com/user-attachments/assets/772b90a9-7b48-4b4f-9690-d381780fc14f)

---

### Troubleshooting ArgoCD Deployments

1. Go to ArgoCD, select your service.
2. View Kubernetes resource graph.
3. Click red-marked resources to view error messages.

  ![Screenshot](https://github.com/user-attachments/assets/e08db6ba-4531-487d-96af-c6da709505cf)

---

### Manual Syncing in ArgoCD

Click the **Sync** button if ArgoCD doesn’t detect changes.

  ![Screenshot](https://github.com/user-attachments/assets/6e6aee39-b64d-4e35-a80d-a981be8dea18)

---

### Viewing Container Logs

- Via Lens IDE: Select pods → Logs.
- For historical logs:  
  Go to [https://logs.staging.ani.tech](https://logs.staging.ani.tech), log in as `elastic`, open **Analyze** tab.

  Example query:

```text
kubernetes.labels.app_kubernetes_io/name : "ani-api"
```

  ![Screenshots](https://github.com/user-attachments/assets/2b9eea4e-f0f6-47ee-bc49-4f526ae9cf5a)
  ![Screenshots](https://github.com/user-attachments/assets/31e3c7b2-d2fc-4ecf-a1be-f0f925259e1c)

---

### Monitoring Kubernetes Cluster Resources

Access Grafana dashboards:

- **Staging**: [grafana.staging.ani.tech](https://grafana.staging.ani.tech)
- **Production**: [grafana.ani.tech](https://grafana.ani.tech)

Dashboards of interest:

- Kubernetes / Compute Resources / Namespace (Pods)
- Kubernetes / Compute Resources / Node (Pods)
- Kubernetes / Compute Resources / Pod
- Kubernetes / Networking / Namespace (Pods)

Use filters to refine metrics by namespace, pod, etc.

  ![Screenshot](https://github.com/user-attachments/assets/719636ba-9836-4bd7-bfcf-3616ee2f8096)

---

### Updating Infrastructure Services

1. Clone `ani-iacc`, navigate to `k8s-init`.
2. Create an `.env` file (refer to `.env.example`).
3. Generate SSH keys and update `.env`.
4. Run `k8s-init` and apply Helm chart.

For non-ArgoCD services, use `ani_gitops` to apply changes to `values.yaml`.

---

### How to add CronJobs?

If you need to add a new CronJob, go to the ani-app-gitops repository and select the appropriate environment and service. Then, open the values.yaml file for your service and look for the cronjobs key.

If the cronjobs key is not present, it means that templating for CronJobs has not been configured yet.

If the cronjobs key is present, you can add your new CronJob to the list under this key. The name of the CronJob should be used as the key—for example, in this case, etl-test-script-job is the name of the CronJob.
```bash
etl-test-script-job:
    schedule: "15 3 * * *" # schedule when job should run in cron format
    args:
      - /app/app/scripts/etl/cron_jobs/scripts/etl_test_script.py # which script to run for cronjob (it is path shown in container, to place script yous should go to repository of service and place your script into specified path, as here etl_test_script.py has been placed)
    backoffLimit: 0 # do not restart the job on failure
    restartPolicy: OnFailure # To restart on failure 
    nodeSelector: # Needed to place pod jobs into Specific seperate NodeGroup, do not touch it
      target: etl 
    tolerations: # Needed to place pod jobs into Specific seperate NodeGroup, do not touch it
      - key: "target"
        operator: "Equal"
        value: "etl"
        effect: "NoSchedule"
      - key: kubernetes.azure.com/scalesetpriority
        operator: Equal
        value: spot
        effect: NoSchedule
```


---

### How to get inside a pod 

There are two common methods to access a running pod for debugging or operational purposes: **kubectl CLI** and **Lens IDE**.

#### Option 1: Using Lens IDE (Recommended for UI-based workflows)

1. Open **Lens IDE** and navigate to the **Pods** section in the desired namespace.
2. Locate the specific pod you want to access.
3. Click on the pod to open its detail view.
4. Inside the pod details, locate the terminal icon (shell access symbol).
5. Click the icon to open an interactive terminal session attached to the pod’s container.
6. You can now execute commands inside the pod environment just as you would via `kubectl exec`.

<img width="995" alt="Screenshot 2025-05-09 at 01 59 40" src="https://github.com/user-attachments/assets/1c14ceb0-ffb4-423a-9538-7cb7ca76d6e4" />

#### Option 2: Using kubectl CLI

If you prefer CLI-based access or are working outside Lens:

```bash
kubectl exec -it <pod-name> -n <namespace> -- /bin/sh
```
---

### How to Access Databases?

Our current infrastructure includes the following primary database services:

- **Redis**: Deployed internally within the Kubernetes cluster.
- **MongoDB**: Hosted externally using MongoDB Atlas.

####  Accessing Redis (Kubernetes-Internal)

Redis is deployed as a Kubernetes-managed service. To access it, follow these steps:

1. Identify the running Redis pod in the relevant namespace using either Lens IDE or kubectl.

   kubectl get pods -n <namespace> | grep redis

2. Connect to the Redis pod using the terminal:

   - **Using Lens**: Select the Redis pod → click the terminal icon.
   - **Using CLI**:

     kubectl exec -it <redis-pod-name> -n <namespace> -- /bin/sh

3. Once inside the pod, authenticate using the Redis CLI:

   redis-cli -a <your_redis_password>

4. The Redis password is stored securely in your environment’s values.yaml file under sealedSecrets.  
   - To retrieve the password:
     - Locate the correct environment and service path in ani_app_gitops.
     - Decrypt the sealed secret using kubeseal with the private key.

> Ensure you're using the correct Kubernetes context and namespace before

#### Accessing MongoDB (Hosted on MongoDB Atlas)

MongoDB is not hosted within Kubernetes; instead, it is managed through MongoDB Atlas, providing cloud scalability and high availability.

- **Access Instructions**:  
  TODO 


### Accessing RabbitMQ Web UI

RabbitMQ provides a user-friendly Web UI for managing queues, exchanges, bindings, and message flow. It is deployed within Kubernetes and accessed through port-forwarding.

#### Steps to Access via Lens IDE:

1. Open Lens IDE and navigate to the Services section.
2. Locate the RabbitMQ service within the appropriate namespace.
3. Click the port-forward icon on the RabbitMQ service as shown in the screenshot:

   <img width="995" alt="Screenshot 2025-05-09 at 01 59 40" src="https://github.com/user-attachments/assets/7a72ea41-534e-4b14-943a-ed37e388d3bd" />

4. Forward port 15672, which is RabbitMQ’s default management UI port.

5. Open a web browser and navigate to:

   http://localhost:15672

6. Log in using the RabbitMQ username and password.

#### Retrieving RabbitMQ Credentials

The credentials for accessing the RabbitMQ Web UI are stored in the values.yaml file for the rmq service in the ani_git_ops repository.

- Direct link to configuration:  
  https://github.com/byobeta-team/ani_git_ops/blob/main/prod/rmq/values.yaml

> Make sure you have read access to the repository and your Kubernetes context is correctly configured if you need to modify or apply changes to the RabbitMQ deployment.
