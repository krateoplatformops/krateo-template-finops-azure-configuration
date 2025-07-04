# Krateo Composable FinOps - Prepare your Azure Environment
This repository shows how to prepare your Azure environment for Krateo Composable FinOps

## Summary
1. [Overview](#krateo-composable-finops---prepare-your-azure-environment)
2. [Installation](#installation)
3. [Cost and Usage Metrics](#cost-and-usage-metrics)

## Installation
You need a Kubernetes cluster with version greater or equal to 1.31 (OpenShift 4.18) for the pricing example to work. If you have Kubernetes version 1.30 (OpenShift 4.17) you need to enable the feature gate `CustomResourceFieldSelectors` (see [here](https://github.com/kubernetes/kubernetes/pull/122717)). On OpenShift 4.17 you can enable it with:
```yaml
apiVersion: config.openshift.io/v1
kind: FeatureGate
metadata:
  name: cluster
spec:
  featureSet: CustomNoUpgrade
  customNoUpgrade:
    enabled:
      - CustomResourceFieldSelectors
```
Note that this will disable automatic updates between minor versions in OpenShift.
> [!NOTE]
> If you did not have the feature gate enabled when you installed the finops-operator-focus, you need to manually re-apply the CRD, as the `selectableFields` [field](https://github.com/krateoplatformops/finops-operator-focus-chart/blob/c6ee9f0b3d361100e5b5893c83e9231cbae5077b/chart/crds/finops.krateo.io_focusconfigs.yaml#L18) in the CRD will get pruned without the feature gate (applies to both standard Kubernetes and OpenShift). To install the Feature Gate on OpenShift see [fg.yaml](samples/fg.yaml)

Install the cert-manager for the Azure operator:
```sh
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.16.3/cert-manager.yaml
```

Install the Azure operator (Azure Service Operator v2):
```sh
helm repo add aso2 https://raw.githubusercontent.com/Azure/azure-service-operator/main/v2/charts
helm repo update
helm upgrade --install aso2 aso2/azure-service-operator \
    --create-namespace \
    --namespace=azureserviceoperator-system \
    --set crdPattern='resources.azure.com/*;compute.azure.com/*;network.azure.com/*'
```
On OpenShift use:
```sh
helm upgrade --install aso2 aso2/azure-service-operator \
    --create-namespace \
    --namespace=azureserviceoperator-system \
    --set crdPattern='resources.azure.com/*;compute.azure.com/*;network.azure.com/*' \
    --set securityContext.runAsUser=null \
    --set securityContext.runAsGroup=null \
    --set networkPolicies.enable=false
```
It removes the user and group since Openshift runs with randomized user and group uids. Additionally, it disables network policies, since they are not supported by default on Openshift (see [issue#3259 of ASO](https://github.com/Azure/azure-service-operator/issues/3259)).

Create the secret with the credentials for the Azure operator
```sh
kubectl create ns azure-pricing-system
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
 name: aso-credential
 namespace: azure-pricing-system
stringData:
 AZURE_SUBSCRIPTION_ID: # insert your subscription id
 AZURE_TENANT_ID:       # insert your tenant id
 AZURE_CLIENT_ID:       # insert your client id
 AZURE_CLIENT_SECRET:   # insert your client secret
EOF
```

# Billing, Cost and Usage Metrics
This section will cover how to configure the Krateo Composable FinOps to collect cost and usage metrics from the virtual machine just created on Azure.

> [!NOTE]  
> The Azure Portal UI may change over time, so some steps or visuals might differ slightly from what is described here.

## Prepare your Billing Report in FOCUS
To start, you will need a storage account to store the FOCUS exports. If you do not already have it, you need to configure it: 
1. go on your Azure dashboard and `Storage Accounts`, hit `Create` and select the `Blob Storage` mode. Then, follow the guided steps to complete the configuration. 
2. go to `Access Control (IAM)`, then `Role Assignments` and add a new role assignment. Search for `Azure Blob Data Owner` and hit forward. Select the application that you use for the authentication. Finally, assign it.

To configure the FOCUS exports on the storage account just created: 
1. go on your dashboard and search for `Cost Management`.
2. on the left pane, select `Exports`, then `Create`
3. select the FOCUS export and how often the export should run. 
4. select the storage account that you just created. 
5. to use the `Logic App` provided below, set the container to `focus`, the directory to `exports` and the format to `CSV`, without compression and with overwrite.

Now the export will run automatically to the storage account. However, it will place it inside a folder with the `uid`of the run, creating a dynamic path that cannot be known at runtime to configure the exporters and scrapers. Therefore, we will use a `Logic App` to copy the export from the `uid` folder to a static location.

Go back to the Azure dashboard and search for `Logic apps`. Then, create a new `Logic App` with the tier you most prefer (the consumption tier is pretty cheap if used for this task only). When created, hit `Edit`, then graphic editor will open. Select `Code view` and paste the following code. 

```json
{
    "definition": {
        "$schema": "https://schema.management.azure.com/providers/Microsoft.Logic/schemas/2016-06-01/workflowdefinition.json#",
        "contentVersion": "1.0.0.0",
        "triggers": {
            "Recurrence": {
                "type": "Recurrence",
                "recurrence": {
                    "frequency": "Day",
                    "interval": 1,
                    "schedule": {
                        "hours": [
                            1
                        ],
                        "minutes": [
                            0
                        ]
                    },
                    "timeZone": "UTC"
                }
            }
        },
        "actions": { }
    }
}
```

Select `Designer` to go back to the graphical view.

Then, create the following objects:
1. click on the plus below `Recurrence`, add an action, and on the right pane search `azure blob storage` and add `list blobs`, this will prompt to create a connection. Configure the connection to your storage account. Use the `Service principal authentication` and provide the client id and secret of the application to which you granted the `Azure Blob Data Owner` permissions (which is also the same you will use for the exporters' authentication). Set `Flat listing` to `Yes`.
2. add another action by searching for `data operations` and adding `filter array`. Click on parameters, then the lightning and finally `value`. Then, in `Filter query`, select the lightning and `Name`, then in the filter `starts with` and in the value `part_`.
3. add another `filter array`, in the data value choose the `fx` button, switch to `dynamic content` and select `Body`. Then, in the `Filter query` select `Body Name`, `ends with` and set the value to `.csv`.
4. add one more action of the type `data operations`, but this time choose `compose`. In the `inputs` click on the `fx` button and then write `sort()` in the box that opens, select dynamic content and click `Body`, then add `LastModified`. The output should look like this:
```
sort(body('<YOUR PREVIOUS ACTION NAME>', 'LastModified'))
```
5. finally, add one last action by serach for `azure blob storage` and selecting `copy blobs`. Set the storage account name to your storage account, then the destination blob to `/focus/exports/static-export.csv`. For the source url, click on the `fx` button. Write `last()` then select `Dynamic content` and choose `Outputs` from the previous `Compose` action. Then add `?['Path']` at the end to select the right field of the object. Your `fx` should look like this:
```
last(outputs('<YOUR COMPOSE ACTION NAME>'))?['Path']
```
Set `Overwrite` to `Yes`.

Now you can save and run the logic app. If you did everything right, you should see a successful run.

## Export Billing and Cost Metrics
To handle the access tokens for Azure, we suggest the [external-secrets](https://github.com/external-secrets/external-secrets) operator. The configuration is included in the sample. It handles retrieving bearer tokens from Azure automatically, every hour. See the installation instructions here: [Getting started](https://external-secrets.io/latest/introduction/getting-started/). Or use the following commands:
```
helm repo add external-secrets https://charts.external-secrets.io
helm repo update
helm install external-secrets external-secrets/external-secrets -n externalsecrets-system --create-namespace
```

Finally, enable the cluster-wide external secrets for the krateo-system namespace with:
```
kubectl label namespace krateo-system azure-secrets=enabled
```

To start the exporter/scraper for all costs, install this chart in the Krateo install namespace and configure all required parameters:
```sh
helm install krateo-template-finops-azure-configuration krateo/krateo-template-finops-azure-configuration -n krateo-system \
  --set tenantId=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx \
  --set clientId=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx \
  --set clientSecret=xxxxxxxxxx \
```