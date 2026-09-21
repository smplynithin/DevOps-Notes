# DevOps Interview Preparation — CV + 4G Project + LGB Reference

## Source Basis

This interview bank is based on:
- Current CV
- 4G Telecom DevSecOps Interview Master Guide
- LGB Project Explanation reference

> **Interview rule:** Clearly distinguish what you personally owned, what your team/architect owned, and what is only a recommended design. Do not claim reference-project details as personal experience unless they are actually yours.

---

# 1. Highest-Priority Questions

## Q1. Tell me about yourself.

**Answer:**

Good morning. I'm Srinithin, a Cloud DevOps Engineer with 3.5+ years of experience primarily working on AWS, CI/CD, Kubernetes, Terraform, Docker and DevSecOps. Currently at HCL Technologies, I'm working on a 4G microservices cloud platform where I work mainly on Jenkins CI/CD, EKS deployments, Helm, ArgoCD and monitoring using Prometheus and Grafana. I have also worked extensively with Terraform and Ansible for infrastructure and automation. One of my key contributions was improving the deployment process by around 40% through pipeline automation and security gates.

---

## Q2. Explain your current project.

**Answer:**

Currently, I’m working on an internal application used by our teams and vendors to manage 4G internet service components such as routers, switches, billing, and monitoring. The application has 50+ containerized microservices running on AWS EKS with a multi-AZ architecture.

We use Jenkins for CI, Docker for containerization, Helm for packaging, and ArgoCD for GitOps-based deployments. Prometheus and Grafana are used for monitoring. Our production environment has 76+ worker nodes across multiple Availability Zones. My primary responsibilities are CI/CD, Kubernetes operations, Helm, ArgoCD deployments, and monitoring.

---

## Q3. What exactly is your role in the project?

**Answer:**

My primary responsibility is around CI/CD, Kubernetes operations, Helm/GitOps deployment and monitoring. I maintain Jenkins pipelines with security gates, manage Kubernetes workloads and Helm configurations, work with ArgoCD for GitOps deployments, and monitor applications using Prometheus and Grafana. I also work with Terraform and Ansible for infrastructure and automation. For the underlying VPC architecture, IAM strategy and DR decisions, I worked with the respective cloud/architecture teams rather than claiming sole ownership.

---

## Q4. Explain your complete CI/CD pipeline.

**Answer:**

A developer pushes code to GitHub and raises a pull request. After code review and approval, Jenkins starts the CI pipeline. It performs checkout, unit testing, SonarQube analysis, quality-gate validation, dependency scanning, Docker image build, Trivy vulnerability scanning and image signing using Cosign. The image is then pushed to Amazon ECR. We validate the Helm chart using Helm lint and kubeconform, update the image tag in the GitOps repository, and ArgoCD detects the Git change and synchronizes it to EKS. Production has an additional approval gate and post-deployment smoke testing.

### Pipeline Flow

**GitHub → Jenkins → Unit Test → SonarQube → Quality Gate → SCA → Docker Build → Trivy → Cosign → ECR → Helm Lint → Kubeconform → GitOps Repo → ArgoCD → EKS → Smoke Test**

---

# 2. AWS Architecture Questions

## Q5. Explain the AWS architecture of your project.

**Answer:**

We used a multi-AZ AWS architecture. The EKS workloads were distributed across three Availability Zones. The worker nodes were placed in private subnets, while public subnets were used for internet-facing components. NAT Gateways were deployed per AZ for outbound connectivity, and security groups were separated by tier. Traffic could flow through Route 53, ALB, ingress, Kubernetes Service and finally the application pods.

---

## Q6. Why did you use multiple Availability Zones?

**Answer:**

Mainly for high availability and fault tolerance. If one Availability Zone becomes unavailable, workloads can continue running in the other AZs. Our architecture distributed EKS nodes across three AZs, and other critical components such as ALB and database infrastructure were also designed for redundancy. This contributed to the 99.99% availability objective.

---

## Q7. Why are EKS worker nodes in private subnets?

**Answer:**

Worker nodes don't need direct inbound internet access. They are placed in private subnets for security. For outbound communication, such as pulling packages or reaching AWS services, traffic goes through a NAT Gateway. Public-facing traffic is handled through the load balancer rather than directly exposing worker nodes.

---

## Q8. Why did you use one NAT Gateway per AZ?

**Answer:**

Primarily for availability and to avoid unnecessary cross-AZ traffic. Each private subnet routes outbound traffic through the NAT Gateway in the same AZ. This removes a single NAT dependency and also avoids cross-AZ data-transfer charges.

---

## Q9. Explain the traffic flow from user to pod.

**Answer:**

The request first reaches Route 53 for DNS resolution. It then reaches the Application Load Balancer, which routes traffic to the Kubernetes ingress layer. The ingress routes the request to the appropriate Kubernetes Service, and the Service forwards it to one of the application pods.

### Flow

**User → Route 53 → ALB → Ingress → Service → Pod**

---

## Q10. What does AWS manage in EKS and what do you manage?

**Answer:**

AWS manages the EKS control plane, including the API server, scheduler, controller manager and etcd. We manage the worker nodes, node groups, AMIs, scaling and Kubernetes workloads. We also manage the Kubernetes add-ons and their upgrade timing.

---

# 3. EKS / Kubernetes Questions

## Q11. How many worker nodes were there?

**Answer:**

The production platform had 76+ worker nodes distributed across three Availability Zones. The architecture divided them into critical On-Demand nodes, general Spot nodes and a separate platform-addons node group.

If asked for exact distribution:

> Approximately 30 critical On-Demand nodes, 40 general Spot nodes and 6 platform-add-on nodes.

---

## Q12. Why did you use On-Demand and Spot instances?

**Answer:**

We used On-Demand capacity for critical workloads where availability was important, and Spot capacity for workloads that could tolerate interruption, such as stateless internal services and batch jobs. This helped optimize cost without putting critical services on interruptible capacity.

---

## Q13. What are taints and tolerations?

**Answer:**

Taints are applied to nodes to prevent unwanted workloads from being scheduled there. A pod needs a matching toleration to be scheduled on that node. In our architecture, critical nodes were isolated using taints so only workloads with the appropriate toleration could run there.

---

## Q14. What is HPA?

**Answer:**

HPA stands for Horizontal Pod Autoscaler. It automatically increases or decreases the number of pod replicas based on metrics such as CPU or memory utilization. We used HPA to allow stateless workloads to scale based on demand rather than maintaining a fixed replica count.

---

## Q15. What happens if a pod is unhealthy?

**Answer:**

Kubernetes uses liveness and readiness probes. A failed liveness probe can cause the container to be restarted, while a failed readiness probe removes the pod from Service endpoints so it doesn't receive traffic until it becomes healthy again.

---

## Q16. Difference between liveness and readiness probe?

| Probe | Purpose |
|---|---|
| **Liveness** | Is the application alive? |
| **Readiness** | Can the application receive traffic? |

If liveness fails, Kubernetes can restart the container. If readiness fails, Kubernetes stops sending traffic to that pod.

---

## Q17. Deployment vs StatefulSet?

**Answer:**

A Deployment is normally used for stateless applications where any replica can handle a request and the pod can be recreated anywhere. StatefulSet is used when pods require stable identity or persistent storage. In our architecture, most microservices were stateless and persistent data was kept in managed services such as RDS.

---

## Q18. What is a Kubernetes Service?

**Answer:**

A Service provides stable networking and service discovery for a group of pods. Instead of directly connecting to changing pod IPs, applications communicate through the Service DNS name. We primarily use ClusterIP Services for internal microservice communication.

---

## Q19. What is Ingress?

**Answer:**

Ingress provides HTTP/HTTPS routing into Kubernetes. It can route traffic based on host or path to the appropriate Kubernetes Service. In our architecture, the external request reaches the ALB and then the ingress layer before reaching the Service and pods.

---

# 4. ArgoCD / GitOps Questions

## Q20. Why did you use ArgoCD?

**Answer:**

We used ArgoCD to implement GitOps-based continuous delivery. Instead of Jenkins directly modifying the Kubernetes cluster, Jenkins updates the desired state in the GitOps repository. ArgoCD continuously monitors that repository and synchronizes the desired state to the Kubernetes cluster. This improves deployment consistency, auditability and rollback capability.

---

## Q21. Why shouldn't Jenkins directly deploy to Kubernetes?

**Answer:**

Jenkins can technically deploy directly, but GitOps gives us a cleaner separation between CI and CD. Jenkins builds and validates the artifact and updates the desired configuration in Git. ArgoCD is responsible for reconciling that desired state with Kubernetes. This gives us Git-based audit history and easier rollback.

---

## Q22. What happens when a developer pushes new code?

**Answer:**

Jenkins builds and validates the new image and pushes it to ECR. Jenkins then updates the image tag in the GitOps repository. ArgoCD detects the Git change and synchronizes the corresponding Kubernetes manifests into the target EKS cluster. Dev and staging can use automated sync, while production requires approval/manual synchronization.

---

## Q23. What is App-of-Apps in ArgoCD?

**Answer:**

App-of-Apps is a pattern where one parent ArgoCD Application manages multiple child Applications. Instead of manually creating every application in ArgoCD, we maintain application definitions in Git and let the parent application manage them. The architecture uses `root-app.yaml` as the App-of-Apps entry point.

---

## Q24. How do you handle ArgoCD OutOfSync?

**Answer:**

First I compare the desired state in Git with the live state in Kubernetes. I check the ArgoCD diff to identify which resource differs. Then I check whether the difference was caused by a manual Kubernetes change, an incorrect Helm value, or an application reconciliation issue. If Git is correct, I synchronize the application. If the live configuration is intentionally different, I correct the Git source rather than manually modifying production.

---

# 5. Jenkins Questions

## Q25. Explain your Jenkins pipeline stages.

**Answer:**

The pipeline starts with checkout and unit testing. Then SonarQube performs static code analysis followed by a blocking quality gate. Dependency scanning is performed next. After that we build the Docker image, scan it using Trivy, sign it using Cosign and push it to ECR. Then we validate the Helm chart using Helm lint and kubeconform, update the GitOps repository and finally perform the production approval and smoke-test stages.

---

## Q26. How did you achieve the 40% deployment-time reduction?

**Answer:**

We reduced deployment time through several improvements rather than one change. We parallelized independent security stages, used Docker layer caching, reduced manual handoffs in lower environments through ArgoCD auto-sync, and standardized pipelines using a Jenkins Shared Library. We measured the improvement using Jenkins build-duration history before and after the framework implementation.

---

## Q27. What is Jenkins Shared Library?

**Answer:**

A Jenkins Shared Library allows us to keep common pipeline logic in one reusable repository. Instead of every microservice implementing checkout, security scanning, Docker build, deployment and notification stages independently, services can consume standardized pipeline functions. This improves consistency and reduces duplication.

Example reusable stages:

- `standardBuildStage`
- `standardSecurityScan`
- `standardDeployStage`
- `notifySlack`

---

## Q28. How do you implement approval gates?

**Answer:**

We use multiple approval layers. Pull requests require reviewer approval. Terraform changes require plan review. SonarQube and security scans act as blocking gates. Helm validation must pass before GitOps changes are committed. Production deployment has an explicit Jenkins approval and ArgoCD production synchronization is manual.

---

# 6. DevSecOps Questions

## Q29. Why SonarQube?

**Answer:**

SonarQube performs static code analysis. It identifies issues such as bugs, vulnerabilities, code smells and poor code quality. In our pipeline, the SonarQube quality gate is blocking, so the pipeline doesn't continue when the defined quality criteria are not met.

---

## Q30. What is Trivy?

**Answer:**

Trivy is used for vulnerability scanning. In our pipeline, after building the Docker image, Trivy scans the image for vulnerabilities. HIGH and CRITICAL findings are configured to fail the pipeline, preventing vulnerable images from progressing.

---

## Q31. What is Gitleaks?

**Answer:**

Gitleaks detects secrets accidentally committed to source repositories, such as passwords, API keys or tokens. We integrate it into the CI security process so credentials don't enter the source-code repository.

---

## Q32. What is Cosign and why did you use it?

**Answer:**

Cosign is used to digitally sign container images. After the image passes vulnerability scanning, we sign it using a signing key. This allows the deployment environment to verify that the image came from a trusted build process and wasn't replaced by an unauthorized image.

---

# 7. Secrets / IAM / IRSA

## Q33. How do you manage secrets?

**Answer:**

Secrets such as database credentials, API keys and tokens are stored in AWS Secrets Manager rather than Git, Helm values or Jenkins logs. External Secrets Operator synchronizes the secrets into Kubernetes Secrets, and pods consume them through environment variables or mounted volumes. Access is controlled using IRSA so each service gets only the permissions it requires.

---

## Q34. Explain ESO.

**Answer:**

External Secrets Operator acts as the bridge between an external secret store and Kubernetes. In our case, the actual secret is stored in AWS Secrets Manager. ESO authenticates to AWS using an appropriate IAM role, retrieves the secret and creates or updates the corresponding Kubernetes Secret. The pod then consumes that Kubernetes Secret.

### Flow

**AWS Secrets Manager → ESO → Kubernetes Secret → Pod**

---

## Q35. What is IRSA?

**Answer:**

IRSA means IAM Roles for Service Accounts. It allows individual Kubernetes workloads to assume specific AWS IAM roles rather than giving every pod the permissions of the EC2 node. We associate the EKS cluster with an OIDC provider, create an IAM role with a trust relationship to a specific Kubernetes service account, and annotate the ServiceAccount with that role. The pod then receives temporary AWS credentials through the web identity mechanism.

---

## Q36. IRSA vs RBAC?

**Answer:**

IRSA controls **AWS permissions**, while Kubernetes RBAC controls **Kubernetes API permissions**.

- IRSA → Can this pod call `secretsmanager:GetSecretValue`?
- RBAC → Can this user/service account get, create or delete Kubernetes Deployments?

They are separate and complementary security layers.

---

## Q37. Do you need to install IRSA in the cluster?

**Answer:**

No. IRSA isn't a software component that we install as an add-on. We configure the EKS cluster with an OIDC provider, create IAM roles with appropriate trust policies, and associate those roles with Kubernetes ServiceAccounts.

---

## Q38. Why not store AWS credentials in Jenkins?

**Answer:**

We avoid static long-lived AWS credentials wherever possible. Jenkins workloads can assume an IAM role through workload identity, and permissions are scoped to what the pipeline actually needs, such as ECR push/pull. This follows the least-privilege model.

---

# 8. Terraform Questions

## Q39. Explain your Terraform architecture.

**Answer:**

We used reusable Terraform modules for components such as VPC, EKS, IAM, RDS and ECR. Environment-specific configurations were maintained separately for dev, staging, production and DR. Terraform state was stored remotely in S3 with encryption and versioning, while state locking was handled through DynamoDB.

---

## Q40. Why Terraform modules?

**Answer:**

Modules allow us to create reusable infrastructure components. Instead of writing separate VPC or EKS configurations for every environment, we create a generic module and pass environment-specific variables. This improves consistency, maintainability and reduces duplication.

---

## Q41. Why separate state files instead of workspaces?

**Answer:**

For production environments, separate state files provide stronger isolation. We maintain separate backend keys for dev, staging, prod and DR rather than relying on Terraform workspaces. This reduces the risk of accidentally running an operation against the wrong environment.

Example:

```text
s3://tfstate/dev/terraform.tfstate
s3://tfstate/staging/terraform.tfstate
s3://tfstate/prod/terraform.tfstate
s3://tfstate/dr/terraform.tfstate
```

---

## Q42. How do you secure Terraform?

**Answer:**

We use remote encrypted state with versioning and locking. Terraform changes go through pull requests and plan review. We also run tools such as tfsec or Checkov to identify security misconfigurations such as open security groups, unencrypted resources or public buckets. Production apply requires an additional approval.

---

## Q43. What happens when two people run Terraform simultaneously?

**Answer:**

Remote state locking prevents concurrent Terraform operations against the same state. In our architecture, DynamoDB provides the state-locking mechanism. This prevents concurrent applies from corrupting or conflicting with the state.

---

# 9. Ansible Questions

## Q44. Why did you use Ansible when you already had Terraform?

**Answer:**

Terraform and Ansible solve different problems. Terraform is primarily used to provision infrastructure, while Ansible is used for configuration management and operational tasks on existing machines. In my Xitadel experience, I used Ansible for EC2 patching, package management, Linux configuration and troubleshooting automation.

---

## Q45. What does idempotency mean in Ansible?

**Answer:**

Idempotency means that running the same playbook multiple times produces the same desired state without unnecessarily changing already-correct resources. We prefer native Ansible modules rather than raw shell commands because the modules are designed to maintain idempotent behavior.

---

## Q46. How do you safely execute an Ansible change?

**Answer:**

First we run linting and then `ansible-playbook --check --diff` to perform a dry run. The expected changes are reviewed before the actual playbook execution. Production execution is performed during an approved maintenance window.

---

# 10. Docker Questions

## Q47. Why multi-stage Docker builds?

**Answer:**

Multi-stage builds separate the build environment from the runtime environment. For example, Maven and build dependencies are required to compile a Java application but aren't required when running the application. We use one stage to build the application and a smaller runtime image for execution. This reduces image size, attack surface and deployment time.

---

## Q48. Why should you avoid `latest` tags?

**Answer:**

`latest` is mutable, so the same tag can point to different images over time. That makes deployments and rollbacks difficult to reproduce. We use immutable Git commit SHA-based image tags so every deployment maps to a specific image version.

---

## Q49. Why run containers as non-root?

**Answer:**

Running as a non-root user reduces the impact if the container is compromised. The architecture uses a non-root user in the Dockerfile, and Kyverno also enforces `runAsNonRoot` as an additional admission-control layer.

---

# 11. Monitoring Questions

## Q50. What do you monitor using Prometheus?

**Answer:**

We monitor infrastructure, Kubernetes and application-level metrics.

- Node-level: CPU, memory, disk and network usage
- Kubernetes: pod restarts, pending pods and replica status
- Application: request rate, error rate and latency such as p50, p95 and p99

---

## Q51. What is ServiceMonitor?

**Answer:**

ServiceMonitor is a Prometheus Operator CRD used to define which Kubernetes Services should be scraped by Prometheus. For Spring Boot services, the application exposes metrics through an endpoint, and ServiceMonitor tells Prometheus how to discover and scrape that endpoint.

---

## Q52. What dashboards did you create in Grafana?

**Answer:**

We used dashboards for cluster health, individual service performance and SLO tracking. Service dashboards included RED metrics — request rate, error rate and duration. We also tracked availability against the 99.99% target and monitored resource utilization for right-sizing decisions.

---

## Q53. How did you achieve 99.99% uptime?

**Answer:**

It was achieved through multiple layers rather than a single tool. We used multi-AZ infrastructure, proper readiness and liveness probes, HPA and node autoscaling, proactive Prometheus alerting and fast GitOps rollback. A 99.99% availability target corresponds to roughly 52 minutes of allowed downtime per year.

---

# 12. Troubleshooting Questions

## Q54. A pod is in CrashLoopBackOff. What will you do?

**Answer:**

First I check the pod status and events using `kubectl describe pod`. Then I check the previous container logs using `kubectl logs --previous`. I verify configuration, Secrets, ConfigMaps, environment variables, resource limits and application startup errors. If necessary, I compare the current image with the previous working version and check recent deployment changes.

### Commands

```bash
kubectl get pods -n <namespace>
kubectl describe pod <pod> -n <namespace>
kubectl logs <pod> -n <namespace>
kubectl logs <pod> -n <namespace> --previous
kubectl get events -n <namespace> --sort-by=.lastTimestamp
```

---

## Q55. Pod is stuck in Pending. What do you check?

**Answer:**

I start with `kubectl describe pod` and look at the scheduler events. Then I check CPU/memory requests, node capacity, taints and tolerations, node selectors, affinity rules and PVC status. If there isn't enough node capacity, I check whether autoscaling can provision a suitable node.

---

## Q56. Pod is running but application is not accessible. What do you check?

**Answer:**

I follow the request path layer by layer:

**Ingress → Service → Endpoints → Pod → Application**

I verify the Service selector, endpoints, target port, container port, ingress rules, ALB health checks and application logs.

---

## Q57. A deployment succeeded but users receive 5xx errors. What do you do?

**Answer:**

First I check application error-rate and latency metrics in Grafana. Then I check pod logs, readiness status, Service endpoints, ingress and ALB health. I compare the deployment version with the previous version. If the issue is introduced by the release and cannot be fixed quickly, I roll back through GitOps rather than manually changing production.

What are the most common 4xx errors?
Code	Meaning	Typical reason
400	Bad Request	Invalid request/body/parameters
401	Unauthorized	Missing/invalid authentication
403	Forbidden	Authenticated but not authorized
404	Not Found	Resource/API endpoint doesn't exist
409	Conflict	Request conflicts with current state
429	Too Many Requests	Rate limit exceeded

Most important: 400, 401, 403, 404, 429.

What are the most common 5xx errors?
Code	Meaning	Typical reason
500	Internal Server Error	Application-side unexpected error
502	Bad Gateway	Proxy/load balancer received invalid response from upstream
503	Service Unavailable	Service unavailable, overloaded, or no healthy backend
504	Gateway Timeout	Upstream didn't respond within timeout

Most important: 500, 502, 503, 504.

---

# 13. Production / Incident Questions

## Q58. What was the biggest problem you faced?

**Answer:**

During migration, some legacy services had tightly coupled and stateful dependencies, so replacing them directly could have caused downtime. The solution was to deploy the new containerized version alongside the legacy system and gradually shift traffic. We monitored error rate, latency and saturation during the rollout and maintained the legacy system as the rollback path.

### Traffic Shift

**1% → 5% → 25% → 100%**

If metrics degraded, traffic could immediately be shifted back to the legacy system.

---

## Q59. What is your biggest contribution?

**Answer:**

My biggest contribution was helping build a reusable DevOps framework consisting of standardized Jenkins pipeline logic, Helm patterns and ArgoCD GitOps structure. The objective was to avoid every microservice implementing its own pipeline and security gates from scratch. This improved consistency and contributed to the deployment-time improvement.

---

## Q60. How did you handle rollback?

**Answer:**

With GitOps, the desired deployment version is stored in Git. If a deployment introduces an issue, we can revert the Git change to the previous image tag and ArgoCD reconciles the cluster back to the previous desired state. This provides a controlled and auditable rollback mechanism.

---

# 14. Resource Optimization Questions

## Q61. How did you achieve 25% resource-utilization improvement?

**Answer:**

We analyzed actual CPU and memory utilization through Grafana and CloudWatch and used that data for right-sizing. We also used HPA to scale pods according to demand and node autoscaling to adjust infrastructure capacity. Additionally, we used a combination of On-Demand and Spot capacity for appropriate workloads.

---

## Q62. How would you identify over-provisioned pods?

**Answer:**

I compare the configured CPU and memory requests with historical actual utilization. If a service consistently requests significantly more resources than it consumes, I reduce the requests after validating application behavior and performance. I then monitor the workload after the change.

---

# 15. Disaster Recovery Questions

## Q63. Explain your DR architecture.

**Answer:**

The primary region runs the full EKS environment, while the secondary region maintains a pilot-light or warm-standby setup. RDS data is replicated cross-region, ECR images are replicated, and Kubernetes configuration is backed up using Velero and stored in S3. Route 53 health checks can detect a regional failure. During failover, the RDS replica is promoted, DR compute capacity is scaled up, ArgoCD synchronizes the workloads and Route 53 is redirected to the DR environment.

---

## Q64. What is RTO and RPO in your DR design?

**Answer:**

Because the DR architecture is pilot-light rather than active-active, the RTO is in the tens-of-minutes range. RPO is bounded by database replication lag, which is measured in minutes rather than being near-zero. This was a deliberate cost and complexity trade-off.

---

# 16. Storage Questions

## Q65. How do you dynamically attach persistent storage to Kubernetes pods?

**Answer:**

We use PersistentVolumeClaims with the EBS CSI driver. A StatefulSet requests storage through a PVC, and the CSI driver dynamically provisions the corresponding EBS volume. If the pod is rescheduled, Kubernetes can reattach the persistent volume to the appropriate node, subject to the storage and topology constraints.

### Flow

**StatefulSet → PVC → StorageClass → EBS CSI Driver → EBS Volume**

---

## Q66. What is the difference between EBS and S3?

**Answer:**

EBS is block storage primarily used by EC2/EKS workloads that require a filesystem or persistent block device. S3 is object storage used for objects such as backups, logs, artifacts and Terraform state. In our architecture, EBS was used through the CSI driver, while S3 was used for backups, state and other objects.

---

# 17. Git Questions

## Q67. What branching strategy did you follow?

**Answer:**

The reference flow uses feature branches followed by controlled promotion through development and release branches into the main branch. Developers create feature branches, raise pull requests, undergo code review and automated validation, and only then merge into the protected branch.

---

## Q68. Why are direct commits to main restricted?

**Answer:**

To protect production-quality code. Changes should go through pull requests, code review, unit tests, quality checks and security checks before merging. This provides traceability and prevents unauthorized or unvalidated changes.

---

# 18. Xitadel Questions

## Q69. Explain your Xitadel project.

**Answer:**

At Xitadel, I worked on a cloud infrastructure provisioning and CI/CD platform. My responsibilities included AWS infrastructure, Jenkins CI/CD, Terraform, Ansible, Docker and monitoring. I worked with AWS services such as VPC, EC2, ALB, RDS and S3 and used reusable Terraform modules for provisioning. I also automated EC2 configuration and patching using Ansible.

---

## Q70. How did Terraform reduce provisioning time by 40%?

**Answer:**

Previously infrastructure provisioning involved more manual steps. We created reusable Terraform modules for AWS components and standardized environment provisioning. This made infrastructure creation repeatable and reduced manual configuration, resulting in approximately 40% faster provisioning.

---

## Q71. How did Ansible reduce administration effort by 35%?

**Answer:**

We automated repetitive EC2 administration tasks such as package management, patching, user management and service configuration. Instead of performing these operations manually across servers, we used Ansible playbooks to apply the desired configuration consistently.

---

## Q72. How did Jenkins integrate with Nexus?

**Answer:**

The CI pipeline built the application, performed SonarQube analysis and then created the required artifact or Docker image. Nexus was used as the artifact repository, allowing us to centrally store and retrieve build artifacts instead of depending on individual developer or build machines.

---

# 19. LGB-Style Project Questions

> **Note:** Use these as reference questions. Do not claim LGB-specific implementation details as your personal experience unless they are true for your project.

## Q73. What type of application were you supporting?

**Reference Answer:**

It was a microservices-based digital banking application. Different business capabilities were separated into services such as customer, account, loan, payment, transaction, notification and authentication services. This allowed individual services to be developed, deployed and scaled independently.

---

## Q74. What was the end-to-end application delivery process?

**Reference Answer:**

Developer creates a feature → pushes to Git → raises PR → code review → automated validation → merge → CI pipeline → build and unit testing → code-quality and security scans → Docker image build → image push → deployment to Kubernetes using Helm → promotion through Dev, QA, UAT, Staging and Production.

---

## Q75. Why have Dev, QA, UAT, Staging and Production?

**Reference Answer:**

Each environment has a different purpose:

- Dev → development and integration
- QA → functional and integration testing
- UAT → business validation
- Staging → production-like final validation
- Production → actual customer traffic

This allows defects to be identified progressively before reaching production.

---

# 20. Additional Follow-Up Questions

## AWS

76. Why EKS instead of ECS?
77. Why ALB instead of NLB?
78. Why private subnets?
79. Why NAT Gateway?
80. How does ALB discover Kubernetes pods?
81. How do security groups work between ALB, nodes and RDS?
82. How do you troubleshoot an EC2 instance that is unreachable?
83. How do you troubleshoot high CPU in an EC2 instance?
84. How do you secure S3?
85. How do you encrypt AWS resources?

## Kubernetes

86. What happens internally when you create a Deployment?
87. How does Kubernetes schedule a pod?
88. What happens when a node goes down?
89. How does HPA work?
90. HPA vs VPA?
91. Deployment vs StatefulSet?
92. Deployment vs DaemonSet?
93. ConfigMap vs Secret?
94. Service types?
95. ClusterIP vs LoadBalancer?
96. What is kube-proxy?
97. What is CoreDNS?
98. How does pod-to-pod communication work?
99. How does Kubernetes self-healing work?
100. How do you perform a zero-downtime deployment?

## Jenkins

101. Declarative vs scripted pipeline?
102. What is a Jenkins agent?
103. Why use Kubernetes agents?
104. How do you store Jenkins credentials?
105. How do you prevent secrets appearing in Jenkins logs?
106. How do you parallelize Jenkins stages?
107. How do you handle failed builds?
108. How do you implement rollback?

## Terraform

109. Terraform `plan` vs `apply`?
110. What is state?
111. Why remote state?
112. How do you handle state locking?
113. What is state drift?
114. How do you import an existing resource?
115. Module vs workspace?
116. How do you manage different environments?
117. How do you handle Terraform secrets?
118. What happens if Terraform apply fails halfway?

## DevSecOps

119. SAST vs SCA vs image scanning?
120. SonarQube vs Trivy?
121. Gitleaks vs Trivy?
122. Why image signing?
123. What happens when Trivy finds a HIGH vulnerability?
124. How do you handle a critical CVE in production?
125. How do you prevent unsigned images from running?
126. How do you implement least privilege?

---

# 21. Top 15 Questions to Prepare First

| Priority | Question |
|---:|---|
| 1 | Tell me about yourself |
| 2 | Explain your current 4G project |
| 3 | What is your exact role? |
| 4 | Explain complete CI/CD pipeline |
| 5 | Explain AWS architecture |
| 6 | Explain EKS architecture |
| 7 | How does traffic reach a pod? |
| 8 | Explain Terraform implementation |
| 9 | Explain ArgoCD/GitOps |
| 10 | How do you manage secrets? |
| 11 | Explain IRSA |
| 12 | Explain Kubernetes security |
| 13 | How did you achieve 40% deployment reduction? |
| 14 | How did you achieve 25% resource improvement? |
| 15 | How did you achieve 99.99% uptime? |

---

# 22. Numbers You Must Be Ready to Defend

| CV Claim | Interview Explanation |
|---|---|
| **40% deployment reduction** | Parallel pipeline stages + Docker layer caching + reduced manual lower-environment handoffs + reusable Jenkins Shared Library |
| **25% resource improvement** | HPA + node autoscaling + right-sizing using Grafana/CloudWatch + appropriate Spot/On-Demand mix |
| **99.99% uptime** | Multi-AZ + probes + autoscaling + proactive monitoring + fast GitOps rollback |
| **76+ nodes** | ~30 critical On-Demand + ~40 general Spot + ~6 platform nodes across 3 AZs |

---

# 23. Complete Project Story

If the interviewer says:

> **"Okay Srinithin, give me the complete picture of your project."**

Say:

> "I worked as a DevOps Engineer on a 4G microservices cloud platform. The applications were containerized and deployed on AWS EKS across multiple Availability Zones. My primary responsibilities were Jenkins CI/CD, Kubernetes operations, Helm, ArgoCD GitOps and monitoring using Prometheus and Grafana.
>
> The development process starts with GitHub. After a pull request and code review, Jenkins performs unit testing, SonarQube analysis, dependency scanning, Docker image creation, Trivy vulnerability scanning and image signing. The approved image is pushed to ECR. Jenkins then updates the image version in the GitOps repository, and ArgoCD synchronizes that desired state into EKS.
>
> On the Kubernetes side, we manage Deployments, Services, ConfigMaps, Secrets, Ingress, HPA and RBAC. For AWS access from workloads, we use IAM and IRSA following least-privilege principles. Secrets are stored externally and synchronized through the secrets-management process.
>
> For monitoring, Prometheus collects infrastructure, Kubernetes and application metrics, while Grafana provides dashboards and alerting. The platform has 76+ worker nodes across multiple AZs, and we improved resource utilization through right-sizing and autoscaling. Overall, the automation and standardized DevOps framework helped reduce deployment time by around 40% and supported high availability."

---

# 24. Final Interview Rule

For every project statement, classify it mentally:

### "I built/ran this"
Say it directly and give the technical details.

### "My team/architect owned this; I implemented parts"
Say that explicitly.

### "This is how I would design it"
Clearly identify it as a design recommendation, not past experience.

This distinction is especially important when discussing:

- VPC architecture
- IAM architecture
- Disaster Recovery
- EKS architecture
- Security controls
- Exact service counts
- Exact infrastructure sizing

The goal is to sound technically strong **without overstating ownership**.


Here's a clean, interview-ready overview for each — structured as "what/why/how" so you can deliver it in 30-45 seconds and hold up under a follow-up.

Bash Scripts
1. RDS/Postgres backup verification

What: Checks that the latest automated RDS snapshot for a given database actually completed successfully, rather than assuming it did.
Why: Automated snapshots are configured and usually reliable, but "usually" isn't good enough for a production database — a silently failed snapshot means your last real recovery point could be a day or more stale without anyone knowing until it's needed.
How: Queries AWS for the most recent snapshot of a given DB instance, checks its status field, and alerts if it isn't available or if it's older than expected.
Ties to your project: Backs up your DR/backup story directly — "RDS backups cover transactional data with their own point-in-time-recovery semantics" — this script is the verification layer confirming that story is actually true day-to-day, not just configured and forgotten.

2. Terraform plan drift checker

What: Runs a scheduled terraform plan against prod (or any environment) outside of the normal PR flow, to catch when real infrastructure has drifted from what's defined in code.
Why: Someone making a manual console change (an emergency fix, a misclick, a forgotten one-off tweak) is a real risk — if nobody catches it, the next terraform apply could either silently revert their fix or, worse, produce an unexpected change because state and reality disagree.
How: Runs terraform plan -detailed-exitcode on a schedule; that exit code distinguishes "no changes" from "changes pending" cleanly, so the script can alert specifically on drift rather than parsing plan text.
Ties to your project: Reinforces your "strict state isolation per environment" story — this is the proactive half of that discipline, not just reactive state locking.

3. ECR old-image cleanup (cost/hygiene)

What: Deletes container images older than a set age (e.g., 90 days) from an ECR repository, while always preserving the most recent N images regardless of age.
Why: Every pipeline run pushes a new immutable git-SHA-tagged image, and without cleanup that accumulates indefinitely — real storage cost, and clutter that makes it harder to find the image that actually matters.
How: Lists images sorted by push date, calculates age for anything outside the "always keep" window, and batch-deletes what's both old and outside that safety margin.
Ties to your project: Directly supports your "immutable git-commit-SHA image tags" practice — this is the operational housekeeping that makes an immutable-tagging strategy sustainable long-term instead of just growing forever.

Python Scripts
4. Secrets Manager rotation status checker

What: Checks whether secrets in AWS Secrets Manager have actually rotated on schedule, and flags any that are overdue or have rotation disabled entirely.
Why: Secrets Manager's native rotation (via its Lambda integration) is automated, but automation can silently fail — a broken Lambda or a permissions issue means a secret quietly stops rotating with no obvious symptom until someone asks "when did this last rotate?"
How: Lists secrets (optionally filtered by tag/environment), pulls each one's LastRotatedDate and RotationEnabled status via the API, compares against the expected interval, and reports/alerts on anything overdue.
Ties to your project: Reinforces your least-privilege/IAM-governance story — a verification layer on top of "RDS credential rotation handled natively," catching the gap between "rotation is configured" and "rotation is actually happening."

5. Log parser for error-rate extraction (CloudWatch / ELK exports)

What: Pulls logs for a service over a time window and calculates the error rate (5xx responses, exceptions) as a percentage of total requests.
Why: During a canary rollout you need a clear, repeatable go/no-go signal — eyeballing a Grafana dashboard works, but a scripted check gives a consistent, automatable decision point instead of relying on a person watching a graph at exactly the right moment.
How: Queries CloudWatch Logs Insights (or Elasticsearch) for the relevant log group/time range, parses structured log entries for status codes or error markers, calculates the error percentage, and compares it against a threshold.
Ties to your project: Fits directly into your strangler-fig migration story — the same weighted traffic shifts (1% → 5% → 25% → 100%) could gate on this script's output instead of purely manual dashboard watching.
