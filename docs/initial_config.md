
# 📄 Initial Cluster Configuration Guide

This document provides comprehensive, step-by-step instructions for the initial setup of your K3s distributed homelab cluster, covering the master and first worker node, secure inter-node communication via WireGuard, and essential Kubernetes services like Rancher and Cert-Manager.

## 🚀 Introduction

This guide walks you through the foundational steps to get your multi-node K3s cluster up and running.
* A K3s master node running on Oracle Cloud.
* A K3s worker node running on a Proxmox VM at home.
* Secure WireGuard VPN tunnels enabling communication between these nodes.
* Rancher installed for centralized cluster management.
* Cert-Manager configured with Cloudflare for automated SSL certificate provisioning.

## 🏗️ Overall Architecture

The cluster currently consists of:

* **Master Node:** An Oracle Cloud Infrastructure (OCI) Virtual Machine. This node runs the K3s control plane, Traefik, and Rancher.
* **Worker Node:** A Virtual Machine hosted on a local Proxmox server(can be any vm anywhere) at home. This node joins the K3s cluster as a worker.

**Secure Inter-Node Communication:** WireGuard VPN tunnels are established between all cluster nodes, providing a secure and encrypted network overlay for K3s's Flannel CNI.

**Planned Expansion:**
* An additional worker node in Pune (VM).
* An edge worker node on a OnePlus Nord running Ubuntu Touch.
* Persistent storage integration (e.g., using local storage for Jellyfin).

## 📋 Prerequisites

* **Two Virtual Machines (or physical machines):**
    * One for the **Master Node** (e.g., Oracle Cloud VM with Ubuntu 24.04 LTS).
    * One for the **Worker Node** (e.g., Proxmox VM with Ubuntu 24.04 LTS).
* **SSH Access:** To both machines with `sudo` privileges.
* **Domain Name:** For Rancher access and your applications (e.g., `rancher.vishvesh.me`, `vishvesh.me`).
* **Cloudflare Account & API Token:** For Cert-Manager DNS01 challenge.
* **Basic Networking Knowledge:** Understanding of IPs, subnets, firewalls, and VPNs.
* **OCI Security List Access:** To configure ingress rules for your Oracle VM.

## ⚙️ Step 1: Master Node Setup (Oracle Cloud Instance)

### 1.1 Provision Oracle Cloud VM

* Provision an Ubuntu 24.04 LTS VM in Oracle Cloud. Ensure it has a public IP address and configure its VCN Security List to allow SSH (TCP 22) from your IP.
* Note down its **Private IP** (e.g., `10.0.0.80`).

### 1.2 Install K3s with Rancher

K3s includes Traefik as its default Ingress Controller, we will install rancher along with it.

**On Oracle Cloud Master Node:**

```bash
# --tls-san is crucial for adding your Rancher domain to K3s's certificates
curl -sfL [https://get.k3s.io](https://get.k3s.io) | INSTALL_K3S_EXEC="server --tls-san rancher.vishvesh.me" sh -


sudo systemctl status k3s # this will show the status of k3s ensure enabled and running is present in green 

# Check nodes, will display only one for now
kubectl get nodes

# Get K3s token (save for later) 
sudo cat /var/lib/rancher/k3s/server/node-token
```
**Keep the `node-token` safe! we'll need it for joining worker nodes.**

### 1.3 Initial Rancher Access and SSL Certificate

1.  **Access Rancher (Temporary):**
    Initially, Rancher might be accessible via `https://<Master-Node-Public-IP>`. Accept any certificate warnings for now.
2.  **Configure DNS for Rancher:**
    Create an `A` record for your Rancher domain (e.g., `rancher.vishvesh.me`) pointing to the **public IP address of your Oracle Cloud Master Node**. will take some time for DNS propagation.
3.  **Get Rancher Admin Password:**
    ```bash
    sudo kubectl get secret --namespace cattle-system bootstrap-secret -o go-template='{{.data.password|base64decode}}{{"\n"}}'
    ```
    Log in to Rancher using `admin` as the username and the retrieved password.
4.  **Configure Cert-Manager & Cloudflare ClusterIssuer:**

    * **Clean Up Previous Cert-Manager Installation : (we will install jetstack) **
        ```bash
        # 1. Delete the cert-manager Helm release
        helm uninstall cert-manager --namespace cert-manager

        # 2. Wait for resources to be deleted (optional, but good if errors occur)
        echo "Waiting for cert-manager resources to terminate..."
        sleep 10

        # 3. Delete Cert-Manager's Custom Resource Definitions (CRDs) - USE WITH CAUTION if other apps depend on cert-manager CRDs
        # This command is aggressive and removes all cert-manager related CRDs.
        # Make sure no other applications are using cert-manager CRDs before running.
        kubectl delete crd certificates.cert-manager.io \
                           certificaterequests.cert-manager.io \
                           challenges.acme.cert-manager.io \
                           clusterissuers.cert-manager.io \
                           issuers.cert-manager.io \
                           orders.acme.cert-manager.io

        # 4. Delete the cert-manager namespace (this will remove remaining resources)
        kubectl delete namespace cert-manager

        # 5. Recreate the namespace
        kubectl create namespace cert-manager
        ```

    * **Install Cert-Manager:**
        ```bash
        helm repo add jetstack [https://charts.jetstack.io](https://charts.jetstack.io)
        helm repo update
        helm install \
          cert-manager jetstack/cert-manager \
          --namespace cert-manager \
          --version v1.13.0 \
          --create-namespace \
          --set installCRDs=true
        ```
        )

    * **Create a Cloudflare API Token:**
        * Go to Cloudflare dashboard -> My Profile -> API Tokens -> Create Token.
        * Choose "Create Custom Token".
        * **Permissions:**
            * Zone -> Zone -> Read
            * Zone -> DNS -> Edit
        * **Zone Resources:** Specific zone (your domain, e.g., `vishvesh.me`).
        * Save the token securely.

    * **Create a Kubernetes Secret for Cloudflare API Token:**
        ```bash
        kubectl create secret generic cloudflare-api-token-secret \
          --from-literal=api-token=<YOUR_CLOUDFLARE_API_TOKEN> \
          --namespace cert-manager
        ```

    * **Create the ClusterIssuer:**
    * ```vi clusterissuer.yaml```
        ```yaml
        apiVersion: cert-manager.io/v1
        kind: ClusterIssuer
        metadata:
          name: letsencrypt-prod-cloudflare # Name for your ClusterIssuer
        spec:
          acme:
            email: your-email@example.com # <--- IMPORTANT: Change to your email
            server: [https://acme-v02.api.letsencrypt.org/directory](https://acme-v02.api.letsencrypt.org/directory)
            privateKeySecretRef:
              name: letsencrypt-prod-cloudflare-pk # Secret to store the ACME account private key
            solvers:
            - dns01:
                cloudflare:
                  apiTokenSecretRef:
                    name: cloudflare-api-token-secret
                    key: api-token
        ```
      esc + :wq
      Apply it: `kubectl apply -f clusterissuer.yaml`

### 1.4 Install & Configure WireGuard (OCI Master)

WireGuard provides the secure tunnel for inter-node communication.

**On your Oracle Cloud Master Node:**

```bash
# Install WireGuard
sudo apt update
sudo apt install wireguard -y

# Generate server keys
wg genkey | sudo tee /etc/wireguard/privatekey_server
sudo chmod 600 /etc/wireguard/privatekey_server
sudo cat /etc/wireguard/privatekey_server | wg pubkey | sudo tee /etc/wireguard/publickey_server

# Create wg0.conf for the master node
# Adjust 'enp0s6' according to your VM's primary network 
sudo nano /etc/wireguard/wg0.conf
```
Paste the following into `/etc/wireguard/wg0.conf`:

```ini
[Interface]
PrivateKey = <Private key of the server> # Paste content of /etc/wireguard/privatekey_server
Address = 172.16.16.2/24 # Master's WireGuard IP within the VPN subnet
ListenPort = 51822 # CRUCIAL: Must be open in OCI Security List
# MTU = 1420 # Common optimal MTU for VXLAN over WireGuard
PostUp = iptables -A FORWARD -i %i -j ACCEPT; iptables -A FORWARD -o %i -j ACCEPT; iptables -t nat -A POSTROUTING -o enp0s6 -j MASQUERADE
PreDown = iptables -D FORWARD -i %i -i %i -j ACCEPT; iptables -D FORWARD -o %i -j ACCEPT; iptables -t nat -D POSTROUTING -o enp0s6 -j MASQUERADE

[Peer]
PublicKey = <Public key of Home Node Worker> # This will be generated in Step 2.1
Endpoint = 192.168.5.1:51822 # Worker's internal IP : WireGuard ListenPort
AllowedIPs = 172.16.16.0/24, 192.168.5.0/24, 10.42.0.0/16 # VPN subnet, Worker's internal subnet, K3s Pod CIDR
PersistentKeepalive = 25 # Helps maintain connection through NATs
```
**Save and exit.**

```bash
# Enable and start WireGuard
sudo systemctl enable wg-quick@wg0
sudo systemctl start wg-quick@wg0
sudo wg show # Verify status
```

### 1.5 Configure OCI Security List (Ingress Rule)

**Ensure the following Ingress Rule** is added to the Security List attached to your Oracle Cloud VM's Virtual Cloud Network (VCN) subnet:

* **Source Type:** CIDR
* **Source CIDR:** `0.0.0.0/0` (or restrict to your home/VPN IP if known)
* **IP Protocol:** UDP
* **Destination Port Range:** `51822` (for WireGuard)
* **IP Protocol:** TCP
* **Destination Port Range:** `6443` (K3s API), `10250` (Kubelet)
* **IP Protocol:** UDP
* **Destination Port Range:** `8472` (Flannel VXLAN - though K3s handles this, explicitly allowing is safer)

## ⚙️ Step 2: Worker Node Setup (Home Proxmox VM)

### 2.1 Provision Proxmox VM & Install WireGuard

* Provision an Ubuntu 24.04 LTS VM in Proxmox. Ensure it has a static internal IP address (e.g., `192.168.5.1`).

* **On your Home Proxmox Worker Node (SSH into it):**

    ```bash
    # Install WireGuard
    sudo apt update
    sudo apt install wireguard -y

    # Generate worker keys
    wg genkey | sudo tee /etc/wireguard/privatekey_worker
    sudo chmod 600 /etc/wireguard/privatekey_worker
    sudo cat /etc/wireguard/privatekey_worker | wg pubkey | sudo tee /etc/wireguard/publickey_worker

    # Create wg0.conf for the worker node
    # IMPORTANT: Adjust 'br0' if your Proxmox VM's primary network interface (bridge) is different
    sudo nano /etc/wireguard/wg0.conf
    ```
    Paste the following into `/etc/wireguard/wg0.conf`:

    ```ini
    [Interface]
    PrivateKey = <Private key of node> # Paste content of /etc/wireguard/privatekey_worker
    Address = 172.16.16.1/24 # Worker's WireGuard IP within the VPN subnet
    ListenPort = 51822
    MTU = 1420 # Use a consistent MTU with the master node
    PostUp = iptables -A FORWARD -i %i -j ACCEPT; iptables -A FORWARD -o %i -j ACCEPT; iptables -t nat -A POSTROUTING -o br0 -j MASQUERADE
    PreDown = iptables -D FORWARD -i %i -j ACCEPT; iptables -D FORWARD -o %i -j ACCEPT; iptables -t nat -D POSTROUTING -o br0 -j MASQUERADE

    [Peer]
    PublicKey = <Public key of OCI server> # Paste content of /etc/wireguard/publickey_server from master node
    Endpoint = <Public IP of OCI Master Node>:51822 # e.g., 1.2.3.4:51822
    AllowedIPs = 172.16.16.0/24, 10.0.0.80/32, 10.42.0.0/16 # VPN subnet, Master's internal IP, K3s Pod CIDR
    PersistentKeepalive = 25
    ```
    **Save and exit.**

    ```bash
    # Enable and start WireGuard
    sudo systemctl enable wg-quick@wg0
    sudo systemctl start wg-quick@wg0
    sudo wg show # Verify status
    ```

### 2.2 Configure Master WireGuard Peer

Now that you have the worker's public key, go back to your **Oracle Cloud Master Node** and edit `/etc/wireguard/wg0.conf` to add the worker as a peer:

```bash
sudo nano /etc/wireguard/wg0.conf
```
Add the following `[Peer]` section at the end:

```ini
# Add this section for the Home Node Worker
[Peer]
PublicKey = <Public key of Home Node Worker> # Paste content of /etc/wireguard/publickey_worker
Endpoint = 192.168.5.1:51822 # Worker's internal IP : WireGuard ListenPort
AllowedIPs = 172.16.16.0/24, 192.168.5.0/24, 10.42.0.0/16 # VPN subnet, Worker's internal subnet, K3s Pod CIDR
PersistentKeepalive = 25
```
**Save and exit.**

```bash
# Restart WireGuard on the master node to apply changes
sudo systemctl restart wg-quick@wg0
sudo wg show # Verify peer connection
```

### 2.3 Join K3s Cluster as Worker

**On your Home Proxmox Worker Node:**

```bash
# Get the K3s server URL (master's WireGuard IP or internal IP)
# K3S_URL=[https://172.16.16.2:6443](https://172.16.16.2:6443) # Using WireGuard IP
K3S_URL=[https://10.0.0.80:6443](https://10.0.0.80:6443) # Using Master's Internal IP (if WireGuard not working fully yet)

# Get the K3s node token from your master node (if you didn't save it)
# K3S_TOKEN=$(sudo cat /var/lib/rancher/k3s/server/node-token)

# Install K3s as a worker agent
# Replace <YOUR_K3S_NODE_TOKEN> with the token obtained in Step 1.2
curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION="v1.32.5+k3s1" K3S_URL=https://<master node url>:6443 K3S_TOKEN="<node token>" sh -

# Verify K3s agent status
sudo systemctl status k3s-agent
# Now do 
kubectl get nodes
#if you get something like this "E0612 11:27:57.915326   26117 memcache.go:265] "Unhandled Error" err="couldn't get current server API group list: Get \"http://localhost:8080/api?timeout=32s\": dial tcp [::1]:8080: connect: connection refused"
#The connection to the server localhost:8080 was refused - did you specify the right host or port?"
#then go to master node 
cat ~/.kube/config #master node
#copy the config and change the 0.0.0.0:6443 to <ip of master>:6443
#next on worker node
mkdir -p ~/.kube
vi ~/.kube/config
#paste the config and change server: <master node ip>:6443 & esc + :wq 
export KUBECONFIG=~/.kube/config
chmod 600 ~/.kube/config
echo 'export KUBECONFIG=~/.kube/config' >> ~/.bashrc #makes persistant 
source ~/.bashrc # Apply changes immediately
#now try on worker node
kubectl get nodes
#you shoudld see something like this
root@vishvesh-server-2:~# kubectl get nodes
NAME                     STATUS     ROLES                  AGE    VERSION
instance-20250530-2005   Ready      control-plane,master   5d4h   v1.32.5+k3s1
vishvesh-server-2        Ready      <none>                 64m    v1.32.5+k3s1
vishveshserver           Ready      worker                 4d7h   v1.32.5+k3s1 

```

## ✅ Verification

**On your Oracle Cloud Master Node:**

1.  **Check WireGuard tunnel status:**
    ```bash
    sudo wg show
    ```
    You should see your worker node as a `peer` with `latest handshake` time, indicating active connection.

2.  **Check K3s Nodes:**
    ```bash
    kubectl get nodes -o wide
    ```
    Both `instance-20250530-2005` (master) and `vishveshserver` (worker) should show `Ready` status. If `vishveshserver` is `Ready,SchedulingDisabled`, you can enable scheduling:
    ```bash
    kubectl uncordon vishveshserver
    ```

3.  **Test Inter-Pod Communication:**

    * Get the IP of a pod on the worker node (e.g., your `portfolio-website` pod):
        ```bash
        kubectl get pod -l app.kubernetes.io/name=portfolio-website -o jsonpath='{.items[0].status.podIP}'
        # Let's say it returns 10.42.2.15
        ```
    * Get a shell into a pod on your master node (e.g., `future-city-dash` or a simple `busybox` pod if you don't have one):
        ```bash
        kubectl exec -it future-city-dash-dash-chart-dc96d6846-qjz44 -- bash
        # If no other pod, create a temporary busybox pod for testing
        # kubectl run -it --rm --restart=Never busybox --image=busybox -- sh
        ```
    * **From inside the master node's pod**, `curl` the worker node's pod IP:
        ```bash
        curl [http://10.42.2.15:3000](http://10.42.2.15:3000) # Use the actual pod IP and port
        ```
        This *must* return HTML from your portfolio website. If it hangs, your Flannel/WireGuard tunnel is still not passing traffic correctly.

## 🚀 Future Enhancements (Planned)

* **Persistent Storage:** Implement a distributed storage solution (e.g., Longhorn, Rook-Ceph) or local persistent volumes (PVs) using K3s's Local Path Provisioner, especially for integrating `Jellyfin` with local hard drives on the Proxmox node.

* **Additional Worker Nodes:** Expand the cluster with the Pune VM and the OnePlus Nord edge device, configuring WireGuard for each.

* **Monitoring:** Set up Prometheus and Grafana for comprehensive cluster and application monitoring.

* **GitOps:** Implement Flux CD or Argo CD for automated, declarative deployments.

## ⚠️ Important Notes & Troubleshooting

* **Firewalls:** Ensure `ufw`, `firewalld`, or `iptables` on *all* nodes (master and workers) allow traffic on WireGuard's `ListenPort` (UDP 51822), K3s API (TCP 6443), Kubelet (TCP 10250), and Flannel VXLAN (UDP 8472).

* **MTU:** Inconsistent MTU settings between WireGuard and Flannel can cause packet fragmentation and network issues. Ensure your WireGuard MTU is appropriate (e.g., 1420 for VXLAN over WireGuard) and test it.

* **IP Addresses:** Double-check all IP addresses (private, public, WireGuard subnet) in your configurations.

* **`PostUp`/`PreDown` Interfaces:** The network interfaces (`enp0s6`, `br0`) in your WireGuard config are specific to your environment. Adjust them if your VM's network interfaces are named differently.

* **K3s Token Security:** Never commit your K3s node token or WireGuard private keys to a public Git repository!
```