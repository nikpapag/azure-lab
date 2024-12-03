
# Lab - Azure Custom Deployment Templates

## Summary
Understand how Harness can address custom deployment types and adopt different patterns.

## Learning Objectives
- Create multiple application services.
- Create multiple infrastructure definitions based on type, region, tenant.
- Create variable overrides to adjust to dynamic configuration.
- Detect running instances.
- Deploy an nginx web app into a given stamp.

---

## Steps

### Initial Setup
1. Resource groups are pre-provisioned as shown below.
2. Two stamps are created with resource definitions and additional tagging for identification.

#### Identify Stamps
Use the Fetch Instances functionality.

---

### Create a Template
1. Navigate to **Project Settings**.
2. Select **Templates**.
3. Click **+New Template**.
4. From the dropdown, select **Deployment**.

#### Input Values
- **Name**: Azure Stamps
- **Version Label**: v1
- **Save To**: Project

5. Configure variables in the Variables section by clicking **+New Variable**.

#### Variables
| Variable Name      | Value                                      |
|--------------------|--------------------------------------------|
| serviceName        | `<+service.name>`                          |
| clientId           | `<+env.variables.AZURE_CLIENT_ID>`         |
| clientSecret       | `<+env.variables.AZURE_CLIENT_SECRET>`     |
| tenantId           | `<+env.variables.AZURE_TENANT_ID>`         |
| subscriptionId     | `<+env.variables.AZURE_SUBSCRIPTION_ID>`   |
| environment        | `<+env.variables.ENVIRONMENT_TYPE>`        |
| projectName        | `<+project.name>`                          |

---

### Fetch Instance Script
1. Add the following code:

```bash
echo "Start of Fetch Instance Script"
echo "DEBUG: AZ Login"
az login --service-principal \
  --username <+infra.variables.clientId> \
  --password <+infra.variables.clientSecret> \
  --tenant <+infra.variables.tenantId>
  
echo "DEBUG: AZ Account Set"
az account set --subscription <+infra.variables.subscriptionId>
echo "DEBUG: Identify stamps/resource groups that the service belongs to"
SERVICE_TO_FIND=<+infra.variables.stampName>
az group list --tag stamp=true --tag environment=<+infra.variables.environment> --tag project_name=<+infra.variables.projectName> \
  --query "{items: [?contains(tags.services, '${SERVICE_TO_FIND}')].{name: name, location: location, tags: tags}}" \
  --output json > $INSTANCE_OUTPUT_PATH
cat $INSTANCE_OUTPUT_PATH
az logout
```

2. Save the deployment template.

---

### Create the Service
1. Navigate to **Services**.
2. Click **+New Service**.

#### Input Values
- **Name**: s3

3. Save the service.
4. Use the **Azure Stamps Deployment Template** and save.

---

### Create the Environment
1. Navigate to **Environments**.
2. Click **+New Environment**.

#### Input Values
| Field               | Value          |
|---------------------|----------------|
| Name                | azure_staging  |
| Environment Type    | Pre-Production |

3. Save the environment.
4. Add Infrastructure Definitions.

#### Input Values
| Field             | Value          |
|-------------------|----------------|
| Name              | westeurope     |
| Deployment Type   | Azure_Stamps   |

---

### Service Overrides
1. Navigate to **Overrides**.
2. Click **+New Override**.

#### Override Details
| Variable Name         | Type   | Value                           |
|-----------------------|--------|---------------------------------|
| AZURE_TENANT_ID       | String | `<+variable.org.AZURE_TENANT_ID>` |
| AZURE_SUBSCRIPTION_ID | String | `<+variable.org.AZURE_SUBSCRIPTION_ID>` |
| AZURE_CLIENT_ID       | String | `<+variable.org.AZURE_CLIENT_ID>` |
| AZURE_CLIENT_SECRET   | Secret | Select azure_secret            |
| ENVIRONMENT_TYPE      | String | staging                        |

---

### Pipeline Creation
1. Navigate to **Pipelines**.
2. Click **+Create Pipeline**.

#### Input Values
| Field       | Value          |
|-------------|----------------|
| Name        | Azure_Stamp    |

3. Add a **Deploy Stage** with the following values:

#### Stage Details
| Field             | Value          |
|-------------------|----------------|
| Stage Name        | Deploy Staging |
| Deployment Type   | Azure_Stamps   |

4. Configure the deployment steps, save, and run the pipeline.

---

### Deploy Service to Stamps
1. Use the following shell script:

```bash
echo "Resource group Name"
echo <+matrix.item>
echo "Service Name"
echo <+service.name>
echo "Application Region"
echo <+infra.name>
echo "Application Name"
echo <+project.name>
echo "Stamp"
echo <+pipeline.stages.Deploy_Staging.spec.execution.steps.fetchInstances.deploymentInfoOutcome.serverInstanceInfoList[<+strategy.iteration>].properties.stamp>
```

2. Save, validate output, and deploy.
