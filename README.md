# pangeo-forge Azure Bakery ☁️🍞

This repository serves as the provider of an Terraform Application which deploys the necessary infrastructure to provide a `pangeo-forge` Bakery on Azure

# Contents

* [🧑‍💻 Development - Requirements](#requirements)
* [🧑‍💻 Development - Getting Started](#getting-started-🏃‍♀️)
* [🧑‍💻 Development - Makefile goodness](#makefile-goodness)
* [🚀 Deployment - Standard Deployments](#standard-deployments)

# Development

## Requirements

To develop on this project, you should have the following installed:

* [Python 3.8.*](https://www.python.org/downloads/) (We recommend using [Pyenv](https://github.com/pyenv/pyenv) to handle Python versions)
* [Poetry](https://github.com/python-poetry/poetry)
* [Azure CLI](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli)
* [kubectl](https://kubernetes.io/docs/tasks/tools/#kubectl)
* [Terraform 0.15.0](https://www.terraform.io/downloads.html)

If you're developing on MacOS, all of the above can be installed using [homebrew](https://brew.sh/)

If you're developing on Windows, we'd recommend using either [Git BASH](https://gitforwindows.org/) or [Windows Subsystem for Linux](https://docs.microsoft.com/en-us/windows/wsl/install-win10)


## Getting Started 🏃‍♀️

### Installing dependencies

This project requires some Python dependencies (Namely `prefect` and `dotenv`), these are so that:

* We can register flows for testing (it was also used to generate `prefect_agent_conf.yaml`)
* We can use `.env` files to provide both Prefect Flows and Terraform environment variables

To install the dependencies, run either:

```bash
$ make install # For deploying the infrastructure only
# or
$ make install-dev # For developing on this project
```

### Azure Credential setup

To develop and deploy this project, you will first need to setup some credentials and permissions on Azure

#### Logging in

With the Azure CLI installed, run:

```bash
$ az login # Opens up a browser window to login with
[
  {
    "cloudName": "AzureCloud",
    "homeTenantId": "<a-home-tenant-id>",
    "id": "<a-id>",
    "isDefault": true,
    "managedByTenants": [],
    "name": "SuperAwesomeSubscription",
    "state": "Enabled",
    "tenantId": "<a-tenant-id>",
    "user": {
      "name": "<your-username>",
      "type": "user"
    }
  },
  ...
]
```

If you notice that the `Subscription` you intend to use has `"isDefault": false`, then refer to [this documentation](https://docs.microsoft.com/en-us/cli/azure/manage-azure-subscriptions-azure-cli#change-the-active-subscription) on how to switch your default `Subsciption`.

#### Creating a Service Principal

You will then need to create a `Service Principal` to deploy as:

```bash
$ az ad sp create-for-rbac --name "<name-for-your-service-principal>"
{
  "appId": "<an-app-id>",
  "displayName": "<name-for-your-service-principal>",
  "name": "http://<name-for-your-service-principal>",
  "password": "<a-password>",
  "tenant": "<a-tenant-id>"
}
```

#### Adding Service Principal permissions

Your service principal will need a few permissions added to it, for these you'll need to get its `objectId`, you can get this by running:

```bash
$ az ad sp list --display-name "<name-for-your-service-principal>"
[
  {
    "accountEnabled": "True",
    "addIns": [],
    "alternativeNames": [],
    "appDisplayName": "<name-for-your-service-principal>",
    "appId": "<a-app-id>",
    "appOwnerTenantId": "<a-tenant-id>",
    "appRoleAssignmentRequired": false,
    "appRoles": [],
    "applicationTemplateId": null,
    "deletionTimestamp": null,
    "displayName": "<name-for-your-service-principal>",
    "errorUrl": null,
    "homepage": "https://<name-for-your-service-principal>",
    "informationalUrls": {
      "marketing": null,
      "privacy": null,
      "support": null,
      "termsOfService": null
    },
    "keyCredentials": [],
    "logoutUrl": null,
    "notificationEmailAddresses": [],
    "oauth2Permissions": [
      {
        "adminConsentDescription": "Allow the application to access <name-for-your-service-principal> on behalf of the signed-in user.",
        "adminConsentDisplayName": "Access <name-for-your-service-principal>",
        "id": "<an-id>",
        "isEnabled": true,
        "type": "User",
        "userConsentDescription": "Allow the application to access <name-for-your-service-principal> on your behalf.",
        "userConsentDisplayName": "Access <name-for-your-service-principal>",
        "value": "user_impersonation"
      }
    ],
    "objectId": "<a-object-id>",
    "objectType": "ServicePrincipal",
    "odata.type": "Microsoft.DirectoryServices.ServicePrincipal",
    "passwordCredentials": [],
    "preferredSingleSignOnMode": null,
    "preferredTokenSigningKeyEndDateTime": null,
    "preferredTokenSigningKeyThumbprint": null,
    "publisherName": "<a-publisher-name>",
    "replyUrls": [],
    "samlMetadataUrl": null,
    "samlSingleSignOnSettings": null,
    "servicePrincipalNames": [
      "http://<name-for-your-service-principal>",
      "<a-service-principal-id>"
    ],
    "servicePrincipalType": "Application",
    "signInAudience": "AzureADMyOrg",
    "tags": [],
    "tokenEncryptionKeyId": null
  }
]
```

From the resultant JSON output to the screen, copy the value of `objectID`.

You can then run:

```bash
$ az role assignment create --assignee "<objectId>" --role "Storage Blob Data Contributor"
$ az role assignment create --assignee "<objectId>" --role "User Access Administrator"
$ az role assignment create --assignee "<objectId>" --role  "Azure Kubernetes Service Cluster User Role"
```

You should now be setup with the correct permissions to deploy the infrastructure onto Azure. Further reading on Azure Service Principals can be found [here](https://docs.microsoft.com/en-us/cli/azure/ad/sp?view=azure-cli-latest).

### `.env` file

A `.env` file is expected at the root of the repository to store variables used within deployment, the expected values are:

```bash
TF_VAR_owner="<your-name>"
TF_VAR_identifier="<a-unique-value-to-tie-to-your-deployment>"
TF_VAR_region="<azure-region-name-to-deploy-to>"
BAKERY_NAMESPACE="<the-name-for-your-prefect-agent-k8s-configs-namespace>"
PREFECT__CLOUD__AGENT__AUTH_TOKEN="<value-of-runner-token>" # See https://docs.prefect.io/orchestration/agents/overview.html#tokens - This is required for your Agent to communicate to Prefect Cloud
```

An example called `example.env` is available for you to copy, rename, and fill out accordingly.

## Makefile goodness

A `Makefile` is available in the root of the repository to abstract away commonly used commands for development:

**`make install`**

> This will run `poetry install --no-dev` with the contents of `poetry.lock` - Use this if you only want to deploy the infrastructure

**`make install-dev`**

> This will run `poetry install` with the contents of `poetry.lock` - Use this if you're developing this project

**`make init`**

> This will run `terraform init` within the `terraform/` directory, installing any providers required for deployment

**`make lint`**

> This will run `terraform validate` within the `terraform/` directory, showing you anything that is incorrect in your Terraform scripts

**`make format`**

> This will run `terraform fmt` within the `terraform/` directory, this **will** modify files if issues were found

**`make plan`**

> This will run `terraform plan` within the `terraform/` directory using the contents of your `deployment.tfvars.json` file

**`make apply`**

> This will run `terraform apply` within the `terraform/` directory using the contents of your `deployment.tfvars.json` file. The deployment is auto-approved, so **make sure** you know what you're changing with your deployment first! (Best to run `make plan` to check!)

**`make destroy`**

> This will run `terraform destroy` within the `terraform/` directory using the contents of your `deployment.tfvars.json` file. The destroy is auto-approved, so **make sure** you know what you're destroying first!

**`make configure-kubectl`**

> This will setup `kubectl` to point to your deployed AKS cluster using the output variables terraform creates. You **must** have deployed the cluster first

**`make setup-agent`**

> This will create a namespace on the AKS cluster with the name of `BAKERY_NAMESPACE`, then the agent configuration in `prefect_agent_conf.yaml` will be applied to the cluster. You **must** have deployed the cluster first

# Deployment

Before you deploy the infrastructure, ensure you've taken all the steps outlined under [Azure Credential setup](#azure-credential-setup) and have run [`make install`](#makefile-goodness)

## Standard Deployments

For standard deploys, you can check _what_ you'll be deploying by running:

```bash
$ make plan # Outputs the result of `terraform plan`
```

To deploy the infrastructure, you can run:

```bash
$ make apply # Deploys the Bakery
$ make configure-kubectl # Uses the output from Terraform to configure kubectl to point to the newly deployed cluster
$ make setup-agent # This will create a namespace on the AKS cluster with the name of `BAKERY_NAMESPACE`, then the agent configuration in `prefect_agent_conf.yaml` will be applied to the cluster
```

You could also chain the above commands in one line:

```bash
$ make apply configure-kubectl setup-agent
```

To destroy the infrastructure, you can run:

```bash
$ make destroy # Destroys the Bakery
