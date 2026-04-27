---
name: azure-batch
description: Azure Batch — HPC workloads, pools, jobs, tasks, auto-scaling, container tasks, multi-instance tasks, rendering, low-priority VMs
license: MIT
version: 1.0.0
updated: 2026-04-24
allowed-tools: shell, read_file, write_file, edit_file, glob, grep
---

# Azure Batch

## Overview

Azure Batch is a managed HPC service for running large-scale parallel and batch
compute jobs. You provision pools of VMs, submit jobs with tasks, and Azure Batch
schedules and executes them — no cluster management required.

---

## Batch Accounts

Two allocation modes:

| Mode | Description | Use Case |
|------|-------------|----------|
| **Batch service** | Pools are in Batch-managed subscription | Default; simplest |
| **User subscription** | Pool VMs appear in your subscription | Required for reserved instances, specific policies |

```bash
# Create Batch account (Batch service mode)
az batch account create \
  --name mybatchaccount \
  --resource-group rg-batch \
  --location eastus \
  --storage-account mystorageaccount

# Create in user subscription mode
az batch account create \
  --name mybatchaccount \
  --resource-group rg-batch \
  --location eastus \
  --storage-account mystorageaccount \
  --keyvault myKeyVault \
  --pool-allocation-mode UserSubscription

# Log in to Batch account for az batch commands
az batch account login \
  --name mybatchaccount \
  --resource-group rg-batch \
  --shared-key-auth

# Get account keys
az batch account keys list \
  --name mybatchaccount \
  --resource-group rg-batch
```

---

## Pools

### VM Images

```bash
# List available Marketplace images supported by Batch
az batch pool supported-images list \
  --filter "verificationType eq 'verified'" \
  --query "[?contains(imageReference.offer,'ubuntu')]" \
  -o table

# Create pool with Marketplace image
az batch pool create \
  --id mypool \
  --vm-size Standard_D4s_v3 \
  --image canonical:0001-com-ubuntu-server-focal:20_04-lts:latest \
  --node-agent-sku-id "batch.node.ubuntu 20.04" \
  --target-dedicated-nodes 4 \
  --target-low-priority-nodes 2

# Create pool with custom VHD image
az batch pool create \
  --id mypool-custom \
  --vm-size Standard_D4s_v3 \
  --image /subscriptions/<sub>/resourceGroups/rg-images/providers/Microsoft.Compute/galleries/myGallery/images/myImage/versions/1.0.0 \
  --node-agent-sku-id "batch.node.ubuntu 20.04" \
  --target-dedicated-nodes 2
```

### Pool JSON (full configuration)

```json
{
  "id": "mypool",
  "vmSize": "Standard_D4s_v3",
  "virtualMachineConfiguration": {
    "imageReference": {
      "publisher": "canonical",
      "offer": "0001-com-ubuntu-server-focal",
      "sku": "20_04-lts",
      "version": "latest"
    },
    "nodeAgentSkuId": "batch.node.ubuntu 20.04",
    "containerConfiguration": {
      "type": "dockerCompatible",
      "containerImageNames": ["pytorch/pytorch:2.3.0-cuda12.1-cudnn8-runtime"]
    }
  },
  "targetDedicatedNodes": 4,
  "targetLowPriorityNodes": 8,
  "enableAutoScale": false,
  "startTask": {
    "commandLine": "/bin/bash -c 'apt-get update && apt-get install -y ffmpeg'",
    "userIdentity": { "autoUser": { "elevationLevel": "admin" } },
    "waitForSuccess": true,
    "maxTaskRetryCount": 2
  },
  "networkConfiguration": {
    "subnetId": "/subscriptions/<sub>/resourceGroups/rg-batch/providers/Microsoft.Network/virtualNetworks/myVnet/subnets/batchSubnet"
  },
  "mountConfiguration": [
    {
      "azureBlobFileSystemConfiguration": {
        "accountName": "mystorageaccount",
        "containerName": "input-data",
        "relativeMountPath": "input",
        "sasKey": "<sas_token>"
      }
    },
    {
      "azureFileShareConfiguration": {
        "accountName": "mystorageaccount",
        "azureFileUrl": "https://mystorageaccount.file.core.windows.net/share",
        "accountKey": "<key>",
        "relativeMountPath": "share"
      }
    }
  ],
  "taskSlotsPerNode": 4
}
```

```bash
az batch pool create --json-file pool.json
```

---

## Auto-Scale

### Enable Auto-Scale on a Pool

```bash
az batch pool autoscale enable \
  --pool-id mypool \
  --auto-scale-formula "
    startingNumberOfVMs = 1;
    maxNumberOfVMs = 25;
    pendingTaskSamplePercent = $PendingTasks.GetSamplePercent(180 * TimeInterval_Second);
    pendingTaskSamples = pendingTaskSamplePercent < 70 ? startingNumberOfVMs : avg(\$PendingTasks.GetSample(180 * TimeInterval_Second));
    \$TargetDedicatedNodes = min(maxNumberOfVMs, pendingTaskSamples);
    \$TargetLowPriorityNodes = 0;
    \$NodeDeallocationOption = taskcompletion;
  " \
  --auto-scale-evaluation-interval "PT5M"
```

### Common Auto-Scale Formula Variables

| Variable | Description |
|----------|-------------|
| `$PendingTasks` | Tasks queued and not yet running |
| `$RunningTasks` | Tasks actively executing |
| `$ActiveTasks` | Active + pending tasks |
| `$TargetDedicatedNodes` | Desired dedicated node count |
| `$TargetLowPriorityNodes` | Desired low-priority/spot node count |
| `$NodeDeallocationOption` | `requeue`, `terminate`, `taskcompletion`, `retaineddata` |
| `$PreemptedNodeCount` | Nodes preempted in current interval |
| `$CurrentDedicatedNodes` | Current dedicated node count |

### Scale to Zero When Idle

```
// Scale to zero after 10 minutes of no tasks
lifespan = (now() - \$PoolLifetimeInSeconds);
\$TargetDedicatedNodes = (\$PendingTasks.GetSample(1) > 0 || \$RunningTasks.GetSample(1) > 0) ? 4 : 0;
\$TargetLowPriorityNodes = 0;
\$NodeDeallocationOption = taskcompletion;
```

```bash
# Evaluate formula without applying
az batch pool autoscale evaluate \
  --pool-id mypool \
  --auto-scale-formula "$formula"
```

---

## Jobs

```bash
# Create a job
az batch job create \
  --id myjob \
  --pool-id mypool

# Create a job with constraints
az batch job create \
  --id myjob \
  --pool-id mypool \
  --max-wall-clock-time "PT8H" \
  --max-task-retry-count 2 \
  --priority 100

# Terminate a job (stops new tasks, cancels pending)
az batch job stop --job-id myjob

# Delete a job
az batch job delete --job-id myjob --yes
```

### Job Manager Task

Runs first in a job; can submit more tasks dynamically and optionally terminate the job when done.

```json
{
  "id": "myjob",
  "poolInfo": { "poolId": "mypool" },
  "jobManagerTask": {
    "id": "job-manager",
    "commandLine": "/bin/bash -c 'python3 submit_tasks.py'",
    "resourceFiles": [
      { "httpUrl": "https://mystorageaccount.blob.core.windows.net/scripts/submit_tasks.py",
        "filePath": "submit_tasks.py" }
    ],
    "killJobOnCompletion": true,
    "userIdentity": { "autoUser": { "scope": "pool", "elevationLevel": "nonadmin" } }
  },
  "onAllTasksComplete": "terminatejob"
}
```

### Job Preparation & Release Tasks

```json
{
  "jobPreparationTask": {
    "commandLine": "/bin/bash -c 'mkdir -p /mnt/work && mount ...'",
    "waitForSuccess": true,
    "rerunOnNodeRebootAfterSuccess": false
  },
  "jobReleaseTask": {
    "commandLine": "/bin/bash -c 'rm -rf /mnt/work && echo Job cleaned up'"
  }
}
```

### Job Schedules

```bash
az batch job-schedule create \
  --id weekly-render \
  --schedule '{"doNotRunUntil":"2026-05-01T00:00:00Z","recurrenceInterval":"P7D"}' \
  --job-specification '{"poolInfo":{"poolId":"mypool"},"onAllTasksComplete":"terminatejob"}'
```

---

## Tasks

### Basic Command-Line Task

```bash
az batch task create \
  --job-id myjob \
  --task-id task001 \
  --command-line "/bin/bash -c 'echo Hello from task001 && sleep 10'" \
  --environment-settings "FRAME=001" "SCENE=forest"
```

### Resource Files & Output Files

```bash
az batch task create \
  --job-id myjob \
  --task-id render001 \
  --command-line "/bin/bash -c 'blender -b scene.blend -f 1 -o /output/frame_####'" \
  --resource-files '[
    {"httpUrl":"https://mystorageaccount.blob.core.windows.net/scenes/scene.blend?<sas>",
     "filePath":"scene.blend"}
  ]' \
  --output-files '[
    {"filePattern":"../output/**/*",
     "destination":{"container":{"containerUrl":"https://mystorageaccount.blob.core.windows.net/output?<sas>"}},
     "uploadOptions":{"uploadCondition":"taskCompletion"}}
  ]'
```

### Container Tasks

```python
from azure.batch.models import (
    TaskContainerSettings, TaskAddParameter, EnvironmentSetting
)

task = TaskAddParameter(
    id="container-task-001",
    command_line="python3 /app/process.py --input /mnt/input/data.csv",
    container_settings=TaskContainerSettings(
        image_name="myregistry.azurecr.io/myapp:latest",
        container_run_options="--rm --user 1000:1000",
    ),
    environment_settings=[
        EnvironmentSetting(name="BATCH_FRAME", value="001"),
    ],
)
batch_client.task.add(job_id="myjob", task=task)
```

### Multi-Instance Tasks (MPI)

```python
from azure.batch.models import MultiInstanceSettings

task = TaskAddParameter(
    id="mpi-task",
    command_line="mpirun -n $AZ_BATCH_NODE_LIST_FILE python3 /app/train.py",
    multi_instance_settings=MultiInstanceSettings(
        number_of_instances=4,
        coordination_command_line="/bin/bash -c 'cat $AZ_BATCH_HOST_LIST > /tmp/hostfile'",
    ),
)
```

### Task Dependencies

```python
from azure.batch.models import TaskDependencies, TaskIdRange

task = TaskAddParameter(
    id="merge-task",
    command_line="/bin/bash -c 'python3 merge_results.py'",
    depends_on=TaskDependencies(
        task_ids=["task001", "task002", "task003"],  # by ID list
        # or by range:
        task_id_ranges=[TaskIdRange(start=1, end=100)],
    ),
)
```

---

## Application Packages

```bash
# Create application
az batch application create \
  --resource-group rg-batch \
  --account-name mybatchaccount \
  --application-name myapp

# Upload package version
az batch application package create \
  --resource-group rg-batch \
  --account-name mybatchaccount \
  --application-name myapp \
  --version 1.0.0 \
  --package-file myapp-1.0.0.zip

# Set default version
az batch application set \
  --resource-group rg-batch \
  --account-name mybatchaccount \
  --application-name myapp \
  --default-version 1.0.0

# Reference in pool (makes package available at $AZ_BATCH_APP_PACKAGE_MYAPP)
az batch pool create \
  --id mypool \
  --application-package-references "myapp#1.0.0"
```

---

## Python SDK — Full Example

```python
import azure.batch as batch
from azure.batch.models import (
    PoolAddParameter, VirtualMachineConfiguration, ImageReference,
    JobAddParameter, PoolInformation, TaskAddParameter,
    OutputFile, OutputFileBlobContainerDestination,
    OutputFileDestination, OutputFileUploadOptions,
    OutputFileUploadCondition
)
from azure.common.credentials import ServicePrincipalCredentials

# Batch Management SDK (for pool/account provisioning via ARM)
from azure.mgmt.batch import BatchManagementClient

# Batch Service SDK (for submitting jobs/tasks)
BATCH_ACCOUNT_URL = "https://mybatchaccount.eastus.batch.azure.com"
BATCH_ACCOUNT_NAME = "mybatchaccount"
BATCH_ACCOUNT_KEY = "<key>"

credentials = batch.auth.SharedKeyCredentials(BATCH_ACCOUNT_NAME, BATCH_ACCOUNT_KEY)
batch_client = batch.BatchServiceClient(credentials, batch_url=BATCH_ACCOUNT_URL)

# Create pool
pool = PoolAddParameter(
    id="mypool",
    vm_size="Standard_D4s_v3",
    virtual_machine_configuration=VirtualMachineConfiguration(
        image_reference=ImageReference(
            publisher="canonical",
            offer="0001-com-ubuntu-server-focal",
            sku="20_04-lts",
            version="latest",
        ),
        node_agent_sku_id="batch.node.ubuntu 20.04",
    ),
    target_dedicated_nodes=4,
    target_low_priority_nodes=8,
)
batch_client.pool.add(pool)

# Create job
job = JobAddParameter(
    id="myjob",
    pool_info=PoolInformation(pool_id="mypool"),
    on_all_tasks_complete="terminateJob",
)
batch_client.job.add(job)

# Submit tasks
tasks = [
    TaskAddParameter(
        id=f"task-{i:04d}",
        command_line=f"/bin/bash -c 'python3 process.py --frame {i}'",
    )
    for i in range(1, 101)
]
batch_client.task.add_collection(job_id="myjob", value=tasks)

# Poll for completion
import time
while True:
    job_state = batch_client.job.get("myjob").state
    if job_state == "completed":
        break
    time.sleep(30)
    print(f"Job state: {job_state}")
```

---

## Rendering

Azure Batch has built-in rendering support for DCC applications.

### Supported Applications

| App | License Model |
|-----|--------------|
| Blender | Open source (free on Batch nodes) |
| Arnold (Autodesk) | Pay-per-use via Azure Batch rendering |
| V-Ray (Chaos Group) | Pay-per-use via Azure Batch rendering |
| 3ds Max | Pay-per-use or BYOL |
| Maya | Pay-per-use or BYOL |

```bash
# Create rendering pool (pre-installed DCC apps)
az batch pool create \
  --id render-pool \
  --vm-size Standard_D8s_v3 \
  --image microsoftwindowsserver:windowsserver:2019-datacenter:latest \
  --node-agent-sku-id "batch.node.windows amd64" \
  --application-package-references "3dsmax#2024" "arnold#7.2" \
  --target-dedicated-nodes 0 \
  --target-low-priority-nodes 10

# Submit Blender render job
az batch task create \
  --job-id blender-job \
  --task-id frame-0001 \
  --command-line "cmd /c blender -b C:\input\scene.blend -f 1 -o C:\output\frame_####.exr" \
  --resource-files '[{"httpUrl":"https://storage.blob.core.windows.net/scenes/scene.blend?<sas>","filePath":"C:\\input\\scene.blend"}]'
```

---

## Monitoring

### Metrics (az monitor)

```bash
# View pool node counts over time
az monitor metrics list \
  --resource "/subscriptions/<sub>/resourceGroups/rg-batch/providers/Microsoft.Batch/batchAccounts/mybatchaccount/pools/mypool" \
  --metric "DedicatedCoreCount" "LowPriorityCoreCount" "RunningTaskCount" \
  --interval PT5M \
  --start-time 2026-04-24T00:00:00Z

# Set alert: alert when task failure count > 10
az monitor metrics alert create \
  --name "batch-task-failures" \
  --resource-group rg-batch \
  --scopes "/subscriptions/<sub>/resourceGroups/rg-batch/providers/Microsoft.Batch/batchAccounts/mybatchaccount" \
  --condition "total TaskFailCount > 10" \
  --window-size 5m \
  --evaluation-frequency 1m \
  --action myActionGroup
```

### Diagnostic Logs to Log Analytics

```bash
az monitor diagnostic-settings create \
  --name batch-diag \
  --resource "/subscriptions/<sub>/resourceGroups/rg-batch/providers/Microsoft.Batch/batchAccounts/mybatchaccount" \
  --workspace "/subscriptions/<sub>/resourceGroups/rg-logs/providers/Microsoft.OperationalInsights/workspaces/myWorkspace" \
  --logs '[{"category":"ServiceLog","enabled":true}]' \
  --metrics '[{"category":"AllMetrics","enabled":true}]'
```

### Application Insights Integration

```python
from applicationinsights import TelemetryClient

tc = TelemetryClient("<instrumentation_key>")

def on_task_complete(task_id: str, duration_seconds: float, exit_code: int):
    tc.track_event("BatchTaskComplete", {
        "task_id": task_id,
        "job_id": "myjob",
    }, {
        "duration_seconds": duration_seconds,
        "exit_code": exit_code,
    })
    tc.flush()
```

### KQL Queries

```kusto
// Failed tasks in last 24h
AzureDiagnostics
| where ResourceProvider == "MICROSOFT.BATCH"
| where Category == "ServiceLog"
| where OperationName == "TaskComplete"
| where ResultType == "Failed"
| project TimeGenerated, poolId_s, jobId_s, taskId_s, exitCode_d
| order by TimeGenerated desc

// Node preemption events
AzureDiagnostics
| where ResourceProvider == "MICROSOFT.BATCH"
| where OperationName == "PoolResizeComplete"
| project TimeGenerated, poolId_s, targetDedicated_d, targetLowPriority_d
```

---

## Security

### Managed Identity

```bash
# Assign system-assigned managed identity to pool
az batch pool create \
  --id secure-pool \
  --vm-size Standard_D4s_v3 \
  --identity SystemAssigned \
  ...

# Grant pool identity access to storage
az role assignment create \
  --assignee "<pool_principal_id>" \
  --role "Storage Blob Data Reader" \
  --scope "/subscriptions/<sub>/resourceGroups/rg-batch/providers/Microsoft.Storage/storageAccounts/mystorageaccount"
```

### Private Endpoints

```bash
# Disable public network access on Batch account
az batch account set \
  --name mybatchaccount \
  --resource-group rg-batch \
  --public-network-access Disabled

# Create private endpoint
az network private-endpoint create \
  --name batch-pe \
  --resource-group rg-batch \
  --vnet-name myVnet \
  --subnet batchSubnet \
  --private-connection-resource-id "/subscriptions/<sub>/resourceGroups/rg-batch/providers/Microsoft.Batch/batchAccounts/mybatchaccount" \
  --group-id batchAccount \
  --connection-name batch-pe-conn
```

### Simplified Compute Node Communication (2024+)

Eliminates the need to open inbound ports to pool nodes; all communication
flows outbound from nodes to the Batch service endpoint.

```json
{
  "networkConfiguration": {
    "enableAcceleratedNetworking": true,
    "nodePlacementPolicy": "regional",
    "dynamicVnetAssignmentScope": "job"
  }
}
```

---

## Cost Optimization

### Low-Priority / Spot VMs

Up to **80% discount** vs. dedicated VMs. VMs can be preempted with ~30s notice.

```bash
# Mixed dedicated + low-priority pool
az batch pool create \
  --id cost-pool \
  --vm-size Standard_D8s_v3 \
  --target-dedicated-nodes 2 \
  --target-low-priority-nodes 20 \
  ...
```

Tasks interrupted by preemption are automatically requeued. Set `maxTaskRetryCount`
to handle transient failures.

### Auto-Scale to Zero

```bash
az batch pool autoscale enable \
  --pool-id mypool \
  --auto-scale-formula "
    tasks = max(\$PendingTasks.GetSample(1), \$RunningTasks.GetSample(1));
    \$TargetDedicatedNodes = tasks > 0 ? 4 : 0;
    \$TargetLowPriorityNodes = tasks > 0 ? 12 : 0;
    \$NodeDeallocationOption = taskcompletion;
  " \
  --auto-scale-evaluation-interval "PT5M"
```

---

## Bicep / Terraform Examples

### Bicep

```bicep
param location string = resourceGroup().location
param batchAccountName string = 'mybatchaccount'
param storageAccountName string = 'mybatchstorage'

resource storageAccount 'Microsoft.Storage/storageAccounts@2023-01-01' = {
  name: storageAccountName
  location: location
  sku: { name: 'Standard_LRS' }
  kind: 'StorageV2'
}

resource batchAccount 'Microsoft.Batch/batchAccounts@2024-02-01' = {
  name: batchAccountName
  location: location
  properties: {
    autoStorage: {
      storageAccountId: storageAccount.id
      authenticationMode: 'BatchAccountManagedIdentity'
    }
    poolAllocationMode: 'BatchService'
    publicNetworkAccess: 'Enabled'
  }
  identity: {
    type: 'SystemAssigned'
  }
}

resource batchPool 'Microsoft.Batch/batchAccounts/pools@2024-02-01' = {
  parent: batchAccount
  name: 'mypool'
  properties: {
    vmSize: 'Standard_D4s_v3'
    deploymentConfiguration: {
      virtualMachineConfiguration: {
        imageReference: {
          publisher: 'canonical'
          offer: '0001-com-ubuntu-server-focal'
          sku: '20_04-lts'
          version: 'latest'
        }
        nodeAgentSkuId: 'batch.node.ubuntu 20.04'
      }
    }
    scaleSettings: {
      autoScale: {
        formula: '$TargetDedicatedNodes = 0; $TargetLowPriorityNodes = 0;'
        evaluationInterval: 'PT5M'
      }
    }
    taskSlotsPerNode: 4
  }
}

output batchAccountEndpoint string = batchAccount.properties.accountEndpoint
```

### Terraform

```hcl
resource "azurerm_batch_account" "batch" {
  name                                = "mybatchaccount"
  resource_group_name                 = azurerm_resource_group.rg.name
  location                            = azurerm_resource_group.rg.location
  pool_allocation_mode                = "BatchService"
  storage_account_id                  = azurerm_storage_account.storage.id
  storage_account_authentication_mode = "BatchAccountManagedIdentity"
}

resource "azurerm_batch_pool" "pool" {
  name                = "mypool"
  resource_group_name = azurerm_resource_group.rg.name
  account_name        = azurerm_batch_account.batch.name
  display_name        = "My Batch Pool"
  vm_size             = "Standard_D4s_v3"
  node_agent_sku_id   = "batch.node.ubuntu 20.04"

  storage_image_reference {
    publisher = "canonical"
    offer     = "0001-com-ubuntu-server-focal"
    sku       = "20_04-lts"
    version   = "latest"
  }

  auto_scale {
    evaluation_interval = "PT5M"
    formula = <<-EOF
      $TargetDedicatedNodes = ($PendingTasks.GetSample(1) > 0) ? 4 : 0;
      $TargetLowPriorityNodes = 0;
      $NodeDeallocationOption = taskcompletion;
    EOF
  }

  start_task {
    command_line       = "/bin/bash -c 'apt-get update && apt-get install -y ffmpeg'"
    wait_for_success   = true
    task_retry_maximum = 2
    user_identity {
      auto_user {
        elevation_level = "Admin"
        scope           = "Pool"
      }
    }
  }
}
```

---

## Batch Explorer

Batch Explorer is a free GUI tool for managing Batch accounts:

```bash
# Download from:
# https://azure.github.io/BatchExplorer/

# Features:
# - Pool / job / task visualization
# - Node RDP/SSH access
# - Heatmap of node utilization
# - Log streaming from running tasks
# - Template library (rendering, ML, HPC workloads)
```

---

## Quick Reference — az batch CLI

```bash
# Account
az batch account create / show / login / keys list

# Pool
az batch pool create --json-file pool.json
az batch pool list / show --pool-id <id>
az batch pool resize --pool-id <id> --target-dedicated-nodes 10
az batch pool autoscale enable / disable / evaluate
az batch pool delete --pool-id <id> --yes

# Job
az batch job create --id <id> --pool-id <pool>
az batch job list / show --job-id <id>
az batch job stop --job-id <id>
az batch job delete --job-id <id> --yes

# Task
az batch task create --job-id <id> --task-id <tid> --command-line "..."
az batch task list --job-id <id>
az batch task show --job-id <id> --task-id <tid>
az batch task file list --job-id <id> --task-id <tid>
az batch task file download --job-id <id> --task-id <tid> --file-path stdout.txt --destination ./stdout.txt

# Node
az batch node list --pool-id <id>
az batch node remote-login-settings show --pool-id <id> --node-id <nid>

# Application packages
az batch application create / list / set
az batch application package create / list / activate
```
