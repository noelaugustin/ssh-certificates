# Hardened SSH Certificate Authority Infrastructure

This repository implements a production-ready, SSH-native Certificate Authority (CA) infrastructure. It utilizes a Dual-Root CA architecture, protocol isolation via dual SSH processes, and principal-based identity mapping to eliminate password reliance and Host Key (TOFU) prompts.

## Architecture Overview

The system consists of three primary components:

### 1. CA Server (`ca-server`)
The trusted authority. It runs two independent `sshd` processes:
*   **Issuance Port (22)**: Dedicated to signing certificates.
    *   **Auth**: Password only (`username=password`).
    *   **Restricted**: Users are locked into issuance scripts via `ForceCommand`.
    *   **Harden**: Only presents a CA-signed host certificate.
*   **Management Port (2222)**: Dedicated to administrative maintenance.
    *   **Auth**: Public Key only (using `user_identity`).
    *   **Access**: Full shell access to the container.

### 2. Target Server (`target-server`)
The managed compute node.
*   **Native Trust**: Trust for the `user_ca.pub` is pre-baked.
*   **Host Identity**: Automatically generates a host key and requests a Host Certificate from the CA during boot.
*   **Role Mapping**: Maps certificate principals to local accounts (`npc` -> standard, `mc` -> sudo, `god` -> root).

### 3. Client (`client`)
The user workstation.
*   **Global Trust**: Pre-configured to trust the `host_ca.pub` for all domains, ensuring no "Unknown Host" prompts.

---

## Security Model

*   **Dual-Root CA**: Separate Root CAs are used for `Host` verification (so clients trust servers) and `User` verification (so servers trust users).
*   **Privileged Port Assertion**: Host certificates are only signed if the request originates from a privileged port (< 1024), preventing unprivileged users from impersonating servers.
*   **Short-Lived Identities**: User certificates are typically issued with a 1-hour TTL, enforcing frequent re-authentication.
*   **Zero-Interaction Trust**: By using `@cert-authority` in `known_hosts`, the entire fleet is trusted natively without manual fingerprint confirmation.

---

## Getting Started

### 1. Initialize Infrastructure Keys
Run the setup script to generate your Root CAs and administrative identities. These keys are ignored by Git for security.
```bash
./setup-keys.sh
```

### 2. Launch the Infrastructure
```bash
docker compose up -d --build
```

---

## Manual Verification Guide

Follow these steps to verify the infrastructure without relying on automated demo scripts.

### 1. Verify Host Trust
From the `client` container, attempt to connect to the CA. Since trust is pre-baked globally, you should be prompted for a password **immediately** without many "Host identity not established" warnings.
```bash
docker exec -it client ssh -p 22 npc@ca-server
```
*(Accept the password 'npc' if you see the prompt. It will immediately close the connection as issuance is a ForceCommand, but the lack of a host warning proves trust is working.)*

### 2. Issue a User Certificate Manually
You must use an SSH Agent because the issuance script adds the certificate directly to your active session.

```bash
# Enter the client container
docker exec -it client bash

# 1. Start an agent and add your base identity
eval $(ssh-agent)

# 2. Connect to the CA Port 22 with Agent Forwarding (-A)
# Use 'npc' as both the username and password
SSHPASS=npc sshpass -e ssh -p 22 -A npc@ca-server

# 3. Verify the certificate has been added to your agent
ssh-add -L
```
*You should see a line starting with `ssh-ed25519-cert-v01@openssh.com...`.*

### 3. Access the Target Server
Using the certificate you just issued, log into the target server. Because of principal mapping, you do not need to manage `authorized_keys` on the target.
```bash
ssh npc@target-server
```
*You should be logged in instantly without a password.*

### 4. Verify Identity Escalation (Sudo)
Repeat the issuance process for the `mc` (Main Character) user and verify sudo access.
```bash
# Request MC certificate
SSHPASS=mc sshpass -e ssh -p 22 -A mc@ca-server

# Login and test sudo
ssh mc@target-server "sudo whoami"
# Should return 'root'
```

### 5. Verify CA Management Port (2222)
Verify that you can access the CA's management interface using your pre-baked `user_identity` key (skipping passwords).
```bash
ssh -i /home/user/.ssh/id_ed25519 -p 2222 npc@ca-server
```
*This should drop you into a shell on the CA server.*

---

## Cleanup
```bash
docker compose down --volumes
```
