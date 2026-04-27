# 🚀 Jenkins SSH Setup Guide
> Step-by-step guide to connect Jenkins to remote servers using SSH keys and trigger scripts/deployments — no passwords needed!

---

## 📋 Table of Contents
- [Part 0: Run Jenkins with Docker](#part-0-run-jenkins-with-docker)
- [Overview](#overview)
- [Part 1: Generate SSH Key Pair on Jenkins Server](#part-1-generate-ssh-key-pair-on-jenkins-server)
- [Part 2: Set Up Remote Server](#part-2-set-up-remote-server)
- [Part 3: Managing Multiple Servers](#part-3-managing-multiple-servers)
- [Part 4: Add SSH Credentials to Jenkins](#part-4-add-ssh-credentials-to-jenkins)
- [Part 5: Create the Jenkins Pipeline](#part-5-create-the-jenkins-pipeline)
- [Part 6: Pipeline for Multiple Servers](#part-6-pipeline-for-multiple-servers)
- [Quick Reference Checklist](#quick-reference-checklist)

---

## Part 0: Run Jenkins with Docker

### Prerequisites
- Docker installed
- Dockerfile built: `docker build -t myjenkins-blueocean:2.555.1-1 .`

### Step 1 — Create Docker Network

```bash
docker network create jenkins
```

---

### Step 2 — Run Docker-in-Docker (dind)

Allows Jenkins to run Docker commands inside pipelines.

```bash
docker run \
  --name jenkins-docker \
  --detach \
  --restart=unless-stopped \
  --privileged \
  --network jenkins \
  --network-alias docker \
  --env DOCKER_TLS_CERTDIR=/certs \
  --volume jenkins-docker-certs:/certs/client \
  --volume jenkins-data:/var/jenkins_home \
  docker:dind \
  --storage-driver overlay2
```

| Flag | Why |
|------|-----|
| `--restart=unless-stopped` | Auto-restarts after EC2/server reboot |
| `--privileged` | Required for Docker-in-Docker |
| `--network-alias docker` | Jenkins container reaches it via hostname `docker` |
| No `--publish 2376` | Docker daemon not exposed to host — unnecessary attack surface |

> ⚠️ Pull image first if needed: `docker image pull docker:dind`

---

### Step 3 — Run Jenkins

```bash
docker run \
  --name jenkins-blueocean \
  --detach \
  --restart=unless-stopped \
  --network jenkins \
  --env DOCKER_HOST=tcp://docker:2376 \
  --env DOCKER_CERT_PATH=/certs/client \
  --env DOCKER_TLS_VERIFY=1 \
  --publish 4000:8080 \
  --publish 50000:50000 \
  --volume jenkins-data:/var/jenkins_home \
  --volume jenkins-docker-certs:/certs/client:ro \
  myjenkins-blueocean:2.555.1-1
```

| Flag | Why |
|------|-----|
| `--restart=unless-stopped` | Auto-restarts after EC2/server reboot |
| `--publish 4000:8080` | Jenkins UI available at `http://YOUR_IP:4000` |
| `--publish 50000:50000` | Required if external Jenkins build agents connect to this controller |
| `jenkins-data` volume | All Jenkins config/jobs persist across container restarts |
| `jenkins-docker-certs` volume | TLS certs shared between dind and Jenkins |

> 💡 Port `50000` is only needed if you have remote inbound Jenkins agents. Skip `--publish 50000:50000` for standalone setups.

---

### Step 4 — Get Initial Admin Password

```bash
docker exec -it jenkins-blueocean cat /var/jenkins_home/secrets/initialAdminPassword
```

Access Jenkins at: `http://YOUR_SERVER_IP:4000`

---

### Surviving Server Reboots

Both containers use `--restart=unless-stopped`. On EC2 reboot:
- `jenkins-docker` (dind) restarts automatically ✅
- `jenkins-blueocean` restarts automatically ✅
- `jenkins-data` Docker volume persists — all jobs/config safe ✅

---

### Stop / Start

```bash
# Stop
docker stop jenkins-blueocean jenkins-docker

# Start
docker start jenkins-docker jenkins-blueocean
```

> ⚠️ Always start `jenkins-docker` (dind) **before** `jenkins-blueocean`.

---

## Overview

```
Jenkins Server                        Remote Server
(Local or Cloud)                      (Code lives here)
                                      
  🔑 Private Key          SSH ──►    🔒 authorized_keys
     (stays here)                       (public key stored here)
```

> **Rule:** Private key stays on Jenkins. Public key goes to every remote server.

---

## Part 1: Generate SSH Key Pair on Jenkins Server

Run these commands on your **Jenkins server terminal** (or local machine if Jenkins is local).

### Step 1 — Generate the SSH Key Pair

```bash
ssh-keygen -t rsa -b 4096

# Press Enter for all prompts:
# Enter file: (press Enter)
# Passphrase: (press Enter)
# Confirm:    (press Enter)
```

This creates **two files** inside `~/.ssh/`:

| File | Purpose |
|------|---------|
| `~/.ssh/id_rsa` | 🔴 **PRIVATE KEY** — stays on Jenkins, paste into Jenkins credentials |
| `~/.ssh/id_rsa.pub` | 🟢 **PUBLIC KEY** — copy to remote server's `authorized_keys` |

---

### Step 2 — View Your Private Key
> Copy this — you will paste it into Jenkins credentials later.

```bash
cat ~/.ssh/id_rsa
```

Output looks like:
```
-----BEGIN RSA PRIVATE KEY-----
MIIEowIBAAKCAQEA1234abcd....
.....many lines.....
-----END RSA PRIVATE KEY-----
```

> ⚠️ Copy the **ENTIRE** output including `-----BEGIN RSA PRIVATE KEY-----` and `-----END RSA PRIVATE KEY-----`

---

### Step 3 — View Your Public Key
> Copy this — you will paste it into the remote server later.

```bash
cat ~/.ssh/id_rsa.pub
```

Output looks like:
```
ssh-rsa AAAAB3NzaC1yc2E.... user@machine
```

---

## Part 2: Set Up Remote Server

Run these commands on your **remote server** (the server where your code lives).

### Step 1 — Create `.ssh` Folder and `authorized_keys` File
> Only needed if they don't exist yet.

```bash
mkdir -p ~/.ssh
touch ~/.ssh/authorized_keys
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

> ⚠️ `chmod 700` and `chmod 600` are **required** — SSH will refuse to work without correct permissions!

**What each command does:**

| Command | What it does |
|---------|-------------|
| `mkdir -p ~/.ssh` | Creates the `.ssh` folder (no error if already exists) |
| `touch ~/.ssh/authorized_keys` | Creates an empty file to store public keys |
| `chmod 700 ~/.ssh` | Only YOU can access the `.ssh` folder |
| `chmod 600 ~/.ssh/authorized_keys` | Only YOU can read/write the file |

---

### Step 2 — Add Jenkins Public Key to Remote Server

```bash
# Replace with your actual public key from Part 1 Step 3
echo "ssh-rsa AAAAB3NzaC1yc2E.... user@machine" >> ~/.ssh/authorized_keys
```

> ✅ Use `>>` (append) — NOT `>` (overwrite)!  
> `>>` adds a new line. `>` deletes everything and replaces it.

---

### Step 3 — Verify the Key Was Added

```bash
cat ~/.ssh/authorized_keys

# Should show your public key ✅
```

---

### Step 4 — Test SSH Connection
> From your **Jenkins server**, test that SSH works without a password.

```bash
ssh developer@YOUR_REMOTE_SERVER_IP

# Should log in WITHOUT asking for password ✅
```

> 💡 If it still asks for password — double check `chmod` permissions and that the public key was added correctly.

---

## Part 3: Managing Multiple Servers

One key pair on Jenkins can control **many servers** — just copy the public key to each server.

```bash
# On EACH remote server, run:
echo "YOUR_PUBLIC_KEY" >> ~/.ssh/authorized_keys

# Same public key → ALL servers
# One private key in Jenkins → controls them all ✅
```

> 💡 `authorized_keys` can hold **multiple public keys** — one per line. Each line = one machine allowed in!

**Example `authorized_keys` with multiple machines:**
```
ssh-rsa AAAAB3NzaC1yc2E.... jenkins@machineA
ssh-rsa AAAAB3NzaC1yc2E.... jenkins@machineB
```

### Key Management Strategy

| Scenario | Recommendation |
|----------|---------------|
| Same team, same owner servers | ONE key pair for all — simple! |
| Different teams, different owners | Separate key pairs per team — more secure |
| Moving to a new Jenkins machine | Generate new key pair, append new public key to all servers |
| Revoke a machine's access | Delete that machine's public key line from `authorized_keys` |

---

## Part 4: Add SSH Credentials to Jenkins

### Step 1 — Install SSH Agent Plugin

```
Jenkins Dashboard
  → Manage Jenkins
    → Plugins
      → Available Plugins
        → Search: "SSH Agent"
          → Install ✅
```

---

### Step 2 — Navigate to Credentials

```
Jenkins Dashboard
  → Manage Jenkins
    → Credentials
      → System
        → Global credentials (unrestricted)
          → + Add Credentials
```

---

### Step 3 — Fill in the Form

| Field | Value |
|-------|-------|
| **Kind** | SSH Username with private key |
| **Scope** | Global |
| **ID** | `remote-server-key` ← remember this, used in pipeline! |
| **Description** | Any label e.g. `My Remote Server` |
| **Username** | Your SSH username e.g. `developer` or `ubuntu` |
| **Private Key** | Click `Enter directly` → paste entire private key |
| **Passphrase** | Leave empty (unless you set one during keygen) |

> Click **Create** ✅

---

## Part 5: Create the Jenkins Pipeline

### Step 1 — Create a New Pipeline Job

```
Jenkins Dashboard
  → New Item
    → Enter name: my-deploy-pipeline
    → Select: Pipeline
    → Click OK
```

---

### Step 2 — Paste Your Jenkinsfile

Scroll down to the **Pipeline** section and paste:

```groovy
pipeline {
    agent any

    stages {

        stage('Deploy to Remote Server') {
            steps {
                sshagent(credentials: ['remote-server-key']) {
                    sh '''
                        ssh -o StrictHostKeyChecking=no developer@YOUR_SERVER_IP "
                            echo 'Connected to server!'
                            cd ~/your-project-folder
                            git pull origin main
                            bash deploy.sh
                        "
                    '''
                }
            }
        }

    }

    post {
        success {
            echo '✅ Deployment Done!'
        }
        failure {
            echo '❌ Deployment Failed - Check Console Output!'
        }
    }
}
```

> 💡 Replace:
> - `remote-server-key` → the ID you set in credentials
> - `developer@YOUR_SERVER_IP` → your actual username and IP
> - `~/your-project-folder` → your actual project path
> - `deploy.sh` → your actual script name

---

### Step 3 — Save and Run

```
→ Click Save
→ Click "Build Now" (left sidebar)
→ Click build number #1
→ Click "Console Output" to see live logs
```

---

### Step 4 — Expected Console Output

```
[ssh-agent] Using credentials: remote-server-key
$ ssh-agent
Identity added: /var/jenkins_home/.../private_key
[ssh-agent] Started.
+ ssh -o StrictHostKeyChecking=no developer@YOUR_SERVER_IP
  Connected to server!
  Already up to date.        <- git pull output
  Build started...
  Build successful!
✅ Deployment Done!
Finished: SUCCESS
```

---

## Part 6: Pipeline for Multiple Servers

To deploy to multiple servers in one pipeline, add multiple stages:

```groovy
pipeline {
    agent any

    stages {

        stage('Deploy to Server 1') {
            steps {
                sshagent(credentials: ['remote-server-key']) {
                    sh '''
                        ssh -o StrictHostKeyChecking=no developer@SERVER_1_IP "
                            cd /app && bash deploy.sh
                        "
                    '''
                }
            }
        }

        stage('Deploy to Server 2') {
            steps {
                sshagent(credentials: ['remote-server-key']) {
                    sh '''
                        ssh -o StrictHostKeyChecking=no developer@SERVER_2_IP "
                            cd /app && bash deploy.sh
                        "
                    '''
                }
            }
        }

        stage('Deploy to Server 3') {
            steps {
                sshagent(credentials: ['remote-server-key']) {
                    sh '''
                        ssh -o StrictHostKeyChecking=no developer@SERVER_3_IP "
                            cd /app && bash deploy.sh
                        "
                    '''
                }
            }
        }

    }
}
```

## Part 7: Sample scripts for reading logs.

```

pipeline {
    agent any

    parameters {
        choice(
            name: 'DURATION',
            choices: ['1', '2', '5'],
            description: 'How many minutes to tail logs'
        )
        choice(
            name: 'TARGET',
            choices: ['api_livekit', 'api_livekit_sip_dispatcher', 'both'],
            description: 'Which container to stream logs from'
        )
    }

    stages {
        stage('Live Logs') {
            parallel {

                stage('Control (api_livekit)') {
                    when {
                        expression {
                            params.TARGET == 'api_livekit' || params.TARGET == 'both'
                        }
                    }
                    steps {
                        sshagent(['livekit-agent-server-key']) {
                            sh """
                                ssh -o StrictHostKeyChecking=no developer@13.234.150.174 \
                                'timeout ${params.DURATION}m docker logs -f --tail=50 api_livekit || true'
                            """
                        }
                    }
                }

                stage('Dispatcher (api_livekit_sip_dispatcher)') {
                    when {
                        expression {
                            params.TARGET == 'api_livekit_sip_dispatcher' || params.TARGET == 'both'
                        }
                    }
                    steps {
                        sshagent(['livekit-agent-server-key']) {
                            sh """
                                ssh -o StrictHostKeyChecking=no developer@13.234.150.174 \
                                'timeout ${params.DURATION}m docker logs -f --tail=50 api_livekit_sip_dispatcher || true'
                            """
                        }
                    }
                }

            }
        }
    }

    post {
        success {
            echo "✅ Log streaming completed after ${params.DURATION} minute(s)"
        }
        failure {
            echo "❌ Log streaming failed — check SSH credentials or container name"
        }
        aborted {
            echo "⚠️ Log streaming was manually stopped"
        }
    }
}

```

---

## Quick Reference Checklist

Use this every time you set up a new Jenkins → Server connection:

- [ ] Generate SSH key pair on Jenkins server (`ssh-keygen -t rsa -b 4096`)
- [ ] Create `~/.ssh` folder and `authorized_keys` on remote server
- [ ] Set correct permissions (`chmod 700 ~/.ssh` and `chmod 600 ~/.ssh/authorized_keys`)
- [ ] Copy public key (`id_rsa.pub`) into remote server's `authorized_keys` using `>>`
- [ ] Test SSH connection manually from Jenkins to remote server (no password = success ✅)
- [ ] Install **SSH Agent** plugin in Jenkins
- [ ] Add private key (`id_rsa`) to Jenkins Credentials with a memorable ID
- [ ] Create Pipeline job and use `sshagent(credentials: ['your-id'])` in Jenkinsfile
- [ ] Run build and check **Console Output** for logs

---

## Security Tips

> 🔐 **Never share your private key** with anyone.  
> 🔐 Only copy/share the **public key** (`id_rsa.pub`) to servers.  
> 🔐 For production servers — disable password login completely:

```bash
# On remote server
sudo nano /etc/ssh/sshd_config

# Find and change:
PasswordAuthentication yes  →  PasswordAuthentication no

# Restart SSH
sudo systemctl restart sshd
```

After this — only machines with the correct private key can connect. Password login is blocked. ✅

---

*Jenkins SSH Setup Guide — Save this for future reference!*