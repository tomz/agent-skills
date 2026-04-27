# JaaiCode Cloud Skills Library

86 comprehensive cloud platform skills for [JaaiCode](https://github.com/tomz/jaaicode) — the self-hosted CLI AI coding assistant.

## Installation

```bash
# Clone and copy all skills
git clone https://github.com/tomz/agent-skills.git /tmp/cloud-skills
cp -r /tmp/cloud-skills/*-*/ ~/.jaaicode/skills/

# Or install specific platforms
cp -r /tmp/cloud-skills/azure-*/ ~/.jaaicode/skills/
cp -r /tmp/cloud-skills/aws-*/ ~/.jaaicode/skills/

# Or use jaaicode's built-in install command
/skill install github:tomz/agent-skills
/skill install github:tomz/agent-skills --only "azure-*"
```

## Skills Catalog

### Azure (37 skills)

| Skill | Description |
|-------|-------------|
| `azure-acr` | Azure Container Registry — image management, ACR Tasks, geo-replication, vulnerability scanning, O |
| `azure-ai` | Azure OpenAI Service, Azure AI Search, Azure AI Services (Cognitive Services), Azure Machine Learnin |
| `azure-apim` | Azure API Management — APIs, products, subscriptions, policies, developer portal, gateway, OAuth,  |
| `azure-app-config` | Azure App Configuration — centralized config management, feature flags, key-value store, Key Vault |
| `azure-arc` | Azure Arc — hybrid and multi-cloud management, Arc-enabled servers, Kubernetes, SQL, data services |
| `azure-avd` | Azure Virtual Desktop — host pools, session hosts, application groups, FSLogix, MSIX app attach, s |
| `azure-backup` | Azure Backup — Recovery Services vault, Backup vault, VM backup, SQL/SAP HANA backup, MARS agent,  |
| `azure-batch` | Azure Batch — HPC workloads, pools, jobs, tasks, auto-scaling, container tasks, multi-instance tas |
| `azure-cli` | Azure CLI (az) patterns, authentication, resource management, output formats, JMESPath queries, and  |
| `azure-communication` | Azure Communication Services — SMS, email, voice calling, video calling, chat, phone numbers, room |
| `azure-compute` | Azure compute — VMs, VMSS, App Service, Container Apps, Functions, AKS, ACI — sizing, availabili |
| `azure-cosmosdb` | Azure Cosmos DB — multi-model globally distributed database, SQL API, MongoDB API, Cassandra, Grem |
| `azure-costs` | Azure Cost Management — Cost Analysis, Budgets, Advisor, Reservations, Savings Plans, right-sizing |
| `azure-databases` | Azure managed databases — SQL Database, SQL Managed Instance, MySQL Flexible Server, PostgreSQL Fl |
| `azure-databricks` | Azure Databricks — workspace provisioning, Unity Catalog, clusters, SQL warehouses, Delta Lake, ML |
| `azure-devops` | Azure DevOps Pipelines (YAML), Repos, Artifacts, GitHub Actions for Azure, ACR, blue-green/canary de |
| `azure-devtest-labs` | Azure DevTest Labs — lab environments, VM management, formulas, artifacts, cost management, auto-s |
| `azure-fabric` | Microsoft Fabric unified analytics platform — OneLake, lakehouses, warehouses, Data Factory, Real- |
| `azure-firewall` | Azure Firewall — Standard/Premium/Basic SKUs, firewall policies, DNAT/network/application rules, I |
| `azure-hdinsight` | Azure HDInsight — managed Hadoop ecosystem (Spark, Hive, HBase, Kafka, Storm), Enterprise Security |
| `azure-iac` | Infrastructure as Code for Azure — Bicep (preferred), ARM templates, Terraform azurerm, deployment |
| `azure-iot` | Azure IoT — IoT Hub, IoT Central, IoT Edge, Device Provisioning Service, Digital Twins, device man |
| `azure-lighthouse` | Azure Lighthouse — cross-tenant management, delegated resource management, managed service offers, |
| `azure-load-testing` | Azure Load Testing — JMeter-based load tests, CI/CD integration, server-side metrics, auto-stop cr
Triggered from CI pipeline |
| `azure-logic-apps` | Azure Logic Apps — workflow automation, connectors, triggers, actions, Standard vs Consumption, in |
| `azure-messaging` | Azure messaging — Event Hubs, Service Bus, Event Grid, Queue Storage, stream processing, pub/sub,  |
| `azure-migrate` | Azure Migrate — discovery, assessment, migration of servers, databases, web apps, VDI; Azure Site  |
| `azure-monitoring` | Azure Monitor, Log Analytics, KQL query patterns, Application Insights, Alerts, Diagnostic Settings, |
| `azure-networking` | Azure networking — VNets, NSGs, ASGs, UDRs, Load Balancer, App Gateway, Front Door, Private Link,  |
| `azure-powerbi` | Azure Power BI — datasets, reports, dashboards, embedded analytics, Power BI Service, Gateway, dat |
| `azure-power-platform` | Microsoft Power Platform — Power Apps, Power Automate, Power Pages, Dataverse, environments, solut |
| `azure-purview` | Microsoft Purview — data governance, data catalog, data map, lineage, classification, sensitivity  |
| `azure-redis` | Azure Cache for Redis — caching patterns, tiers, clustering, geo-replication, data persistence, Re |
| `azure-security` | Azure security — Microsoft Entra ID, RBAC, Key Vault, Managed Identity, Defender for Cloud, Sentin |
| `azure-signalr` | Azure SignalR Service & Web PubSub — real-time messaging, hubs, groups, serverless mode, upstream, |
| `azure-static-web-apps` | Azure Static Web Apps — JAMstack hosting, API backends, authentication, custom domains, staging en |
| `azure-synapse` | Azure Synapse Analytics — dedicated SQL pools, serverless SQL, Spark pools, pipelines, Synapse Lin |

### AWS (10 skills)

| Skill | Description |
|-------|-------------|
| `aws-ai` | AWS AI/ML services — Bedrock, SageMaker, Comprehend, Rekognition, Textract, Lex, Polly, Transcribe |
| `aws-cli` | AWS CLI v2 patterns, profiles, SSO, JMESPath queries, pagination, waiters, and productivity aliases |
| `aws-compute` | AWS compute services — EC2, ECS (Fargate/EC2), EKS, Lambda, App Runner, Elastic Beanstalk, Lightsa |
| `aws-costs` | AWS cost management — Cost Explorer, Budgets, Savings Plans, Reserved Instances, Spot, Trusted Adv |
| `aws-data` | AWS data services — RDS, Aurora, DynamoDB, S3, Redshift, ElastiCache, DocumentDB, Glue |
| `aws-devops` | AWS DevOps — CodePipeline, CodeBuild, CodeDeploy, ECR, GitHub Actions OIDC, CodeArtifact, deployme |
| `aws-iac` | Infrastructure as Code for AWS — CloudFormation, CDK (TypeScript/Python), Terraform, SAM, and Rain |
| `aws-monitoring` | AWS monitoring — CloudWatch metrics/logs/alarms, X-Ray, CloudTrail Lake, Config, EventBridge, SNS/ |
| `aws-networking` | AWS networking — VPC, Security Groups, NACLs, ALB/NLB, CloudFront, Route 53, Transit Gateway, Priv |
| `aws-security` | AWS security — IAM, KMS, Secrets Manager, GuardDuty, Security Hub, WAF, Config, CloudTrail, SCPs,  |

### GCP (10 skills)

| Skill | Description |
|-------|-------------|
| `gcp-ai` | GCP AI/ML — Vertex AI, Gemini API, Document AI, Vision AI, Natural Language AI, AutoML, Vertex AI  |
| `gcp-cli` | gcloud CLI patterns, authentication, configurations, output formats, gsutil, bq CLI |
| `gcp-compute` | GCP compute — Compute Engine, GKE (Autopilot/Standard), Cloud Run, Cloud Functions, App Engine, Ba |
| `gcp-costs` | GCP cost management — billing accounts, budgets, BigQuery export, Recommender, CUDs, Spot pricing, |
| `gcp-data` | GCP data services — Cloud SQL, Firestore, Bigtable, BigQuery, Cloud Storage, Pub/Sub, Dataflow, Sp |
| `gcp-devops` | GCP DevOps — Cloud Build, Artifact Registry, Cloud Deploy, GitHub Actions, Source Repositories, Sk |
| `gcp-iac` | Infrastructure as Code for GCP — Terraform google provider, Deployment Manager, Pulumi, Config Con |
| `gcp-monitoring` | GCP observability — Cloud Monitoring, Cloud Logging, Cloud Trace, Error Reporting, Profiler |
| `gcp-networking` | GCP networking — VPC, Load Balancing, Cloud CDN, DNS, Armor, NAT, Interconnect, VPN, VPC Service C |
| `gcp-security` | GCP security — IAM, Secret Manager, SCC, VPC Service Controls, Binary Authorization, Certificate A |

### Oracle Cloud (10 skills)

| Skill | Description |
|-------|-------------|
| `oci-ai` | OCI AI services — Generative AI, Data Science, AI Language, AI Vision, AI Speech, Anomaly Detectio |
| `oci-cli` | Oracle Cloud Infrastructure CLI patterns, authentication, config, and common commands |
| `oci-compute` | OCI Compute — instances, shapes, OKE, Container Instances, Functions, and Autoscaling |
| `oci-costs` | OCI cost management — Cost Analysis, Budgets, Usage Reports, quotas, Always Free, Reserved capacit |
| `oci-data` | OCI data services — Autonomous Database, MySQL HeatWave, NoSQL, Object Storage, GoldenGate, Stream |
| `oci-devops` | OCI DevOps service — build/deploy pipelines, OCIR container registry, GitHub Actions integration,  |
| `oci-iac` | Infrastructure as Code for OCI — Terraform provider, Resource Manager, Ansible, and state manageme |
| `oci-monitoring` | OCI observability — Monitoring metrics/MQL/alarms, Logging, Logging Analytics, Events, Notificatio |
| `oci-networking` | OCI networking — VCN, subnets, security, load balancers, gateways, DRG, VPN, DNS, and peering |
| `oci-security` | OCI security — IAM with Identity Domains, policies, Vault, Cloud Guard, Bastion, Security Zones, W |

### DigitalOcean (6 skills)

| Skill | Description |
|-------|-------------|
| `do-app-platform` | DigitalOcean App Platform — app specs, components, deploys, scaling, env vars, custom domains, and |
| `do-cli` | DigitalOcean doctl CLI — authentication, contexts, output formats, and core subcommand patterns |
| `do-compute` | DigitalOcean Compute — Droplets, volumes, snapshots, firewalls, reserved IPs, SSH keys, and tags |
| `do-data` | DigitalOcean managed databases (PostgreSQL, MySQL, Redis, MongoDB, Kafka) and Spaces object storage |
| `do-k8s` | DigitalOcean Kubernetes (DOKS), node pools, auto-scaling, Container Registry, and monitoring |
| `do-networking` | DigitalOcean networking — VPC, Load Balancers, DNS, CDN, and firewall strategies |

### Salesforce (6 skills)

| Skill | Description |
|-------|-------------|
| `sf-apex` | Apex language patterns — triggers, classes, batch/queueable/schedulable, governor limits, bulkific |
| `sf-cli` | Salesforce CLI (sf command) — auth, scratch orgs, sandboxes, deployments, metadata API, project st |
| `sf-data` | SOQL, SOSL, Data Loader, sf data CLI, Bulk API 2.0, Record Types, sharing rules, field-level securit |
| `sf-devops` | Salesforce DevOps — scratch org model, unlocked packages (1GP/2GP), CI/CD with GitHub Actions, Dev |
| `sf-integrations` | Salesforce integrations — REST API, Bulk API, Streaming API (Platform Events, CDC), Connected Apps |
| `sf-lwc` | Lightning Web Components — component structure, reactive properties, lifecycle hooks, wire service |

### IBM Cloud (4 skills)

| Skill | Description |
|-------|-------------|
| `ibm-cli` | IBM Cloud CLI (ibmcloud) — login, targeting, IAM, plugins, resource management, output formats |
| `ibm-data` | Db2 on Cloud, IBM Cloud Databases (PostgreSQL/MongoDB/Redis/etc via ibmcloud cdb), Cloud Object Stor |
| `ibm-openshift` | Red Hat OpenShift on IBM Cloud (ROKS) — cluster lifecycle, worker pools, oc CLI, networking, IBM C |
| `ibm-watsonx` | watsonx.ai (foundation models, prompt lab, tuning), watsonx.data (Presto/Iceberg lakehouse), watsonx |

### CoreWeave (3 skills)

| Skill | Description |
|-------|-------------|
| `coreweave-gpu` | CoreWeave GPU workloads — requesting GPUs, multi-GPU jobs, MIG, spot pricing, NCCL tuning, and nod |
| `coreweave-inference` | CoreWeave inference endpoints — vLLM/TGI/Triton on Kubernetes, autoscaling, traffic splitting, A/B |
| `coreweave-k8s` | CoreWeave Kubernetes platform — kubeconfig, namespaces, storage, networking, node affinity, and th |

## Skill Format

Each skill is a directory with a `SKILL.md` file following the [Open Agent Skills spec](https://github.com/your/spec):

```
skill-name/
├── SKILL.md          # YAML frontmatter + markdown instructions
├── scripts/          # Optional executables
├── references/       # Optional docs
└── assets/           # Optional static files
```

## License

MIT
