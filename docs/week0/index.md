# Week 0 – Prerequisites and environment setup

Week 0 gets every student onto the lab cloud, GitHub and Kubernetes before graded work starts. Nothing this week is graded, but every lab from Week 1 assumes it is done.

## Assignment: readiness check

Week 0 carries no points. Complete all four parts before the Week 1 live session.

**Part 1 – Self-assessment.** Tick what you can already do; anything left unticked is your Week 0 study list.

- SSH to a server with a key and copy files with scp
- Read a service log with journalctl and find a listening port with ss
- Clone, branch, commit, push and tag with Git
- Explain a /24 network, a port and a DNS lookup
- Read and edit YAML without breaking the indentation
- Write a 20-line script in one programming language

**Part 2 – Prerequisite quiz.** Answer each question in a few lines in `week0/quiz.md`. Fewer than 7 correct means booking extra time at the environment check.

1. What does `chmod 600 ~/.ssh/id_ed25519` do, and why does SSH care?
2. How many usable host addresses does 10.20.0.0/24 have?
3. What are the default ports for SSH, DNS and HTTPS?
4. What is the difference between `git commit` and `git push`?
5. You pushed an API key to GitHub five minutes ago. What do you do first, and why?
6. What is the difference between a container image and a running container?
7. What does a Kubernetes Service give you that a Pod's IP address does not?
8. What does `terraform plan` show, and why should someone review it before `apply`?
9. What does a reverse proxy do? Name one security control it can enforce.
10. Why is `curl https://example.com/install.sh | sh` risky? Give two safer options.

**Part 3 – Introduce yourself on the discussion board.** In about 150 words: your security background, one system you have been responsible for, and one thing you want to be able to do by Week 8. Reply to two classmates.

**Part 4 – Submit your environment check.** Finish Hands-on Lab 0, then push `week0/check.txt`, `week0/quiz.md` and `week0/notes.md`, and the tag `week0-submit`.

## Hands-on Lab 0: Set up and touch the cloud (ungraded, about 3 h)

By the end, you have every tool installed, reach the lab cloud through the public bastion, have run Kubernetes on a throwaway VM, and have pushed your first tagged commit.

Your instructor sends you the bastion address and username, your Proxmox URL and username, and the template repository link. Replace the UPPERCASE placeholders below with those values.

### Step 1 – Install the tools on your laptop

| Tool | Used for | Install from |
| --- | --- | --- |
| Git | Version control | git-scm.com |
| VS Code | Editing code and YAML | code.visualstudio.com |
| kubectl | Talking to Kubernetes | kubernetes.io/docs/tasks/tools |
| Terraform or OpenTofu | Infrastructure as code | developer.hashicorp.com/terraform/install or opentofu.org |
| SSH client | SSH to the bastion and your VM | Built in on macOS, Linux and WSL |
| jq | Reading JSON output | jqlang.github.io/jq |

On Windows, run `wsl --install` in an administrator terminal, then do all command-line work in the Ubuntu shell it creates. Install VS Code on Windows itself.

```bash
git --version
kubectl version --client
terraform version      # or: tofu version
ssh -V
jq --version
```

### Step 2 – Create an SSH key

```bash
ssh-keygen -t ed25519 -C "you@example.edu"
ls -l ~/.ssh/id_ed25519 ~/.ssh/id_ed25519.pub
```

The private key must show `-rw-------`. Share only the `.pub` file: add it to GitHub under Settings, SSH and GPG keys, and submit it through the course key form so the bastion lets you in.

### Step 3 – Reach the public bastion

The bastion has a public address, so you connect to it directly over SSH; there is no VPN.

Add the bastion to `~/.ssh/config`:

```
Host cyb601-bastion
    HostName BASTION_IP
    User BASTION_USER
    IdentityFile ~/.ssh/id_ed25519
```

```bash
ssh cyb601-bastion hostname
```

**Check:** the bastion prints its hostname. If it does not, use the troubleshooting table at the end.

### Step 4 – Sign in to Proxmox and create an API token

1. Open your Proxmox console at `https://PROXMOX_URL:8006`, sign in and change your password.
2. Upload your SSH public key where your instructor directs (you also add it through cloud-init in Lab 1), named `LASTNAME-key`.
3. In Datacenter, then Permissions, then API Tokens, create a token for your user. Store the token ID and secret in a password manager, never in a file inside your repository.
4. Load them into your shell without echoing them or saving them in history:

```bash
export PROXMOX_VE_ENDPOINT="https://PROXMOX_URL:8006"
printf 'API token (user@realm!id=secret): '; read -rs PROXMOX_VE_API_TOKEN; echo; export PROXMOX_VE_API_TOKEN
```

These variables vanish when the terminal closes, which is the point. Week 1's first `terraform plan` is the real test of the keys.

### Step 5 – Build a practice VM by clicking

1. In Proxmox, clone a VM from the course Ubuntu 24.04 template. Give it 2 vCPU and 6 GB, attach the student network, and add your SSH public key through the template's cloud-init. Name it `practice-LASTNAME`.
2. Note its IP address; it is `VM_IP` below.
3. Write down every choice you made in the console. In Week 1 you rewrite them as Terraform.
4. Connect through the bastion:

```bash
ssh -J cyb601-bastion ubuntu@VM_IP
```

### Step 6 – Kubernetes in 30 minutes

On the practice VM, download the k3s installer, read it, then run it:

```bash
curl -sfL https://get.k3s.io -o install-k3s.sh
less install-k3s.sh
sudo sh install-k3s.sh
```

Give your user a private copy of the cluster credentials:

```bash
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown "$USER" ~/.kube/config
chmod 600 ~/.kube/config
echo 'export KUBECONFIG=$HOME/.kube/config' >> ~/.bashrc
export KUBECONFIG=$HOME/.kube/config
kubectl get nodes
```

Deploy, expose and break something:

```bash
kubectl create namespace practice
kubectl -n practice create deployment web --image=nginx:stable
kubectl -n practice expose deployment web --port=80
kubectl -n practice get pods,svc -o wide
kubectl -n practice run curl --rm -it --restart=Never --image=curlimages/curl -- curl -s http://web
kubectl -n practice scale deployment web --replicas=3
kubectl -n practice delete pod -l app=web --wait=false
kubectl -n practice get pods -w        # Ctrl+C once all three are Running again
kubectl -n practice exec deploy/web -- id
kubectl -n practice logs deploy/web --tail=20
```

Answer these in `week0/notes.md`:

1. Why did the pods come back after you deleted them?
2. Which user does nginx run as inside the container, and why will that fail Week 4's restricted policy?
3. Who can read `/etc/rancher/k3s/k3s.yaml`, and what could someone do with it?
4. Why read `install-k3s.sh` before running it as root?

### Step 7 – See the site from your laptop

```bash
ssh -J cyb601-bastion -L 8080:localhost:8080 ubuntu@VM_IP \
  'kubectl --kubeconfig ~/.kube/config -n practice port-forward svc/web 8080:80'
```

Open http://localhost:8080. You are seeing nginx through two tunnels: SSH to the VM, then kubectl to the Service. Press Ctrl+C to close both.

### Step 8 – Create your course repository

1. On GitHub, open the template repository, choose "Use this template", and create a private repository named `cyb601-LASTNAME`. Add your instructor as a collaborator.
2. Clone it on your laptop and run the environment check:

```bash
git clone git@github.com:YOUR_GITHUB_USER/cyb601-LASTNAME.git
cd cyb601-LASTNAME
mkdir -p week0
BASTION=BASTION_IP BASTION_USER=BASTION_USER ./tools/week0-check.sh | tee week0/check.txt
```

1. Add `quiz.md` and `notes.md` to `week0/`, then commit, push and tag:

```bash
git add week0/
git commit -m "Week 0: environment check, quiz and notes"
git push
git tag week0-submit
git push origin week0-submit
```

### Step 9 – Clean up

Destroy `practice-LASTNAME` in Proxmox (stop it, then remove), and confirm it is gone from the VM list. Your quota allows one VM, and Week 1 needs it.

### Troubleshooting
| Symptom | Likely cause | Fix |
| --- | --- | --- |
| `Permission denied (publickey)` at the bastion | Key not registered, or SSH offers a different key | Run `ssh -v cyb601-bastion` to see which key is offered; resubmit the `.pub` file |
| Cannot reach the bastion | Wrong bastion address, or your key is not authorized yet | `Check the bastion address and that your public key was submitted` |
| `kubectl` reports connection refused | `KUBECONFIG` not set, or k3s not running | `export KUBECONFIG=$HOME/.kube/config`; `sudo systemctl status k3s` |
| Proxmox refuses the VM, or the pool quota is exceeded | Your one-VM quota is used | Remove old VMs, including any practice VM |
| `git push` is rejected | Wrong remote, or SSH key missing on GitHub | `git remote -v`; `ssh -T git@github.com` |
| Scripts fail with `$'\r': command not found` on Windows | Windows line endings | In WSL, run `git config --global core.autocrlf input`, then clone again |