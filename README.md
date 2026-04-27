# Agent Skills — Cloud Platform Library

86 comprehensive cloud platform skills for AI coding agents. Each skill is a standalone `SKILL.md` file that any AI coding assistant with skills support can use — including [Claude Code](https://docs.anthropic.com/en/docs/claude-code), [JaaiCode](https://github.com/tomz/jaaicode), and other agents that follow the SKILL.md convention.

## What Are Skills?

Skills are markdown instruction files (`SKILL.md`) that give AI coding agents domain expertise on demand. When a user's task matches a skill's trigger patterns, the agent loads the relevant instructions — CLI commands, best practices, architectural patterns, gotchas — so it can work competently in that domain without prior training.

These skills are agent-agnostic: any AI coding tool that supports the `SKILL.md` format can use them.

## Installation

```bash
# Clone all skills
git clone https://github.com/tomz/agent-skills.git /tmp/agent-skills

# Copy all into your agent's skills directory
cp -r /tmp/agent-skills/*-*/ ~/.jaaicode/skills/    # JaaiCode
cp -r /tmp/agent-skills/*-*/ ~/.claude/skills/       # Claude Code (if supported)

# Or install specific platforms
cp -r /tmp/agent-skills/azure-*/ ~/.jaaicode/skills/
cp -r /tmp/agent-skills/aws-*/ ~/.jaaicode/skills/
```

## Skills Catalog

### Azure (37 skills)

| Skill | Description |
|-------|-------------|
| `azure-acr` | Azure Container Registry — image management, ACR Tasks, geo-replication, vulnerability scanning |
| `azure-ai` | Azure OpenAI Service, Azure AI Search, Azure AI Services, Azure Machine Learning |
| `azure-apim` | Azure API Management — APIs, products, subscriptions, policies, developer portal, gateway, OAuth |
| `azure-app-config` | Azure App Configuration — centralized config management, feature flags, key-value store, Key Vault |
| `azure-arc` | Azure Arc — hybrid and multi-cloud management, Arc-enabled servers, Kubernetes, SQL, data services |
| `azure-avd` | Azure Virtual Desktop — host pools, session hosts, application groups, FSLogix, MSIX app attach |
| `azure-backup` | Azure Backup — Recovery Services vault, Backup vault, VM backup, SQL/SAP HANA backup, MARS agent |
| `azure-batch` | Azure Batch — HPC workloads, pools, jobs, tasks, auto-scaling, container tasks |
| `azure-cli` | Azure CLI (az) patterns, authentication, resource management, output formats, JMESPath queries |
| `azure-communication` | Azure Communication Services — SMS, email, voice calling, video calling, chat, phone numbers |
| `azure-compute` | Azure compute — VMs, VMSS, App Service, Container Apps, Functions, AKS, ACI |
| `azure-cosmosdb` | Azure Cosmos DB — multi-model globally distributed database, SQL API, MongoDB API, Cassandra |
| `azure-costs` | Azure Cost Management — Cost Analysis, Budgets, Advisor, Reservations, Savings Plans |
| `azure-databases` | Azure managed databases — SQL Database, SQL Managed Instance, MySQL, PostgreSQL Flexible Server |
| `azure-databricks` | Azure Databricks — workspace provisioning, Unity Catalog, clusters, SQL warehouses, Delta Lake |
| `azure-devops` | Azure DevOps Pipelines (YAML), Repos, Artifacts, GitHub Actions for Azure, blue-green/canary |
| `azure-devtest-labs` | Azure DevTest Labs — lab environments, VM management, formulas, artifacts, cost management |
| `azure-fabric` | Microsoft Fabric unified analytics — OneLake, lakehouses, warehouses, Data Factory, Real-Time |
| `azure-firewall` | Azure Firewall — Standard/Premium/Basic SKUs, policies, DNAT/network/application rules |
| `azure-hdinsight` | Azure HDInsight — managed Hadoop ecosystem (Spark, Hive, HBase, Kafka, Storm) |
| `azure-iac` | Infrastructure as Code for Azure — Bicep, ARM templates, Terraform azurerm |
| `azure-iot` | Azure IoT — IoT Hub, IoT Central, IoT Edge, Device Provisioning Service, Digital Twins |
| `azure-lighthouse` | Azure Lighthouse — cross-tenant management, delegated resource management |
| `azure-load-testing` | Azure Load Testing — JMeter-based load tests, CI/CD integration, server-side metrics |
| `azure-logic-apps` | Azure Logic Apps — workflow automation, connectors, triggers, actions |
| `azure-messaging` | Azure messaging — Event Hubs, Service Bus, Event Grid, Queue Storage, stream processing |
| `azure-migrate` | Azure Migrate — discovery, assessment, migration of servers, databases, web apps |
| `azure-monitoring` | Azure Monitor, Log Analytics, KQL, Application Insights, Alerts, Diagnostic Settings |
| `azure-networking` | Azure networking — VNets, NSGs, Load Balancer, App Gateway, Front Door, Private Link |
| `azure-powerbi` | Azure Power BI — datasets, reports, dashboards, embedded analytics |
| `azure-power-platform` | Microsoft Power Platform — Power Apps, Power Automate, Power Pages, Dataverse |
| `azure-purview` | Microsoft Purview — data governance, data catalog, data map, lineage, classification |
| `azure-redis` | Azure Cache for Redis — caching patterns, tiers, clustering, geo-replication |
| `azure-security` | Azure security — Entra ID, RBAC, Key Vault, Managed Identity, Defender for Cloud, Sentinel |
| `azure-signalr` | Azure SignalR Service & Web PubSub — real-time messaging, hubs, groups, serverless mode |
| `azure-static-web-apps` | Azure Static Web Apps — JAMstack hosting, API backends, authentication, custom domains |
| `azure-synapse` | Azure Synapse Analytics — dedicated SQL pools, serverless SQL, Spark pools, pipelines |

### AWS (10 skills)

| Skill | Description |
|-------|-------------|
| `aws-ai` | AWS AI/ML — Bedrock, SageMaker, Comprehend, Rekognition, Textract, Lex, Polly, Transcribe |
| `aws-cli` | AWS CLI v2 patterns, profiles, SSO, JMESPath queries, pagination, waiters |
| `aws-compute` | AWS compute — EC2, ECS (Fargate/EC2), EKS, Lambda, App Runner, Elastic Beanstalk, Lightsail |
| `aws-costs` | AWS cost management — Cost Explorer, Budgets, Savings Plans, Reserved Instances, Spot |
| `aws-data` | AWS data — RDS, Aurora, DynamoDB, S3, Redshift, ElastiCache, DocumentDB, Glue |
| `aws-devops` | AWS DevOps — CodePipeline, CodeBuild, CodeDeploy, ECR, GitHub Actions OIDC |
| `aws-iac` | Infrastructure as Code for AWS — CloudFormation, CDK, Terraform, SAM |
| `aws-monitoring` | AWS monitoring — CloudWatch, X-Ray, CloudTrail Lake, Config, EventBridge |
| `aws-networking` | AWS networking — VPC, Security Groups, ALB/NLB, CloudFront, Route 53, Transit Gateway |
| `aws-security` | AWS security — IAM, KMS, Secrets Manager, GuardDuty, Security Hub, WAF, Config |

### GCP (10 skills)

| Skill | Description |
|-------|-------------|
| `gcp-ai` | GCP AI/ML — Vertex AI, Gemini API, Document AI, Vision AI, Natural Language AI, AutoML |
| `gcp-cli` | gcloud CLI patterns, authentication, configurations, output formats, gsutil, bq CLI |
| `gcp-compute` | GCP compute — Compute Engine, GKE, Cloud Run, Cloud Functions, App Engine |
| `gcp-costs` | GCP cost management — billing accounts, budgets, BigQuery export, Recommender, CUDs |
| `gcp-data` | GCP data — Cloud SQL, Firestore, Bigtable, BigQuery, Cloud Storage, Pub/Sub, Dataflow |
| `gcp-devops` | GCP DevOps — Cloud Build, Artifact Registry, Cloud Deploy, GitHub Actions |
| `gcp-iac` | Infrastructure as Code for GCP — Terraform google provider, Deployment Manager, Pulumi |
| `gcp-monitoring` | GCP observability — Cloud Monitoring, Cloud Logging, Cloud Trace, Error Reporting |
| `gcp-networking` | GCP networking — VPC, Load Balancing, Cloud CDN, DNS, Armor, NAT, Interconnect |
| `gcp-security` | GCP security — IAM, Secret Manager, SCC, VPC Service Controls, Binary Authorization |

### Oracle Cloud (10 skills)

| Skill | Description |
|-------|-------------|
| `oci-ai` | OCI AI — Generative AI, Data Science, AI Language, AI Vision, AI Speech |
| `oci-cli` | Oracle Cloud Infrastructure CLI patterns, authentication, config, common commands |
| `oci-compute` | OCI Compute — instances, shapes, OKE, Container Instances, Functions, Autoscaling |
| `oci-costs` | OCI cost management — Cost Analysis, Budgets, Usage Reports, quotas, Always Free |
| `oci-data` | OCI data — Autonomous Database, MySQL HeatWave, NoSQL, Object Storage, GoldenGate |
| `oci-devops` | OCI DevOps — build/deploy pipelines, OCIR container registry, GitHub Actions |
| `oci-iac` | Infrastructure as Code for OCI — Terraform provider, Resource Manager, Ansible |
| `oci-monitoring` | OCI observability — Monitoring metrics/MQL/alarms, Logging, Events, Notifications |
| `oci-networking` | OCI networking — VCN, subnets, security, load balancers, gateways, DRG, VPN, DNS |
| `oci-security` | OCI security — IAM with Identity Domains, policies, Vault, Cloud Guard, Bastion |

### DigitalOcean (6 skills)

| Skill | Description |
|-------|-------------|
| `do-app-platform` | DigitalOcean App Platform — app specs, components, deploys, scaling, env vars |
| `do-cli` | DigitalOcean doctl CLI — authentication, contexts, output formats |
| `do-compute` | DigitalOcean Compute — Droplets, volumes, snapshots, firewalls, reserved IPs |
| `do-data` | DigitalOcean managed databases (PostgreSQL, MySQL, Redis, MongoDB, Kafka) and Spaces |
| `do-k8s` | DigitalOcean Kubernetes (DOKS), node pools, auto-scaling, Container Registry |
| `do-networking` | DigitalOcean networking — VPC, Load Balancers, DNS, CDN, firewalls |

### Salesforce (6 skills)

| Skill | Description |
|-------|-------------|
| `sf-apex` | Apex language — triggers, classes, batch/queueable/schedulable, governor limits |
| `sf-cli` | Salesforce CLI (sf) — auth, scratch orgs, sandboxes, deployments, metadata API |
| `sf-data` | SOQL, SOSL, Data Loader, Bulk API 2.0, Record Types, sharing rules |
| `sf-devops` | Salesforce DevOps — scratch org model, unlocked packages, CI/CD with GitHub Actions |
| `sf-integrations` | Salesforce integrations — REST API, Bulk API, Streaming API, Platform Events, CDC |
| `sf-lwc` | Lightning Web Components — structure, reactive properties, lifecycle, wire service |

### IBM Cloud (4 skills)

| Skill | Description |
|-------|-------------|
| `ibm-cli` | IBM Cloud CLI (ibmcloud) — login, targeting, IAM, plugins, resource management |
| `ibm-data` | Db2 on Cloud, IBM Cloud Databases, Cloud Object Storage |
| `ibm-openshift` | Red Hat OpenShift on IBM Cloud (ROKS) — cluster lifecycle, worker pools, oc CLI |
| `ibm-watsonx` | watsonx.ai (foundation models, prompt lab, tuning), watsonx.data, watsonx.governance |

### CoreWeave (3 skills)

| Skill | Description |
|-------|-------------|
| `coreweave-gpu` | CoreWeave GPU workloads — requesting GPUs, multi-GPU jobs, MIG, spot pricing, NCCL |
| `coreweave-inference` | CoreWeave inference endpoints — vLLM/TGI/Triton on Kubernetes, autoscaling |
| `coreweave-k8s` | CoreWeave Kubernetes platform — kubeconfig, namespaces, storage, networking |

## Skill Format

Each skill is a directory containing a `SKILL.md` file with YAML frontmatter (name, trigger patterns, description) followed by markdown instructions:

```
skill-name/
└── SKILL.md          # YAML frontmatter + markdown instructions
```

The SKILL.md format is compatible with any AI coding agent that supports markdown-based skill loading.

## Contributing

To add a new skill, create a directory with a `SKILL.md` file. The frontmatter should include `name`, `triggers` (keyword patterns that activate the skill), and a `description`. The markdown body contains the actual instructions the agent will follow.

## License

MIT
