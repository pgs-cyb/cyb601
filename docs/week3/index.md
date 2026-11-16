# Week 3 – Identity and access control engineering

Week 3 has students break access control first, then rebuild it with a standards-based identity provider and least-privilege Kubernetes RBAC. CLO 5.
## Assignments

Week 3 has three pieces of work: the discussion, the Google Skills lab and Lab 3. All are due by Sunday at 11:59 p.m. Pacific, except the primary discussion post, due Thursday.

### Discussion (20 pts)

**Prompt.** RBAC is the enterprise default, yet role explosion pushes large organizations toward ABAC or relationship-based models. Using one recent incident that involved privilege escalation or over-permissioned identities, argue which model would have limited the damage. Include where DAC and MAC still fit.

**What a full-marks post does**

- Summarizes a real incident with a public account, and identifies the access-control failure.
- Argues for one model with reasons tied to the incident, not a generic comparison.
- Says where DAC and MAC still make sense, with an example of each.
- Cites at least one primary source beyond Anderson, in APA 7, in at least 200 words.
- Replies to two classmates (50 words or more each), engaging with their model choice.

### Google Skills lab (20 pts)

*Managing Security in Google Cloud.* Upload the badge or summary screen showing your name, plus 3–5 sentences: how do Cloud Identity and the resource hierarchy express the access-control matrix you wrote in Lab 3? Which parts of your matrix map cleanly, and which do not?

### Lab 3 submission (40 pts)

- Push your work and the tag `lab3-submit`. Remove client secrets and passwords from the Keycloak export before committing.
- Upload a 2-page lab report PDF: the three attacks with request, response and root cause; what each token test rejected and why; the RBAC change with its `kubectl auth can-i` evidence; and the least-privilege violations in your matrix.
- Grading uses the engineering lab rubric.

## Hands-on Lab 3: OIDC, Kubernetes RBAC and broken access control (40 pts, about 4 h)

You exploit broken access control in Juice Shop, rebuild authentication and authorization with Keycloak and oauth2-proxy, test what the tokens reject, then tighten the cluster's own RBAC.

**Before you start:** your cluster from Lab 1 running with the port 6443 tunnel up; ZAP or Burp Suite; the Lab 3 starter manifests in `app/identity/`; the hostnames your instructor publishes.

### Part A – Attack (about 45 min)

With the shop open through your browser and proxied in ZAP or Burp, solve three broken-access-control challenges. Suggested set: view another user's basket, reach the admin section, and forge or tamper with a JWT. For each, save the request and response and write the root cause.

```
lab3/attacks/basket-idor.md
lab3/attacks/admin-access.md
lab3/attacks/jwt-forgery.md
```

Each file: what you changed, the response that proved success, and the one server-side check that was missing.

### Part B – Build the identity provider (about 60 min)

Deploy Keycloak from the starter manifests, then configure a realm. Do this through the admin console, then export it so it is reproducible.

1. Create realm `shop`.
2. Create realm roles `customer` and `admin`.
3. Create users `alice` (customer) and `bob` (admin), each with a password.
4. Require OTP for the `admin` role (Authentication, then Required actions, then bind to the role), so bob sets up an authenticator app at first login. This is the second factor your Lab 2 attack tree called for.
5. Create an OIDC client `shop-proxy`: confidential, standard flow on, with PKCE required (Proof Key for Code Exchange set to S256).
6. Export the realm to `lab3/keycloak/realm-export.json`, then delete the client secret and any credentials from the file before committing.

### Part C – Enforce at the proxy (about 45 min)

Put oauth2-proxy in front of the customer API so that reaching `/admin` requires the `admin` role. The starter `app/identity/oauth2-proxy.yaml` has the gaps marked. Here is the flow you are building.

[embed: node/03ae8d41-aecb]

The proxy, not the API, talks to Keycloak and checks the role. The API only ever sees requests the proxy has already authorized.

Show two results and save them:

- `lab3/proxy/allowed.txt`: bob reaching `/admin` after login, with the decoded ID token (claims only, signature removed).
- `lab3/proxy/denied.txt`: alice getting 403 at `/admin`, with her decoded token showing only the customer role.

Decode tokens with a local tool; never paste a real token into a website.

### Part D – Test the tokens (about 20 min)

Against the proxy, try three bad tokens and record what happens and why:

| Test | Expected result |
| --- | --- |
| A token past its expiry | Rejected: expired |
| A token with one claim changed and the old signature | Rejected: signature does not verify |
| A token re-signed with alg: none | Rejected: algorithm not allowed |

Save the three requests and responses to `lab3/tokens/`.

### Part E – Fix Kubernetes RBAC (about 30 min)

Lab 1 flagged a workload whose ServiceAccount has more cluster access than it needs. Confirm it, then fix it.

```bash
# Confirm the over-permissive binding
kubectl get clusterrolebindings,rolebindings -A -o wide | grep -i SERVICE_ACCOUNT
kubectl auth can-i --list --as=system:serviceaccount:NS:SERVICE_ACCOUNT
```

Replace the broad binding with a Role granting only what the workload needs, in `lab3/rbac/least-privilege.yaml`. Apply it, then prove the change:

```bash
kubectl apply -f lab3/rbac/least-privilege.yaml
# Before: can-i showed secrets/list etc.; after: only what it needs
kubectl auth can-i list secrets --as=system:serviceaccount:NS:SERVICE_ACCOUNT   # want: no
kubectl auth can-i get configmaps --as=system:serviceaccount:NS:SERVICE_ACCOUNT # want: yes, if required
```

Save the before-and-after `can-i` output to `lab3/rbac/can-i.txt`.

### Part F – Access-control matrix (about 20 min)

Write `lab3/access-matrix.md` with subjects as rows and resources as columns, covering application users (anonymous, customer, admin) and platform identities (the API ServiceAccount, your kubeconfig). Put the allowed action in each cell, and mark with an asterisk every cell that still breaks least privilege, with a note on your plan for it.

```bash
git add lab3/ app/identity/
git commit -m "Lab 3: access-control attacks, Keycloak with MFA, oauth2-proxy, RBAC fix"
git push
git tag lab3-submit
git push origin lab3-submit
```

### Deliverables checklist

- `lab3/attacks/` with three write-ups, each naming the missing server-side check
- `lab3/keycloak/realm-export.json`, secrets removed, with OTP required for admin
- `lab3/proxy/allowed.txt` and `denied.txt` with decoded tokens
- `lab3/tokens/` with the three rejected tokens
- `lab3/rbac/least-privilege.yaml` and `can-i.txt` showing before and after
- `lab3/access-matrix.md`; tag `lab3-submit` pushed and report uploaded

### Troubleshooting

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| Keycloak pod stuck pending | 6 GB VM under load | Scale the shop deployment to zero while configuring Keycloak, then back up |
| Login loops back to the login page | Redirect URI mismatch | Match the client's redirect URI to the proxy's callback URL exactly |
| The proxy returns 500 after login | Wrong issuer URL or client secret | Check the proxy's OIDC issuer matches the realm's, and the secret matches the client |
| JWT forgery challenge will not solve | Testing against the hardened proxy, not Juice Shop | Part A attacks Juice Shop directly; the proxy is what you build in Part C |
| `can-i --as` always says yes | You are testing as a cluster-admin user, not the ServiceAccount | Use the full `system:serviceaccount:NS:NAME` form |

### Expected results

- Part A: all three challenges are solvable in an unmodified Juice Shop. The JWT challenge relies on a weak or omitted signature check.
- Part C: alice gets 403 at `/admin`; bob succeeds only after completing OTP.
- Part D: all three bad tokens are rejected once the proxy verifies issuer, signature and algorithm.
- Part E: after the fix, the ServiceAccount can no longer list secrets, and the application still runs.
### Problems to expect

- Keycloak slow to start on a loaded VM: suggest scaling the shop to zero during configuration.
- Students decoding tokens on jwt.io: remind them not to paste real tokens into websites; provide a local decode snippet.
- OTP confusion: bob must scan the authenticator QR at first login; if skipped, reset the user's required action in Keycloak.