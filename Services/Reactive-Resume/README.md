# Reactive Resume on Kubernetes

This repository provides a comprehensive guide and Kubernetes manifests to deploy [Reactive Resume](https://rxresu.me/) onto a Kubernetes cluster. This setup leverages essential cloud-native components like Traefik as an Ingress Controller, Cert-Manager for automatic TLS certificates, and MinIO for object storage, ensuring a robust, secure, and scalable environment for your resume builder.

## Table of Contents

1.  [Introduction](#introduction)
2.  [Features](#features)
3.  [Prerequisites](#prerequisites)
4.  [Architecture Diagram](#architecture-diagram)
5.  [Deployment Steps](#deployment-steps)
    * [Step 1: Clone this Repository](#step-1-clone-this-repository)
    * [Step 2: Generate and Configure Secrets](#step-2-generate-and-configure-secrets)
    * [Step 3: Install Traefik Ingress Controller](#step-3-install-traefik-ingress-controller)
    * [Step 4: Install Cert-Manager](#step-4-install-cert-manager)
    * [Step 5: Create Reactive Resume Namespace](#step-5-create-reactive-resume-namespace)
    * [Step 6: Apply Core Application Components](#step-6-apply-core-application-components)
    * [Step 7: Initial MinIO Setup (Create Bucket)](#step-7-initial-minio-setup-create-bucket)
    * [Step 8: Verify Deployment](#step-8-verify-deployment)
    * [Step 9: Access Reactive Resume](#step-9-access-reactive-resume)
6.  [Troubleshooting](#troubleshooting)
7.  [Customization](#customization)
8.  [Why Deployments (and why StatefulSets are often preferred)](#why-deployments-and-why-statefulsets-are-often-preferred)
9.  [Contributing](#contributing)
10. [License](#license)

---

## 1. Introduction

This guide walks you through deploying Reactive Resume, a free and open-source resume builder, to your Kubernetes cluster. It provides all the necessary Kubernetes YAML manifests for the application and its dependencies, including:

* **Reactive Resume Application:** The main web application.
* **PostgreSQL:** The database for Reactive Resume.
* **MinIO:** An S3-compatible object storage server for storing uploaded images and generated PDFs.
* **Browserless:** A headless Chrome instance used by Reactive Resume for PDF generation.
* **Traefik:** As your Ingress Controller to expose services externally and handle TLS.
* **Cert-Manager:** For automated TLS certificate provisioning from Let's Encrypt.

## 2. Features

* Full Reactive Resume functionality, including PDF generation.
* Automated HTTPS with Let's Encrypt certificates.
* Persistent storage for database and MinIO data.
* Modular Kubernetes manifests for easy management.

## 3. Prerequisites

Before you begin, ensure you have the following:

* **A running Kubernetes Cluster:** This guide assumes a basic Kubernetes cluster is already set up (e.g., K3s, MicroK8s, EKS, GKE, AKS).
* **`kubectl`:** Configured to connect to your cluster.
* **`helm`:** Package manager for Kubernetes (version 3+ recommended).
* **`openssl`:** For generating secure random strings for secrets.
* **Domain Names:** Two fully qualified domain names (FQDNs) pointing to your cluster's external IP address or LoadBalancer.
    * `resumebuilder.your_domain.com` (for Reactive Resume app)
    * `minio.your_domain.com` (for MinIO S3 API)
    * **DNS Provider:** Access to your domain's DNS settings (e.g., Cloudflare is highly recommended for `DNS01` challenge with Cert-Manager).
* **`mc` client (MinIO Client):** Used for initial MinIO bucket creation. [Installation Guide](https://min.io/docs/minio/linux/reference/minio-mc/mc-install.html)

## 4. Architecture Diagram

*(Link to your diagram image here. For example: `![Architecture Diagram](images/architecture-diagram.png)`)`*

---

## 5. Deployment Steps

### Step 1: Clone this Repository

```bash
git clone [https://github.com/your-username/reactive-resume-k8s-guide.git](https://github.com/your-username/reactive-resume-k8s-guide.git) # Replace with your repo URL
cd reactive-resume-k8s-guide
````

### Step 2: Generate and Configure Secrets

Sensitive information like database passwords, API secrets, and access keys should be stored securely as Kubernetes Secrets.

**Important:** Replace all placeholder values (`YOUR_...`) with strong, randomly generated strings.

1.  **Open the main manifest file:** `reactive-resume-secrets.yaml`.
2.  **Generate secrets for `postgres-secrets`:**
      * `POSTGRES_PASSWORD`:
        ```bash
        openssl rand -base64 24 | tr -d '\n' ; echo
        ```
      * Update `POSTGRES_DB_URL` in the secret: `postgresql://postgres:YOUR_POSTGRES_PASSWORD@postgres:5432/postgres`.
3.  **Generate secrets for `minio-secrets`:**
      * `MINIO_ROOT_PASSWORD`: (Use a strong password, default is `minioadmin` but change it)
        ```bash
        openssl rand -base64 24 | tr -d '\n' ; echo
        ```
4.  **Generate secrets for `reactive-resume-app-secret`:**
      * `APP_SECRET`:
        ```bash
        openssl rand -base64 32 | tr -d '\n' ; echo
        ```
      * `ACCESS_TOKEN_SECRET`:
        ```bash
        openssl rand -base64 32 | tr -d '\n' ; echo
        ```
      * `REFRESH_TOKEN_SECRET`:
        ```bash
        openssl rand -base64 32 | tr -d '\n' ; echo
        ```
      * `CHROME_TOKEN`: (This is used by Browserless for authentication)
        ```bash
        openssl rand -hex 32 | tr -d '\n' ; echo # Hex is often preferred for tokens
        ```
      * Update `STORAGE_URL`: Ensure `https://minio.your_domain.com/reactive-resume-files` is correct.
      * Update `MAIL_FROM`: `noreply@resumebuilder.your_domain.com`
5.  **Optional: Configure `google-oauth-secrets` (if using Google Login):**
      * Uncomment the `google-oauth-secrets` block.
      * Obtain `GOOGLE_CLIENT_ID` and `GOOGLE_CLIENT_SECRET` from Google Cloud Console API -> Credentials -> Create Credentials.
      * Update `GOOGLE_CALLBACK_URL` in the `app` Deployment environment variables: `https://resumebuilder.your_domain.com/api/auth/google/callback`.


### Step 3: Install Traefik Ingress Controller

Traefik will handle incoming requests and route them to your services. It also manages TLS termination.

1.  **Add Traefik Helm repository:**

    ```bash
    helm repo add traefik [https://traefik.github.io/charts](https://traefik.github.io/charts)
    helm repo update
    ```

2.  **Install Traefik:**
    **Crucial:** We need to configure Traefik to watch for Kubernetes Custom Resources (like Middlewares) in *all* namespaces. This is done by adding `--providers.kubernetescrd.namespaces=` to its arguments.

    ```bash
    helm install traefik traefik/traefik \
      --namespace kube-system \
      --set providers.kubernetesIngress.enabled=true \
      --set providers.kubernetesCRD.enabled=true \
      --set "providers.kubernetesCRD.namespaces={}" \
      --set "service.annotations.helm\\.sh/hook"="post-install,post-upgrade" \
      --set "service.type=LoadBalancer" \
      --set "ports.web.exposedPort=80" \
      --set "ports.websecure.exposedPort=443" \
      --wait # Wait for Traefik to be ready
    ```

    *Adjust `kube-system` if you install Traefik in a different namespace.*

3.  **Get Traefik's External IP:**

    ```bash
    kubectl get svc -n kube-system traefik -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
    ```

    Take this IP and configure your DNS `A` records for `resumebuilder.your_domain.com` and `minio.your_domain.com` to point to it.

### Step 4: Install Cert-Manager

Cert-Manager automatically provisions and manages TLS certificates using Let's Encrypt.

1.  **Add Cert-Manager Helm repository:**
    ```bash
    helm repo add jetstack [https://charts.jetstack.io](https://charts.jetstack.io)
    helm repo update
    ```
2.  **Install Cert-Manager CRDs (required first):**
    ```bash
    kubectl apply -f [https://github.com/cert-manager/cert-manager/releases/download/v1.14.5/cert-manager.crds.yaml](https://github.com/cert-manager/cert-manager/releases/download/v1.14.5/cert-manager.crds.yaml)
    ```
3.  **Install Cert-Manager itself:**
    ```bash
    helm install cert-manager jetstack/cert-manager \
      --namespace cert-manager \
      --create-namespace \
      --version v1.14.5 \
      --set installCRDs=false \
      --wait 
    ```
4.  **Create a ClusterIssuer (for Let's Encrypt with Cloudflare DNS01):**
    Create a file named `cluster-issuer.yaml`:
    ```yaml
    apiVersion: cert-manager.io/v1
    kind: ClusterIssuer
    metadata:
      name: letsencrypt-prod-cloudflare 
      namespace: reactive-resume # Can be in any namespace, ClusterIssuer is cluster-scoped
    spec:
      acme:
        email: your-email@example.com # <<< CHANGE THIS 
        server: [https://acme-v02.api.letsencrypt.org/directory](https://acme-v02.api.letsencrypt.org/directory)
        privateKeySecretRef:
          name: letsencrypt-prod-cloudflare-account-key
        solvers:
          - dns01:
              cloudflare:
                email: your-cloudflare-email@example.com # <<< CHANGE THIS 
                apiTokenSecretRef:
                  name: cloudflare-api-token-secret # Name of the secret holding your CF API token
                  key: api-token # Key within that secret
    ```
    **Create the Cloudflare API Token Secret:**
      * Generate a Cloudflare API Token with "Edit DNS" permissions for your zone.
      * Create the secret:
        ```bash
        kubectl create secret generic cloudflare-api-token-secret \
          --namespace reactive-resume \
          --from-literal=api-token='YOUR_CLOUDFLARE_API_TOKEN' # <<< CHANGE THIS
        ```
    **Apply the ClusterIssuer:**
    ```bash
    kubectl apply -f cluster-issuer.yaml
    ```
    **Verify ClusterIssuer status:**
    ```bash
    kubectl get clusterissuer letsencrypt-prod-cloudflare -o yaml
    ```
    Ensure `Ready: True` is displayed in its status conditions.

### Step 5: Create Reactive Resume Namespace

All Reactive Resume components will live in this dedicated namespace.

```bash
kubectl create namespace reactive-resume
```
### Step 6: Create respective Yamls in your cluster
Ensure all 6 yamls are present.
### Step 7: Apply these in specified order
```bash
kubectl apply -f namespace.yaml
kubectl apply -f secrets.yaml
kubectl apply -f postgers.yaml
kubectl apply -f minio.yaml
kubectl apply -f Reactive-resume-pdf-generator.yaml
kubectl apply -f Reactive-resume-app.yaml
```

### Step 8: Verify Deployment

Monitor all components to ensure they are healthy.

```bash
# Watch all pods in the reactive-resume namespace
kubectl get pods -n reactive-resume -w

# Check deployments
kubectl get deployments -n reactive-resume

# Check services
kubectl get services -n reactive-resume

# Check ingresses and their external IPs
kubectl get ingresses -n reactive-resume

# Check certificate status (look for Ready: True)
kubectl get certificate -n reactive-resume minio-tls-secret
kubectl get certificate -n reactive-resume reactive-resume-tls-secret

# Check application logs for errors
kubectl logs -f $(kubectl get pods -n reactive-resume -l app=reactive-resume -o jsonpath='{.items[0].metadata.name}') -n reactive-resume
kubectl logs -f $(kubectl get pods -n reactive-resume -l app=minio -o jsonpath='{.items[0].metadata.name}') -n reactive-resume
kubectl logs -f $(kubectl get pods -n reactive-resume -l app=reactive-resume-pdf-generator -o jsonpath='{.items[0].metadata.name}') -n reactive-resume
```

### Step 9: Access Reactive Resume

Once all pods are `Running` and `Ready`, and Ingresses have their external IPs:

  * Open your web browser and navigate to: `https://resumebuilder.your_domain.com`

You should see the Reactive Resume application. Try generating a PDF to test MinIO storage and Browserless integration.

-----

## 6\. Troubleshooting

  * **`Pending` pods:** Check `kubectl describe pod <pod-name>` for `Events` (e.g., `FailedScheduling` due to resource limits, `ImagePullBackOff` for image pull issues).
  * **`CrashLoopBackOff` pods:** Check `kubectl logs <pod-name>` to see application errors (e.g., `STORAGE_URL undefined`, `EPROTO` SSL errors).
  * **`404 Not Found` for domain:**
      * Check `dig +short your_domain.com` to confirm DNS points to cluster's external IP.
      * Check Traefik logs: `kubectl logs <traefik-pod> -n <traefik-namespace> | grep your_domain.com` for routing errors.
      * Verify `kubectl get ingress` shows correct host, path, and service.
      * Remove Cloudflare proxy from minio as it sometimes interferes with PDF download.
  * **TLS (`HTTPS`) errors / Certificate Not Ready:**
      * Check `kubectl describe certificate <secret-name>` for Cert-Manager logs/events.
      * Verify `ClusterIssuer` status.
      * Ensure Cloudflare API token secret is correct if using `DNS01`.
      * Confirm port 443 is open to your cluster.
  * **PDF Generation Issues:**
      * Check logs for `reactive-resume-app` and `reactive-resume-pdf-generator` for errors during generation or MinIO communication.
      * Verify MinIO bucket `reactive-resume-files` exists using `mc ls myminio`.

-----

## 7\. Customization

  * **Domains:** Update `PUBLIC_URL`, `NEXTAUTH_URL` in `app` Deployment, and `host` in Ingresses.
  * **Passwords/Secrets:** Generate new random strings for all secrets.
  * **Resource Limits:** Add `resources.limits` and `requests` to Deployments for production stability.
  * **SMTP Email:** Uncomment and configure `SMTP_URL` environment variable in `app` Deployment.
  * **OAuth (GitHub, Google, OpenID):** Uncomment relevant sections in `app` Deployment and `google-oauth-secrets` if applicable.

-----

## 8\. Why Deployments (and why StatefulSets are often preferred)

In this guide, we used standard `Deployments` for PostgreSQL and MinIO, which are typically considered stateful applications. This was done for simplicity and to focus on getting the core application running with a single replica for each backend service.

However, for **production environments** requiring high availability, data consistency, and robust scaling, `StatefulSets` are generally the preferred.

**Advantages of StatefulSets:**

  * **Stable, Unique Network Identifiers:** Each pod in a StatefulSet gets a predictable, sticky hostname (e.g., `postgres-0`, `postgres-1`), which is crucial for distributed databases and quorum-based systems.
  * **Ordered, Graceful Deployment and Scaling:** Pods are started, updated, and deleted in a defined order (e.g., `0` then `1` then `2`). This prevents data corruption or service disruption during scaling or rolling updates in clustered environments.
  * **Stable, Persistent Storage per Replica:** Each pod has its own dedicated `PersistentVolumeClaim` that follows it even if the pod restarts or is rescheduled to a different node.

**When to consider StatefulSets:**

  * **PostgreSQL:** For a highly available PostgreSQL cluster, you would use a StatefulSet.
  * **MinIO:** For a distributed, highly available MinIO object storage cluster, you would deploy it as a StatefulSet or use the which manages the underlying StatefulSet.

While the provided `Deployments` with `PersistentVolumeClaims` work for a single instance.

-----

## 9\. Contributing

Feel free to open issues or submit pull requests if you have suggestions, improvements, or find any bugs.

-----

## 10\. License

This project is open-source and available under the [MIT License](https://www.google.com/search?q=LICENSE).

```
```