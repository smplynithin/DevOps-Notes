# Thermo Fisher — DevOps Engineer Interview: Detailed Answer Notes

> Tailored to your background: ~4.7 yrs total IT / ~3.5 yrs DevOps, 4G Microservices Cloud Enablement Platform (Australian telecom client) at HCLTech, EKS + ArgoCD + Helm + Velero + Terraform stack, AWS SAA + Cloud Practitioner certified.

---

## 1. Introduction & Experience

**How many years of experience do you have? / relevant DevOps experience?**
- Total IT experience: ~4.7 years. DevOps-specific: ~3.5 years.
- Frame it as: "Started as an engineer, transitioned into DevOps in Dec 2021, and have been full-time DevOps since — currently on a production 4G microservices platform for a telecom client."

**What were you doing prior to DevOps?**
- Keep this simple and forward-looking. Say you began in an engineering support/technical role and moved into DevOps because you gravitated toward automation, infrastructure, and CI/CD. Don't over-elaborate on the CAE background unless pressed — pivot quickly to your DevOps journey and achievements.

**Have you worked on-premises as well as on cloud, or only on AWS?**
- Be honest: your primary depth is AWS (EKS, EC2, ECS, RDS, S3, VPC, ALB, Route 53, IAM, Secrets Manager). If you've touched any on-prem/hybrid setups (e.g., an on-prem worker node scenario like the one they asked about), mention it — otherwise say your production experience is AWS-native, but you understand hybrid patterns conceptually (VPN/Direct Connect, on-prem node joining a cluster via kubeadm, etc.) and can speak to how you'd approach it.

**Can you explain how your infrastructure is set up?**
Structure your answer top-down:
1. **Source control & CI**: GitHub → GitHub Actions/Jenkins pipelines build and test.
2. **Container build**: Docker images built, scanned (Trivy), pushed to ECR/GCR.
3. **GitOps deployment**: ArgoCD watches a Git repo of Helm charts/Kustomize manifests and syncs to EKS across Dev/Stage/Prod namespaces or clusters.
4. **Cluster**: EKS with managed node groups, Cluster Autoscaler/HPA, Istio or native networking, Network Policies for isolation.
5. **Storage**: EBS-backed GP3 StorageClass via the AWS EBS CSI driver; Persistent Volumes/Claims for stateful workloads.
6. **Secrets**: Sealed Secrets / External Secrets Operator pulling from AWS Secrets Manager.
7. **Observability**: Prometheus + Grafana for metrics, ELK/Fluentd for logs, CloudWatch/CloudTrail for AWS-level visibility.
8. **Backup/DR**: Velero for cluster and PV backups to S3.
9. **IaC**: Terraform/CloudFormation for provisioning VPC, EKS, RDS, IAM.

**What is your application?**
- Describe it as a 4G microservices platform for a telecom client — multiple microservices deployed on EKS, multi-environment (Dev/Stage/Prod), exposed via ALB, backed by RDS/S3, monitored with Prometheus/Grafana, deployed via ArgoCD GitOps.

---

## 2. Kubernetes / EKS

**Cluster setup — did you do it, or was it done by someone else?**
- Answer honestly based on your actual involvement. If you inherited the cluster, say so, but pivot to what you *do* own: upgrades, add-ons, storage classes, scaling, troubleshooting, backups. If you did set it up (via Terraform/eksctl), describe the node groups, VPC/subnet design, and IAM roles for service accounts (IRSA).

**Do you take care of cluster upgrades?**
- Describe the process:
  1. Check EKS version deprecation notices and add-on compatibility (VPC CNI, CoreDNS, kube-proxy, EBS CSI driver).
  2. Upgrade control plane first (`aws eks update-cluster-version`), one minor version at a time.
  3. Upgrade managed node groups next (rolling replacement, respecting PodDisruptionBudgets).
  4. Update add-ons and Helm charts for API deprecations (check `kubectl deprecations` / `pluto`).
  5. Validate via a staging cluster first, then production, with a rollback plan (keep previous node group AMI/launch template).

**Cluster backups / Velero:**
- **What Velero is used for**: backing up Kubernetes object state (deployments, configmaps, secrets, PVCs) and the underlying EBS volume snapshots, stored in an S3 bucket.
- **What kind of backups**: full-namespace backups for critical namespaces, and scheduled snapshots of PVs (via the Velero AWS plugin, which triggers EBS snapshots).
- **Scheduling**: Velero `Schedule` CRD with a cron expression (e.g., nightly full backup, hourly for critical namespaces), with a defined TTL (retention) per backup.
  ```yaml
  apiVersion: velero.io/v1
  kind: Schedule
  metadata:
    name: nightly-backup
  spec:
    schedule: "0 2 * * *"
    template:
      includedNamespaces: ["prod"]
      ttl: 720h
  ```
- Mention restore testing — periodically restoring into a scratch namespace/cluster to validate backups are actually usable (a strong point to raise proactively).

**Add-ons used in the cluster:**
- VPC CNI (networking), CoreDNS (DNS), kube-proxy, AWS EBS CSI driver (GP3 storage), Cluster Autoscaler or Karpenter, AWS Load Balancer Controller (for ALB/NLB ingress), metrics-server (for HPA).

**StorageClass — GP3:**
- GP3 is not a default EKS StorageClass out of the box; you need the **AWS EBS CSI driver** add-on installed (IRSA-based IAM permissions) to provision GP3 volumes dynamically.
  ```yaml
  apiVersion: storage.k8s.io/v1
  kind: StorageClass
  metadata:
    name: gp3
  provisioner: ebs.csi.aws.com
  parameters:
    type: gp3
    fsType: ext4
  volumeBindingMode: WaitForFirstConsumer
  ```

---

## 3. Kubernetes Storage — NFS / PV / PVC

**Using an existing NFS server as a StorageClass → PV → PVC → Pod:**

Since NFS doesn't have a native dynamic provisioner by default (unlike EBS/GP3), the standard flow is **static provisioning**, or dynamic via the `nfs-subdir-external-provisioner`.

**Option A — Static (you already have server + path):**
1. **PersistentVolume** pointing directly at the NFS export:
   ```yaml
   apiVersion: v1
   kind: PersistentVolume
   metadata:
     name: nfs-pv
   spec:
     capacity:
       storage: 10Gi
     accessModes: ["ReadWriteMany"]
     nfs:
       server: <nfs-server-ip>
       path: /exported/path
     persistentVolumeReclaimPolicy: Retain
   ```
2. **PersistentVolumeClaim** requesting matching size/access mode (binds to the PV):
   ```yaml
   apiVersion: v1
   kind: PersistentVolumeClaim
   metadata:
     name: nfs-pvc
   spec:
     accessModes: ["ReadWriteMany"]
     resources:
       requests:
         storage: 10Gi
   ```
3. **Attach to pod** via `volumes` + `volumeMounts` referencing the PVC.

**Option B — Dynamic (StorageClass-backed):**
- Deploy `nfs-subdir-external-provisioner` (Helm chart) — this runs a small controller pod that watches PVCs referencing an `nfs-client` StorageClass and auto-creates subdirectories on the NFS share as PVs.
  ```yaml
  apiVersion: storage.k8s.io/v1
  kind: StorageClass
  metadata:
    name: nfs-client
  provisioner: cluster.local/nfs-subdir-external-provisioner
  ```

**Why do you need a ConfigMap for NFS (vs. GP3 which just needs an add-on)?**
- GP3/EBS provisioning is handled by the **EBS CSI driver**, which talks to the AWS API directly — no extra config needed beyond IAM permissions.
- NFS has no AWS-native CSI driver by default, so the `nfs-subdir-external-provisioner` needs a **ConfigMap** to tell it the NFS server IP, export path, and provisioner name — this is how the provisioner pod knows *which* NFS share to talk to and how to name/organize the subdirectories it creates per PVC. In short: GP3 config lives in AWS/IAM; NFS config has to be supplied manually via ConfigMap because there's no cloud API backing it.

**hostPath storage:**
- `hostPath` mounts a directory from the **node's local filesystem** directly into the pod — no external storage system involved.
  ```yaml
  volumes:
    - name: local-data
      hostPath:
        path: /data/myapp
        type: DirectoryOrCreate
  ```
- Caveats to mention (shows depth): data is tied to that specific node (not portable if the pod reschedules), no built-in replication, generally used only for node-level agents (logging/monitoring daemonsets) or single-node dev/test — not recommended for production stateful apps.

---

## 4. Kubernetes Troubleshooting

**CrashLoopBackOff — possible causes:**
- Application crashing on startup (bad config, missing env var/secret, unhandled exception).
- Failing liveness probe repeatedly killing the container before it's ready.
- OOMKilled — container exceeding memory limits.
- Missing dependency (DB/downstream service unreachable) causing the app to exit.
- Wrong command/entrypoint or image tag issue.
- Permission issues (e.g., trying to write to a read-only filesystem, wrong securityContext).
- **Troubleshooting steps**: `kubectl describe pod` (check Events section), `kubectl logs <pod> --previous` (logs from the crashed instance), check resource limits, check probe configuration.

**Liveness vs Readiness probes:**
- **Liveness probe**: "Is the container alive?" — if it fails repeatedly, Kubernetes **restarts** the container.
- **Readiness probe**: "Is the container ready to serve traffic?" — if it fails, the pod is removed from the Service endpoints (no restart), so traffic isn't routed to it until it passes again.

**Slow-starting container (5-15 min) causing CrashLoopBackOff:**
- Root cause: liveness probe's `initialDelaySeconds` is too short, so Kubernetes kills the container before it finishes initializing — it never gets a chance to become healthy.
- **Fix**: use a **startup probe** — it runs first and disables liveness/readiness checks until the app has actually started, however long that takes.
  ```yaml
  startupProbe:
    httpGet:
      path: /healthz
      port: 8080
    failureThreshold: 30
    periodSeconds: 10
  livenessProbe:
    httpGet:
      path: /healthz
      port: 8080
    periodSeconds: 10
  ```
  (30 × 10s = 300s / 5 min grace period before liveness even starts checking.)
- Alternative/simpler fix: increase `initialDelaySeconds` on the liveness probe itself, though startupProbe is the cleaner, more scalable solution.

**What happens if you remove both probes?**
- Kubernetes assumes the container is alive and ready the moment the process starts (as soon as the container's main process is running).
- Risk: traffic gets routed to a pod before the app is actually ready to handle it (causing errors/timeouts for users), and a hung/deadlocked-but-still-running process will never be automatically restarted, since there's no health check to catch it. You lose both self-healing and safe traffic cutover.

---

## 5. Kubernetes Networking

**CNI in EKS:**
- Default is the **AWS VPC CNI** — it assigns each pod a real IP address from the VPC's subnet CIDR (via ENIs attached to worker nodes), so pods are directly routable within the VPC.
- If asked about alternatives/why you'd choose something else: Calico (for advanced NetworkPolicy enforcement) or Cilium (eBPF-based, better observability and policy) can be layered on top of/instead of VPC CNI when you need L3/L4 network policies beyond what VPC CNI alone provides. If your project uses Network Policies (which your skills list mentions), be ready to say whether you use VPC CNI's native policy support or a Calico overlay for that.

---

## 6. Kubernetes Cluster Architecture — Scenario (Hybrid Cluster)

**Scenario: 6-node cluster — 3 control-plane + 3 worker nodes, 1 worker node on-premises.**

This is a **kubeadm-based hybrid cluster** scenario (not something EKS supports natively, since EKS control planes are AWS-managed) — so frame your answer around self-managed Kubernetes:

1. **Control plane (3 nodes, for HA)**: Set up via `kubeadm init` on the first node with `--control-plane-endpoint` pointing to a load balancer (or a stable DNS/VIP) in front of all 3 API servers. Join the other 2 as control-plane nodes using `kubeadm join --control-plane`. Use a stacked or external etcd (3-node etcd cluster for quorum).
2. **Worker nodes (2 in-cloud)**: Standard `kubeadm join` using the join token/cert generated during init.
3. **On-prem worker node — the key challenge is network connectivity**:
   - Establish a **VPN or AWS Direct Connect** between the on-prem network and the VPC hosting the control plane, so the on-prem node can reach the API server endpoint and other nodes' pod CIDRs.
   - Ensure the on-prem node's pod CIDR range doesn't overlap with the cloud nodes' ranges (plan CIDR allocation up front).
   - Open required ports (6443 for API server, 2379-2380 for etcd if applicable, kubelet 10250, and CNI-specific ports).
   - Join it like any other worker: `kubeadm join <control-plane-endpoint>:6443 --token ... --discovery-token-ca-cert-hash ...`.
   - Use a CNI that supports cross-network routing well (Calico with BGP, or a VPN-aware overlay like Flannel with VXLAN) since VPC CNI (ENI-based) won't work for a non-AWS node.
4. **Considerations to mention**: latency between on-prem and cloud nodes affects etcd/control-plane health if the on-prem node were a control-plane node (this is why it's specified as a *worker*, which tolerates latency much better); node labels/taints to intentionally schedule specific on-prem-only workloads there (e.g., `kubectl taint nodes onprem-node dedicated=onprem:NoSchedule` + matching tolerations).

---

## 7. Docker

**Docker experience — framing:**
- Speak to: writing multi-stage Dockerfiles for microservices, image optimization (small base images, layer caching), pushing to ECR/GCR/Docker Hub, and integrating builds into CI/CD.

**Scenario: UI code in GitHub → build a Docker image:**
1. Write a **multi-stage Dockerfile**:
   ```dockerfile
   FROM node:18-alpine AS build
   WORKDIR /app
   COPY package*.json ./
   RUN npm ci
   COPY . .
   RUN npm run build

   FROM nginx:alpine
   COPY --from=build /app/build /usr/share/nginx/html
   EXPOSE 80
   CMD ["nginx", "-g", "daemon off;"]
   ```
2. Build: `docker build -t myapp-ui:<tag> .` — tag with a meaningful version (git SHA or semver), not just `latest`.

**Where/how to push the image:**
- Tag for the registry: `docker tag myapp-ui:<tag> <account>.dkr.ecr.<region>.amazonaws.com/myapp-ui:<tag>`
- Authenticate: `aws ecr get-login-password | docker login --username AWS --password-stdin <account>.dkr.ecr.<region>.amazonaws.com`
- Push: `docker push <account>.dkr.ecr.<region>.amazonaws.com/myapp-ui:<tag>`
- Mention: scan the image with **Trivy** before/after push as part of the pipeline gate.

**Deploying via Kubernetes/EKS:**
- Update the image tag in the Helm chart/Kustomize overlay, commit to the GitOps repo, and let **ArgoCD** auto-sync the change to the cluster (rather than `kubectl set image` directly, to keep GitOps as the source of truth). Rolling update strategy ensures zero downtime.

---

## 8. Docker Build Troubleshooting — "Latest changes not appearing in UI"

**How to check whether the latest file was copied into the image during build:**
1. **Disable/bypass the build cache** first to rule out stale layers: `docker build --no-cache -t myapp-ui:test .`
2. **Inspect the image contents directly**:
   - `docker run --rm -it myapp-ui:test sh` then `cat`/`ls -la` the relevant file to confirm the change is actually inside the image.
   - Or without running it: `docker cp` from a temporary container, or `docker export <container_id> | tar -t` to list files.
3. **Check layer history**: `docker history myapp-ui:test` to see which layers changed and confirm the `COPY` step actually re-ran (a cached `COPY` layer is the classic cause of "my changes aren't showing up").
4. **Common root causes** (this is really what the question is testing):
   - **Docker layer caching**: if `package.json` didn't change but source did, and the Dockerfile copies `package.json` before source but Docker still thinks nothing changed, or more commonly — the `COPY . .` layer got cached because build context hashing thought nothing changed (rare, but check `.dockerignore` isn't accidentally excluding the changed file).
   - **Browser/CDN/ALB caching** the old static assets — even if the image is correct, users see stale UI, so also check browser cache headers/cache-busting on static assets.
   - **Wrong image tag deployed** — CI built a new image but the deployment manifest/Helm values still reference an old tag (or `imagePullPolicy` isn't `Always` with a mutable tag like `latest`, so the node reuses a locally cached old image).
   - **ArgoCD/K8s didn't actually roll out** — check `kubectl rollout status` and `kubectl describe deployment` to confirm a new ReplicaSet was actually created, not just that the manifest changed.

---

## 9. CI/CD Pipeline — Major Scenario (Dev → Stage → Prod with Gates)

**Design overview** — a GitOps-based multi-stage pipeline (GitHub Actions/Jenkins for CI, ArgoCD for CD):

```
Commit → CI (build, test, scan, push image)
   → Auto-deploy to Dev (ArgoCD auto-sync)
      → Post-deploy health check on Dev
         → Pre-checks before Stage:
              - Dev deployment status check
              - Dev app health check (UI reachability / backend API health)
         → Auto-deploy to Stage (ArgoCD auto-sync)
            → Post-deploy health check on Stage
               → Pre-checks before Prod:
                    - Dev success check
                    - Stage success check
                    - App health checks (UI + backend)
                    - Manual Approval Gate (authorized approvers only)
               → Deploy to Production (ArgoCD sync, manually triggered/promoted)
```

**Key design points to mention:**
- **Dev**: fully automated CD — every merge to `main`/`develop` triggers CI build → image push → ArgoCD auto-sync updates the Dev `Application` manifest with the new tag.
- **Stage**: also continuous, but gated by pre-checks (below) so a broken Dev deploy never silently promotes.
- **Production**: gated by pre-checks **and** a manual approval step — this is the critical control point.
- Use **separate ArgoCD Applications per environment** (or an ApplicationSet with environment overlays via Kustomize/Helm values), each pointing to its own branch/folder/values file in the GitOps repo, so promotion = a controlled PR/merge or image-tag bump between environment folders — not redeploying from scratch.

---

## 10. CI/CD Pre-Checks — Detailed Configuration

**1. Check whether Dev/Stage deployment was successful:**
- Kubernetes-native: `kubectl rollout status deployment/<name> -n dev --timeout=120s` — exits non-zero if the rollout didn't complete, which the pipeline step treats as a hard failure.
- ArgoCD-native (since you're on ArgoCD): query the Application's sync status via the ArgoCD API/CLI: `argocd app get <app-name> -o json` and assert `status.sync.status == Synced` and `status.health.status == Healthy`. This is the stronger option since it reflects the CD tool's own source of truth, not just a point-in-time kubectl check.

**2. Check whether the deployed application is running:**
- `kubectl get pods -n <env> -l app=<name>` and assert all pods are `Running` and `Ready` (e.g., via `kubectl wait --for=condition=Ready pod -l app=<name> --timeout=60s`).

**3. UI health check:**
- A simple HTTP check against the exposed endpoint/ALB URL: `curl -sf -o /dev/null -w "%{http_code}" https://<dev-ui-url>` and assert `200`. Can be a dedicated pipeline step using `curl`, or a synthetic check via a monitoring tool.

**4. Backend API health check:**
- Every service should expose a `/health` or `/actuator/health` (Spring Boot) endpoint. The pipeline calls it and parses the JSON status (`{"status":"UP"}`), rather than just checking HTTP 200, so it also catches cases where the app is up but a downstream dependency (DB, cache) is unhealthy.

**5. Approval gate for Production:**
- **GitHub Actions**: use **Environments** with **required reviewers** — configure the `production` environment in repo settings with specific approvers/teams; the workflow job targeting that environment pauses and sends a notification until approved.
  ```yaml
  jobs:
    deploy-prod:
      environment:
        name: production
      runs-on: ubuntu-latest
      steps:
        - run: ./deploy.sh
  ```
- **Jenkins**: use the `input` step with a `submitter` restriction:
  ```groovy
  stage('Approve Production') {
    steps {
      timeout(time: 24, unit: 'HOURS') {
        input message: 'Deploy to Production?', submitter: 'release-managers'
      }
    }
  }
  ```
- **ArgoCD angle**: keep Production's ArgoCD Application on **manual sync** (`syncPolicy: {}` with no `automated:` block) so even after CI passes, someone has to explicitly click "Sync" in the ArgoCD UI or run `argocd app sync prod-app` — this doubles as a natural approval gate tied directly to the deployment tool itself, which is worth mentioning since it shows GitOps-native thinking rather than bolting approval only onto CI.

**Putting it together**: each pre-check is a discrete pipeline stage that fails fast (so a bad Dev deploy blocks Stage automatically), the approval gate is enforced at the tooling level (GitHub Environments/Jenkins input) so it can't be bypassed by re-running the pipeline, and Production sync stays decoupled from CI by keeping ArgoCD on manual sync for that environment.

---

## Quick Prep Tips
- Wherever a question doesn't map to something you've personally done, say so plainly ("I haven't done that exact setup, but here's how I'd approach it based on how I've handled X") — interviewers value that honesty and it opens well-scoped technical discussion instead of guesswork.
- Have 1-2 real war stories ready: a Velero restore you ran, a CrashLoopBackOff you debugged, an EKS upgrade you performed. Concrete stories beat textbook definitions.
- Given the on-prem/kubeadm hybrid cluster question, it's worth spending 10 minutes before the interview refreshing exact `kubeadm init/join` flag names, since that's the one area outside your EKS-managed comfort zone.
