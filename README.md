# essential-azure-event-hub

Tutorial: Stream big data into a data warehouse

<https://learn.microsoft.com/en-us/azure/event-grid/event-grid-event-hubs-integration>

<https://github.com/Azure/azure-event-hubs/tree/master/samples/e2e/EventHubsCaptureEventGridDemo>

Choose between Azure messaging services - Event Grid, Event Hubs, and Service Bus

<https://learn.microsoft.com/en-us/azure/event-grid/compare-messaging-services?toc=%2Fazure%2Fservice-bus-messaging%2Ftoc.json&bc=%2Fazure%2Fservice-bus-messaging%2Fbreadcrumb%2Ftoc.json>

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

## Step 2 Deploy event hub stack with dedicated SQL pool

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

## Step 3 Create table in dedicated SQL pool

Run in Portal > My Resource Group > My Dedicated SQL pool > Query editor (Preview)

```SQL
CREATE TABLE [dbo].[Fact_WindTurbineMetrics] (
    [DeviceId] nvarchar(50) COLLATE SQL_Latin1_General_CP1_CI_AS NULL, 
    [MeasureTime] datetime NULL, 
    [GeneratedPower] float NULL, 
    [WindSpeed] float NULL, 
    [TurbineSpeed] float NULL
)
WITH (CLUSTERED COLUMNSTORE INDEX, DISTRIBUTION = ROUND_ROBIN);
```

## Step 4 Install function core and .net sdk to publish function

Function Core

```bash
curl https://packages.microsoft.com/keys/microsoft.asc | gpg --dearmor > microsoft.gpg
sudo mv microsoft.gpg /etc/apt/trusted.gpg.d/microsoft.gpg
sudo sh -c 'echo "deb [arch=amd64] https://packages.microsoft.com/debian/$(lsb_release -rs | cut -d'.' -f 1)/prod $(lsb_release -cs) main" > /etc/apt/sources.list.d/dotnetdev.list'
sudo apt-get update
sudo apt-get install azure-functions-core-tools-4
```

.Net SDK

```bash
wget https://packages.microsoft.com/config/debian/11/packages-microsoft-prod.deb -O packages-microsoft-prod.deb
sudo dpkg -i packages-microsoft-prod.deb
rm packages-microsoft-prod.deb
sudo apt-get update
sudo apt-get install -y dotnet-sdk-7.0
sudo apt-get install -y dotnet-sdk-6.0
sudo apt-get install -y dotnet-sdk-3.1
```

Try function locally

```bash
cd EventHubsCaptureEventGridDemo/FunctionEGDWDumper/
func start
```

Publish to app

```bash
source .env
cd EventHubsCaptureEventGridDemo/FunctionEGDWDumper/
func azure functionapp fetch-app-settings $FUNCTION_APP
func azure storage fetch-connection-string $STORAGE_ACCOUNT
func azure functionapp publish $FUNCTION_APP
cd ../..
```

## Step 5 Create a system topic & subscription

<https://learn.microsoft.com/en-us/azure/event-grid/create-view-manage-system-topics>
<https://learn.microsoft.com/en-us/azure/active-directory/managed-identities-azure-resources/overview?WT.mc_id=Portal-Microsoft_Azure_EventGrid>

Create a topic

```bash
# coming soon...
```

Create a subscription to the topic

```bash
# coming soon...
```

## Step 6 Generate wind turbine data to event hub

Get event hub connection string

```bash
source .env
export PRIMARY_CONNECTION_STRING="$(az eventhubs namespace authorization-rule keys list --resource-group $RESOURCE_GROUP --namespace-name $EVENTHUB_NAMESPACE --name RootManageSharedAccessKey | jq '.primaryConnectionString' | tr -d '"')"
export SECONDARY_CONNECTION_STRING="$(az eventhubs namespace authorization-rule keys list --resource-group $RESOURCE_GROUP --namespace-name $EVENTHUB_NAMESPACE --name RootManageSharedAccessKey | jq '.secondaryConnectionString' | tr -d '"')"
```

6.1 VSCode > Right-click WindTurbineDataGenerator project > Set as Startup project.

6.2 Find program.cs > Replace <EVENT HUBS NAMESPACE CONNECTION STRING> & <EVENT HUB NAME>

```C#
private const string EventHubConnectionString = System.Environment.GetEnvironmentVariable("EVENTHUB_NAMESPACE");
private const string EventHubName = System.Environment.GetEnvironmentVariable("EVENTHUB_NAME");
```

maybe...

```bash
cat ./ARMTemplate/EventHubsDataMigration/parameters.json | jq '.parameters.eventHubNamespaceName.value' | tr -d '"'
test2raynamespace
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
