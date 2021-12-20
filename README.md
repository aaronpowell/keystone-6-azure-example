# Deploying Keystone 6 on Azure

This article will cover how to deploy [Keystone](https://keystonejs.com/) to Azure using Platform-as-a-Service (PaaS).

You will need an Azure account for this, you can sign up for a [free trial](https://azure.microsoft.com/free/?WT.mc_id=javascript-38807-aapowell) if you don't already have one.

## Required Resources

There are three resources in Azure that are required to run Keystone in a PaaS model, [AppService](https://azure.microsoft.com/services/app-service/?WT.mc_id=javascript-38807-aapowell#overview) to host the Keystone web application, [Storage](https://azure.microsoft.com/product-categories/storage/?WT.mc_id=javascript-38807-aapowell) to store images/uploaded assets, and a [managed Postgres database](https://azure.microsoft.com/services/postgresql/?WT.mc_id=javascript-38807-aapowell#overview).

## Table of Contents

- Creating Resoueces
  - [Using the portal](#creating-resources-via-the-azure-portal)
  - [Using the Azure CLI](#creating-resources-via-the-azure-cli)
  - [Using an Azure Resource Manager (ARM) template](#deploy-with-an-azure-resource-manager-template)
- [Storing files and images with Azure Storage](#storing-files-and-images)
- [Automated Deploymnets](#deploying-and-running-keystone)
- [Azure AD B2C for user management](#optional---using-azure-ad-b2c-for-user-accounts)

---

## Creating Resources via the Azure Portal

In this section we'll use the Azure Portal to create the required resources to host Keystone.

1. Navigate to the [Azure Portal](https://portal.azure.com/?WT.mc_id=javascript-38807-aapowell)

1. Click **Create a resource** and search for _Resource group_ from the provided search box

1. Provide a name for your Resource Group, `my-keystone-app`, and select a region

1. Click **Review + create** then **Create**

1. Navigate to the Resource Group once it's created, click **Create resources**
   and search for _Web App_

1. Ensure the _Subscription_ and _Resource Group_ are correct, then provide the following configuration for the app:

   - _Name_ - `my-keystone-app`
   - _Publish_ - `Code`
   - _Runtime stack_ - `Node 14 LTS`
   - _Operating System_ - `Linux`
   - _Region_ - Select an appropriate region

1. Use the _App Service Plan_ to select the appropriate Sku and size for the level fo scale your app will need (refer to [the Azure docs](https://azure.microsoft.com/pricing/details/app-service/linux/?WT.mc_id=javascript-38807-aapowell) for more information on the various Sku and sizes)

1. Click **Review + create** then **Create**

1. Navigate back to the Resource Group and click **Create** then search for _Storage account_ and click **Create**

1. Ensure the _Subscription_ and _Resource Group_ are correct, then provide the following configuration for the storage account:

   - _Name_ - `mykeystoneapp`
   - _Region_ - Select an appropriate region
   - _Performance_ - `Standard`
   - _Redundancy_ - Select the appropriate level of redundancy for your files

1. Click **Review + create** then **Create**

1. Navigate back to the Resource Group and click **Create** then search for _Azure Database for PostgreSQL_ and click **Create**

1. Select _Single server_ for the service type

1. Ensure the _Subscription_ and _Resource Group_ are correct, then provide the following configuration for the storage account:

   - _Name_ - `my-keystone-db`
   - _Data source_ - `None` (unless you're wanting to import from a backup)
   - _Location_ - Select an appropriate region
   - _Version_ - `11`
   - _Compute + storage_ - Select an appropriate scale for your requirements (Basic is adequate for many Keystone workloads)

1. Enter a username and password for the _Administrator account_, click **Review + create** then **Create**

### Configuring the Resources

Once all the resources are created, you will need to get the connection information for the Postgres and Storage account to the Web App (App Service), as well as configure the resources for use.

#### Configure the Storage Account

1. Navigate to the **Storage Account** resource, then **Data storage** - **Containers**
1. Create a new Container, provide a _Name_, `keystone-uploads`, and set _Public access level_ to `Blob`, then click **Create**
1. Navigate to **Security + networking** - **Access keys**, copy the _Storage account name_ and _key1_
1. Navigate to the **Web App (App Service)** you created and go to **Settings** - **Configuration**
1. Create three **New application settings**:
   - `AZURE_STORAGE_ACCOUNT` as the _Storage account name_ value you copied above.
   - `AZURE_STORAGE_KEY` as the _key1_ value you copied above.
   - `AZURE_STORAGE_CONTAINER` as the name of the container, which you specified as `keystone-uploads` above.
1. These will become the environment variables available to Keystone, click _Save_.

#### Configure Postgres

1. Navigate to the **Azure Database for PostgreSQL server** resource then **Settings** - **Connection security**
1. Set `Allow access to Azure services` to `Yes` and click **Save**
1. Navigate to **Overview** and copy _Server name_ and _Server admin login name_
1. Open the [Azure Cloud Shell](https://shell.azure.com?WT.mc_id=javascript-38807-aapowell) and log into the `psql` cli:

   - `psql --host <server> --user <username> --port=5432 --dbname postgres`

1. Create a database for Keystone to use `CREATE DATABASE keystone;` then close the Cloud Shell
   - Optional - create a separate non server admin user (see [this doc](https://docs.microsoft.com/azure/mysql/howto-create-users?tabs=single-server&WT.mc_id=javascript-38807-aapowell) for guidance)
1. Navigate to the **Web App (App Service)** you created and go to **Settings** - **Configuration**
1. Create a **New application setting** named `DATABASE_URL`, which contains the connection string, encoded [per Prisma's requirements](https://www.prisma.io/docs/concepts/database-connectors/postgresql) (this will become the environment variables available to Keystone) and click _Save_, it will look something like:
   
   - `postgres://$username%40$serverName:$password@$serverName.postgres.database.azure.com:5432/$dbName`

---

## Creating Resources via the Azure CLI

In this section, we'll use the [Azure CLI](https://docs.microsoft.com/cli/azure/?WT.mc_id=javascript-38807-aapowell) to create the required resources. This will assume you have some familiarity with the Azure CLI and how to find the right values.

1. Create a new Resource Group

   ```bash
   rgName=my-keystone-app
   location=westus
   az group create --name $rgName --location $location
   ```

1. Create a new Linux App Service Plan (ensure you change the `number-of-workers` and `sku` to meet your scale requirements)

   ```bash
   appPlanName=keystone-app-service-plan
   az appservice plan create --resource-group $rgName --name $appPlanName --is-linux --number-of-workers 4 --sku S1 --location $location
   ```

1. Create a Web App (App Service) running Node.js 14

   ```bash
   webAppName=my-keystone-app
   az webapp create --resource-group $rgName --name $webAppName --plan $appPlanName --runtime "NODE|14-lts"
   ```

1. Create a Storage Account

   ```bash
   saName=mykeystoneapp
   az storage account create --resource-group $rgName --name $saName --location $location

   # Get the access key
   saKey=$(az storage account keys list --account-name $saName --query "[?keyName=='key1'].value" --output tsv)

   # Add a container to the storage account
   container=keystone-uploads
   az storage container create --name $container --public-access blob --access-key $saKey --account-name $saName
   ```

1. Create a Postgres database

   ```bash
   serverName=my-keystone-db
   dbName=keystone
   username=keystone
   password=...

   # Create the server
   az postgres server create --resource-group $rgName --name $serverName --location $location --admin-user $username --admin-password $password --version 11 --sku-name B_Gen5_1

   # Create the database
   az postgres db create --resource-group $rgName --name $dbName --server-name $serverName

   # Allow Azure resources through the firewall
   az postgres server firewall-rule create --resource-group $rgName --server-name $serverName --name AllowAllAzureIps --start-ip-address 0.0.0.0 --end-ip-address 0.0.0.0
   ```

1. Add configuration values to the Web App (App Service)

   ```bash
   az webapp config appsettings set --resource-group $rgName --name $webAppName --setting AZURE_STORAGE_ACCOUNT=$saName
   az webapp config appsettings set --resource-group $rgName --name $webAppName --setting AZURE_STORAGE_KEY=$saKey
   az webapp config appsettings set --resource-group $rgName --name $webAppName --setting AZURE_STORAGE_CONTAINER=$container
   az webapp config appsettings set --resource-group $rgName --name $webAppName --setting DATABASE_URL=postgres://$username%40$serverName:$password@$serverName.postgres.database.azure.com:5432/$dbName
   ```

---

## Deploy with an Azure Resource Manager template

A Resource Manager template has been created that will provision all the resources that are required, as well as setup the appropriate settings across the AppService. Click the button below and fill out the parameters as required to create the resources. Alternatively, you can create a custom template deployment via the portal and [upload the template file](https://github.com/aaronpowell/keystone-6-azure-example/blob/main/.azure/azuredeploy.json), or [run the template form the Azure CLI](https://docs.microsoft.com/azure/azure-resource-manager/templates/deploy-cli?WT.mc_id=javascript-38807-aapowell).

[![Deploy to Azure](https://aka.ms/deploytoazurebutton)](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2Faaronpowell%2Fkeystone-6-azure-example%2Fmain%2F.azure%2Fazuredeploy.json)

_The template is located at [`.azure/azuredeploy.json`](https://github.com/aaronpowell/keystone-6-azure-example/blob/main/.azure/azuredeploy.json)_

---

## Storing files and images

As AppService is a PaaS hosting model, it is not possible to store files on disk, as it is not persisted between deployments nor is it shared between scale-out machines. The solution for this in Keystone is to use a custom field type https://github.com/keystonejs-contrib/k6-contrib/tree/main/packages/fields-azure.

### Local development

For local development, you can either use the standard Keystone file/image field types (which stored on the local disk), or the [Azurite emulator](https://docs.microsoft.com/azure/storage/common/storage-use-azurite?tabs=visual-studio-code&WT.mc_id=javascript-38807-aapowell#install-and-run-azurite).

## Deploying and running Keystone

Azure AppService can be deployed to using CI/CD pipelines or via FTPS, refer to the [Azure docs](https://docs.microsoft.com/azure/app-service/deploy-continuous-deployment?tabs=github&WT.mc_id=javascript-38807-aapowell) on how to do this for your preferred manner.

To start the Node.js application, AppService will run the `npm start` command. As there is no guarantee that the symlinks created by `npm install` were preserved (in the case of an upload from a CI/CD pipeline) it is recommended that the `npm start` command directly references the Keystone entry point:

```json
"scripts": {
    "start": "node ./node_modules/@keystone-6/core/bin/cli.js start"
}
```

## Optional - Using Azure AD B2C for user accounts

Keystone provides built in authentication but this can be replaced with a custom provider that supports Azure AD B2C. For this, you'll need the https://github.com/OpenSaasAU/keystone-nextjs-auth auth package and then follow the [NextAuth docs on setting up Azure AD B2C](https://next-auth.js.org/v3/providers/azure-ad-b2c).

Once Azure is configured, update your auth provider with nextjs-auth to use AzureADB2C like so:

```js
const auth = createAuth({
  listKey: "User",
  identityField: "subjectId",
  sessionData: `id name email`,
  autoCreate: true,
  userMap: { subjectId: "id", name: "name" },
  accountMap: {},
  profileMap: { email: "mail" },
  providers: [
    Providers.AzureADB2C({
      clientId: process.env.AZURE_CLIENT_ID,
      clientSecret: process.env.AZURE_CLIENT_SECRET,
      scope: "offline_access User.Read",
      tenantId: process.env.AZURE_TENANT_ID,
    }),
  ],
});
```

## Running the sample locally

To run the sample locally, you will need to use a Postgres database (either locally hosted, or hosted on Azure) and provide an Azure storage account (either an emulator like [Azurite](https://docs.microsoft.com/azure/storage/common/storage-use-azurite?tabs=visual-studio-code&WT.mc_id=javascript-38807-aapowell#install-and-run-azurite) or an Azure resource).

This connection information will need to be made available as environment variables:

```plaintext
AZURE_STORAGE_ACCOUNT=...
AZURE_STORAGE_KEY=...
AZURE_STORAGE_CONTAINER=...
DATABASE_URL=...
AZURE_STORAGE_ACCOUNT_HOST=http://127.0.0.1:10000/
```
