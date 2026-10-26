

## Course learning outcomes and alignment

The eight CLOs keep the original wording word for word; each now has a week that teaches it and a graded artifact that assesses it.

| # | Course learning outcome | PLOs | Taught | Assessed by |
| --- | --- | --- | --- | --- |
| 1 | Analyze software/service architectures (enterprise, cloud, container/K8s) to identify security dependencies, weaknesses, and design risks. | 1 | Wk 1 | Lab 1, Final |
| 2 | Apply security frameworks (e.g., NIST SSDF, OWASP SAMM/ASVS, NIST SP 800-160, Zero Trust) to produce secure system designs. | 1, 2 | Wk 1, 2, 5 | Lab 2, Lab 5, Final |
| 3 | Perform threat modeling (STRIDE, attack trees) and risk analysis to inform requirements, design, and controls. | 2, 3 | Wk 2 | Lab 2, Final |
| 4 | Author a secure design specification demonstrating defense-in-depth, least privilege, and secure-by-design principles. | 2, 3 | Wk 4 | Lab 4, Summative 1 |
| 5 | Engineer authN/authZ, cryptographic protections, secrets, keys, and data-at-rest/in-transit controls aligned to business risk. | 3, 4 | Wk 3, 4, 5 | Lab 3, Lab 4, Lab 5 |
| 6 | Instrument systems for observability (logs, metrics, traces) and implement runtime protections (policy as code, WAF/Armor, Binary Authorization). | 3, 4 | Wk 5, 6, 7 | Lab 5 (WAF), Lab 6 (logs, metrics, traces, runtime detection), Lab 7 (admission control, the open-source equivalent of Binary Authorization) |
| 7 | Establish DevSecOps verification (SAST/DAST/IAST, IaC scanning, artifact integrity, SBOM, supply-chain controls) and document evidence. | 3, 4 | Wk 7 | Lab 7, Summative 2 |
| 8 | Present and defend software/system security design and verification results to both technical and executive audiences. | 1–4 | Wk 8 | Capstone debrief, Final presentation and defense |

IAST is covered conceptually in Week 7 and in Summative 2's coverage analysis rather than as a tool exercise.

## Lab environment and running case study

All seven labs run on one CloudStack server with 128 GB RAM, 32 CPU cores and 2 TB of storage. That fits 15 students at one 6 GB single-node Kubernetes cluster each, plus shared class services.

### Running case study

Students are the security engineering team for a fictional retailer taking its web shop to production. The system is OWASP Juice Shop plus a small customer API and its Postgres database, deployed to each student's own k3s cluster. Each lab's output feeds the next: the Week 2 threat model decides what Week 5 blocks and what Week 6 detects, and the final plan cites all seven labs.

lab architecture · one of 15 student VMs shown

Students enter only through the VPN; clusters call the internet only for code, images and signature checks.
### What runs in each student cluster

| Namespace | Runs | Added in |
| --- | --- | --- |
| kube-system | k3s, CoreDNS, Traefik (k3s's bundled ingress) | Lab 1 |
| shop | OWASP Juice Shop | Lab 1 |
| api, data | Customer API and the Postgres database holding mock customer records | Lab 1 |
| identity | Keycloak and oauth2-proxy | Lab 3 |
| cert-manager | Internal certificate authority and certificates | Lab 4 |
| waf | Nginx with ModSecurity and the OWASP Core Rule Set | Lab 5 |
| istio-system | Istio in ambient mode | Lab 5 |
| falco, observability | Falco and Grafana Alloy | Lab 6 |
| kyverno, flux-system | Kyverno admission control and Flux GitOps | Lab 7 |

The instructor's dry run confirms that the full stack, with a ZAP scan running, fits in 6 GB before the term starts.

### Network and access

- Students reach their VM only through the course WireGuard VPN or the SSH bastion. Only the VPN endpoint has a public address.
- Security groups block traffic between student VMs. The instructor opens red-team-to-target paths only during the capstone.
- Outbound traffic is limited to DNS and TCP 80/443, and it is logged. Clusters need it to pull images, reach GitHub and check Sigstore signatures.
- Images build in GitHub Actions, and from Lab 7 Flux pulls deployments from GitHub. Nothing on the internet ever connects into a cluster.

### Provisioning

Each student gets a CloudStack account with an API key and a limit of one VM, 2 vCPU, 6 GB RAM, 50 GB disk and one snapshot. In Week 1 they write Terraform with the proxmox provider to create that VM, and cloud-init installs k3s. Terraform code and Kubernetes manifests are graded like any other code, and Checkov scans them in Lab 7.

### Tool stack by lab

| Lab | Tools |
| --- | --- |
| Provisioning | Terraform, cloud-init, Ansible, kubectl, Kustomize |
| 1 | nmap, Syft, kube-bench |
| 2 | OWASP Threat Dragon |
| 3 | Keycloak, oauth2-proxy, ZAP or Burp Suite, Kubernetes RBAC |
| 4 | OpenSCAP or Lynis, Ansible, kube-bench, Pod Security Admission, Trivy, cert-manager, testssl.sh, gitleaks, k3s secrets encryption |
| 5 | ZAP, Nginx with ModSecurity and the OWASP Core Rule Set, NetworkPolicy, Istio ambient mode |
| 6 | Grafana Alloy with its Beyla eBPF component, Falco, Sigma with the pySigma Loki backend; the class Loki, Tempo, Prometheus and Grafana |
| 7 | GitHub Actions, gitleaks, Semgrep, Trivy, Checkov, ZAP baseline, Syft, Cosign, Conftest, Flux, Kyverno |
| Capstone | Caldera or Atomic Red Team |


### Rules

| Item | Rule |
| --- | --- |
| Repository | One private GitHub repository per student, cyb601-lastname, with a folder per lab plus Terraform and manifests. The instructor is a collaborator; commit history is evidence. |
| Rebuildable | Every lab must rebuild from a clean clone using its README. The instructor rebuilds a random sample each week. |
| Ethics and scope | Students attack only their own VM, and in the capstone only their assigned target. Touching the CloudStack host, bastion, class services or a classmate's VM is an integrity violation and may be illegal. A rules-of-engagement form is signed in Week 1 and re-signed before the capstone. |
| Internet use | Outbound access is for packages, images, GitHub and signature checks. Scanning any outside system is prohibited. |
| AI tools | Allowed for explaining errors and tool syntax. Analysis, risk ratings and conclusions are the student's own, and AI use is disclosed in the lab README. |
| Living plan | From Week 2, each student keeps SECURITY-PLAN.md in the repository and extends it after each lab. Week 8 assembles the final plan rather than starting it. |

## Week 1 – Foundations: thinking like a security engineer

Students learn Anderson's frame of policy, mechanism, assurance and incentives, then provision their own environment and map its attack surface. CLO 1.

**Readings**

- Anderson (2020), Ch. 1 "What Is Security Engineering?" and Ch. 2 "Who Is the Opponent?"
- NIST SP 800-160 Vol. 1 Rev. 1 (2022), Chapters 1–2
- Saltzer and Schroeder (1975), "The Protection of Information in Computer Systems", Section I.A.3 (design principles)

**Discussion.** Pick one breach from the last three years that has a public post-incident report. Was the root cause a software security failure, a system security failure or an incentive failure in Anderson's sense? Name the one Saltzer and Schroeder principle that would have changed the outcome, and the lifecycle stage where it should have been applied.

**Hands-on Lab 1: Provision, deploy and map the attack surface (40 pts, about 4 h)**

1. Write Terraform with the CloudStack provider, starting from the course module, that creates your VM from the course template; cloud-init installs k3s. Commit the code and the terraform plan output.
2. Deploy the reference system from the starter repository with Kustomize: Juice Shop, the customer API, Postgres and a pass-through placeholder for the WAF.
3. Inventory the system: namespaces, Services and ports, ServiceAccounts and RBAC bindings, images and data stores. Generate an SBOM for every image with Syft.
4. Scan your VM with nmap from the bastion, then from inside a pod. Explain why the two views differ.
5. Run kube-bench with its k3s benchmark and record the score as the baseline you will improve in Lab 4.
6. Draw a data-flow diagram with trust boundaries, entry points and assets, including the CloudStack API, the bastion and GitHub.
7. Rank the five assets an attacker would want most, and name the opponent from Anderson Ch. 2 who would want each.
8. Sign and commit the rules of engagement.

*Deliverable:* lab1/ with the Terraform code, data-flow diagram, inventory, SBOMs, kube-bench baseline and a one-page architecture risk memo.

**Additional assignment: Google Skills lab (20 pts).** *A Tour of Google Cloud Hands-on Labs.* Badge plus a 3–5 sentence reflection comparing Google Cloud's project and IAM boundaries with the CloudStack account and project boundaries you used in Lab 1.

## Week 2 – Secure SDLC and threat modeling

Students turn the Week 1 diagram into a threat model and derive testable security requirements from it. CLOs 2 and 3.

**Readings**

- Anderson (2020), Ch. 27 "Secure Systems Development"
- NIST SP 800-218, Secure Software Development Framework (current version)
- OWASP SAMM v2 model overview; OWASP ASVS (current version), chapter list and levels
- Threat Modeling Manifesto (2020)

**Discussion.** Microsoft SDL and OWASP SAMM prescribe practices; BSIMM measures what firms actually do. For a 15-developer startup shipping weekly, which would you adopt first, and which three practices would you start with? Support the choice with one breach where a missing SDLC practice was the documented cause.

**Hands-on Lab 2: STRIDE threat model, attack tree and risk register (40 pts, about 4 h)**

1. Rebuild the Lab 1 data-flow diagram in OWASP Threat Dragon.
2. Apply STRIDE per element. Record at least 15 threats, each with the asset affected and the trust boundary crossed.
3. Include at least three threats against the platform itself: the Kubernetes API, the CloudStack API keys and the CI pipeline.
4. Build one attack tree for the highest-value asset, at least three levels deep, with cost or skill on the leaves.
5. Rate each threat on a 5 × 5 likelihood-impact matrix and justify the top five ratings.
6. Write a testable security requirement for each of the top ten threats, mapped to an ASVS requirement and an SSDF practice.
7. Start SECURITY-PLAN.md with the system description and the top risks.

*Deliverable:* lab2/ with the Threat Dragon JSON and report, the attack tree and a risk register in CSV. The register is reused in Labs 5, 6 and 7, the capstone and the final plan.

**Additional assignment: Google Skills lab (20 pts).** *Implement Cloud Security Fundamentals on Google Cloud.* Badge plus reflection: which STRIDE categories do IAM roles and service accounts mitigate, and which do they not touch?

## Week 3 – Identity and access control engineering

Students break access control first, then rebuild it with a standards-based identity provider and least-privilege Kubernetes RBAC. CLO 5.

**Readings**

- Anderson (2020), Ch. 6 "Access Control"
- NIST SP 800-63B, Digital Identity Guidelines: Authentication (current revision), authenticator assurance levels
- NIST SP 800-162, Guide to Attribute Based Access Control, Sections 1–2
- ISO/IEC 27001:2022 Annex A controls 5.15–5.18 (access control, identity management, authentication information, access rights)
- OWASP Top 10, Broken Access Control
- Kubernetes documentation, "Role Based Access Control Good Practices"

**Discussion.** RBAC is the enterprise default, yet role explosion pushes large organizations toward ABAC or relationship-based models. Using one recent incident involving privilege escalation or over-permissioned identities, argue which model would have limited the damage. Include where DAC and MAC still fit.

**Hands-on Lab 3: OIDC, Kubernetes RBAC and broken access control (40 pts, about 4 h)**

1. Attack: from your laptop over the VPN, use ZAP or Burp to solve three Juice Shop broken-access-control challenges, such as viewing another user's basket, reaching the admin section or forging a JWT. Record the request, response and root cause for each.
2. Build: configure Keycloak in your cluster with a realm, two roles (customer, admin), two users and an OIDC client using authorization code flow with PKCE. Require a one-time-password second factor for the admin role.
3. Enforce: put oauth2-proxy in front of the customer API so that /admin needs the admin role. Show an allowed and a denied request with the decoded tokens.
4. Test the tokens: an expired token, a tampered signature and alg: none. Record what is rejected and why.
5. Platform identity: the starter repository grants one workload far more Kubernetes access than it needs. Find it, replace it with a least-privilege Role, and prove the change with kubectl auth can-i.
6. Write an access control matrix (subjects × resources × actions) covering application users and platform identities, and mark every cell that violates least privilege.

*Deliverable:* lab3/ with the Keycloak realm export (secrets removed), oauth2-proxy and RBAC manifests, attack write-ups and the access matrix.

**Additional assignment: Google Skills lab (20 pts).** *Managing Security in Google Cloud.* Badge plus reflection: how do Cloud Identity and the resource hierarchy express the matrix you wrote in Lab 3?

## Week 4 – Hardening, patching, cryptography and secrets

Students measure the host, the cluster and the images against baselines, fix them as code, and fix how the system handles keys and secrets. CLOs 4 and 5.

**Readings**

- Anderson (2020), Ch. 5 "Cryptography": read for what engineers must get right; the mathematics can be skimmed
- CIS Benchmarks for Ubuntu Linux and for Kubernetes, Level 1 (free with registration); the k3s CIS hardening guide
- Kubernetes documentation, "Pod Security Standards" and "Encrypting Confidential Data at Rest"
- NIST SP 800-40 Rev. 4 (2022), Guide to Enterprise Patch Management Planning
- U.S. House Committee on Oversight and Government Reform (2018), *The Equifax Data Breach*, executive summary and patch management findings

**Discussion.** Equifax had a patch policy and a vulnerability scanner, yet ran a known-vulnerable Apache Struts for months. Using the House report, identify the two process failures that mattered most. Then state a patch SLA you would defend for internet-facing systems, and what you would do when a patch cannot be applied in that time.

**Hands-on Lab 4: Host, cluster and image hardening, TLS and secrets (40 pts, about 5 h)**

Optional: snapshot your VM before you start, and delete the snapshot after you submit. Your one snapshot slot is needed before the capstone.

1. Host: scan your VM with OpenSCAP and the CIS profile for your Ubuntu release, or with Lynis. Record the score and the ten highest-risk findings.
2. Write an Ansible playbook that fixes at least eight of them, rescan, and show before and after.
3. Cluster: rerun kube-bench and fix at least five findings in the k3s configuration, including API audit logging (Lab 6 uses that log). Compare against your Lab 1 baseline.
4. Workloads: label the application namespaces to enforce the "restricted" Pod Security Standard. Fix every violation: non-root user, read-only root filesystem, all capabilities dropped, seccomp RuntimeDefault, no privilege escalation.
5. Images: scan the Juice Shop image with Trivy. Write a hardened Dockerfile on a minimal base, build it with the starter repository's build workflow, rescan and compare CVE counts.
6. Transport: issue certificates from an internal CA with cert-manager and terminate TLS at Traefik. Test with testssl.sh and remove every weak protocol and cipher it reports.
7. Secrets: run gitleaks over the repository history and remove the database password planted in the starter manifests. Move it into a Kubernetes Secret, turn on k3s secrets encryption, and show the stored value is ciphertext in the datastore. Then rotate the encryption key.

*Deliverable:* lab4/ with before-and-after reports for host, cluster and image; the playbook, k3s configuration diff, Dockerfile and namespace labels; the TLS report; the encryption evidence; and a list of findings deliberately not fixed, each with a reason.

**Summative Assessment 1: System Hardening Report (60 pts, 3–4 pages, APA 7).** A secure design specification for the hardened host, cluster and workloads, written from the Lab 4 evidence. Required sections: baselines chosen and why; defense-in-depth layers with the control at each; least-privilege decisions; before-and-after metrics; exceptions with compensating controls and a named risk owner; patch management policy with SLAs by severity; key and secret lifecycle, including rotation. CLO 4.

**Additional assignment: Google Skills lab (20 pts).** *Mitigating Security Vulnerabilities on Google Cloud.* Badge plus reflection linking vulnerability management to software resilience.

## Week 5 – Network and application defense, Zero Trust

Students put a WAF in front of the application and measure what it stops, then enforce Zero Trust inside the cluster with network policy and mesh identity. CLOs 2, 5 and 6.

**Readings**

- Anderson (2020), Ch. 21 "Network Attack and Defence" and Ch. 7 "Distributed Systems" (selected sections)
- NIST SP 800-207 (2020), Zero Trust Architecture, Sections 2–3
- Ward and Beyer (2014), "BeyondCorp: A New Approach to Enterprise Security"
- OWASP Top 10 (current edition)
- Istio documentation, ambient mode overview and "Ambient and Kubernetes NetworkPolicy"

**Discussion.** Critics say Zero Trust is a rebranding of least privilege and network segmentation. Using SP 800-207's tenets and one documented deployment, such as Google BeyondCorp or the U.S. federal strategy in OMB M-22-09, argue what is new and what is not. Then place a WAF, a NetworkPolicy and a mesh authorization policy in the 800-207 logical model, and say what each cannot see.

**Hands-on Lab 5: WAF, network policy and service-mesh mTLS (40 pts, about 5 h)**

1. Baseline: run a ZAP active scan plus manual SQL injection, XSS and path traversal against Juice Shop. Save the alerts.
2. WAF: replace the Lab 1 placeholder with Nginx, ModSecurity and the OWASP Core Rule Set in detection-only mode. Replay the attacks and read the audit log.
3. Switch to blocking at paranoia level 1, then 2. Record blocked attacks and false positives on normal shopping traffic at each level.
4. Tune: write one rule exclusion for a false positive, and one custom rule for a Lab 2 threat that the Core Rule Set misses.
5. Segment: apply default-deny NetworkPolicies to every application namespace, then allow only the flows in your data-flow diagram. Prove it with allowed and blocked requests from a test pod, and repeat the Lab 1 in-pod nmap scan.
6. Mesh: install Istio in ambient mode with the starter values file and enroll the application namespaces. Traffic will break until your NetworkPolicies allow port 15008; explain why.
7. Enforce strict mTLS, and write authorization policies so that only the API's service account can reach Postgres and only the WAF can reach Juice Shop. Prove each with an allowed and a denied request, and show the workload certificate on a live connection.

*Deliverable:* lab5/ with configs and policies, before-and-after scans, a table of blocked versus missed attacks, the policy test evidence, and the residual risks that a WAF and a mesh cannot address.

**Additional assignment: Google Skills lab (20 pts).** *Securing Your Network with Cloud Armor.* Badge plus reflection: compare Cloud Armor's preconfigured rules and bot management with your ModSecurity deployment. What does the managed service give you, and what control do you lose?

## Week 6 – Observability and detection engineering

Students send logs, metrics and traces from every layer to the class observability stack, add runtime detection, and measure how well their detections work. CLO 6.

**Readings**

- Anderson (2020), Ch. 28 "Assurance and Sustainability"
- NIST SP 800-137 (2011), Information Security Continuous Monitoring, Sections 2–3
- NIST SP 800-92, Guide to Computer Security Log Management (current revision), Sections 2–4
- MITRE ATT&CK Enterprise and Containers matrices: Initial Access, Credential Access and Exfiltration tactics
- Sigma rule specification (SigmaHQ) and the Falco rules documentation

**Discussion.** In the 2013 Target breach the malware alerts fired and nobody acted; in 2020 FireEye found the SolarWinds intrusion from a single unusual device registration. Using both cases, argue whether the harder engineering problem is collecting telemetry or acting on it. Propose two metrics a security lead should report monthly to show detection is improving.

**Hands-on Lab 6: Telemetry, runtime detection and ATT&CK-mapped rules (40 pts, about 5 h)**

1. Logs: with Grafana Alloy, ship application logs, Traefik access logs, ModSecurity audit logs, Keycloak authentication events and the Lab 4 Kubernetes audit log to your tenant in the class stack.
2. Metrics and traces: enable Alloy's Beyla component for Juice Shop and the customer API. Send RED metrics to the class Prometheus and traces to the class Tempo, then follow one admin API request from its trace to its log lines.
3. Runtime: deploy Falco with its modern eBPF probe. Enable or write rules for a shell spawned in a container, an unexpected outbound connection, a write below a read-only path and a read of a service-account token.
4. Detections: pick five threats from the Lab 2 register. Write each as a Sigma rule tagged with its ATT&CK technique, convert it to LogQL with the pySigma Loki backend, and load it as an alert in your tenant. At least one detection must use a metric or trace signal.
5. Replay the Lab 3 and Lab 5 attacks plus 10 minutes of normal traffic. For each rule, record fired or missed, time to alert and false positives.
6. Build one dashboard with four panels an on-call engineer would need, and write a half-page triage runbook for the noisiest alert.
7. Threat-model your own telemetry: Alloy and Falco run with elevated privileges. Add their risks and mitigations to your register.

*Deliverable:* lab6/ with the Sigma rules and generated LogQL, Falco rules, Alloy configuration, dashboard JSON, a detection coverage table (threat, technique, rule, result) and the runbook.

**Additional assignment: Google Skills lab (20 pts).** *Logging and Monitoring in Google Cloud.* Badge plus reflection on which of your detections could be built with Cloud Logging and log-based alerts, and which assurance metrics you would report.

## Week 7 – DevSecOps and software supply chain security

Students build a pipeline that ships only scanned, signed and inventoried images, and a cluster that refuses anything else. CLOs 6 and 7.

**Readings**

- Anderson (2020), Ch. 8 "Economics" (why vendors ship insecure software and who pays)
- OWASP DevSecOps Guideline, pipeline stages overview
- SLSA specification (current version), build track levels
- NIST SP 800-204D (2024), Strategies for the Integration of Software Supply Chain Security in DevSecOps CI/CD Pipelines, Sections 3–5
- Sigstore documentation on keyless signing; Kyverno documentation on image verification
- Case files: SolarWinds (2020), Log4Shell (2021), the xz-utils backdoor (2024)

**Discussion.** SolarWinds compromised the build, Log4Shell was a flaw in a trusted dependency, and xz-utils was a maintainer takeover. For each, state which control (SAST, SCA, SBOM, signing, provenance, admission control, reproducible builds) would have prevented or shortened it, and which would have done nothing. Conclude with what automation cannot fix.

**Hands-on Lab 7: Signed pipeline, GitOps and admission control (40 pts, about 5 h)**

1. Extend the starter build workflow in GitHub Actions to run on every pull request, in this order: gitleaks, Semgrep (SAST), Trivy filesystem scan (dependencies), Checkov on your Terraform and Kubernetes manifests (IaC), image build, Trivy image scan, and a ZAP baseline against the container in the runner (DAST).
2. Generate a CycloneDX SBOM with Syft and attach it to the image as an attestation.
3. Sign the image and the SBOM attestation with Cosign keyless signing. Show cosign verify passing, then failing on a tampered image.
4. Pipeline policy: write a Conftest (Rego) policy that fails the build on any critical CVE, a root user or a missing SBOM. Show one red build and one green build.
5. GitOps: install Flux in your cluster and point it at your repository. The pipeline updates the image digest by pull request and Flux deploys it, so nothing connects into the cluster.
6. Admission: install Kyverno. Write policies that admit only images signed by your repository's workflow, require a non-root user and resource limits, and generate a default-deny NetworkPolicy in every new namespace. Show one deployment rejected and one admitted.
7. Triage: from all findings, pick ten, classify each as true positive, false positive or accepted risk, and fix at least three in code.

*Deliverable:* lab7/ with the workflow YAML, Rego and Kyverno policies, the SBOM, Flux configuration, links to the red and green runs, rejection evidence and the triage table.

**Summative Assessment 2: DevSecOps Risk and Verification Report (60 pts, 4–5 pages, APA 7).** An evidence-backed assessment of the Lab 7 pipeline and deployment path. Required sections: pipeline architecture and trust boundaries; verification coverage by stage against SSDF practices, including what IAST would add; findings triage with risk ratings; SBOM analysis (riskiest components, license issues); artifact integrity, the SLSA build level achieved and the gap to the next; the admission gate as the last line of defense; residual risk and a 90-day improvement plan. CLO 7.

**Additional assignment: Google Skills lab (20 pts).** *Using DevSecOps in Your Google Cloud Environment.* Badge plus reflection comparing Artifact Registry scanning and Binary Authorization with your Trivy, Cosign and Kyverno gates.

## Week 8 – Governance, risk communication and the red/blue capstone

Students test their systems against a live red team, then assemble seven weeks of evidence into one plan and defend it to two audiences. CLO 8, integrating CLOs 1–7.

**Readings**

- Anderson (2020), Ch. 27, sections on organizational issues (re-read), and Ch. 29 "Beyond 'Computer Says No'"
- NIST Cybersecurity Framework 2.0 (2024), the Govern function
- ISACA, COBIT 2019 Framework: Introduction and Methodology, governance overview
- ISO/IEC 27001:2022, Clauses 5, 6 and 9 (leadership, planning, performance evaluation)

**Discussion.** You have budget for one of three things next year: a second detection engineer, a developer security champions program, or a commercial tool replacing your open-source pipeline scanners. Choose one for the case-study retailer and defend it to the CFO in plain language, with one metric that would prove you right in 12 months. Use your capstone results as evidence.

### Capstone: red team versus blue team (Week 8 live session; counts as Week 8 participation, 20 pts)

- **Targets.** Twenty-four hours before the session, the instructor randomly picks one member's cluster per blue team and restores it from that student's pre-capstone snapshot. Everyone therefore keeps their own cluster ready.
- **Teams and rounds.** The class forms four teams of three or four. Two engagements run in parallel, then teams swap roles, so every student plays red once and blue once. Each round lasts 60 minutes, followed by a 30-minute joint debrief.
- **Red starting point.** The public URL, one low-privilege shop account, and a shell in a pod in the shop namespace, simulating a compromised dependency.
- **Red objectives.** Read mock customer records in Postgres, steal a service-account token, persist or escalate in the cluster, and exfiltrate data, using Caldera or Atomic Red Team plus manual technique. Every action is logged with a timestamp and its ATT&CK technique.
- **Blue rules.** Detect, contain and evict using only the controls built in Labs 3–7. Policies may change during the round; new tools may not be installed.
- **Out of scope.** Denial of service, the CloudStack host and its API, the bastion, the class services and any cluster not assigned as a target. Rules of engagement are re-signed before the session.
- **Scoring.** Red scores on objectives reached and time taken. Blue scores on time to detect, time to contain and detection coverage. The debrief reconciles the red timeline against the blue team's alerts to find the gaps.
- **Cannot attend live.** Within 48 hours, replay the red team's recorded Caldera operation against your own cluster and reconcile it against your alerts, for the same participation points.

Within 48 hours, each student writes a one-page after-action note: what their side did, what worked, what the other side did that they missed, and the two highest-priority fixes. It becomes the capstone section of the final plan.

### Final Project: Comprehensive Software and Systems Security Plan and Presentation (140 pts)

The plan covers the student's own system as built in Labs 1–7 and tested in the capstone. It is assembled from SECURITY-PLAN.md, which students have extended since Week 2.

*Part A, written plan (8–10 pages plus evidence appendix, APA 7, 90 pts)*

- Executive summary: one page, no jargon, top three risks and the ask
- System architecture and threat model summary
- Software lifecycle controls mapped to SSDF and SAMM
- IAM model and network defense
- Monitoring and incident detection, with Lab 6 coverage results
- DevSecOps automation and supply chain integrity
- Governance and compliance: control mapping to NIST CSF 2.0 and ISO/IEC 27001:2022 Annex A
- Residual risk register with owners, and a 12-month improvement roadmap with metrics
- Capstone findings (20 of the 90 points): the after-action note, the reconciled timeline and the changes it drove
- Appendix: a link to repository evidence for every claim

*Part B, presentation and defense (10–12 slides, 50 pts)*

- A 12-minute presentation in two parts: 4 minutes to an executive audience (risk, cost, decision needed) and 8 minutes to a technical audience (architecture, evidence, trade-offs).
- Then 5 minutes of questions from the instructor and two assigned peer reviewers. Students who present by recording answer written questions within 48 hours.

Submit both parts to the LMS by Sunday of Week 8.

## References

All readings except ISO/IEC 27001 and COBIT are free online. Confirm edition and section numbers against current versions before each term.

- Anderson, R. (2020). *Security engineering: A guide to building dependable distributed systems* (3rd ed.). Wiley. Author's page
- Center for Internet Security. (n.d.). *CIS Benchmarks*. cisecurity.org/cis-benchmarks
- Chandramouli, R., Kautz, F., & Torres-Arias, S. (2024). *Strategies for the integration of software supply chain security in DevSecOps CI/CD pipelines* (NIST SP 800-204D). doi.org/10.6028/NIST.SP.800-204D
- Dempsey, K., Chawla, N. S., Johnson, A., Johnston, R., Jones, A. C., Orebaugh, A., Scholl, M., & Stine, K. (2011). *Information security continuous monitoring (ISCM) for federal information systems and organizations* (NIST SP 800-137). doi.org/10.6028/NIST.SP.800-137
- Hu, V. C., Ferraiolo, D., Kuhn, R., Schnitzer, A., Sandlin, K., Miller, R., & Scarfone, K. (2014). *Guide to attribute based access control (ABAC) definition and considerations* (NIST SP 800-162). doi.org/10.6028/NIST.SP.800-162
- International Organization for Standardization. (2022). *Information security, cybersecurity and privacy protection: Information security management systems: Requirements* (ISO/IEC 27001:2022).
- ISACA. (2018). *COBIT 2019 framework: Introduction and methodology*.
- Kent, K., & Souppaya, M. (2006). *Guide to computer security log management* (NIST SP 800-92). doi.org/10.6028/NIST.SP.800-92 (check for a newer revision)
- MITRE. (n.d.). *ATT&CK Enterprise and Containers matrices*. attack.mitre.org
- National Institute of Standards and Technology. (2024). *The NIST Cybersecurity Framework (CSF) 2.0* (NIST CSWP 29). doi.org/10.6028/NIST.CSWP.29
- National Institute of Standards and Technology. *Digital identity guidelines: Authentication and authenticator management* (NIST SP 800-63B, current revision).
- Office of Management and Budget. (2022). *Moving the U.S. government toward zero trust cybersecurity principles* (M-22-09).
- Open Source Security Foundation. (n.d.). *SLSA specification*. slsa.dev
- OWASP Foundation. (n.d.). *Application Security Verification Standard*. owasp.org
- OWASP Foundation. (n.d.). *DevSecOps Guideline*. owasp.org
- OWASP Foundation. (n.d.). *OWASP Top 10*. owasp.org
- OWASP Foundation. (n.d.). *Software Assurance Maturity Model v2*. owaspsamm.org
- Rose, S., Borchert, O., Mitchell, S., & Connelly, S. (2020). *Zero trust architecture* (NIST SP 800-207). doi.org/10.6028/NIST.SP.800-207
- Ross, R., Winstead, M., & McEvilley, M. (2022). *Engineering trustworthy secure systems* (NIST SP 800-160 Vol. 1 Rev. 1). doi.org/10.6028/NIST.SP.800-160v1r1
- Saltzer, J. H., & Schroeder, M. D. (1975). The protection of information in computer systems. *Proceedings of the IEEE, 63*(9), 1278–1308.
- SigmaHQ. (n.d.). *Sigma rule specification*. sigmahq.io
- Souppaya, M., & Scarfone, K. (2022). *Guide to enterprise patch management planning* (NIST SP 800-40 Rev. 4). doi.org/10.6028/NIST.SP.800-40r4
- Souppaya, M., Scarfone, K., & Dodson, D. (2022). *Secure software development framework (SSDF) version 1.1* (NIST SP 800-218). doi.org/10.6028/NIST.SP.800-218 (check for a newer revision)
- Threat Modeling Manifesto Working Group. (2020). *Threat modeling manifesto*. threatmodelingmanifesto.org
- U.S. House of Representatives, Committee on Oversight and Government Reform. (2018). *The Equifax data breach* (Majority staff report, 115th Congress).
- Ward, R., & Beyer, B. (2014). BeyondCorp: A new approach to enterprise security. *;login:, 39*(6), 6–11.

**Platform documentation used in the labs**

- Kubernetes: Pod Security Standards, RBAC good practices, encrypting data at rest. kubernetes.io/docs
- K3s: CIS hardening guide and secrets encryption. docs.k3s.io
- Istio: Ambient and Kubernetes NetworkPolicy
- Grafana Alloy: beyla.ebpf component
- Grafana: pySigma Loki backend
- Apache CloudStack: Terraform provider 0.6.0 release
- Kyverno, Sigstore and Falco documentation: kyverno.io/docs, docs.sigstore.dev, falco.org/docs

Lab tools: OWASP Juice Shop, Threat Dragon, ZAP, Keycloak, oauth2-proxy, OpenSCAP, Lynis, Ansible, kube-bench, Trivy, cert-manager, testssl.sh, gitleaks, ModSecurity with the OWASP Core Rule Set, Istio, Grafana Alloy, Falco, Semgrep, Checkov, Syft, Cosign, Conftest, Flux, Kyverno, Caldera and Atomic Red Team. All are free for coursework and open source, except Burp Suite Community (free) and Terraform (source-available; OpenTofu is a compatible open-source alternative).