# High Availability Vault Cluster with Auto-Unseal & Snapshot Agent

This repository contains Ansible playbooks and roles to deploy a production-ready, High Availability (HA) HashiCorp Vault cluster. The deployment includes a Transit Vault for auto-unsealing, Raft integrated storage, automated snapshots to S3 (MinIO/AWS), and a Load Balancer entrypoint.

## 🏗 Architecture Overview

The infrastructure consists of four logical components:

1. **Main Vault Cluster (3 Nodes):**
* Uses **Raft (Integrated Storage)** for HA.
* Configured with **Auto-Unseal** via the Transit node (no manual unsealing required after restarts).
* TLS enabled for internal communication.


2. **Transit Vault (1 Node):**
* A standalone Vault instance solely responsible for holding the unseal keys of the Main Cluster.
* Automatically initialized and configured with the `autounseal` policy.


3. **Snapshot Agent (Sidecar):**
* A lightweight container deployed alongside Vault nodes.
* Uses **AppRole** authentication to connect to Vault.
* Periodically takes snapshots and uploads them to an S3-compatible backend (e.g., MinIO, AWS S3).


4. **Load Balancer (2 Nodes - Optional but Recommended):**
* **HAProxy**: Terminates SSL (Port 443) and forwards traffic to the active Vault leader (Port 8200).
* **Keepalived**: Provides a Virtual IP (VIP) to ensure the Load Balancer itself is highly available.



---

## 📋 Prerequisites

1. **Ansible Installed** (Control Node).
2. **Target Servers**:
* 3x VMs for Main Vault Cluster.
* 1x VM for Transit Vault.
* 2x VMs for Load Balancers (Optional).


3. **SSH Access**: Passwordless SSH access to all target nodes (configured in `ansible.cfg`).
4. **S3 Storage Details**: Endpoint, Bucket, Access Key, and Secret Key for backups.

---

## ⚙️ Configuration

### 1. Inventory Setup

Edit `inventory.ini` to match your server IPs and node IDs.

```ini
[vault_nodes]
vault1 ansible_host=192.168.13.211 node_id=vault1
vault2 ansible_host=192.168.13.212 node_id=vault2
vault3 ansible_host=192.168.13.213 node_id=vault3

[transit_node]
vault-transit ansible_host=192.168.13.214 node_id=vault-transit

[loadbalancers]
LB1 ansible_host=192.168.13.215 keepalived_state=MASTER keepalived_priority=100
LB2 ansible_host=192.168.13.216 keepalived_state=BACKUP keepalived_priority=90

[loadbalancers:vars]
vip_address=192.168.13.217
vip_interface=eth0

```

### 2. Global Variables

Edit `group_vars/all.yml` to configure S3 settings and Vault addresses.

```yaml
# S3 Configuration for Snapshots
backup_s3_endpoint: "https://s3.example.local"
backup_s3_bucket: "vault-cluster-snapshots"
backup_s3_access_key: "admin"
backup_s3_secret_key: "SUPERSECRET"

# Snapshot Schedule
snapshot_frequency: "10m"
snapshot_retain: 3

```

---

## 🚀 Deployment Guide

Follow these steps in order to bring up the entire stack.

### Step 1: Generate Self-Signed Certificates

Generates a CA-signed certificate including all Vault Node IPs and the Transit Node IP in the SAN (Subject Alternative Names).

```bash
ansible-playbook playbooks/certificates/create.yml

```

*Output:* Creates `local_certs/tls.crt` and `local_certs/tls.key`.

### Step 2: Distribute Certificates

Uploads the generated certificates to all Vault nodes (Main + Transit).

```bash
ansible-playbook playbooks/certificates/deploy.yml

```

### Step 3: Deploy Vault Infrastructure

This is the core playbook. It performs the following sequence:

1. Installs Docker on all nodes.
2. **Sets up Transit Node**: Starts Vault, initializes it, enables the `transit` engine, creates the `autounseal` key, and generates a client token.
3. **Sets up Main Cluster**: Configures `vault.hcl` to use the Transit token for auto-unseal.
4. **Initializes Main Cluster**: Automatically initializes the first node (`vault1`) and saves the recovery keys locally.

```bash
ansible-playbook playbooks/vault/deploy.yml

```

> **Note:**
> * Transit Keys are saved to: `keys/transit_keys.json`
> * Transit Token is saved to: `keys/transit_token.txt`
> * Main Cluster Recovery Keys are saved to: `keys/main_cluster_keys.json`
> 
> 

### Step 4: Deploy Snapshot Agent

Configures AppRole authentication and deploys the snapshot agent container.

1. Creates `snapshot-agent` policy and AppRole in Vault (both Main and Transit).
2. Fetches `RoleID` and `SecretID`.
3. Deploys the Docker container with S3 credentials injected.

```bash
ansible-playbook playbooks/snapshot-agent/deploy.yml

```

### Step 5: Deploy Load Balancers (Recommended)

Deploys HAProxy and Keepalived. Ensure you have placed your external SSL certificates in the `lb_cert/` directory named `tls.crt` and `tls.key` before running this.

```bash
# Ensure external certs exist
# cp /path/to/your/domain.crt lb_cert/tls.crt
# cp /path/to/your/domain.key lb_cert/tls.key

ansible-playbook playbooks/load-balancers/deploy.yml

```

---

## 🔍 Verification & Usage

### Accessing Vault

If you deployed the Load Balancer, access Vault via the VIP:

* **URL:** `https://192.168.13.217` (or your configured VIP)

If accessing nodes directly (Main Cluster):

* `https://192.168.13.211:8200`

### Checking Cluster Health

Log into `vault1` or any main node:

```bash
export VAULT_ADDR='https://127.0.0.1:8200'
export VAULT_SKIP_VERIFY=true

# Check Status (Should show "Sealed: false" and "Recovery Seal Type: transit")
docker exec vault vault status

# List Peers
docker exec vault vault operator raft list-peers

```

### Checking Snapshots

Check the logs of the snapshot agent container:

```bash
docker logs vault-snapshot-agent

```

*You should see logs indicating successful upload to your S3 bucket.*

---

## 🛠 Maintenance & Operations

### Rolling Restart

To restart nodes one by one without downtime (updates container configurations):

**Main Cluster:**

```bash
ansible-playbook playbooks/vault/restart.yml

```

**Load Balancers:**

```bash
ansible-playbook playbooks/load-balancers/restart.yml

```

### Destroying the Cluster

**⚠️ DANGER:** This will wipe all data, volumes, and containers.

```bash
# Destroy Main and Transit Vaults
ansible-playbook playbooks/vault/destroy.yml

# Destroy Load Balancers
ansible-playbook playbooks/load-balancers/destroy.yml

```

---

## 🔐 Security Best Practices

1. **Secure the `keys/` directory**: This folder contains root tokens and unseal keys. **Do not commit this folder to Git**.
2. **S3 Bucket Access**: Ensure the IAM user or S3 credentials used for snapshots have **write-only** or limited permissions if possible.
3. **TLS**: The current setup generates self-signed certificates for internal node communication. For production, replace `local_certs/` with certificates signed by a trusted internal CA.
