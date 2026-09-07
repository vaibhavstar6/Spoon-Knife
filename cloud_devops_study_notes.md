# Cloud & DevOps Engineer — Study Notes (5-Year Level, Big 4 Prep)

These notes are written to be read once for understanding, then skimmed again before the interview. Each topic has: **what it is → why it matters → key terms → things people get tripped up on.**

---

# 1. Cloud Fundamentals & Architecture

### IaaS vs PaaS vs SaaS
- **IaaS** (Infrastructure as a Service) — you rent raw compute/storage/network (EC2, Azure VM). You manage the OS upward.
- **PaaS** (Platform as a Service) — the provider manages the OS/runtime, you just deploy code (Elastic Beanstalk, App Service, Cloud Run).
- **SaaS** (Software as a Service) — fully managed application, you just use it (Gmail, Salesforce).
- Interview angle: know the **shared responsibility model** — as you move from IaaS → SaaS, more security/operational responsibility shifts to the provider. You're always responsible for your data and access management, regardless of tier.

### Regions, Availability Zones (AZs), Edge Locations
- **Region** = a geographic area (e.g., `us-east-1`) containing multiple isolated data centers.
- **AZ** = one or more discrete data centers within a region, with independent power/cooling/networking, but low-latency links to other AZs in the same region. Deploying across multiple AZs = high availability within a region.
- **Edge location** = smaller PoPs used for CDN caching (CloudFront) — closer to end users, not for compute.
- Rule of thumb: multi-AZ protects against a data center failure; multi-region protects against a regional disaster (and helps with latency for global users).

### Landing Zones & Account Strategy
- Already covered in depth earlier — the pre-built secure multi-account foundation (guardrails, identity, networking, logging) new workloads land into. Know AWS Control Tower, Azure Landing Zones, GCP Foundation as the tool names per cloud.

### Well-Architected Framework (know the 5–6 pillars, AWS naming, others map similarly)
1. **Operational excellence** — run and monitor systems, continuously improve processes.
2. **Security** — protect data, systems, and assets.
3. **Reliability** — recover from failures, meet demand.
4. **Performance efficiency** — use resources efficiently, adapt as needs change.
5. **Cost optimization** — avoid unnecessary spend.
6. **Sustainability** (added later) — minimize environmental impact.
Interviewers love asking "walk me through how you'd evaluate an architecture" — structure your answer around these pillars.

### HA vs DR (a very common mix-up)
- **High Availability (HA)** — designed to keep running through *component* failures (e.g., one server dies, load balancer routes around it). Usually same region, multi-AZ.
- **Disaster Recovery (DR)** — plan for recovering after a *major* event (region outage, data corruption, ransomware). Usually involves another region.
- **RTO** (Recovery Time Objective) — how long can you be down before it's a problem?
- **RPO** (Recovery Point Objective) — how much data can you afford to lose (time since last backup)?
- DR strategies (know these, cost/speed trade-off, cheapest→most expensive):
  - **Backup & restore** — cheapest, slowest recovery.
  - **Pilot light** — minimal core infra always running in DR region, scaled up when needed.
  - **Warm standby** — smaller-scale but fully functional copy always running, scaled up on failover.
  - **Multi-site active-active** — full production capacity in two+ regions simultaneously, fastest recovery, most expensive.

---

# 2. Compute

### VMs / Instances
- Instance types are usually organized by workload profile: general purpose, compute-optimized, memory-optimized, storage-optimized, GPU/accelerated.
- **Pricing models**:
  - **On-demand** — pay per second/hour, no commitment, most expensive per unit.
  - **Reserved / Savings Plans** — commit to 1 or 3 years for a discount (up to ~70%).
  - **Spot instances** — bid on unused capacity, cheapest (up to ~90% off), but can be reclaimed with short notice — good for fault-tolerant/batch workloads, bad for stateful production traffic.

### Auto Scaling
- Scales instance count based on metrics (CPU, memory, custom CloudWatch/Monitor metrics, or request count).
- **Scaling policies**: target tracking (maintain a metric at X%), step scaling (scale in increments based on alarm severity), scheduled scaling (predictable traffic patterns, e.g., scale up before 9am).
- Always paired with a load balancer for traffic distribution.

### Containers — Docker Fundamentals
- **Image** — a read-only template built in layers (each Dockerfile instruction = a layer, cached for faster rebuilds).
- **Container** — a running instance of an image, with its own writable layer on top.
- **Volume** — persistent storage that survives container restarts/removal (containers themselves are ephemeral).
- **Networking** — bridge network (default, isolated per host), host network (shares host's network stack), overlay network (multi-host, used in Swarm/Kubernetes).
- Common interview trap: "why is my container's data gone after restart?" → answer: didn't use a volume, wrote to the container's writable layer instead.

### Kubernetes
- **Pod** — smallest deployable unit, one or more tightly-coupled containers sharing network/storage.
- **Deployment** — manages a set of identical pod replicas, handles rolling updates/rollbacks.
- **Service** — stable network endpoint in front of a changing set of pods (ClusterIP, NodePort, LoadBalancer types).
- **Ingress** — HTTP(S) routing into the cluster (path/host-based routing, TLS termination), needs an Ingress Controller (nginx, ALB Ingress Controller, etc.) to actually function.
- **ConfigMap** — non-sensitive config data injected into pods.
- **Secret** — same idea but for sensitive data (base64-encoded by default, NOT encrypted at rest unless you enable encryption — a common misconception to correct in interviews).
- **HPA (Horizontal Pod Autoscaler)** — scales pod count based on metrics.
- **Namespace** — logical isolation boundary within a cluster (used for multi-team/multi-env separation).
- **RBAC** — Role-Based Access Control — Roles/ClusterRoles define permissions, RoleBindings/ClusterRoleBindings assign them to users/service accounts.
- Managed Kubernetes (EKS/AKS/GKE) — cloud provider manages the control plane (API server, etcd, scheduler); you manage worker nodes (or go fully serverless with Fargate/AKS virtual nodes/GKE Autopilot).

### Serverless (Lambda / Azure Functions / Cloud Functions)
- Event-driven, no server management, billed per invocation/duration.
- **Cold start** — latency penalty on the first invocation after idle (or scale-up), because a new execution environment must be provisioned. Mitigations: provisioned concurrency (AWS), keeping functions warm, smaller deployment packages.
- **Triggers** — HTTP requests, queue messages, storage events, scheduled (cron-like) events.
- **Limits** — execution time limits (e.g., 15 min for Lambda), memory limits, and this shapes what workloads are appropriate (short, stateless tasks — not long-running processes).

---

# 3. Networking

### VPC/VNet Design
- A **VPC** is your own isolated virtual network within the cloud.
- **Subnets** — a VPC is divided into subnets, tied to a specific AZ.
  - **Public subnet** — has a route to an Internet Gateway, resources can have public IPs.
  - **Private subnet** — no direct internet route; outbound internet access (if needed) goes through a **NAT Gateway** sitting in a public subnet.
- **Route tables** — define where traffic from a subnet is allowed to go.

### Security Groups vs NACLs
- **Security Group** — stateful, attached to instances/ENIs, allow rules only (no explicit deny — if not allowed, it's denied). Return traffic is automatically allowed.
- **NACL (Network ACL)** — stateless, attached to subnets, supports both allow AND deny rules, evaluated in rule-number order. Because it's stateless, you must explicitly allow return traffic too.
- Common interview question: "why would you use both?" — defense in depth: NACLs as a coarse subnet-level filter, security groups as fine-grained per-resource control.

### Load Balancers
- **Layer 4 (Network/Transport)** — routes based on IP/port, very fast, protocol-agnostic (TCP/UDP).
- **Layer 7 (Application)** — routes based on HTTP content (path, host header, headers), can do SSL termination, path-based routing to different services.
- **Health checks** — LB periodically pings targets; unhealthy targets are removed from rotation automatically.

### DNS
- Route 53 / Azure DNS / Cloud DNS — managed DNS services.
- Know: A record (IP), CNAME (alias to another domain), record TTL, routing policies (weighted, latency-based, geolocation, failover).

### Connectivity between on-prem and cloud
- **VPN** — encrypted tunnel over the public internet, cheaper, higher latency/variable.
- **Direct Connect (AWS) / ExpressRoute (Azure)** — dedicated private physical connection, more expensive, consistent low latency — used for large enterprises/Big-4-style client environments with heavy hybrid traffic.
- **VPC Peering** — connects two VPCs directly (non-transitive — A↔B and B↔C doesn't mean A↔C).
- **Transit Gateway** — hub-and-spoke model to connect many VPCs/on-prem networks without needing full-mesh peering.

### CDN
- Caches content at edge locations close to users, reduces latency and origin load. Know cache invalidation as a common operational task/interview scenario ("how do you push an urgent content update through a CDN cache?").

---

# 4. Identity & Security

### IAM
- **Users** — individual identities (people or apps).
- **Roles** — a set of permissions assumed temporarily (no long-term credentials) — the preferred way to grant access to services/cross-account access.
- **Policies** — JSON documents defining allowed/denied actions on resources.
- **Least privilege principle** — grant only the minimum permissions needed — this is the single most repeated phrase in cloud security interviews.

### Cross-Account & Federation
- **AssumeRole** — a mechanism to temporarily gain permissions in another account/role, commonly used in CI/CD pipelines and cross-account architectures (very relevant to the multi-account landing zone discussion earlier).
- **OIDC (OpenID Connect)** — modern way to let external identity providers (e.g., GitHub Actions, Okta) federate into cloud accounts without long-lived access keys — the current best practice, replacing static IAM access keys in CI/CD.

### Encryption
- **At-rest** — data encrypted while stored (disk, database, object storage) — typically via KMS/Key Vault-managed keys.
- **In-transit** — data encrypted while moving over the network (TLS/SSL).
- **KMS / Key Vault** — managed key storage/rotation services; know the difference between provider-managed keys vs customer-managed keys (CMK) — CMKs give you rotation/revocation control, which matters for compliance-heavy client work.

### Secrets Management
- Never hardcode secrets in code or plain env files. Use Vault, AWS Secrets Manager, Azure Key Vault — supports automatic rotation, access auditing, and fine-grained access policies.

### Perimeter Security
- **WAF (Web Application Firewall)** — filters HTTP traffic against rules (SQL injection, XSS patterns, rate limiting).
- **DDoS protection** — provider-level services (AWS Shield, Azure DDoS Protection) that absorb/mitigate volumetric attacks.

### Compliance (high-level awareness — critical for Big 4)
- **SOC 2** — audit standard around security, availability, processing integrity, confidentiality, privacy — very common in vendor/client due-diligence conversations.
- **ISO 27001** — international standard for information security management systems.
- **HIPAA** — US healthcare data protection regulation, relevant if the client is healthcare-adjacent.
- You don't need to be a compliance expert, but you should be able to say things like "I made sure logging and encryption were enabled to support the client's SOC 2 audit requirements" — this signals maturity beyond pure tooling knowledge.

---

# 5. Infrastructure as Code & Automation

*(Terraform is covered in the earlier dedicated notes — this section covers the surrounding ecosystem.)*

### Terraform vs Alternatives
- **Pulumi** — IaC using real programming languages (Python, TypeScript, Go) instead of HCL — appeals to teams wanting full programming constructs (loops, classes) natively rather than HCL's more limited expression language.
- **AWS CDK** — similar idea but AWS-specific, compiles down to CloudFormation.
- **CloudFormation/ARM templates/Bicep** — provider-native IaC (AWS/Azure respectively) — tightly integrated with the provider but not multi-cloud.

### Configuration Management: Ansible vs Terraform
- **Terraform** — provisions infrastructure (the "what exists").
- **Ansible** — configures what's running on that infrastructure (the "what's installed/configured on it") — push-based, agentless (SSH), procedural/task-based (though it aims for idempotency too).
- **Chef/Puppet** — older configuration management tools, pull-based with agents — less commonly greenfield-adopted now but you'll still find them in legacy client environments (common in Big 4 work).
- Common combo in real projects: Terraform provisions the VM/cluster, Ansible (or cloud-init/user-data scripts) configures the software on top.

### GitOps
- Concept: Git is the single source of truth for both application and infrastructure state; a controller (ArgoCD, Flux) continuously reconciles the live cluster state to match what's declared in Git.
- Especially relevant for Kubernetes-heavy environments — instead of `kubectl apply` manually, you commit to Git and the GitOps controller auto-syncs the cluster, giving you full audit history and easy rollback (git revert).

---

# 6. CI/CD & DevOps Practices

### Pipeline Stages (typical flow)
`Build → Unit test → Static/security scan → Package/artifact → Deploy to staging → Integration test → Deploy to prod (with approval gate)`

### Common Tools
- **Jenkins** — highly customizable, self-hosted, plugin-heavy (older but still very common in enterprise/Big 4 client environments).
- **GitHub Actions / GitLab CI** — YAML-based, tightly integrated with the source repo, increasingly the default for greenfield projects.
- **Azure DevOps** — common in Microsoft-heavy enterprise shops.

### Deployment Strategies
- **Blue/Green** — two full environments; traffic is switched all at once from old (blue) to new (green). Instant rollback (switch back), but doubles infrastructure cost during the switch.
- **Canary** — gradually shift a small percentage of traffic to the new version, monitor, then increase — lower blast radius if something's wrong, but more complex to set up (traffic splitting, monitoring gates).
- **Rolling update** — replace instances/pods incrementally, no full duplicate environment needed, but rollback is slower and you briefly run mixed versions.

### Artifact Management
- Nexus, JFrog Artifactory, or cloud-native registries (ECR, ACR, GCR) — store built artifacts (Docker images, packages) with versioning, used as the handoff point between build and deploy stages.

### Branching Strategies
- **GitFlow** — long-lived `develop`/`release`/`feature` branches, heavier process, common in traditional enterprise release cycles.
- **Trunk-based development** — everyone commits to `main` frequently with short-lived feature branches, relies heavily on feature flags and strong CI — favored by high-velocity teams; be ready to discuss trade-offs (trunk-based needs strong test automation to be safe).

---

# 7. Monitoring, Logging & Observability

### The Three Pillars
- **Metrics** — numeric time-series data (CPU %, request count, latency) — good for dashboards/alerting/trends.
- **Logs** — discrete, timestamped event records — good for debugging specific incidents ("what exactly happened at 3:42pm").
- **Traces** — follow a single request across multiple services (distributed tracing) — essential in microservices to pinpoint which service caused latency/failure.

### Common Tools
- **CloudWatch / Azure Monitor / Cloud Monitoring** — provider-native metrics/logs/alarms.
- **Prometheus + Grafana** — open-source metrics collection (pull-based) + visualization, the default combo for Kubernetes environments.
- **ELK/EFK stack** (Elasticsearch, Logstash/Fluentd, Kibana) — centralized log aggregation and search.
- **Datadog / New Relic** — commercial full-stack observability platforms, common in enterprise/client environments because they unify metrics/logs/traces/APM in one place (reduces tool sprawl — often preferred by consulting clients over stitching open-source tools together).

### Alerting Design
- Alert on **symptoms that affect users** (high error rate, high latency) rather than every possible internal metric — avoids alert fatigue where real issues get lost in noise.
- Use severity tiers (page immediately vs log for later review).

### SLI / SLO / SLA
- **SLI (Service Level Indicator)** — the actual measured metric (e.g., 99.95% of requests succeeded).
- **SLO (Service Level Objective)** — the internal target you aim for (e.g., "99.9% success rate over 30 days").
- **SLA (Service Level Agreement)** — the external, often contractual, commitment to a customer/client — usually looser than the internal SLO, with financial penalties if missed. This distinction is a very common interview question.

---

# 8. Storage & Databases

### Storage Types
- **Object storage** (S3/Blob/GCS) — flat namespace, accessed via HTTP API, scales massively, used for unstructured data (files, backups, static assets, data lakes). Know **storage classes/tiers** (hot/standard → infrequent access → archive/glacier) and **lifecycle policies** that auto-transition/delete objects based on age — a very common cost-optimization interview topic.
- **Block storage** (EBS/Managed Disks) — attached to a single VM like a virtual hard drive, low latency, used for OS disks/databases.
- **File storage** (EFS/Azure Files) — shared network file system, mountable by multiple instances simultaneously — used when several servers need to read/write the same files.

### Managed Relational Databases (RDS / Azure SQL / Cloud SQL)
- Handles patching, backups, and failover for you.
- **Multi-AZ deployment** — synchronous standby replica in another AZ, automatic failover on primary failure — this is for **availability**, not read scaling.
- **Read replicas** — asynchronous copies used to offload read traffic — this is for **read scaling**, not typically used for failover (though can be manually promoted). This distinction (Multi-AZ vs read replica purpose) is a classic interview trap.
- Automated backups + point-in-time recovery — standard expectation for production databases.

### NoSQL (DynamoDB / Cosmos DB)
- Schema-less/flexible schema, horizontally scalable, used for high-throughput, simple-access-pattern workloads (key-value lookups) rather than complex joins/queries.

### Migration Strategies
- Lift-and-shift (rehost) vs replatform vs full refactor — know these terms; Big 4 engagements very frequently involve "assess and migrate legacy client workload to cloud" projects, so being able to talk through migration strategy trade-offs is valuable.

---

# 9. Cost Management (FinOps)

### Purchasing Models (recap with cost lens)
- On-demand = flexibility, highest cost.
- Reserved/Savings Plans = commitment discount, best for predictable steady-state workloads.
- Spot = cheapest, only for interruption-tolerant workloads (batch jobs, CI runners, stateless workers).

### Tagging Strategy
- Consistent tags (e.g., `team`, `environment`, `cost-center`, `project`) applied across all resources — this is what makes cost allocation/chargeback reporting possible. A very real, very common Big 4 consulting deliverable: "help the client understand which team/project is driving their cloud spend."

### Budgets & Anomaly Detection
- Set budget alerts (e.g., notify at 80% of monthly budget) and use anomaly detection tools to catch unexpected spend spikes early (e.g., a misconfigured auto-scaling group running wild).

### Right-Sizing
- Regularly review actual utilization vs provisioned capacity (CPU/memory usage) and downsize over-provisioned resources — one of the most common, easiest cost wins to describe in an interview if you have a real example.

---

# 10. Reliability & Resilience

### Chaos Engineering
- Deliberately injecting failure (killing instances, adding network latency) into a system to verify it behaves resiliently — "test in production, safely." Tools: Chaos Monkey, AWS Fault Injection Simulator, Gremlin.

### Backup & DR Strategies
- Already covered under Cloud Fundamentals (pilot light, warm standby, active-active) — be ready to map a real/hypothetical workload to the right strategy based on RTO/RPO requirements and budget.

### Incident Management
- **Blameless postmortems** — after an incident, the focus is on *what* failed in the system/process, not *who* made a mistake — this creates psychological safety so people report issues honestly instead of hiding them. Interviewers ask about this to gauge team culture maturity, not just technical skill.
- Know the basic incident lifecycle: detect → triage/severity → mitigate → resolve → postmortem → follow-up action items.

---

# 11. Scripting & Linux

### Bash
- Be comfortable with: variables, loops, conditionals, piping/redirection, basic `awk`/`sed`/`grep` for log parsing — this comes up constantly in "how would you find X in these logs" style questions.

### Python for Automation
- `boto3` (AWS SDK for Python) / Azure SDK — used for automation scripts beyond what Terraform/CLI easily covers (custom reporting, cleanup scripts, one-off migrations).

### Linux Fundamentals
- File permissions (`chmod`, `chown`, read/write/execute for owner/group/others).
- Process management (`ps`, `top`, `kill`, understanding zombie/orphan processes).
- `systemd` — how services are started/managed/enabled on boot (`systemctl status/start/enable`).
- Networking commands — `netstat`/`ss`, `curl`, `dig`/`nslookup`, `traceroute` — used constantly for troubleshooting connectivity issues, and a very common live/whiteboard interview scenario ("a service isn't reachable, walk me through how you'd debug it").

---

# 12. Big 4 / Consulting-Specific Preparation

These aren't pure technical topics, but Big 4 interviewers weight them heavily for a 5-year profile because the job is as much about client trust as it is about infrastructure:

- **Change management** — how do you roll out infrastructure changes safely across multiple client environments without surprising them? (Think: change advisory boards, maintenance windows, communication plans — not just "I ran terraform apply.")
- **Documentation & runbooks** — be ready to describe how you document architecture decisions and operational procedures so another engineer (or the client's own team) can pick up where you left off.
- **Cross-team/stakeholder communication** — translating technical trade-offs into business language for non-technical client stakeholders is a core consulting skill; have a real example ready.
- **Cost optimization storytelling** — have one concrete story (even a modest one) where you identified and delivered a cost saving, framed in business terms ("reduced monthly spend by X% by right-sizing Y").

---

### How to use these notes
Read once fully, then go back through and, for each topic, try to recall a **real example from your own experience** — Big 4 interviewers for a 5-year profile care much more about "tell me about a time you dealt with X" than "define X." The definitions here are just the scaffolding; the stories are what get you hired.
