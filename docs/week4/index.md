# Week 4 – Hardening, patching, cryptography and secrets

Week 4 has students measure the host, cluster and images against baselines, fix them as code, and fix how the system handles keys and secrets. It also carries the first summative. CLOs 4 and 5.

## Assignments

Week 4 has four pieces of work: the discussion, the Google Skills lab, Lab 4 and Summative 1. All are due by Sunday at 11:59 p.m. Pacific, except the primary discussion post, due Thursday. This is a heavier week; start the lab early, because the report depends on it.

### Discussion (20 pts)

**Prompt.** Equifax had a patch policy and a vulnerability scanner, yet ran a known-vulnerable Apache Struts for months. Using the House report, identify the two process failures that mattered most. Then state a patch SLA you would defend for internet-facing systems, and what you would do when a patch cannot be applied in that time.

**What a full-marks post does**

- Names two specific process failures with support from the House report, not general observations.
- States a concrete SLA by severity (for example, critical internet-facing within 15 days) and justifies it.
- Describes compensating controls for when a patch cannot be applied in time.
- Cites at least one primary source beyond Anderson, in APA 7, in at least 200 words.
- Replies to two classmates (50 words or more each), engaging with their SLA.

### Google Skills lab (20 pts)

*Mitigating Security Vulnerabilities on Google Cloud.* Upload the badge or summary screen showing your name, plus 3–5 sentences linking vulnerability management to software resilience: how does Google's approach compare with the Trivy and kube-bench workflow you ran in Lab 4?

### Lab 4 submission (40 pts)

- Push your work and the tag `lab4-submit`.
- Upload a 2-page lab report PDF: before-and-after numbers for host, cluster and image; what you remediated and what you deliberately did not; and the secret you found and how you handled it.
- Grading uses the engineering lab rubric.

### Summative 1 (60 pts) – see the next section for the full brief.

## Hands-on Lab 4: Host, cluster and image hardening, TLS and secrets (40 pts, about 5 h)

You measure the host, cluster and images against baselines, remediate as code, fix TLS, and move the planted secret out of Git and into encrypted storage. The numbers you record here are the evidence for Summative 1.

**Before you start:** your cluster from Lab 1; `gitleaks`, `testssl.sh` and `trivy` installed on your laptop or available on the VM; the Lab 4 starter additions applied. Optional: snapshot the VM first and delete the snapshot after you submit, since your one slot is needed before the capstone.

The six layers you build:

[embed: node/0c0edb07-4480]

### Part A – Host baseline and remediation (about 75 min)

```bash
# Baseline
sudo apt-get update && sudo apt-get install -y lynis
sudo lynis audit system | tee lab4/lynis-before.txt
grep -E "Hardening index|Warning|Suggestion" lab4/lynis-before.txt | head -n 30
```

Record the hardening index and the ten highest-risk findings. Write an Ansible playbook `lab4/harden.yml` that fixes at least eight of them (SSH configuration, unused services, file permissions, kernel parameters). Run it, then rescan:

```bash
ansible-playbook -i "localhost," -c local lab4/harden.yml
sudo lynis audit system | tee lab4/lynis-after.txt
```

Report the index before and after.

### Part B – Cluster hardening (about 45 min)

Rerun kube-bench (the Lab 1 command) and save `lab4/kube-bench-before.txt`. Fix at least five findings in the k3s configuration. Turn on API server audit logging, since Lab 6 ships that log:

```yaml
# add to /etc/rancher/k3s/config.yaml, then restart k3s
kube-apiserver-arg:
  - "audit-log-path=/var/log/k3s-audit.log"
  - "audit-policy-file=/etc/rancher/k3s/audit-policy.yaml"
```

The starter repository provides `audit-policy.yaml`. Rerun kube-bench to `lab4/kube-bench-after.txt` and compare the FAIL count against Week 1.

### Part C – Restricted Pod Security (about 45 min)

Label the application namespaces to enforce the restricted standard:

```bash
kubectl label ns shop api data \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/warn=restricted --overwrite
kubectl get pods -n shop     # some pods now fail to schedule
```

Fix each violation in the manifests: `runAsNonRoot`, a non-root `runAsUser`, `readOnlyRootFilesystem: true` with writable `emptyDir` mounts where the app needs them, `capabilities.drop: [ALL]`, `seccompProfile.type: RuntimeDefault` and `allowPrivilegeEscalation: false`. Save the patched manifests under `app/` and the evidence to `lab4/pod-security.txt`.

### Part D – Harden the Juice Shop image (about 45 min)

```bash
trivy image --severity HIGH,CRITICAL JUICE_SHOP_IMAGE | tee lab4/trivy-before.txt
```

Write `lab4/Dockerfile` on a minimal base, as a non-root user, and build it with the starter repository's build workflow. Rescan and compare:

```bash
trivy image --severity HIGH,CRITICAL YOUR_HARDENED_IMAGE | tee lab4/trivy-after.txt
grep -c CVE lab4/trivy-before.txt lab4/trivy-after.txt
```

### Part E – TLS (about 30 min)

Install cert-manager from the starter manifests, create an internal CA issuer and a certificate for the shop hostname, and have Traefik serve TLS. Then test and remove weak settings:

```bash
testssl.sh --quiet --protocols --ciphers https://SHOP_HOSTNAME | tee lab4/testssl.txt
```

Remove every weak protocol and cipher it flags, then rerun until the report is clean. Save the final report.

### Part F – Secrets (about 30 min)

```bash
gitleaks detect --source . --report-path lab4/gitleaks-before.json
```

The planted database password will appear. Remove it from the manifests, put it in a Kubernetes Secret, and turn on k3s secrets encryption:

```yaml
# /etc/rancher/k3s/config.yaml
secrets-encryption: true
```

Show the stored value is ciphertext, then rotate the key:

```bash
sudo k3s secrets-encrypt status
kubectl -n data get secret db-credentials -o jsonpath='{.data.password}' | base64 -d | head -c 0; echo "(value exists)"
sudo ETCDCTL_API=3 ... # or the documented datastore read, to show ciphertext at rest
sudo k3s secrets-encrypt rotate-keys
```

Rescan to confirm the secret is gone from the working tree, and record in your report that it was rotated, not just moved.

```bash
git add lab4/ app/ harden.yml
git commit -m "Lab 4: host, cluster and image hardening, TLS, secrets"
git push
git tag lab4-submit
git push origin lab4-submit
```

### Deliverables checklist

- `lab4/lynis-before.txt`, `lynis-after.txt` and `harden.yml`
- `lab4/kube-bench-before.txt` and `kube-bench-after.txt`
- `lab4/pod-security.txt` and the patched manifests
- `lab4/trivy-before.txt`, `trivy-after.txt` and `Dockerfile`
- `lab4/testssl.txt`, clean of weak protocols and ciphers
- `lab4/gitleaks-before.json` and a note that the secret was rotated
- `lab4/exceptions.md`: findings deliberately not fixed, each with a reason
- Tag `lab4-submit` pushed and report uploaded

## Summative Assessment 1: System Hardening Report (60 pts)

**Due Sunday of Week 4, 11:59 p.m. Pacific. 3–4 pages plus appendix, APA 7. CLO 4.**

Write a secure design specification for the host, cluster and workloads you hardened in Lab 4. This is not a lab log; it is the document you would hand an architect to justify the design. Write it in your own words from your own evidence. AI may help with wording and error messages, but the analysis, risk ratings and conclusions are yours and are disclosed in the report.

### Required sections

1. **Executive summary** (about half a page): the system, the top three residual risks, and what you recommend.
2. **Baselines chosen and why**: which CIS profiles and benchmarks you used for host, cluster and image, and why they fit a production retailer.
3. **Defense-in-depth layers**: each of the six layers, the control at it, and the attack-tree path it closes. Reference your Lab 2 register.
4. **Least-privilege decisions**: the restricted Pod Security settings and the RBAC posture, and what each denies.
5. **Before-and-after metrics**: a table of the Lynis index, kube-bench FAIL count, image CVE count and TLS grade, before and after.
6. **Exceptions**: findings you did not fix, each with a compensating control and a named risk owner.
7. **Patch management policy**: SLAs by severity for internet-facing and internal systems, and what you do when a patch cannot be applied in time.
8. **Key and secret lifecycle**: how secrets are generated, stored, injected, encrypted and rotated, using your Lab 4 evidence.
9. **Appendix**: links to the repository evidence (commit or tag) for every claim.


### Expected results

- Lynis hardening index typically rises from the 50s to the 70s after eight fixes.
- kube-bench FAIL count drops noticeably once audit logging and a few defaults are fixed.
- The Juice Shop image CVE count falls sharply on a minimal base; exact numbers vary with the scan date, so grade the delta and the method, not an absolute target.
- testssl.sh should end with no weak protocols and a clean cipher list.