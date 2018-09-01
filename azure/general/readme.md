# General

[Home](../../readme.md) > [Azure](../readme.md) > [General](./readme.md)

## Table of Contents

1. [CLI](#cli)


## CLI

* Install azure cli from [here](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest)
* Open cmd line
* Type `az login`. After login , you will have a json result with the list of subscriptions.
* Type `az account show` to get the current subscription.

### Create Resource group

Documentation available [here](https://docs.microsoft.com/en-us/azure/azure-resource-manager/cli-azure-resource-manager#create-a-resource-group).

Example to create a resource group

```bash
az group create --name MyTestResourceGroup --location "francecentral"
```

To get the list of resource groups:

```bash
az group list
```

### Create Hosting plan

Documentation available [here](https://docs.microsoft.com/en-us/cli/azure/appservice/plan?view=azure-cli-latest#az-appservice-plan-create).

Example to create an app service plan

```bash
az appservice plan create --name MyFreeAppServicePlan --resource-group MyTestResourceGroup  --location "francecentral" --sku FREE
```

To get the list of app service plans:

```bash
az appservice plan list
```

### Create Web App

Documentation available [here](https://docs.microsoft.com/en-us/cli/azure/webapp?view=azure-cli-latest#az-webapp-create).

Example to create a web app

```bash
az webapp create --name MyAwesomeWebApp --resource-group MyTestResourceGroup --plan MyFreeAppServicePlan
```

* Get available deployment profiles [doc here](https://docs.microsoft.com/en-us/cli/azure/webapp/deployment?view=azure-cli-latest). Useful to get user login and password, for ftp connection or manual deployment.

```bash
az webapp deployment list-publishing-profiles --name MyAwesomeWebApp --resource-group MyTestResourceGroup
```

### Package and Deploy a .NET Framework Web App

Create a package using msbuild.

```bash
"C:\Program Files (x86)\Microsoft Visual Studio\2017\Community\MSBuild\15.0\Bin\msbuild.exe" MyTestWebApplication.csproj /t:package /p:Configuration=Release
```

Deploy the package using deploy.cmd file and information in deployment profile (site name and password).

```bash
"obj\Release\Package\MyTestWebApplication.deploy.cmd" /y /M:https://myawesomewebapp.scm.azurewebsites.net/msdeploy.axd /U:$MyAwesomeWebApp /P:enterpasswordhere /A:Basic "-setParam:name='IIS Web Application Name',value='MyTestWebApplication'"
```

### SQL

* Create sql server on azure. Doc available [here](https://docs.microsoft.com/en-us/azure/sql-database/sql-database-get-started-cli).

```bash
 az sql server create --name my-awesome-sql-server-dev01 --resource-group MyTestResourceGroup --location "francecentral" --admin-user **** --admin-password ****
```

* Configure firewall, in this example, I allow all.

```bash
az sql server firewall-rule create --resource-group MyTestResourceGroup --server my-awesome-sql-server-dev01 -n AllowAllIp --start-ip-address 0.0.0.0 --end-ip-address 255.255.255.255
```

* Create database

```bash
az sql db create --resource-group MyTestResourceGroup --server my-awesome-sql-server-dev01 --name myAwesomeDatabase01 --service-objective Basic
```

* Migrate an existing database using Data Migration Assistant, doc available [here](https://docs.microsoft.com/en-us/azure/sql-database/sql-database-migrate-your-sql-server-database#migrate-your-database).