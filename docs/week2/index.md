# Week 2 – Secure SDLC and threat modeling

Week 2 shows how secure software gets built and has students turn their Week 1 diagram into a threat model, an attack tree and a risk register that every later lab reuses. CLOs 2 and 3.

## Assignments

Week 2 has three pieces of work: the discussion, the Google Skills lab and Lab 2. All are due by Sunday at 11:59 p.m. Pacific, except the primary discussion post, due Thursday.

### Discussion (20 pts)

**Prompt.** Microsoft SDL and OWASP SAMM prescribe practices; BSIMM measures what firms actually do. For a 15-developer startup shipping weekly, which would you adopt first, and which three practices would you start with? Support the choice with one breach where a missing SDLC practice was the documented cause.

**What a full-marks post does**

- Takes a clear position, and explains why the other two models fit a 15-person team shipping weekly less well.
- Names three concrete starting practices, each tied to an SSDF practice ID, such as PW.7 for code review.
- Supports the choice with one breach whose public report documents the missing practice, with a link.
- Cites at least one primary source beyond Anderson, in APA 7, in at least 200 words.
- Replies to two classmates (50 words or more each) with evidence that supports or challenges their choice.

### Google Skills lab (20 pts)

*Implement Cloud Security Fundamentals on Google Cloud.* Upload the badge or summary screen showing your name, plus 3–5 sentences: which STRIDE categories do IAM roles and service accounts mitigate, and which do they not touch? Give one example of each from your Lab 2 register.

### Lab 2 submission (40 pts)

- Push your work and the tag `lab2-submit`.
- Upload a 2-page lab report PDF: the model's scope, your five highest-rated threats with their justification, the attack tree's cheapest path, and what you would now change in your Week 1 diagram.
- Grading uses the engineering lab rubric. Here, "working implementation" means the Threat Dragon model opens from your repository and the CSV loads cleanly.

## Hands-on Lab 2: STRIDE threat model, attack tree and risk register (40 pts, about 4 h)

You turn the Week 1 data-flow diagram into a STRIDE threat model, build one attack tree, rate the threats, and write the requirements and risk register the rest of the course reuses.

**Before you start:** your Lab 1 data-flow diagram, OWASP Threat Dragon (desktop app or the version your instructor hosts), and the ASVS 5.0 and SSDF 1.1 reference PDFs open.

### Part A – Rebuild the diagram in Threat Dragon (about 30 min)

Recreate your Lab 1 diagram in Threat Dragon: processes, data stores, external entities, data flows and trust boundaries. Save `lab2/model.json` in your repository. If you drew Lab 1 in Threat Dragon already, copy it here and refine it.

### Part B – STRIDE per element (about 75 min)

Work element by element. For each, ask which STRIDE categories apply using the per-element guide from the slides, and record at least 15 threats total. Threat Dragon stores them on the diagram; you will also export them in Part F.

For each threat, record the element, the STRIDE letter, the asset at risk, the trust boundary crossed, and a one-line description. At least three threats must target the platform: the Kubernetes API, the CloudStack API keys and the CI pipeline.

Worked example, to set the level of detail:

| Element | STRIDE | Asset | Boundary crossed | Threat |
| --- | --- | --- | --- | --- |
| Customer to API flow | T | Order data | Internet to application | A manipulated request alters an order's price in transit |
| Postgres store | I | Customer records | Application to data | A reused database password lets a compromised pod read all records |
| API process | E | Admin functions | Within the application | A forged JWT grants a customer the admin role |

### Part C – Attack tree (about 45 min)

Build one attack tree for your highest-value asset, at least three levels deep, with a cost or skill estimate on each leaf. Draw it in Threat Dragon, draw.io or by hand, and save it as `lab2/attack-tree.png`. Identify the cheapest path and name it in your report.

Here is a worked example for the goal of reading customer records. Yours must target your own highest-value asset and reflect your own architecture.

[embed: node/d97a6382-815a]

The red leaf marks the cheapest path. Even with strong database controls, a phished admin who also defeats MFA reaches the data, which is why Lab 3 requires a second factor on the admin role.

### Part D – Rate the threats (about 30 min)

Rate every threat on a 5 × 5 likelihood-impact matrix. Use the course bands: 1–4 low, 5–9 medium, 10–15 high, 16–25 critical. Write a sentence of justification for your top five. Keep the ratings in the register you build next.

### Part E – Requirements for the top ten (about 30 min)

For your ten highest-rated threats, write one testable security requirement each, in the form "The system shall …; verified by …". Map each to an ASVS 5.0 requirement and an SSDF practice. Save them as `lab2/requirements.md`.

| Example requirement | ASVS | SSDF |
| --- | --- | --- |
| The API shall reject any token whose signature does not verify against the realm key; verified by the Lab 3 tampered-token test | V9.2 | PW.4 |
| Every request to /admin shall be authorized server-side against the caller's role; verified by the Lab 3 denied-request evidence | V8.2 | PW.7 |
| Only the API's service account shall connect to Postgres; verified by the Lab 5 network-policy test | V8.2 | PW.9 |

### Part F – Risk register and plan (about 30 min)

Export the model report from Threat Dragon to `lab2/threat-dragon-report.pdf`. Then write `lab2/risk-register.csv` with one row per threat and this header:

```
id,element,stride,asset,boundary,description,likelihood,impact,score,band,requirement_id,asvs,ssdf,status
```

Finally, extend `SECURITY-PLAN.md` from Week 1 with a threat-model summary: the top five risks, the cheapest attack path, and the requirements that address them.

```bash
git add lab2/ SECURITY-PLAN.md
git commit -m "Lab 2: STRIDE model, attack tree, risk register"
git push
git tag lab2-submit
git push origin lab2-submit
```

### Deliverables checklist

- `lab2/model.json` and `lab2/threat-dragon-report.pdf`
- At least 15 threats, including three against the platform
- `lab2/attack-tree.png` with costs on the leaves and the cheapest path named in the report
- `lab2/risk-register.csv` with every threat rated and banded
- `lab2/requirements.md` with ten requirements mapped to ASVS and SSDF
- `SECURITY-PLAN.md` updated; tag `lab2-submit` pushed and report uploaded