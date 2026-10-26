# CYB 601 – Starter Repository (GitHub scaffold)

Also, Available at `https://github.com/PGS-CYB/cyb601-template` .

This is the `cyb601-<lastname>` template students fork in Week 0. Create a new GitHub repository from these files, mark it a template, and add the instructor as a collaborator. Files marked TODO are the ones you finalize for your Proxmox environment before term.

## How to use this scaffold

Each section below is one file in the repository. Copy its contents into the path shown in the heading. When every file is in place:

1. Create a new GitHub repository named `cyb601-template`, private, owned by the course org.
2. Commit these files, then in Settings mark the repository as a **template**.
3. In Week 0, students choose "Use this template" to create their own `cyb601-<lastname>`, add you as a collaborator, and work in it all term.

**What you finalize before term** (every TODO):

- `infra/terraform.tfvars.example`: your Proxmox node name, template VM ID, pool and bridge.
- `app/`: the customer API image (the base ships a stub) and your mock customer dataset.
- `.github/workflows/ci.yaml`: the Cosign identity string for your org, and the registry path.

**Two planted weaknesses** stay in the base on purpose, so the labs have something to find:

- An over-permissive ServiceAccount binding in `app/api/` (students fix it in Lab 3).
- A plaintext database password in `app/data/` (gitleaks finds it in Lab 4).

## Repository tree

```
cyb601-<lastname>/
├─ README.md
├─ .gitignore
├─ infra/                     # Week 1: provision the VM and cluster
│  ├─ versions.tf
│  ├─ variables.tf
│  ├─ main.tf
│  ├─ cloud-init.yaml.tftpl
│  └─ terraform.tfvars.example
├─ app/                       # the reference system (Kustomize base)
│  ├─ kustomization.yaml
│  ├─ namespaces.yaml
│  ├─ shop/                   # OWASP Juice Shop
│  │  └─ deployment.yaml
│  ├─ api/                    # customer API + planted RBAC weakness
│  │  ├─ deployment.yaml
│  │  └─ rbac.yaml
│  ├─ data/                   # Postgres + planted plaintext secret
│  │  └─ postgres.yaml
│  └─ waf/                    # pass-through placeholder (replaced in Lab 5)
│     └─ placeholder.yaml
├─ tools/
│  └─ week0-check.sh
├─ .github/
│  └─ workflows/
│     ├─ lab4-build.yaml     # build the hardened image (Lab 4)
│     └─ ci.yaml             # full pipeline (Lab 7)
└─ labs/
   ├─ README.md              # how labs are submitted
   ├─ lab1/ … lab7/           # one folder per lab, created as you go
   └─ _TEMPLATE.md           # per-lab report template
```

Students commit their own lab evidence into `lab1/` through `lab7/` and `week0/` as the course proceeds; those folders start empty.

## `README.md`

```markdown
# CYB 601 – Software and Systems Security

Your working repository for the term. You provision one Kubernetes environment
on the course Proxmox server, then secure one reference system across seven labs.

## Prerequisites (Week 0)

- An SSH client, Git, VS Code, `kubectl`, Terraform or OpenTofu, and `jq`.
- Access to the course Proxmox URL and the public bastion (your instructor sends both).
- A GitHub account and a Google account (for the Google Skills labs).

Run the environment check with your bastion details:

```bash
BASTION=BASTION_IP BASTION_USER=BASTION_USER ./tools/week0-check.sh | tee week0/check.txt
```

## Reaching your environment

- Proxmox console: `https://PROXMOX_URL:8006`
- Bastion (SSH jump host): `ssh BASTION_USER@BASTION_IP`
- Your VM, through the bastion: `ssh -J BASTION_USER@BASTION_IP ubuntu@VM_IP`
- Your cluster, from your laptop: tunnel the API, then use `kubectl`
  ```bash
  ssh -J BASTION_USER@BASTION_IP -f -N -L 6443:127.0.0.1:6443 ubuntu@VM_IP
  export KUBECONFIG=~/.kube/cyb601.yaml
  kubectl get nodes
  ```

Student VMs have no public address. Everything in reaches you through the bastion;
the cluster only calls out (DNS and 443) for images, GitHub and signature checks.

## Provisioning (Week 1)

```bash
cd infra
cp terraform.tfvars.example terraform.tfvars   # then edit the values
export PROXMOX_VE_ENDPOINT="https://PROXMOX_URL:8006"
read -rs PROXMOX_VE_API_TOKEN; export PROXMOX_VE_API_TOKEN   # user@realm!id=secret
terraform init
terraform plan -out=lab1.tfplan
terraform apply lab1.tfplan
```

## Layout

| Path | What it is |
| --- | --- |
| `infra/` | Terraform for your VM and k3s (bpg/proxmox provider) |
| `app/` | The reference system: Juice Shop, a customer API, Postgres, a WAF placeholder |
| `tools/` | The Week 0 environment check |
| `.github/workflows/` | The Lab 4 image build and the Lab 7 security pipeline |
| `labs/` | Your evidence and reports, one folder per lab |

## Submitting a lab

Commit your work, then tag it:

```bash
git add labs/labN
git commit -m "Lab N: <summary>"
git push
git tag labN-submit && git push origin labN-submit
```

Then upload the 2-page report PDF to the LMS. See `labs/README.md`.

## Rules

- Attack only your own VM and cluster; in the capstone, only your assigned target.
- Never scan or test anything outside the course Proxmox environment.
- Keep secrets out of Git. If one is exposed, rotate it, then clean history.
- AI tools may explain errors and syntax; analysis and conclusions are your own
  and are disclosed in each lab's `README`.

## .gitignore

```
# Terraform state and cache – may contain sensitive values, never commit
.terraform/
*.tfstate
*.tfstate.*
*.tfplan
crash.log
# Commit the lock file: it pins provider checksums (a supply-chain control)
!.terraform.lock.hcl


# Local credentials and kubeconfigs
*.kubeconfig
kubeconfig*
.env
*.pem
*.key

# OS and editor noise
.DS_Store
.vscode/
```

## `infra/` Terraform (bpg/proxmox)

**`infra/versions.tf`**

```hcl
terraform {
  required_version = ">= 1.5"
  required_providers {
    proxmox = {
      source  = "bpg/proxmox"
      version = "~> 0.81"
    }
  }
}

# Credentials come from PROXMOX_VE_ENDPOINT and PROXMOX_VE_API_TOKEN
# in your environment, never from this file.
provider "proxmox" {
  insecure = false
}
```

**`infra/variables.tf`**

```hcl
variable "name"           { type = string }  # lab-LASTNAME
variable "node"           { type = string }  # the Proxmox node name
variable "template_id"    { type = number }  # the course template's VM ID
variable "pool"           { type = string }  # your resource pool
variable "ssh_public_key" { type = string }  # your public key, from Week 0
variable "k3s_version"    { type = string }  # pinned by your instructor

variable "bridge"    { type = string, default = "vmbr1" }
variable "cores"     { type = number, default = 2 }
variable "memory_mb" { type = number, default = 6144 }
variable "disk_gb"   { type = number, default = 50 }
```

**`infra/main.tf`**

```hcl
# Upload cloud-init as a Proxmox snippet (enable the "snippets" content
# type on the local datastore first).
resource "proxmox_virtual_environment_file" "cloud_init" {
  content_type = "snippets"
  datastore_id = "local"
  node_name    = var.node
  source_raw {
    data      = templatefile("${path.module}/cloud-init.yaml.tftpl", { k3s_version = var.k3s_version })
    file_name = "${var.name}-cloud-init.yaml"
  }
}

resource "proxmox_virtual_environment_vm" "lab" {
  name      = var.name
  node_name = var.node
  pool_id   = var.pool

  clone { vm_id = var.template_id }   # the course Ubuntu 24.04 template

  cpu    { cores = var.cores }
  memory { dedicated = var.memory_mb }

  disk {
    datastore_id = "local-lvm"
    interface    = "scsi0"
    size         = var.disk_gb
  }

  network_device { bridge = var.bridge }
  agent          { enabled = true }

  initialization {
    datastore_id      = "local-lvm"
    user_data_file_id = proxmox_virtual_environment_file.cloud_init.id
    ip_config { ipv4 { address = "dhcp" } }
    user_account {
      username = "ubuntu"
      keys     = [trimspace(var.ssh_public_key)]
    }
  }
}

output "vm_ip" {
  # requires the guest agent; [1][0] is the first real NIC's address
  value = proxmox_virtual_environment_vm.lab.ipv4_addresses[1][0]
}
```

**`infra/terraform.tfvars.example`** — copy to `terraform.tfvars` and fill in. Contains no secrets, so the example is committed.

```hcl
name           = "lab-LASTNAME"
node           = "NODE_NAME"      # TODO: your Proxmox node
template_id    = 9000             # TODO: the course template's VM ID
pool           = "POOL_NAME"      # TODO: your resource pool
ssh_public_key = "ssh-ed25519 AAAA... you@example.edu"
bridge         = "vmbr1"          # TODO: the student network bridge
k3s_version    = "K3S_VERSION"    # TODO: pinned by your instructor
```

## `infra/cloud-init.yaml.tftpl`

Installs k3s on first boot and gives the `ubuntu` user a private kubeconfig. The `${k3s_version}` placeholder is filled by Terraform.

```yaml
#cloud-config
package_update: true
packages:
  - jq
  - nmap
write_files:
  - path: /etc/rancher/k3s/config.yaml
    content: |
      # Lab 4 adds hardening settings here (audit log, etc.).
      write-kubeconfig-mode: "0600"
runcmd:
  - curl -sfL https://get.k3s.io -o /root/install-k3s.sh
  - INSTALL_K3S_VERSION="${k3s_version}" sh /root/install-k3s.sh
  - mkdir -p /home/ubuntu/.kube
  - cp /etc/rancher/k3s/k3s.yaml /home/ubuntu/.kube/config
  - chown -R ubuntu:ubuntu /home/ubuntu/.kube
  - chmod 600 /home/ubuntu/.kube/config
  - echo 'export KUBECONFIG=$HOME/.kube/config' >> /home/ubuntu/.bashrc
```

## `tools/week0-check.sh`

Mark it executable before committing: `git update-index --chmod=+x tools/week0-check.sh`. One of `terraform` or `tofu` will report missing; that is expected.

```bash
#!/usr/bin/env bash
# Prints tool versions and connectivity for the Week 0 environment check.
set -u
echo "== Tools =="
for cmd in git ssh kubectl terraform tofu jq; do
  if ! command -v "$cmd" >/dev/null 2>&1; then
    echo "$cmd: missing"
    continue
  fi
  case "$cmd" in
    ssh)       echo "ssh: $(ssh -V 2>&1)" ;;
    kubectl)   echo "kubectl: $(kubectl version --client 2>/dev/null | head -n 1)" ;;
    terraform) echo "terraform: $(terraform version | head -n 1)" ;;
    tofu)      echo "tofu: $(tofu version | head -n 1)" ;;
    *)         echo "$cmd: $("$cmd" --version 2>&1 | head -n 1)" ;;
  esac
done
echo "== Connectivity =="
BASTION="${BASTION:?set BASTION to the bastion address}"
BASTION_USER="${BASTION_USER:-$USER}"
if ssh -o BatchMode=yes -o ConnectTimeout=5 -o StrictHostKeyChecking=accept-new \
     "$BASTION_USER@$BASTION" true 2>/dev/null; then
  echo "SSH to bastion: ok"
else
  echo "SSH to bastion: FAILED"
fi
if ssh -T -o ConnectTimeout=5 -o StrictHostKeyChecking=accept-new git@github.com 2>&1 \
     | grep -q "successfully authenticated"; then
  echo "GitHub SSH: ok"
else
  echo "GitHub SSH: FAILED"
fi
echo "Checked at $(date -u +%Y-%m-%dT%H:%M:%SZ) by $(git config user.name 2>/dev/null || echo unknown)"
```

## `app/` Kustomize base

Apply with `kubectl apply -k app/`. The base runs as-is but ships the two planted weaknesses on purpose.

**`app/kustomization.yaml`**

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - namespaces.yaml
  - shop/deployment.yaml
  - api/deployment.yaml
  - api/rbac.yaml
  - data/postgres.yaml
  - waf/placeholder.yaml
```

**`app/namespaces.yaml`**

```yaml
apiVersion: v1
kind: Namespace
metadata: { name: shop }
---
apiVersion: v1
kind: Namespace
metadata: { name: api }
---
apiVersion: v1
kind: Namespace
metadata: { name: data }
---
apiVersion: v1
kind: Namespace
metadata: { name: waf }
```

**`app/shop/deployment.yaml`** — OWASP Juice Shop. It runs as root by default, which Lab 4's restricted Pod Security forces you to fix.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata: { name: juice-shop, namespace: shop }
spec:
  replicas: 1
  selector: { matchLabels: { app: juice-shop } }
  template:
    metadata: { labels: { app: juice-shop } }
    spec:
      containers:
        - name: juice-shop
          image: bkimminich/juice-shop:v17.1.1   # pin a version; update as needed
          ports: [{ containerPort: 3000 }]
---
apiVersion: v1
kind: Service
metadata: { name: juice-shop, namespace: shop }
spec:
  selector: { app: juice-shop }
  ports: [{ port: 80, targetPort: 3000 }]
```

**`app/data/postgres.yaml`** — holds the mock customer records. PLANTED WEAKNESS: the password is in plaintext in the manifest; gitleaks finds it in Lab 4, and you move it into a Secret with k3s encryption.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata: { name: postgres, namespace: data }
spec:
  replicas: 1
  selector: { matchLabels: { app: postgres } }
  template:
    metadata: { labels: { app: postgres } }
    spec:
      containers:
        - name: postgres
          image: postgres:16
          env:
            - name: POSTGRES_DB
              value: customers
            - name: POSTGRES_PASSWORD
              value: "Sup3rSecret-ChangeMe"   # PLANTED: plaintext secret (Lab 4)
          ports: [{ containerPort: 5432 }]
---
apiVersion: v1
kind: Service
metadata: { name: postgres, namespace: data }
spec:
  selector: { app: postgres }
  ports: [{ port: 5432, targetPort: 5432 }]
```

**`app/api/deployment.yaml`** — TODO: replace the stub image with your customer API. The stub just proves the namespace and wiring.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata: { name: customer-api, namespace: api }
spec:
  replicas: 1
  selector: { matchLabels: { app: customer-api } }
  template:
    metadata: { labels: { app: customer-api } }
    spec:
      serviceAccountName: customer-api
      containers:
        - name: api
          image: hashicorp/http-echo:1.0   # TODO: replace with the real API
          args: ["-text=customer-api stub", "-listen=:8080"]
          ports: [{ containerPort: 8080 }]
---
apiVersion: v1
kind: Service
metadata: { name: customer-api, namespace: api }
spec:
  selector: { app: customer-api }
  ports: [{ port: 80, targetPort: 8080 }]
```

**`app/api/rbac.yaml`** — PLANTED WEAKNESS: the customer API's ServiceAccount is bound to a cluster role that can read every Secret in the cluster. Students find and fix this in Lab 3.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata: { name: customer-api, namespace: api }
---
# PLANTED: far more access than a web API needs. Replace with a least-
# privilege Role in Lab 3.
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata: { name: customer-api-too-much }
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: view        # cluster-wide read, including Secrets in many setups
subjects:
  - kind: ServiceAccount
    name: customer-api
    namespace: api
```

**`app/waf/placeholder.yaml`** — a pass-through so the edge exists from Week 1; Lab 5 replaces it with Nginx + ModSecurity + the OWASP Core Rule Set.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata: { name: waf, namespace: waf }
spec:
  replicas: 1
  selector: { matchLabels: { app: waf } }
  template:
    metadata: { labels: { app: waf } }
    spec:
      containers:
        - name: waf
          image: nginx:stable   # Lab 5: swap for an image with ModSecurity + CRS
          ports: [{ containerPort: 80 }]
```

## `.github/workflows/`

**`.github/workflows/lab4-build.yaml`** — builds and pushes the hardened image from Lab 4. Students extend this into the full pipeline in Lab 7.

```yaml
name: lab4-build
on:
  workflow_dispatch:
  push:
    paths: ["lab4/Dockerfile", ".github/workflows/lab4-build.yaml"]
permissions:
  contents: read
  packages: write
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - uses: docker/build-push-action@v6
        with:
          context: lab4
          push: true
          tags: ghcr.io/${{ github.repository }}/juice-shop:hardened
```

**`.github/workflows/ci.yaml`** — the Lab 7 pipeline. Each stage fails the build on a serious finding; fill the TODO identity before enforcing signing.

```yaml
name: ci
on: [pull_request]
permissions:
  contents: read
  packages: write
  id-token: write        # for Cosign keyless signing
jobs:
  secrets:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with: { fetch-depth: 0 }
      - uses: gitleaks/gitleaks-action@v2

  sast:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: returntocorp/semgrep-action@v1
        with: { config: p/default }

  deps-and-iac:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Trivy filesystem
        uses: aquasecurity/trivy-action@0.28.0
        with: { scan-type: fs, severity: "HIGH,CRITICAL", exit-code: "1" }
      - name: Checkov (IaC)
        uses: bridgecrewio/checkov-action@v12
        with: { directory: . }

  build-scan-sign:
    needs: [secrets, sast, deps-and-iac]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - name: Build
        id: build
        uses: docker/build-push-action@v6
        with:
          context: lab4
          push: true
          tags: ghcr.io/${{ github.repository }}/juice-shop:${{ github.sha }}
      - name: Trivy image scan
        uses: aquasecurity/trivy-action@0.28.0
        with:
          image-ref: ghcr.io/${{ github.repository }}/juice-shop:${{ github.sha }}
          severity: "CRITICAL"
          exit-code: "1"
      - name: SBOM (Syft, CycloneDX)
        uses: anchore/sbom-action@v0
        with:
          image: ghcr.io/${{ github.repository }}/juice-shop:${{ github.sha }}
          format: cyclonedx-json
          output-file: sbom.cdx.json
      - name: Cosign keyless sign + attest SBOM
        uses: sigstore/cosign-installer@v3
      - run: |
          cosign sign --yes ghcr.io/${{ github.repository }}/juice-shop:${{ github.sha }}
          cosign attest --yes --predicate sbom.cdx.json --type cyclonedx \
            ghcr.io/${{ github.repository }}/juice-shop:${{ github.sha }}
      - name: Conftest policy gate
        uses: instrumenta/conftest-action@master
        with: { files: lab7/evidence, policy: lab7/policy }

  dast:
    needs: [build-scan-sign]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: ZAP baseline
        uses: zaproxy/action-baseline@v0.12.0
        with: { target: "http://localhost:3000" }   # TODO: start the container first
```

In Lab 7, students replace the permissive Cosign identity with their workflow's exact `--certificate-identity` / `--certificate-oidc-issuer`, and Kyverno verifies the same identity at admission.

## `labs/`

**`labs/README.md`**

```markdown
# Labs

One folder per lab (`lab1/` through `lab7/`), plus `week0/` for the setup check.
Each folder holds the evidence the lab asks for and a `README.md` written from
the `_TEMPLATE.md` below.

## Submitting

```bash
git add labs/labN
git commit -m "Lab N: <summary>"
git push
git tag labN-submit && git push origin labN-submit
```

Then upload the 2-page report PDF to the LMS. Grading uses the engineering lab
rubric: working implementation (14), evidence (8), analysis and risk reasoning
(12), professional communication (6).

## Rules

- The lab must rebuild from a clean clone using this README.
- Keep secrets out of Git. Report any exposed secret within 24 hours.
- Disclose any AI use; analysis and conclusions are your own.
```

**`labs/_TEMPLATE.md`** — copy into each `labN/README.md`.

```markdown
# Lab N – <title>

## What I built
<one paragraph>

## How to run it
<commands to reproduce from a clean clone>

## Evidence
<command output, configs, scan results — or links to the files in this folder>

## Analysis
<answers to the lab's questions; link findings to the threat model and readings;
state the residual risk>

## What failed and how I fixed it
<short note>

## AI use
<what you used it for, if any>
```

That completes the scaffold. Drop these files into a repo, mark it a template, and you're ready for Week 0.