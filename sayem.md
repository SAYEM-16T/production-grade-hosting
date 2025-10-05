# Setting up a Kubernetes Cluster with Kind and ArgoCD on AWS EC2

This guide provides a step-by-step walkthrough of how to set up a Kubernetes cluster on an AWS EC2 instance using `kind`, and how to install and expose the ArgoCD UI.

## Prerequisites

*   An AWS account.
*   An SSH key pair generated in your AWS account.
*   Basic knowledge of using the AWS console and shell commands.

## Step 1: EC2 Instance Setup

1.  **Launch a new EC2 instance:**
    *   **Name:** `kubernates-cluster`
    *   **AMI:** Ubuntu Server 24.04 LTS (HVM), SSD Volume Type
    *   **Instance type:** `c7i-flex.large` (or any other suitable instance type)
    *   **Key pair:** Select the key pair you created.
2.  **Network settings:**
    *   Create a new security group.
    *   Allow SSH traffic from your IP address (or anywhere, for testing purposes).
    *   Allow HTTP and HTTPS traffic from the internet.
3.  **Storage:**
    *   Configure the root volume with at least 15 GiB of storage.
4.  **Launch the instance.**

## Step 2: Connect to the EC2 Instance

1.  Copy the public DNS of your EC2 instance.
2.  Use the following command to connect to your instance via SSH:

    ```bash
    chmod 400 "your-key.pem"
    ssh -i "your-key.pem" ubuntu@your-ec2-public-dns
    ```

## Step 3: Install Docker

Once connected to the instance, install Docker with the following command:

```bash
sudo apt-get update
sudo apt-get install -y docker.io
```

## Step 4: Install and Configure a 3-Node Kubernetes Cluster with Kind

1.  **Create a directory for the installation scripts:**

    ```bash
    mkdir k8s-install
    cd k8s-install
    ```

2.  **Create an installation script for `kind`:**

    ```bash
    nano install_kind.sh
    ```

    Add the following content to the script:

    ```bash
    #!/bin/bash
    # For AMD64 / x86_64
    [ $(uname -m) = x86_64 ] && curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.20.0/kind-linux-amd64
    # For ARM64
    [ $(uname -m) = aarch64 ] && curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.20.0/kind-linux-arm64
    chmod +x ./kind
    sudo mv ./kind /usr/local/bin/kind
    ```

    Make the script executable and run it:

    ```bash
    chmod +x install_kind.sh
    ./install_kind.sh
    ```

3.  **Create a configuration file for the `kind` cluster:**

    ```bash
    nano config.yml
    ```

    Add the following content to the file to create a cluster with one control-plane node and two worker nodes:

    ```yaml
    kind: Cluster
    apiVersion: kind.x-k8s.io/v1alpha4
    nodes:
    - role: control-plane
    - role: worker
    - role: worker
    ```

4.  **Create the cluster:**

    ```bash
    kind create cluster --config=config.yml --name=my-cluster
    ```

## Step 5: Install kubectl

1.  **Create an installation script for `kubectl`:**

    ```bash
    nano install_kubectl.sh
    ```

    Add the following content to the script:

    ```bash
    #!/bin/bash
    curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
    chmod +x kubectl
    sudo mv kubectl /usr/local/bin/
    ```

    Make the script executable and run it:

    ```bash
    chmod +x install_kubectl.sh
    ./install_kubectl.sh
    ```

2.  **Verify the installation and check the nodes:**

    ```bash
    kubectl get nodes
    ```

## Step 6: Install ArgoCD

1.  **Create a namespace for ArgoCD:**

    ```bash
    kubectl create namespace argocd
    ```

2.  **Apply the ArgoCD installation manifests:**

    ```bash
    kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
    ```

## Step 7: Accessing the ArgoCD UI

1.  **Expose the ArgoCD server using a NodePort service:**

    ```bash
    kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "NodePort"}}'
    ```

2.  **Get the NodePort for the `argocd-server` service:**

    ```bash
    kubectl get svc -n argocd argocd-server
    ```

    Look for the port mapping for port 443. It will be something like `443:3XXXX/TCP`.

3.  **Forward the port to make it accessible from your local machine:**

    ```bash
    kubectl port-forward -n argocd svc/argocd-server 8443:443 --address=0.0.0.0
    ```

4.  **Access the ArgoCD UI** by navigating to `https://<your-ec2-public-ip>:8443` in your browser.

5.  **Get the initial admin password:**

    ```bash
    kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d && echo
    ```

    Use `admin` as the username and the password you just retrieved to log in.

## Troubleshooting

If you are unable to connect to the ArgoCD UI, it might be because of multiple `kubectl port-forward` processes running.

1.  **Find the process IDs (PIDs) of the `kubectl port-forward` processes:**

    ```bash
    ps aux | grep kubectl
    ```

2.  **Kill the processes:**

    ```bash
    sudo kill -9 <PID1> <PID2>
    ```

3.  **Restart the port-forwarding with the correct command:**

    ```bash
    kubectl port-forward -n argocd svc/argocd-server 8443:443 --address=0.0.0.0
    ```




