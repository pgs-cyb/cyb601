# Week 1 – Foundations: thinking like a security engineer

Week 1 teaches to think like security engineers, then has them build their own cluster in code and map its attack surface. CLO 1.

## Assignments

Week 1 has four pieces of work: the discussion, the rules of engagement, the Google Skills lab and Lab 1. All are due by Sunday at 11:59 p.m. Pacific, except the primary discussion post, due Thursday.

### Discussion (20 pts)

**Prompt.** Pick one breach from the last three years that has a public post-incident report. Was the root cause a software security failure, a system security failure or an incentive failure in Anderson's sense? Name the one Saltzer and Schroeder principle that would have changed the outcome, and the lifecycle stage where it should have been applied.

**What a full-marks post does**

- Links the post-incident report and summarizes the attack path in three or four sentences.
- Classifies the root cause with evidence from the report, not adjectives.
- Names exactly one principle, the mechanism that would have applied it, and the lifecycle stage.
- Cites at least one primary source beyond Anderson, in APA 7, in at least 200 words.
- Replies to two classmates (50 words or more each) with new evidence or a reasoned challenge to their classification.

### Rules of engagement (required before any scan in Lab 1)

Copy this into `lab1/rules-of-engagement.md`, fill in your name and the date, and commit it. A typed name counts as your signature unless your instructor asks for a signed PDF.

```markdown
# CYB 601 rules of engagement

I, NAME, agree that for the whole course I will:

1. Test only systems assigned to me: my own VM and cluster, and in the capstone only my assigned target.
2. Not scan, probe or attack anything outside the course cloud, including from my VM.
3. Not access the Proxmox host or console, any account but my own, the bastion beyond my login, the class services, or a classmate's VM.
4. Not run denial-of-service tests against shared infrastructure.
5. Keep secrets out of Git, and report any secret I expose or discover within 24 hours.
6. Report suspected incidents or out-of-scope access to the instructor at once.

I understand that breaking these rules is an academic integrity violation and may be illegal.

Signed: NAME
Date: YYYY-MM-DD
```

### Google Skills lab (20 pts)

*A Tour of Google Cloud Hands-on Labs.* Upload the completion badge or summary screen showing your name, plus 3–5 sentences: compare Google Cloud's project and IAM boundaries with the Proxmox user and token scope you used in Lab 1. Where is the trust boundary in each, and who can cross it?

### Lab 1 submission (40 pts)

- Push your work and the tag `lab1-submit`.
- Upload a 2-page lab report PDF to the LMS: what you built, the evidence, your analysis of the two nmap views and the kube-bench baseline, and what failed and how you fixed it.
- Grading uses the engineering lab rubric: working implementation 14, evidence 8, analysis 12, communication 6.

## Hands-on Lab 1: Provision, deploy and map the attack surface (40 pts, about 4 h)

You rebuild last week's practice VM as code, deploy the reference system, and produce the inventory, scans, baseline and diagram that every later lab builds on.

**Before you start:** Bastion reachable; Proxmox token loaded in this terminal (Lab 0, step 4); practice VM removed; Syft installed on your laptop; rules of engagement committed.

### Part A – Provision your VM with Terraform (about 1 h)

Your repository's `infra/` folder holds the starter module. Complete it so it matches these files.

`infra/versions.tf`

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

`infra/variables.tf`

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

`infra/main.tf`

```hcl
# Upload your cloud-init as a Proxmox snippet (needs the "snippets"
# content type enabled on the local datastore).
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
  value = proxmox_virtual_environment_vm.lab.ipv4_addresses[1][0]
}
```

`infra/cloud-init.yaml.tftpl`

```yaml
#cloud-config
package_update: true
packages:
  - jq
  - nmap
write_files:
  - path: /etc/rancher/k3s/config.yaml
    content: |
      # Lab 4 adds hardening settings here.
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

`infra/terraform.tfvars` holds the values your instructor publishes. It contains no secrets, so it is committed.

```hcl
name           = "lab-LASTNAME"
node           = "NODE_NAME"
template_id    = 9000                               # the course template's VM ID
pool           = "POOL_NAME"
ssh_public_key = "ssh-ed25519 AAAA... you@example.edu"
bridge         = "vmbr1"
k3s_version    = "K3S_VERSION"
```

Your `.gitignore` must contain these lines. Commit `.terraform.lock.hcl`: it pins the provider's checksums, a supply-chain control you meet again in Week 7. Never commit state, which can hold sensitive values.

```
.terraform/
*.tfstate
*.tfstate.*
*.tfplan
```

Plan, read the plan, then apply it:

```bash
cd infra
terraform init
terraform fmt -check
terraform validate
terraform plan -out=lab1.tfplan
mkdir -p ../lab1
terraform show -no-color lab1.tfplan > ../lab1/terraform-plan.txt
terraform apply lab1.tfplan
VM_IP=$(terraform output -raw vm_ip)
ssh -J cyb601-bastion ubuntu@"$VM_IP" 'cloud-init status --wait && kubectl --kubeconfig ~/.kube/config get nodes'
```

**Check:** the node reports Ready. In your report, name one line of the plan you would challenge if a teammate had written it.

### Part B – Reach the cluster from your laptop and deploy (about 30 min)

```bash
scp -J cyb601-bastion ubuntu@"$VM_IP":.kube/config ~/.kube/cyb601.yaml
chmod 600 ~/.kube/cyb601.yaml
ssh -J cyb601-bastion -f -N -L 6443:127.0.0.1:6443 ubuntu@"$VM_IP"
export KUBECONFIG=~/.kube/cyb601.yaml
kubectl get nodes
```

That kubeconfig is full cluster-admin. Keep it outside your repository, readable only by you. Then, from your repository root:

```bash
kubectl apply -k app/
kubectl get pods -A -w        # Ctrl+C when every pod is Running
```

### Part C – Inventory and SBOMs (about 45 min)

```bash
kubectl get ns > lab1/namespaces.txt
kubectl get deploy,sts,ds,svc,ingress -A -o wide > lab1/workloads.txt
kubectl get sa,roles,rolebindings,clusterrolebindings -A > lab1/identities.txt
kubectl get pods -A -o jsonpath='{range .items[*]}{range .spec.containers[*]}{.image}{"\n"}{end}{end}' \
  | sort -u > lab1/images.txt

mkdir -p lab1/sbom
while read -r img; do
  syft "$img" -o cyclonedx-json > "lab1/sbom/$(echo "$img" | tr '/:@' '___').cdx.json"
done < lab1/images.txt

for f in lab1/sbom/*.json; do echo "$(jq '.components | length' "$f") $f"; done | sort -rn
```

In your report, answer: which image carries the most packages? Which data store holds customer records, and who can reach it? Which service account can do more than it needs? Lab 3 fixes that last one.

### Part D – Scan from two places (about 30 min)

From the bastion, scan every port on your VM. Then scan every pod from inside the cluster.

```bash
ssh cyb601-bastion "nmap -sT -Pn -p- $VM_IP" > lab1/nmap-from-bastion.txt

PODS=$(kubectl get pods -A -o jsonpath='{.items[*].status.podIP}')
kubectl run scan --rm -i --quiet --restart=Never --image=instrumentisto/nmap -- \
  -sT -Pn -p 1-10000 $PODS > lab1/nmap-from-pod.txt
```

In your report, explain why the two results differ and which flows Lab 5 should block.

### Part E – CIS baseline with kube-bench (about 20 min)

On the VM:

```bash
KB=KUBE_BENCH_VERSION    # pinned by your instructor, without the leading v
curl -sLO "https://github.com/aquasecurity/kube-bench/releases/download/v${KB}/kube-bench_${KB}_linux_amd64.tar.gz"
sudo mkdir -p /opt/kube-bench
sudo tar -xzf "kube-bench_${KB}_linux_amd64.tar.gz" -C /opt/kube-bench
ls /opt/kube-bench/cfg | grep -i k3s
sudo /opt/kube-bench/kube-bench run \
  --config-dir /opt/kube-bench/cfg --config /opt/kube-bench/cfg/config.yaml \
  --benchmark K3S_BENCHMARK > kube-bench-baseline.txt
grep -A 5 "Summary total" kube-bench-baseline.txt
```

Then, on your laptop:

```bash
scp -J cyb601-bastion ubuntu@"$VM_IP":kube-bench-baseline.txt lab1/
```

### Part F – Data-flow diagram (about 45 min)

Draw it in OWASP Threat Dragon, which you reuse in Lab 2, or in draw.io. Save the image and the source file in `lab1/`. Include:

- External entities: customer browser, admin, GitHub, the Proxmox API, the bastion
- Processes: Traefik, the WAF placeholder, Juice Shop, the customer API
- Data stores: Postgres, Juice Shop's internal database, the k3s datastore, Terraform state on your laptop
- Flows: HTTP and HTTPS, SQL, kubectl, SSH, Terraform API calls
- Trust boundaries: internet to VPN, bastion to VM, edge to application, application to data, cluster to outside, your laptop to the control plane

### Part G – Assets and opponents (about 20 min)

| Rank | Asset | Where it lives | Opponent (Anderson Ch. 2) | What they want with it |
| --- | --- | --- | --- | --- |
| 1 | Customer records | Postgres, data namespace | Crooks | Resale and payment fraud |
| 2 |  |  |  |  |
| 3 |  |  |  |  |
| 4 |  |  |  |  |
| 5 |  |  |  |  |



### Part H – Memo and submission

Write `lab1/risk-memo.md`: one page with your top three architecture risks, the evidence for each, and what you would fix first. Then:

```bash
git add infra/ lab1/
git commit -m "Lab 1: provisioned VM, inventory, scans, baseline, DFD"
git push
git tag lab1-submit
git push origin lab1-submit
```

### Deliverables checklist

- `infra/` with the lock file, plus `lab1/terraform-plan.txt`
- `lab1/namespaces.txt`, `workloads.txt`, `identities.txt`, `images.txt` and `sbom/`
- `lab1/nmap-from-bastion.txt` and `lab1/nmap-from-pod.txt`
- `lab1/kube-bench-baseline.txt`
- Data-flow diagram image and source file
- `lab1/assets.md`, `lab1/risk-memo.md` and `lab1/rules-of-engagement.md`
- Tag `lab1-submit` pushed and report uploaded

### Troubleshooting
| Symptom | Likely cause | Fix |
| --- | --- | --- |
| `terraform plan` fails to authenticate | Proxmox token not loaded in this terminal | Repeat Lab 0, step 4 |
| `apply` fails with a resource limit error | The practice VM still exists | Remove it in Proxmox, then apply again |
| SSH works, but kubectl fails right after apply | cloud-init is still installing k3s | Run `cloud-init status --wait` on the VM |
| kubectl on the laptop times out | The port 6443 tunnel is not running | Rerun the `ssh -f -N -L 6443:...` command |
| A Syft pull fails | Registry rate limit or a typo in the image name | Retry that image alone after a few minutes |
| The pod scan finds nothing | Pods not Running yet, so `$PODS` is empty | Check `kubectl get pods -A -o wide` first |