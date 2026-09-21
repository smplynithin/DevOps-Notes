# Thermo Fisher — Engineer II, DevOps Engineering — Combined Interview Prep

This maps the actual JD to likely questions, with answers built from your CV/project background. It's meant to sit alongside your existing prep docs:
- `Srinithin_Interview_Question_Bank.md` (question list by CV section)
- `Srinithin_Interview_QA_Prep.md` (full answers, CV + HCL project deep dive)
- `Srinithin_CV_QA_with_Followups.md` (main Q + follow-up chains)

Those three still cover all your **project deep-dive** questions (4G platform, Xitadel, IAM/RBAC/Terraform internals) in full — this doc adds the **JD-specific layer**: what this particular role will probe for, including a few real gaps worth preparing an honest answer for.

---

## 0. Role Reality Check — Read This First

A few things about this JD are worth noting honestly before you walk in, because they'll shape how the conversation goes:

1. **This is an Engineer II role supporting product teams, not a platform-owner role.** The JD repeatedly says "under the guidance of senior engineers and technical leads" and "support product teams." Your CV reads as more senior/autonomous (owned a 76-node platform, built org-wide frameworks) than this role's day-to-day scope. Don't undersell yourself, but expect questions probing whether you're comfortable in a *supporting*, less architecturally-autonomous role after running your own platform.
2. **Their toolchain differs from yours in specific spots**: they want GitHub/GitLab (you're strong in GitHub, plus GitHub Actions on your CV) and either GitHub Actions or GitLab CI/CD as the CI/CD platform — your deepest CI/CD experience is Jenkins, not GitHub Actions/GitLab CI, even though it's listed on your CV. Be ready to speak to it honestly at the level you actually know it, and bridge from Jenkins concepts (stages, gates, agents) rather than overclaiming hands-on GitHub Actions depth.
3. **Two application tracks are mentioned — desktop (CMake, C++/Qt, PowerShell, Windows) and web (Kubernetes, Helm, Rancher).** You have zero CMake/C++/Qt experience — the **web track is your clear fit** given EKS/Kubernetes/Helm. Rancher itself isn't on your CV; be ready to say you haven't used Rancher specifically but understand it as a Kubernetes management layer (multi-cluster ops, RBAC, app catalog) and your EKS/Helm experience transfers directly to learning it.
4. **They explicitly want approved GenAI tool usage** ("Use approved AI-assisted engineering tools") — your CV lists GitHub Copilot, which is a direct, genuine match. Lean into this.
5. **3-6 years experience band, you're at ~3.5-4.7 depending how you count** — comfortably in range.

---

## 1. Questions Directly From the JD's Responsibilities

**Q: This role is described as working "under the guidance of senior engineers and technical leads" — how do you feel about that after running more autonomous platform work at HCL?**
A: "I'm genuinely comfortable with that — at HCL I had a lot of ownership, but I was also embedded with cloud architects and senior engineers on VPC design, org-wide IAM, and DR policy, so working within a structured, guided team isn't new to me. I'd actually see it as a chance to learn a different stack (GitHub/GitLab-native tooling, Rancher, potentially the desktop track) from people who've built deep expertise in it, while still contributing the automation and troubleshooting discipline I've built."

- Follow-up: *"Have you ever had to defer to a decision you disagreed with?"* → Use a real, low-drama example; emphasize you raised your view with data, then executed the team's decision professionally.

**Q: Tell me about a time you supported another team's build, test, or deployment issue rather than your own.**
A: Draw from real HCL/Xitadel examples — e.g., helping an application team debug a failed Helm deploy or a broken pipeline stage during migration/onboarding work. If you don't have a clean example, use the "DevOps engineers attended application team ceremonies during active migration/onboarding to unblock fast" pattern from your project as the shape of an answer, but make it concrete to something you actually did.

**Q: Walk me through how you'd investigate a failed deployment or pipeline run, step by step.**
A: "First check the pipeline logs for where exactly it failed — build, test, scan, or deploy stage. If it's a build/test failure, reproduce locally or in a lower environment. If it's a deployment failure, check `kubectl describe`/pod events for scheduling or readiness issues, and check whether it's an infra problem (node capacity, IRSA/permissions) vs. an application problem (bad config, failed migration). I document the root cause and the fix, and if it's a systemic issue, I raise a corrective action so it doesn't recur — not just a one-off patch."

- Follow-up: *"Give me a real example."* → Use your strangler-fig migration story, or a specific Jenkins/Trivy/SonarQube gate failure you've actually debugged.

**Q: What experience do you have with GitHub or GitLab repository administration — branch protection, runners/agents, artifact management?**
A: Speak to what you've actually configured — branch protection rules (required PR approvals, required status checks), and be honest that your artifact/registry experience is mainly Amazon ECR/Nexus rather than GitHub Packages/GitLab Container Registry specifically, but the underlying concepts (image tagging, retention, access scoping) transfer directly.

**Q: Have you set up or maintained GitHub Actions or GitLab CI/CD pipelines?**
A: Be honest about depth here. If your GitHub Actions experience is lighter than Jenkins: "My deepest CI/CD experience is Jenkins — multi-stage pipelines with SAST/dependency/image scanning gates, shared libraries, GitOps handoff to ArgoCD. I've used GitHub Actions for smaller/repo-native workflows. The core concepts — stages, gates, secrets injection, artifact publishing — map directly; I'd expect to be productive in GitHub Actions/GitLab CI quickly given that foundation."

**Q: Describe your experience with automated code, dependency, container, and secrets scanning.**
A: This is a strong match — walk through SonarQube (code/SAST), a dependency/SCA scanner (CVSS-based blocking), Trivy (container image scanning, blocking on HIGH/CRITICAL), and Gitleaks (secrets scanning) — all real tools on your CV and in your pipeline.

- Follow-up: *"What's your process when a scan finds a vulnerability — do you fix it yourself or hand it off?"*
  A: Depends on the finding — a dependency bump or base-image update you'd typically fix directly; something requiring an application-code change you'd raise to the owning developer/team with the scan detail and severity, tracked to resolution rather than just flagged and forgotten.

**Q: What upgrade, migration, or patching activities have you been part of?**
A: EKS minor-version upgrades (control plane via Terraform, node groups via blue/green replacement, add-on version bumps), EC2 patching via Ansible at Xitadel, and Docker base-image updates in response to Trivy findings — all real, defensible examples from your CV/project detail.

**Q: How do you use AI-assisted engineering tools in your work today?**
A: Speak genuinely to how you use GitHub Copilot — code completion, boilerplate for scripts/pipeline YAML, drafting documentation — and be ready to talk about being mindful of reviewing AI-suggested code rather than blindly trusting it, especially for security-sensitive pipeline logic.

---

## 2. Questions From the "Candidate" / Requirements Section

**Q: You have 3.5+ years, they're asking for 3-6 — comfortable with the seniority level of this role?**
A: Yes — frame it as wanting to grow across a broader tooling surface (GitHub/GitLab-native CI/CD, Rancher, potentially cross-platform desktop/web exposure) rather than needing "more experience" to do the job.

**Q: What's your experience with Python specifically?**
A: Be honest and concrete — describe real scripts you've written (automation, CloudWatch/Grafana data pulls, etc.) rather than just citing it as a CV skill. If your Python is lighter than your Bash, say so plainly and pivot to your Bash/scripting depth, since the JD lists Python as one of "several" core technologies, not a hard requirement on its own.

**Q: What's your experience with .NET?**
A: If none, say so directly — "I haven't worked with .NET directly; my application-layer exposure has been with Java/Spring Boot services on the platforms I've supported. I understand it's used on the desktop track here and I'd be ready to ramp up on the build/packaging side without needing to write .NET code myself, since the DevOps role is about the pipeline/environment around it, not the application code."

**Q: Which application track interests you more — desktop or web?**
A: my whole background is Kubernetes-centric: I've run EKS in production, built and maintained Helm charts, and managed RBAC and multi-environment deployments across dev/staging/prod. I haven't personally used Rancher, but conceptually it's a multi-cluster management and app-catalog layer on top of Kubernetes — centralizing access control and Helm-based deployment across clusters, similar in spirit to how I've used ArgoCD for GitOps-based delivery across our dev/staging/prod/DR EKS clusters. The underlying primitives — Deployments, Services, Helm charts, RBAC — are exactly what I already work with daily, so I'd expect to pick up Rancher's specific UI and multi-cluster workflows quickly rather than needing to learn Kubernetes concepts from scratch
A single UI/API to manage multiple clusters from one place, instead of juggling separate kubectl contexts and consoles per cluster
Centralized RBAC and user/access management across all those clusters
An app catalog (Helm charts packaged and exposed through Rancher's UI) for one-click-ish deployment of common tooling
Its own lightweight Kubernetes distribution (RKE — Rancher Kubernetes Engine) if you want Rancher to also provision the clusters themselves, not just manage existing ones like EKS
**Q: What's your understanding of secure SDLC practices?**
A: Dependency management (SCA scanning, CVSS-based blocking), secrets protection (never in Git/pipeline logs, centralized secrets manager, scoped access via IRSA-style identity), automated scanning at every relevant stage (SAST, container, secrets), and quality gates that block rather than just warn — all things you've directly implemented.

**Q: Experience with Agile — Scrum or SAFe?**
A: Scrum, 2-week sprints, Jira-tracked backlog, standard ceremonies (stand-up, planning, review, retro) — direct match from your HCL project. If you haven't used SAFe specifically, say so honestly; Scrum experience transfers reasonably to a SAFe-run organization.

**Q: This role requires strong written and verbal English and cross-functional collaboration with EMEA teams — tell me about working with distributed/international teams.**
A: Speak to any real cross-timezone or cross-team collaboration you've done (client-facing telecom platform work likely involved this) — emphasize clear written documentation (you already author KT/infra docs per your CV) and proactive async communication.

**Q: Would occasional travel be a problem for you?**
A: Answer honestly based on your actual situation.

---

## 3. Likely "Why This Role / Why Us" Questions

**Q: Why do you want to move from a cloud-infrastructure-heavy AWS/EKS role to a role that's more CI/CD-and-developer-tooling focused, supporting desktop and web product teams?**
A: Be genuine here — a real answer might be: "I've built deep infrastructure and platform ownership experience; this role lets me broaden into developer-tooling and CI/CD breadth across a wider set of technologies (GitHub/GitLab ecosystem, Rancher, both desktop and web delivery patterns) inside a larger, more structured engineering organization, which is a deliberate next step in rounding out my DevOps skill set rather than a step back."

**Q: What do you know about Thermo Fisher / the DAIV System Team?**
A: The JD tells you: DAIV is a centralized DevOps/software infrastructure team in Bangalore supporting multiple software product teams across EMEA, building desktop and web applications for data management, image processing, and visualization workflows — providing CI/CD infra, software lifecycle management, developer tooling, AI enablement, and cybersecurity support. Reflect this back in your own words to show you read the JD carefully, and tie it to the mission line ("enable customers to make the world healthier, cleaner, safer") if it feels natural, not forced.

**Q: What questions do you have for us?**
Good ones given this JD specifically:
- "Which of the two tracks — desktop or web — would I likely be supporting first, or is it more fluid across the team?"
- "What does the GitHub/GitLab CI/CD tooling look like today, and is there an ArgoCD/Rancher-style GitOps layer, or is deployment more directly pipeline-driven?"
- "What does 'approved AI-assisted engineering tools' mean in practice here — is there a specific internal Copilot/GenAI setup?"
- "What does success look like for someone in this role at the 6-month mark?"

---

## 4. Quick-Reference: JD Requirement → Your Matching CV Evidence

| JD asks for | Your evidence |
|---|---|
| Git + CI/CD platform, pipeline maintenance, artifact mgmt | Jenkins pipelines (deep), GitHub/GitHub Actions (CV-listed), Nexus/ECR artifact management |
| Docker, Linux | Multi-stage Docker builds, Ubuntu/CentOS/Amazon Linux administration |
| Kubernetes, Helm (web track) | EKS, Helm charts, HPA, RBAC — your strongest technical area |
| Secure SDLC (scanning, secrets, remediation) | SonarQube, Trivy, Gitleaks, IAM, Secrets Manager/IRSA |
| Upgrades/migrations/patching | EKS version upgrades, EC2 patching via Ansible, base-image CVE remediation |
| AI-assisted engineering tools | GitHub Copilot (CV-listed) |
| Agile/Scrum | Scrum, 2-week sprints, Jira — direct match |
| Cross-functional/distributed collaboration | Client-embedded telecom platform team, KT documentation authorship |
| Python, .NET, CMake/C++/Qt | **Gaps** — be honest, bridge from adjacent strengths (see Section 2) |
| GitLab, Rancher specifically | **Gaps** — name-level familiarity only; bridge from GitHub/EKS experience |

---

## 5. How This Combines With Your Other Prep Docs

- Use **this doc** for anything the interviewer frames around the actual job/team/tooling fit.
- Use **`Srinithin_Interview_QA_Prep.md`** and **`Srinithin_CV_QA_with_Followups.md`** for deep technical drill-down on your HCL/Xitadel projects (Terraform internals, IRSA/RBAC, DR strategy, the three CV metrics, etc.) — those don't change based on which company is interviewing you.
- If asked to "walk me through your resume," lead with the 2-minute overview from those docs, then let this doc's honest gap-bridging carry you through anything JD-specific that comes up in follow-ups.
