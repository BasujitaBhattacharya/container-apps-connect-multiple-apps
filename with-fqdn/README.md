# Calling with FQDN

I can call the dotnet-app from the node-app by calling it's FQDN. Even though I use the FQDN, **calls within the environment will stay within the environment and network traffic will not leave**.

```js
const dotnetFQDN = process.env.DOTNET_FQDN;
// ...
var data = await axios.get(`http://${dotnetFQDN}`);
res.send(`${JSON.stringify(data.data)}`);
```

## Deploy with CLI

```bash
# Login to the CLI
az login
az extension add --name containerapp
az provider register --namespace Microsoft.App

# Create a resource group
az group create \
  --name 'sample-rg' \
  --location canadacentral

az monitor log-analytics workspace create \
  --resource-group 'sample-rg' \
  --workspace-name 'logs-for-sample'

LOG_ANALYTICS_WORKSPACE_CLIENT_ID=`az monitor log-analytics workspace show --query customerId -g 'sample-rg' -n 'logs-for-sample' --out tsv`
LOG_ANALYTICS_WORKSPACE_CLIENT_SECRET=`az monitor log-analytics workspace get-shared-keys --query primarySharedKey -g 'sample-rg' -n 'logs-for-sample' --out tsv`

# Create a container app environment
az containerapp env create \
  --name 'sample-env'\
  --resource-group 'sample-rg' \
  --logs-workspace-id $LOG_ANALYTICS_WORKSPACE_CLIENT_ID \
  --logs-workspace-key $LOG_ANALYTICS_WORKSPACE_CLIENT_SECRET \
  --location canadacentral

# Deploy the container-2-dotnet dotnet-app
az containerapp create \
  --name dotnet-app \
  --resource-group 'sample-rg' \
  --environment 'sample-env' \
  --image 'ghcr.io/azure-samples/container-apps-connect-multiple-apps/dotnet:main' \
  --target-port 80 \
  --ingress 'internal'

DOTNET_FQDN=$(az containerapp show \
  --resource-group 'sample-rg' \
  --name dotnet-app \
  --query configuration.ingress.fqdn -o tsv)

# Deploy the container-1-node node-app
az containerapp create \
  --name node-app \
  --resource-group 'sample-rg' \
  --environment 'sample-env' \
  --image 'ghcr.io/azure-samples/container-apps-connect-multiple-apps/node:main' \
  --target-port 3000 \
  --ingress 'external' \
  --environment-variables DOTNET_FQDN=$DOTNET_FQDN \
  --query configuration.ingress.fqdn
```

