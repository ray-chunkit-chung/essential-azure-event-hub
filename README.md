# essential-azure-event-hub

Tutorial: Stream big data into a data warehouse

<https://learn.microsoft.com/en-us/azure/event-grid/event-grid-event-hubs-integration>

<https://github.com/Azure/azure-event-hubs/tree/master/samples/e2e/EventHubsCaptureEventGridDemo>

## Step 1 Login azure

Set app variables by saving in .env file

```bash
export SUBSCRIPTION="xxx"
export TENANT="xxx"
export LOCATION="eastus"
export RESOURCE_GROUP="xxx"
```

Install azure cli & login

```bash
source .env
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash
az login --tenant $TENANT
az account set -s $SUBSCRIPTION
```

## Step 2 Deploy azure service bus

```bash
source .env
az group create --subscription $SUBSCRIPTION \
                --name $RESOURCE_GROUP \
                --location $LOCATION
az deployment group create --subscription $SUBSCRIPTION \
                           --resource-group $RESOURCE_GROUP \
                           --name rollout01 \
                           --template-file ARMTemplate/EventHubsDataMigration/template.json \
                           --parameters ARMTemplate/EventHubsDataMigration/parameters.json
```

## Finally Delete resources

After experiment, delete all resources to avoid charging a lot of money

```bash
source .env
az group delete -y --name $RESOURCE_GROUP
```

There can be some managed resources to delete. Check them by

```bash
az group list --subscription $SUBSCRIPTION
```

Delete them by

```bash
source .env
az group delete --name $(az group list --subscription $SUBSCRIPTION | jq '.[].name' | tr -d '"')
```
