# Harbor Private Registry — Air-Gapped Installation Guide (RHEL 10)

This guide covers the full setup of a Harbor private container registry on an air-gapped RHEL 10 VM using Docker CE and self-signed TLS certificates.

---

## 1. Environment Details

| Component        | Details                              |
|------------------|--------------------------------------|
| Hypervisor       | VMware Workstation                   |
| OS               | RHEL 10                              |
| Docker Version   | 29.x                                 |
| Harbor Version   | v2.14.2 (Offline Installer)          |
| Registry URL     | https://192.168.88.129               |
| Network Type     | Air-Gapped (No Internet)             |

---

## 2. Resources & Official Download Links

### RHEL ISO
- Official Download: [https://access.redhat.com/downloads](https://access.redhat.com/downloads) *(Requires Red Hat account)*
- Documentation: [https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/)

### Docker Engine RPMs
- Official Docker repository: [https://docs.docker.com/engine/install/rhel/](https://docs.docker.com/engine/install/rhel/)
- Download RPM packages: [https://download.docker.com/linux/rhel/](https://download.docker.com/linux/rhel/)

Packages required:
- `docker-ce`
- `docker-ce-cli`
- `containerd.io`
- `docker-compose-plugin`

### Harbor Offline Installer
- Official Harbor GitHub: [https://github.com/goharbor/harbor](https://github.com/goharbor/harbor)
- Releases page: [https://github.com/goharbor/harbor/releases](https://github.com/goharbor/harbor/releases)
- Downloaded file: `harbor-offline-installer-v2.14.2.tgz`
- Official Harbor Docs: [https://goharbor.io/docs/](https://goharbor.io/docs/)

### OpenSSL Documentation
- [https://www.openssl.org/docs/](https://www.openssl.org/docs/)

---

## 3. Architecture Overview

```
Internet Machine
├── Pull Docker images        (docker pull)
├── Download RPMs             (Docker CE packages)
└── Transfer files via SCP   ──────────────────────┐
                                                    ▼
                                          Harbor VM (Air-Gapped)
                                          ├── Docker Engine
                                          ├── Harbor Core
                                          ├── Harbor Registry
                                          ├── Harbor DB
                                          ├── Harbor Redis
                                          └── Harbor Nginx (HTTPS)
```

**Workflow:**

```
Internet → docker pull → docker save → scp → Harbor VM → docker load → docker push → Harbor Registry
```

---

## 4. VM Creation (VMware Workstation)

1. Create a new VM in VMware Workstation
2. Attach the RHEL ISO
3. Assign the following resources:
   - **RAM:** 8 GB
   - **CPUs:** 4
   - **Disk:** 120 GB
4. Set **Network Adapter** to: `Host-Only`
5. Complete RHEL installation and boot into the OS

---

## 5. Prerequisites Checklist

Before starting, ensure the following are ready on your **internet-connected machine**:

- [ ] RHEL 10 ISO downloaded and VM created
- [ ] Harbor offline installer: `harbor-offline-installer-v2.14.2.tgz`
- [ ] Docker CE RPM packages downloaded:
  - `docker-ce`
  - `docker-ce-cli`
  - `containerd.io`
  - `docker-compose-plugin`
- [ ] SCP access from host to VM confirmed

---

## 6. VM Network Setup

After VM creation, get your IP address and configure a static IP.

```bash
ip a
nmcli show connection
```

Set static IP (replace `ens160` with your actual interface name):

```bash
sudo nmcli connection modify ens160 \
  ipv4.method manual \
  ipv4.addresses 192.168.88.129/24 \
  ipv4.gateway "" \
  ipv4.dns ""
```

Restart the connection:

```bash
sudo nmcli connection down ens160
sudo nmcli connection up ens160
```

Verify:

```bash
ip a
ip route
```

### Set Hostname and Hosts Entry

```bash
sudo vi /etc/hosts
```

Add the following line:

```
192.168.88.129  harbor harbor
```

Then set the hostname:

```bash
hostnamectl set-hostname harbor
```

---

## 7. Install Docker CE (Offline)

> ⚠️ RHEL 9 does not ship Docker by default. Harbor does not install Docker. You must install Docker CE manually.

### Transfer RPMs to the VM

From your internet-connected machine, download RPMs from:  
👉 [https://download.docker.com/linux/rhel/9/x86_64/stable/Packages/](https://download.docker.com/linux/rhel/9/x86_64/stable/Packages/)

SCP them to the VM:

```bash
scp *.rpm root@192.168.88.129:/home/sumit/
```

### Install Docker Using DNF

> ⚠️ Use `dnf`, not `rpm -ivh` — DNF resolves dependencies correctly.

```bash
cd /home/sumit
sudo dnf install *.rpm --nogpgcheck -y
```

### Enable and Start Docker

```bash
sudo systemctl enable docker
sudo systemctl start docker
```

### Verify Docker Installation

```bash
docker --version
docker compose version
systemctl status docker
```

---

## 8. Generate TLS Certificates

### Create Certificate Directory

```bash
sudo mkdir -p /data/cert
cd /data/cert
```

### Step 1 — Create Internal CA

```bash
sudo openssl genrsa -out ca.key 4096

sudo openssl req -x509 -new -nodes -sha512 -days 3650 \
  -subj "/C=IN/ST=MH/L=Mumbai/O=AirGapLab/OU=IT/CN=harbor-ca" \
  -key ca.key \
  -out ca.crt
```

### Step 2 — Create Harbor Server Key

```bash
sudo openssl genrsa -out harbor.key 4096
```

### Step 3 — Create SAN Config File

```bash
sudo vi harbor.cnf
```

Paste the following content:

```ini
[req]
default_bits = 4096
prompt = no
default_md = sha512
req_extensions = req_ext
distinguished_name = dn

[dn]
C = IN
ST = MH
L = Mumbai
O = AirGapLab
OU = IT
CN = 192.168.88.129

[req_ext]
subjectAltName = @alt_names

[alt_names]
DNS.1 = harbor
DNS.2 = harbor
IP.1 = 192.168.88.129
```

### Step 4 — Generate CSR

```bash
sudo openssl req -new -key harbor.key \
  -out harbor.csr \
  -config harbor.cnf
```

### Step 5 — Sign Certificate with CA

```bash
sudo openssl x509 -req -sha512 -days 3650 \
  -extfile harbor.cnf \
  -extensions req_ext \
  -CA ca.crt -CAkey ca.key -CAcreateserial \
  -in harbor.csr \
  -out harbor.local.crt
```

### Step 6 — Verify Certificate

```bash
openssl x509 -in harbor.crt -text -noout
```

### Step 7 — Trust the CA Locally (Required for Docker)

```bash
sudo cp ca.crt /etc/pki/ca-trust/source/anchors/
sudo update-ca-trust
```

---

## 9. Install Harbor

### Transfer the Installer to the VM

```bash
scp harbor-offline-installer-v2.x.x.tgz root@192.168.88.129:/home/sumit/
```

### Extract and Move

```bash
mv harbor-offline-installer-*.tgz /opt/
cd /opt
tar -xvf harbor-offline-installer-*.tgz
cd harbor
```

---

## 10. Configure Harbor

Edit `harbor.yml`:

```bash
vi harbor.yml
```

Make the following changes:

```yaml
hostname: 192.168.88.129

# Comment out http:
# http:
#   port: 80

https:
  port: 443
  certificate: /data/cert/harbor.crt
  private_key: /data/cert/harbor.key
```

### Verify Certificate Files Exist

```bash
file /data/cert/harbor.crt
file /data/cert/harbor.key
```

Expected output:
```
/data/cert/harbor.crt: PEM certificate
/data/cert/harbor.key: PEM RSA private key
```

---

## 11. Run Harbor Installation

```bash
cd /opt/harbor
./prepare
./install.sh
```

### Verify Containers Are Running

```bash
docker ps
```

---

## 12. Access Harbor UI

Open your browser and navigate to:

```
https://192.168.88.129
```

> ⚠️ Since you're using a self-signed certificate, the browser will show "Connection not private". Click **Advanced → Proceed to 192.168.88.129**.

**Default credentials:**

| Field    | Value        |
|----------|--------------|
| Username | `admin`      |
| Password | `Harbor12345` |

---

## 13. Create a Project

1. Log in to Harbor UI
2. Go to **Projects → New Project**
3. Name it `myproject`
4. Set visibility to **Public** (for testing)

---

## 14. Configure Docker to Trust Harbor

Since Harbor uses a self-signed certificate, configure Docker to allow it:

```bash
vi /etc/docker/daemon.json
```

Add:

```json
{
  "insecure-registries": ["192.168.88.129"]
}
```

Restart Docker:

```bash
systemctl restart docker
```

### Copy Certificate for Docker Trust

```bash
rm -rf /etc/docker/certs.d/192.168.88.129
mkdir -p /etc/docker/certs.d/192.168.88.129
cp /data/cert/harbor.crt /etc/docker/certs.d/192.168.88.129/ca.crt
```

Restart Docker again:

```bash
systemctl restart docker
```

Then bring Harbor back up (restarting Docker stops all Harbor containers):

```bash
cd /opt/harbor
docker compose up -d
```

Verify Harbor containers are running:

```bash
docker ps
```

---

## 15. Pull, Transfer, and Push an Image to Harbor

Since the VM is air-gapped, you cannot pull images directly on it. The workflow is:

**Host machine (internet) → Save as `.tar` → SCP to VM → Load → Tag → Push to Harbor**

---

### Step 1 — Pull the Image on Your Host Machine

On your **internet-connected laptop/host**:

```bash
docker pull nginx:latest
```

### Step 2 — Save the Image as a TAR File

```bash
docker save -o nginx-latest.tar nginx:latest
```

### Step 3 — Transfer the TAR to the VM

```bash
scp nginx-latest.tar root@192.168.88.129:/home/sumit/
```

---

### Step 4 — Load the Image on the VM

On the **VM**:

```bash
docker load -i /home/sumit/nginx-latest.tar
```

Verify the image is loaded:

```bash
docker images
```

---

### Step 5 — Login to Harbor

```bash
docker login 192.168.88.129
```

Enter credentials when prompted:

| Field    | Value         |
|----------|---------------|
| Username | `admin`       |
| Password | `Harbor12345` |

---

### Step 6 — Tag the Image for Harbor

Use the format: `<harbor-ip>/<project>/<image>:<tag>`

```bash
docker tag nginx:latest 192.168.88.129/myproject/nginx:latest
```

### Step 7 — Push the Image to Harbor

```bash
docker push 192.168.88.129/myproject/nginx:latest
```

---

### Step 8 — Verify in Harbor UI

Open your browser and navigate to:

```
https://192.168.88.129
```

Go to: **Projects → myproject → Repositories**

You should see `nginx` listed with the `latest` tag.

---

> 💡 **Tip — Transferring Multiple Images at Once**
>
> You can save multiple images into a single TAR file:
>
> ```bash
> docker save -o images-bundle.tar nginx:latest redis:7 postgres:15
> ```
>
> Then load them all in one command on the VM:
>
> ```bash
> docker load -i /home/sumit/images-bundle.tar
> ```

---

## Troubleshooting

### Error: `x509: certificate signed by unknown authority`

This happens when Docker doesn't trust the Harbor certificate (e.g., after reinstalling Harbor with a new cert).

**Fix:**

```bash
# Remove old cert directory
rm -rf /etc/docker/certs.d/192.168.88.129

# Recreate and copy new cert
mkdir -p /etc/docker/certs.d/192.168.88.129
cp /data/cert/harbor.crt /etc/docker/certs.d/192.168.88.129/ca.crt

# Restart Docker
systemctl restart docker

# Bring Harbor back up
cd /opt/harbor
docker compose up -d

# Try login again
docker login 192.168.88.129
```

---

## Air-Gap Notes

In air-gapped environments:
- Never pull images from Docker Hub directly
- Import images manually to the VM
- Use Harbor as your internal Docker Hub
- Tag images using the format: `<harbor-ip>/<project>/<image>:<tag>`
