# Docker — Interview Prep (4 YOE DevOps)

## Topic Recall

**1. Architecture**
Docker Engine = client (CLI) + REST API + dockerd (daemon) → containerd → runc (OCI runtime, actually creates the container process using Linux namespaces + cgroups).

**2. Images vs Containers**
Image = read-only layered template (built from a Dockerfile, layers cached and shared).
Container = a running (or stopped) instance of an image with a writable layer on top.

**3. Dockerfile Essentials**
- `FROM`, `WORKDIR`, `COPY`/`ADD`, `RUN`, `ENV`, `ARG`, `EXPOSE`, `CMD`, `ENTRYPOINT`, `USER`, `HEALTHCHECK`
- Multi-stage builds: separate build stage (compiler/toolchain) from final runtime stage → drastically smaller final image, no build tools shipped to prod.
- Layer caching: order instructions least → most frequently changing (deps install before copying app code) to maximize cache hits.
- `.dockerignore`: exclude `.git`, `node_modules`, secrets, build artifacts from build context.

**4. Image Optimization**
- Use slim/alpine/distroless base images.
- Combine `RUN` commands to reduce layers; clean up package cache in the same layer it was created (`apt-get clean && rm -rf /var/lib/apt/lists/*`).
- Multi-stage builds to drop build-only dependencies.

**5. Networking**
- **bridge** (default, isolated NAT network on the host)
- **host** (shares host network namespace, no NAT, faster, less isolation)
- **none** (fully isolated)
- **overlay** (multi-host, used in Swarm/K8s)
- **macvlan** (container gets its own MAC/IP on the physical network)
- Custom bridge networks give automatic DNS resolution between containers by name.

**6. Storage**
- **Volumes**: managed by Docker (`/var/lib/docker/volumes`), best for persistent data, portable, backup-friendly.
- **Bind mounts**: map a host path directly into the container, good for local dev, tied to host filesystem layout.
- **tmpfs**: in-memory only, never persisted, good for secrets/temp data.

**7. Docker Compose**
- Multi-container app definition in YAML: `services`, `depends_on`, `healthcheck`, `networks`, `volumes`, `env_file`.
- `depends_on` controls start order only, not readiness — pair with `healthcheck` + `condition: service_healthy` for real readiness gating.

**8. Registries**
- Docker Hub, Amazon ECR, private Nexus/Harbor.
- Auth: `docker login`, IAM-based auth for ECR (`aws ecr get-login-password`).
- Tag images with commit SHA/build number, never rely on `latest` in production.

**9. Security**
- Run as non-root (`USER` instruction).
- Scan images (Trivy, Grype, Docker Scout) in CI before push.
- Drop unnecessary Linux capabilities (`--cap-drop=ALL`, add back only what's needed).
- Read-only root filesystem (`--read-only`) where possible.
- Never bake secrets into image layers — use secrets manager / runtime env injection.

**10. Resource Control**
`--memory`, `--memory-swap`, `--cpus`, `--pids-limit` — prevent noisy-neighbor issues on shared hosts.

**11. Storage Driver**
`overlay2` is the modern default — union filesystem, efficient layer sharing.

**12. Logging**
Logging drivers: `json-file` (default), `syslog`, `fluentd`, `awslogs`. Centralize logs in production rather than relying on local `json-file`.

**13. Docker Swarm vs Kubernetes**
Swarm = simpler, built into Docker Engine, less feature-rich. Kubernetes = industry standard for production orchestration, richer scheduling/scaling/self-healing — most orgs (like yours, on EKS) use K8s over Swarm.

**14. Troubleshooting Toolkit**
`docker logs`, `docker exec -it <c> sh`, `docker inspect`, `docker stats`, `docker events`, `docker top`, `docker diff`.

---

## 20 Real-Time Interview Questions & Answers

**1. What exactly happens when you run `docker run nginx`?**
Docker checks for the image locally; if missing, pulls it from the registry. It then creates a new container (writable layer + namespaces/cgroups), sets up networking, mounts any volumes, and starts the process defined by `ENTRYPOINT`/`CMD`.

**2. Difference between `COPY` and `ADD`?**
`COPY` does a plain file/directory copy from build context. `ADD` additionally supports remote URLs and auto-extracts local tar archives. Best practice: prefer `COPY` unless you specifically need `ADD`'s extra behavior — it's more predictable.

**3. Difference between `CMD` and `ENTRYPOINT`?**
`ENTRYPOINT` defines the fixed executable that always runs; `CMD` supplies default arguments that can be overridden at `docker run`. Combining both (`ENTRYPOINT ["python","app.py"]`, `CMD ["--prod"]`) gives a fixed binary with overridable flags.

**4. What are multi-stage builds and why use them?**
Multiple `FROM` stages in one Dockerfile — build/compile in an early stage, then `COPY --from=<stage>` only the final artifact into a minimal runtime image. Reduces image size and attack surface by excluding compilers/build tools from production.

**5. How do you reduce a Docker image's size?**
Use a smaller base image (alpine/distroless), multi-stage builds, combine `RUN` layers, clean package manager caches in the same layer, avoid copying unnecessary files (`.dockerignore`).

**6. Explain the Docker networking modes you've used in production.**
Bridge for standard container-to-container comms on a single host; host mode when you need max network performance and don't need isolation; overlay for multi-host communication in Swarm/K8s-style clustering. In practice, most app containers just use a custom bridge network with DNS-based service discovery.

**7. Bind mount vs named volume — when do you use which?**
Bind mounts are ideal for local development (live code reload, mapping your working directory in). Named volumes are preferred for production persistent data (databases, uploaded files) since Docker manages the lifecycle and they're portable across host changes.

**8. How does Docker layer caching work, and how do you optimize a Dockerfile for it?**
Each instruction creates a cached layer; if an instruction and its context haven't changed, Docker reuses the cached layer instead of re-executing it. Order instructions so rarely-changing steps (installing dependencies) come before frequently-changing steps (copying source code), so code changes don't invalidate the dependency-install cache.

**9. A container is running but the application inside is unreachable — how do you troubleshoot?**
Check `docker ps` for correct port mapping, `docker logs <container>` for app errors, `docker exec -it <container> sh` to check the process is actually listening (`netstat`/`ss`), verify the app binds to `0.0.0.0` not `127.0.0.1`, and check network/firewall/security-group rules if it's cloud-hosted.

**10. How do you persist data across container restarts/recreates?**
Attach a named volume (or bind mount) to the path where the app writes data, so the writable container layer isn't the only copy — the volume survives `docker rm` and can be reattached to a new container.

**11. A container exits immediately after starting — how do you debug it?**
`docker logs <container>` first to see the exit error. Check the `CMD`/`ENTRYPOINT` actually runs a long-lived foreground process (containers exit when PID 1 exits — a common mistake is running a script that finishes immediately). `docker inspect` for exit code, and `docker run -it --entrypoint sh <image>` to poke around interactively.

**12. Difference between `docker-compose up` and `docker-compose up --build`?**
`up` uses existing images (pulling if not present locally) as-is. `--build` forces Compose to rebuild images from their Dockerfiles first, picking up any source/Dockerfile changes before starting containers.

**13. How do you handle secrets in Docker — what shouldn't you do?**
Don't bake secrets into `ENV` in the Dockerfile (they persist in image history/layers). Use Docker secrets (Swarm), mounted secret files, or fetch at runtime from Vault/SSM/Key Vault. `--env-file` is fine for non-sensitive config but not for real secrets in shared images.

**14. What is the `overlay2` storage driver and why does it matter?**
It's Docker's default union filesystem driver on Linux — it lets multiple image layers stack efficiently and be shared across containers/images without duplication, improving disk usage and container startup time versus older drivers like `aufs`.

**15. How do you limit CPU/memory for a container, and why is this important in production?**
`docker run --memory=512m --cpus=1.5 <image>` (or the equivalent in Compose/K8s resource limits). Prevents one noisy or leaking container from starving others on the same host — critical in multi-tenant/shared-node environments.

**16. Docker Swarm vs Kubernetes — how would you explain the difference in an interview?**
Swarm is simpler and built into Docker Engine, good for small/simple clustering needs. Kubernetes is the industry-standard orchestrator with far richer scheduling, autoscaling, self-healing, and ecosystem (Helm, operators, service mesh) — which is why most production platforms, including EKS-based ones, standardize on K8s over Swarm.

**17. How would you scan a Docker image for vulnerabilities as part of CI/CD?**
Add a scan stage (Trivy/Grype/Docker Scout) right after the image build step, before push to registry — fail the pipeline on HIGH/CRITICAL findings above an agreed threshold, similar to a SonarQube quality gate for code.

**18. What's a dangling image, and how do you clean it up?**
An image layer left with no tag (usually from repeated builds overwriting a tag) — shows as `<none>` in `docker images`. Clean with `docker image prune` (dangling only) or `docker system prune -a` (more aggressive, removes unused images/containers/networks).

**19. Why run containers as a non-root user, and how do you do it?**
Running as root inside a container means a container breakout gives root on the host's kernel namespace — a major security risk. Set `USER <non-root-user>` in the Dockerfile (create the user first with `RUN useradd`), and ensure file permissions on app directories match that user.

**20. Two containers on the same custom network can't reach each other — how do you debug it?**
Confirm both are actually attached to the same user-defined network (`docker network inspect <net>`), verify the target container's app is listening internally, use `docker exec` into one container and `ping`/`curl` the other by service/container name (DNS resolution only works on user-defined networks, not default bridge), and check that the app isn't only binding to `localhost` inside its own container.
