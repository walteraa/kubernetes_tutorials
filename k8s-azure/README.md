# Setting up the Azure CLI


**_Notice¹: I'm considering that you complete the tutorial "Setting up kubernetes" before start this tutorial._**

**_Notice²: This tutorial was developed over `Ubuntu 16.04`, maybe you'll need change this step depending on your OS_**

Azure CLI will make possible manipulating your Azure account to create groups, VMs, clusters, etc. Install it is pretty simple: you just need download and execute the installer, fill some information,confirming using enter(kinda of "next, next, next...") and after that it will be enable to use the cli to log in and manipulate your account.

## Downloading and executing the installer

For download and execute the installer, you just need to use curl followed by bash command

`$ curl -L https://aka.ms/InstallAzureCli | bash`

After that, some informations will be asked for you(e.g.: place where the cli will be installed, etc). Finishing it, you'll be able to use the Azure CLI.

## Login in your azure Account

All "events" that you want send to Azure, should be sent using the `az` command, for perform this commands successfully, you need do log in first, using the same command(`az`)

`$ az login`

After that, an instruction to log in will appear in terminal with a link and a code

`To sign in, use a web browser to open the page https://aka.ms/devicelogin and enter the code #{SOME_CODE} to authenticate.`

You should open this link in your browser and put the `#{SOME_CODE}` code, after that you will make log on in your azure account. Finally you will be able to manipulate your azure infrastructure.

You can check your credentials

```
$ az account list 
A few accounts are skipped as they don't have 'Enabled' state. Use '--all' to display them.
[
  {
    "cloudName": "AzureCloud",
    "id": "#{ID}",
    "isDefault": true,
    "name": "Free Trial",
    "state": "Enabled",
    "tenantId": "#{TENANT_ID}",
    "user": {
      "name": "walter-alves@outlook.com",
      "type": "user"
    }
  }
]
```

Now you can perform commands on your azure infra.

# Creating your kubernetes cluster in azure infra

In this tutorial, I will use a cluster in west europe with 1 agent(the free trial account core quotes make possible using only 4 cores, to get 2 agents is necessary 6 cores) named `az-europe`.

## Creating resource group

A resource group is a "container" that groups related resources for a solution. The resource group includes all the resources that you can manage as a group.

Create a resource group is very simple, you just need use the azure cli passing the name and location

```
$ az group create --name az-west-europe --location westeurope
[
  {
    "id": "/subscriptions/#{ID}/resourceGroups/az-europe",
    "location": "westeurope",
    "managedBy": null,
    "name": "az-europe",
    "properties": {
      "provisioningState": "Succeeded"
    },
    "tags": null
  }
]
```

Now you can create resources using the resource group created above

## Creating the kubernetes cluster

Now I will create the kubernetes cluster using the azure container service manager(`az acs`) passing the orquestrator type, the resource group which this cluster make part the cluster name and the amount of agents

```
$ az acs create --orchestrator-type kubernetes --resource-group az-west-europe --name az-europe --agent-count 1
waiting for AAD role to propagate.done
\ Running ..
```

If everything is fine, you will have the kubernetes cluster provisioned in your azure infra.

# Authorizing the kubernetes client(kubectl) to operate the Azure cluster

To operate the azure cluster, you should get the necessary credentials to connect your kubernetes client in the cluster.

## Getting Azure cluster credentials

Ok, now you need to use the kubernetes container service manager to get the credentials, passing the resource group and the cluster name

`$ az acs kubernetes get-credentials --resource-group=az-west-europe --name=az-europe`

It will update the kubernetes client(kubectl) configuration file saving the credentials to operate the cluster.

# Making sure that the configurations were made successfully


To make sure that everything until here works fine, we can run two commands

- Verifying if the cluster was added in kubernetes configuration

```
$ kubectl config get-contexts
CURRENT   NAME          CLUSTER                                       AUTHINFO                                      NAMESPACE
*          az-europe-az-west-europe-4fecb0mgmt     az-europe-az-west-europe-4fecb0mgmt           az-europe-az-west-europe-4fecb0mgmt-admin     
```

- Verifying if you are able to list resources from cluster
(TODO)

# Making your life easier

Probably you realized that the name `z-europe-az-west-europe-4fecb0mgmt` is a kinda large, somethimes you will need to use this name to get or create some resource from a specific context, but for our happiness, we can use the command below to make the context name smaller

`$ kubectl config rename-context az-europe-az-west-europe-4fecb0mgmt az-europe`

Then you can check if the context name was changed

```
$ kubectl config get-contexts
CURRENT   NAME          CLUSTER                                       AUTHINFO                                      NAMESPACE
*          az-europe     az-europe-az-west-europe-4fecb0mgmt           az-europe-az-west-europe-4fecb0mgmt-admin     
```

# Creating resources

You can read this tutorial about [Operating a kubernetes cluster](#)(TODO: create tutorial). 

