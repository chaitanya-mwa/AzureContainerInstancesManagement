[![Software License](https://img.shields.io/badge/license-MIT-brightgreen.svg?style=flat-square)](LICENSE)
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg?style=flat-square)](http://makeapullrequest.com)
[![unofficial Google Analytics for GitHub](https://gaforgithub.azurewebsites.net/api?repo=AzureContainerInstancesManagement)](https://github.com/dgkanatsios/gaforgithub)
![](https://img.shields.io/badge/status-beta-orange.svg)

# AzureContainerInstancesManagement

Manage Azure Container Instances using Azure Functions. We suppose that there is an external service that manages running instances (Container Groups) and schedules sessions on them. `Sessions` could be anything that has a fixed lifetime, like multiplayer game sessions.

## Inspiration
This project was heavily inspired by a similar project that deals with VMs called [AzureGameRoomsScaler](https://github.com/PoisonousJohn/AzureGameRoomsScaler).

## One-click deployment

Click the following button to deploy in your Azure subscription:

<a href="https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2Fdgkanatsios%2FAzureContainerInstancesManagement%2Fmaster%2Fdeploy.json" target="_blank"><img src="http://azuredeploy.net/deploybutton.png"/></a>

As soon as the deployment completes, you need to manually add the Event Subscription webhook manually using the instructions [here](https://docs.microsoft.com/en-us/azure/azure-functions/functions-bindings-event-grid#create-a-subscription).

## Technical details

This project allows you to manage [Azure Container Instances](https://azure.microsoft.com/en-us/services/container-instances/) using [Azure Functions](https://azure.microsoft.com/en-us/services/functions/). It contains 8 functions:

- **ACICreate**: Creates a new Azure Container Group
- **ACIDelete**: Deletes a Container Group
- **ACIDetails**: Gets details/logs for a Container Group/Container
- **ACIGC**: [Timer triggered](https://docs.microsoft.com/en-us/azure/azure-functions/functions-bindings-timer), runs every 5', deletes all Container Groups that have no running sessions and have been marked as 'MarkedForDeletion'
- **ACIList**: Returns the details (IP/ number of active sessions) about 'Running' Container Groups
- **ACIMonitor**: Responds to Event Grid events which occur when a Container Instance resource is created/deleted/changed
- **ACISetSessions**: Sets the running sessions for each Container Group
- **ACISetState**: Sets the state of the Container Group (2 options are 'MarkedForDeletion' and 'Failed')

These functions are supposed to be called by an external service (for a game, this would be the matchmaking service).

Azure Container Groups that are created can be in one of the below states:

- **Creating**: Container group has just been created
- **Running**: Docker image pulled, public IP (if available) ready, can accept connections/sessions
- **MarkedForDeletion**: We can mark a Container Group as `MarkedForDeletion` so that it will be deleted when a) there are no more active sessions and b) the **ACIGC** Function runs
- **Failed**: When something has gone bad

## Flow

A typical flow of the app goes like this:

1. External service calls `ACICreate`, so a new Container Group is created and is set to `Creating` state
2. As soon as the Event Grid notification comes to `ACIMonitor` function, this means that the Container Group is ready so the `ACIMonitor` function inserts its public IP into Table Storage
3. External service can call `ACIList` to get Container Groups in `Running` state as well as `ACIDetails` to get logs/details about the Container Group
4. External service or the Docker containers themselves can call `ACISetSessions` to set running sessions count on Table Storage
5. External Service can call `ACISetState` to set Container Group’s state as `MarkedForDeletion` when the Container Group is no longer needed
6. Time triggered `ACIGC` (GC: Garbage Collector) will remove unwanted Container Groups (i.e. Container Groups that have 0 active/running sesions and are `MarkedForDeletion`)

![alt text](media/states.jpg "States and Transition")

## FAQ

#### What is this **.deployment** file at the root of the project?
This guides the [Kudu](https://github.com/projectkudu/kudu) engine as to where the source code for the Functions is located, in the GitHub repo. Check [here](https://github.com/projectkudu/kudu/wiki/Customizing-deployments) for details.

#### Why are there 4 ARM templates instead of one?
Indeed, there 4 4 ARM files on the project. They are executed in the following order:
- **deploy.json**: The master template that deploys the other 3
- **deploy.function.json**: Deploys the Azure Function App that contains the Functions of our project
- **deploy.function.config.json**: As we need to set an environment variable that gets the value of our 'ACISetSessions' Function trigger URL, we need to set up this template that executes *after* the deployment of the Azure Function App has completed.
- **deploy.eventgridsubscription.json**: Again, we need to get the Event Grid webhook/URL so we manually run this *after* the deployment of the Azure Function App has completed (and the URL has been internally created).

#### I want to handle more events from Azure Event Grid. Where is the definition of those events?
Check [here](https://docs.microsoft.com/en-us/azure/event-grid/event-schema-resource-groups) for resource group events and [here](https://docs.microsoft.com/en-us/azure/event-grid/event-schema-subscriptions) for subscription-wide events.

#### How can I troubleshoot my Azure Container Instances?
As always, Azure documentation is your friend, check [here](https://docs.microsoft.com/en-us/azure/container-instances/container-instances-troubleshooting).

#### How can I manage Keys for my Functions?
Check [here](https://github.com/Azure/azure-functions-host/wiki/Key-management-API).

#### How can I test the Functions?
Not direct Function testing on this project (yet), however you can see a testing file on `tests\index.js`. To run it, you need to setup an `tests\.env` file with the following variables properly set:

- SUBSCRIPTIONID = ''
- CLIENTID = ''
- CLIENTSECRET = ''
- TENANT = ''
- AZURE_STORAGE_ACCOUNT = ''
- AZURE_STORAGE_ACCESS_KEY = ''

#### How can I monitor Event Grid message delivery?
Check [here](https://docs.microsoft.com/en-us/azure/event-grid/monitor-event-delivery) on Azure Event Grid documentation.

## Demo
We have created a Docker image of the popular game open source game [Teeworlds](https://www.teeworlds.com/) that can be used to demonstrate this project. Here are the steps that you could use if you wanted to set up a quick demo of the project:
- You need to create an Azure Service Principal. This is an identity that has permission to create/update/delete Azure Resources. Check [here](https://docs.microsoft.com/en-us/azure/azure-resource-manager/resource-group-create-service-principal-portal) for instructions on how to do it from the Azure Portal.
- Deploy the project in your Azure subscription (you can use one-click deployment, as described in the beginning).
- Deploy the Event Grid subscription for the **ACIMonitor** Function. You can either deploy the [deploy.eventgridsubscription.json](deploy.eventgridsubscription.json) file (via the portal, ask to create a resource via `Template Deployment`) or use Azure portal or CLI directly to the **ACIMonitor** Function ([instructions](https://docs.microsoft.com/en-us/azure/azure-functions/functions-bindings-event-grid#create-a-subscription)). When you deploy, make sure that you're monitoring **all** events on either the Resource Group you're planning to create your Container Instances on or your entire subscription.
- Call the **ACICreate** Function to create an Azure Container Instance with the dgkanatsios/docker-teeworlds image. You can get Function's key from the Azure Portal ([instructions](https://docs.microsoft.com/en-us/azure/azure-functions/functions-create-first-azure-function#test-the-function)) and use the provided [Postman](https://www.getpostman.com/) file (located [here](various/ACIManagement.postman_collection.json)) to begin. POST body should be similar to (yeah, half a GB memory/CPU is more than enough):
```javascript
{
    "resourceGroup": "teeworlds",
    "containerGroupName": "teeserver1",
    "containerGroup" : {
        "location": "westeurope",
        "containers": [{
            "name": "teeserver1",
            "image": "dgkanatsios/docker-teeworlds", 
            "environmentVariables": [{
                "name":"SERVER_NAME",
                "value":"Azure-Dimitris-1"
            }],
            "resources": {
                "requests": {
                    "memoryInGB": 0.5,
                    "cpu": 0.5
                }
            },
            "ports": [{
                "protocol": "udp",
                "port": 8303
            }]
        }],
        "ipAddress": {
            "ports": [{
                "protocol": "udp",
                "port": 8303
            }],
            "type": "Public"
        },
        "osType": "Linux"
    }
}
```
As you can easily notice, game requires only one open port (8303/udp) in order to function correctly.
- Once it's deployed, you will see an entry in your Azure Table. You can use [Azure Storage Explorer](https://azure.microsoft.com/en-us/features/storage-explorer/) to monitor it. Container should be in the **Creating** state.
- Hopefully Event Grid integration works, so after a couple of minutes your instance should have transitioned to the **Running** state, having a Public IP. The Storage account that contains this Table should have a name similar to `RANDOM_STRINGacidetails`.
- Call the **ACIList** Function to see your **Running** Container Instances. If result is something like the below, you have successfully set up your Teeworlds game server on Azure!
```javascript
[
    {
        "resourceGroup": "teeworlds",
        "containerGroupName": "teeserver1",
        "PublicIP": "168.63.121.114",
        "ActiveSessions": 0
    }
]
```
- You can use this IP to connect to the Teeworlds server you just set up. Download the [Teeworlds client](https://www.teeworlds.com/?page=downloads) and connect to it.
- If you now check your Table Storage (or call the **ACIList** Function again), you should see that ActiveSessions are 1. This number should increase as more clients connect to your container server. This happens because the `dgkanatsios/docker-teeworlds` image is configured to call the **ACISetSessions** Function when a user connects/disconnect to the server.
- To get the logs from your server, use the **ACIDetails** Function. If you omit the `type:"logs"` from the POST body, you should see the Azure Resource details for your container.
- Now, suppose that you do not need this container instance any more. You may call **ACIDelete** Function, but there may be players that are currently playing the game. Of course, you do not want to ruin their experience, right? What you can do is call the **ACISetState** Function and set the container's state to `MarkedForDeletion`. That would be the POST body:
```javascript
{
    resourceGroup: "teworlds",
    containerGroupName: "teeserver1",
    state:"MarkedForDeletion"
}
```
Of course, we suppose that since it's `MarkedForDeletion`, no other games will be scheduled on this container.
- The **ACIGC** Function, which is being triggered in specific time intervals, will eventually kick-in and delete your Container Group resource when a) it's 'MarkedForDeletion' and b) it has 0 active sessions.
- To cleanup, you should release the Resource Group(s) to where you deployed your resources to as well as your Event Grid Subscriptions. To find them, use [this](https://docs.microsoft.com/en-us/azure/event-grid/monitor-event-delivery#event-subscription-status) article.