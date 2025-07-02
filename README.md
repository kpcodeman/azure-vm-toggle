# Azure VM Toggle Button - Deployment Guide

This guide will walk you through deploying your Azure VM Toggle Button application to Azure Storage as a static website.

## Prerequisites

1. **Azure CLI**: Make sure you have the Azure CLI installed and logged in
2. **Azure Subscription**: You need an active Azure subscription
3. **Built Application**: The React application has been built (the `dist` folder is ready)

## Step 1: Create an Azure Storage Account

```bash
# Replace these values with your own
RESOURCE_GROUP="your-resource-group"
LOCATION="eastus"
STORAGE_ACCOUNT_NAME="vmtoggleapp$RANDOM"

# Create resource group if it doesn't exist
az group create --name $RESOURCE_GROUP --location $LOCATION

# Create storage account
az storage account create \
  --name $STORAGE_ACCOUNT_NAME \
  --resource-group $RESOURCE_GROUP \
  --location $LOCATION \
  --sku Standard_LRS \
  --kind StorageV2
```

## Step 2: Enable Static Website Hosting

```bash
# Enable static website hosting
az storage blob service-properties update \
  --account-name $STORAGE_ACCOUNT_NAME \
  --static-website \
  --index-document index.html \
  --404-document index.html
```

## Step 3: Upload the Built Application

```bash
# Get the storage account key
STORAGE_KEY=$(az storage account keys list \
  --resource-group $RESOURCE_GROUP \
  --account-name $STORAGE_ACCOUNT_NAME \
  --query "[0].value" -o tsv)

# Upload the dist folder contents to the $web container
az storage blob upload-batch \
  --account-name $STORAGE_ACCOUNT_NAME \
  --account-key $STORAGE_KEY \
  --source ./dist \
  --destination '$web'
```

## Step 4: Get the Website URL

```bash
# Get the website URL
WEBSITE_URL=$(az storage account show \
  --name $STORAGE_ACCOUNT_NAME \
  --resource-group $RESOURCE_GROUP \
  --query "primaryEndpoints.web" \
  --output tsv)

echo "Your website is now available at: $WEBSITE_URL"
```

## Step 5: Set Up Azure Function for VM Control (Backend)

For the application to actually control your Azure VM, you'll need to create an Azure Function that:

1. Authenticates with Azure using a service principal or managed identity
2. Provides API endpoints for checking VM status and toggling VM state
3. Secures these endpoints with authentication

### Create an Azure Function App

```bash
# Create a function app
FUNCTION_APP_NAME="vmtoggle-api$RANDOM"

az functionapp create \
  --resource-group $RESOURCE_GROUP \
  --consumption-plan-location $LOCATION \
  --runtime node \
  --functions-version 4 \
  --name $FUNCTION_APP_NAME \
  --storage-account $STORAGE_ACCOUNT_NAME
```

### Enable CORS for Your Static Website

```bash
# Enable CORS for your static website
az functionapp cors add \
  --resource-group $RESOURCE_GROUP \
  --name $FUNCTION_APP_NAME \
  --allowed-origins $WEBSITE_URL
```

## Step 6: Configure Azure Function API Endpoints

Create two HTTP-triggered functions in your function app:

1. **status**: To check the current status of your VM
2. **toggle**: To start or stop your VM

### Example Function Code (JavaScript)

```javascript
// status function
const { DefaultAzureCredential } = require("@azure/identity");
const { ComputeManagementClient } = require("@azure/arm-compute");

module.exports = async function (context, req) {
    try {
        const { subscriptionId, resourceGroup, vmName } = req.body;
        
        // Validate required parameters
        if (!subscriptionId || !resourceGroup || !vmName) {
            context.res = {
                status: 400,
                body: { error: "Missing required parameters" }
            };
            return;
        }
        
        // Authenticate with Azure
        const credential = new DefaultAzureCredential();
        const client = new ComputeManagementClient(credential, subscriptionId);
        
        // Get VM instance view to check power state
        const instanceView = await client.virtualMachines.instanceView(resourceGroup, vmName);
        const powerState = instanceView.statuses.find(status => 
            status.code.startsWith("PowerState/")
        ).code.split("/")[1].toLowerCase();
        
        // Map Azure power states to our app states
        let status = "unknown";
        if (powerState === "running") status = "running";
        else if (powerState === "stopped" || powerState === "deallocated") status = "stopped";
        else if (powerState === "starting") status = "starting";
        else if (powerState === "stopping") status = "stopping";
        
        context.res = {
            status: 200,
            body: { status }
        };
    } catch (error) {
        context.log.error("Error getting VM status:", error);
        context.res = {
            status: 500,
            body: { error: "Failed to get VM status", details: error.message }
        };
    }
};
```

```javascript
// toggle function
const { DefaultAzureCredential } = require("@azure/identity");
const { ComputeManagementClient } = require("@azure/arm-compute");

module.exports = async function (context, req) {
    try {
        const { subscriptionId, resourceGroup, vmName, action } = req.body;
        
        // Validate required parameters
        if (!subscriptionId || !resourceGroup || !vmName || !action) {
            context.res = {
                status: 400,
                body: { error: "Missing required parameters" }
            };
            return;
        }
        
        // Validate action
        if (action !== "start" && action !== "stop") {
            context.res = {
                status: 400,
                body: { error: "Invalid action. Must be 'start' or 'stop'" }
            };
            return;
        }
        
        // Authenticate with Azure
        const credential = new DefaultAzureCredential();
        const client = new ComputeManagementClient(credential, subscriptionId);
        
        // Perform the requested action
        if (action === "start") {
            await client.virtualMachines.beginStart(resourceGroup, vmName);
        } else {
            // Use deallocate to stop billing
            await client.virtualMachines.beginDeallocate(resourceGroup, vmName);
        }
        
        context.res = {
            status: 202,
            body: { message: `VM ${action} operation initiated` }
        };
    } catch (error) {
        context.log.error(`Error ${req.body.action}ing VM:`, error);
        context.res = {
            status: 500,
            body: { error: `Failed to ${req.body.action} VM`, details: error.message }
        };
    }
};
```

## Step 7: Secure Your Azure Function API

Add authentication to your Azure Function using one of these methods:

1. **Function Keys**: Use the built-in function keys for simple authentication
2. **Azure AD**: For enterprise-grade security, configure Azure AD authentication

### Using Function Keys (Simple)

```bash
# Get the default function key
FUNCTION_KEY=$(az functionapp keys list \
  --resource-group $RESOURCE_GROUP \
  --name $FUNCTION_APP_NAME \
  --query "functionKeys.default" \
  --output tsv)

echo "Your function key is: $FUNCTION_KEY"
```

Update your React app settings with:
- API Endpoint: `https://$FUNCTION_APP_NAME.azurewebsites.net/api`
- API Key: The function key you retrieved

## Step 8: Update Your Static Website with API Details

After deploying your Azure Function, you'll need to update your static website with the correct API endpoint and key. You can either:

1. Rebuild the React app with environment variables and redeploy
2. Use the settings UI in the app to enter the API details

## Additional Security Considerations

1. **HTTPS**: Azure Storage static websites automatically use HTTPS
2. **CORS**: Configure CORS on your Azure Function to only allow requests from your static website
3. **Authentication**: Consider using Azure AD authentication for production environments
4. **Managed Identity**: Use managed identities for Azure Functions to securely access Azure resources

## Troubleshooting

- **CORS Issues**: Ensure CORS is properly configured on your Azure Function
- **Authentication Errors**: Verify your API key or authentication token
- **VM Control Errors**: Check that your service principal or managed identity has the necessary permissions to control VMs
- **404 Errors**: Make sure your static website is properly configured with index.html as both the index and error document
