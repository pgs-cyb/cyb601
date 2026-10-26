
In this 3-unit, 8-week graduate course, each student secures one real system from design to production, running on Kubernetes in the department's CloudStack cloud.

| Item | Detail |
| --- | --- |
| Course number and name | CYB 601 – Software and Systems Security |
| Units | 3 |
| Length of class | 8 weeks |
| Instructional method | Online / On-Campus |
| Expected workload | 12–16 hours per week: 3 h class, 3 h reading, 4–5 h lab, 1 h Google Skills lab, 2–3 h discussion and writing. Weeks 4, 7 and 8 run heavier. |
| Recommended preparation | Linux command line, Git, one programming language, networking fundamentals. Week 0 covers SSH, the course VPN, kubectl, Terraform and Kubernetes basics. |

**Description.** This course treats security as an engineering discipline: students reason about adversaries, design controls, build them and prove they work. Each student provisions a Kubernetes environment with Terraform on the department's CloudStack cloud, then secures one reference system for eight weeks. They threat-model it, engineer identity and cryptographic controls, and harden the host, the cluster and the container images. They enforce Zero Trust segmentation with a web application firewall, network policy and service-mesh mTLS, and write detections mapped to MITRE ATT&CK. They then build a pipeline that ships only signed, inventoried artifacts. Google Cloud Security Engineer labs run alongside, showing the same controls as managed services. The course ends with a red-team versus blue-team exercise and a security plan defended to technical and executive audiences.

**Required text.** Anderson, R. (2020). *Security Engineering: A Guide to Building Dependable Distributed Systems* (3rd ed.). Wiley. Every chapter is free on the author's site; the Perlego edition (eBook ISBN 9781119642817) is optional.

**Required standards and papers.** NIST SP 800-160 Vol. 1 Rev. 1, SP 800-218 (SSDF), SP 800-63B, SP 800-162, SP 800-207, SP 800-40 Rev. 4, SP 800-92, SP 800-137 and SP 800-204D; OWASP SAMM, ASVS and Top 10; SLSA; MITRE ATT&CK; NIST CSF 2.0. All are free except ISO/IEC 27001 and COBIT, which the library provides. Full list under References.

**Required tools.** A laptop with a browser, SSH client, Git, VS Code, kubectl and a WireGuard client for the course VPN. OWASP ZAP or Burp Suite Community for the web attacks. A GitHub account and a Google account. Docker Desktop is not required: images build in GitHub Actions and run in the student's own cluster.