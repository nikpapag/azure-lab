
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

   <img width="698" alt="Screenshot 2024-12-03 at 20 44 30" src="https://github.com/user-attachments/assets/2b5cd45c-9d8a-43b4-9902-10219ad6a51d">

---

### Create a Template
1. Navigate to **Project Settings**.
2. Select **Templates**.
3. Click **+New Template**.
4. From the dropdown, select **Deployment**.

   <img width="278" alt="Screenshot 2024-12-03 at 21 02 15" src="https://github.com/user-attachments/assets/65a6f7d2-6d89-4e71-bdde-bdcb6fa4a45d">


#### Input Values
| Variable Name      | Value                                      |
|--------------------|--------------------------------------------|
| name        | `Azure_Stamps`                          |
| Version Label           | `v1`         |
| Save To       | `Project`     |


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
| Field               | Value          |
|---------------------|----------------|
| Name                | s3  |


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
5. Click on **+New Infrastructure Definition**

#### Input Values
| Field             | Value          |
|-------------------|----------------|
| Name              | westeurope     |
| Deployment Type   | Azure_Stamps   |

---

### Service Overrides
Summary: Given the dynamic nature of our deployment we want to be able to switch between tenants, clients and environments at runtime. Harness allows to define variable overrides 

<img width="771" alt="Screenshot 2024-12-03 at 21 03 13" src="https://github.com/user-attachments/assets/73b62eb8-9609-4b4d-8b2f-b611750ad736">


#### Steps
1. Navigate to **Overrides**.
2. From the NavBar select **Infrastructure Specific Overrides**
3. Click **+New Override**.
4. Select the environment and infra for the ovverride

| Variable Name         | Type   | Value                           |
|-----------------------|--------|---------------------------------|
| environment       | String | `azure_staging` |
| infrastructure | String | `westeurope` |

5. Configure the overrides as follows

#### Override Details
| Variable Name         | Type   | Value                           |
|-----------------------|--------|---------------------------------|
| AZURE_TENANT_ID       | String | `<+variable.org.AZURE_TENANT_ID>` |
| AZURE_SUBSCRIPTION_ID | String | `<+variable.org.AZURE_SUBSCRIPTION_ID>` |
| AZURE_CLIENT_ID       | String | `<+variable.org.AZURE_CLIENT_ID>` |
| AZURE_CLIENT_SECRET   | Secret |  org.azure_client_secret         |
| ENVIRONMENT_TYPE      | String | staging                        |

6. Save overrides using the checkbox
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

4. Set Up state following the setup guide

| Field             | Value          |
|-------------------|----------------|
| Service   |`s3` |
| Environment  | `azure_staging`   |
| Infrastructure | `westeurope` |

5. From the NavBar select **Advanced**
6. Select the azure delegate

<img width="512" alt="Screenshot 2024-12-03 at 21 19 49" src="https://github.com/user-attachments/assets/f3679d9a-8883-4f42-b79a-42a3a26a567b">

7. Validate Ouput of the **Fetch Instances** step
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
