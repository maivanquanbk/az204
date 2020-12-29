# AZ-204 Revise Note

- [AZ-204 Revise Note](#az-204-revise-note)
  - [Develop Azure compute solution (25-30%)](#develop-azure-compute-solution-25-30)
    - [Implement IaaS Solutions](#implement-iaas-solutions)
      - [Provision VMs](#provision-vms)
      - [Configure VM for remote access](#configure-vm-for-remote-access)
      - [Manage the availability of your Azure VMs](#manage-the-availability-of-your-azure-vms)
      - [VMs data backup and recovery](#vms-data-backup-and-recovery)
      - [Azure Resource Manager templates](#azure-resource-manager-templates)
      - [Create, publish and deploy container images for solutions](#create-publish-and-deploy-container-images-for-solutions)
    - [Create Azure App Service Web Apps](#create-azure-app-service-web-apps)
    - [Implement Azure Function](#implement-azure-function)
  - [Develop Azure Storage (10-15%)](#develop-azure-storage-10-15)
    - [Develop solutions that use Cosmos DB storage](#develop-solutions-that-use-cosmos-db-storage)
    - [Develop solutions that use blob storage](#develop-solutions-that-use-blob-storage)
  - [Implement Azure Security (15-20%)](#implement-azure-security-15-20)
  - [Monitor, troubleshoot, and optimize Azure solutions (10-15%)](#monitor-troubleshoot-and-optimize-azure-solutions-10-15)
  - [Connect to and consume Azure services and third-party services (25-30%)](#connect-to-and-consume-azure-services-and-third-party-services-25-30)
  - [Additional Tips and Resources](#additional-tips-and-resources)

## Develop Azure compute solution (25-30%)

### Implement IaaS Solutions

***

#### Provision VMs

***

> Compile checklist for creating a VM:
>
> - Start with the network
> - Name the VM
> - Decide location of the VM
> - Decide size of the VM
> - Storage for the VM
> - Select an Operating system
> - Understand pricing model

1. Start with the network

    - It is important to plan before creating resources because it is hard to change latter.
    - If the VNet is to be connected to other VNets or on-premises networks, you must select address ranges that don't overlap.
    - Best practice to use unrouteable IP address: 10.0.0.0/8, 172.16.0.0/12, or 192.168.0.0/16
    - Azure reserves the first four and the last IP in each subnet for its own use.
    - Security: By default, services outside VNets cannot access services inside VNets. However, services in different subnets of a VNets can talk to each other. We can use NSG to control the traffic flow to and from subnet; and to and from VMs.
    - A VM must have at least one NIC but can have more than one. Each NIC attached to a VM must exist in the same location and subscription of the VM.
    - Can change the subnet VM connected to but cannot change the VNet.

    **Terminologies**: Virtual network (VNets), Subnet, Network security group (NSG), Network Interface (NIC), Network address translation ([NAT](https://docs.microsoft.com/en-us/azure/load-balancer/load-balancer-outbound-connections)), Private & Public IP.

    **More information**: [here](https://docs.microsoft.com/en-us/azure/virtual-machines/network-overview?toc=/azure/virtual-machines/windows/toc.json&bc=/azure/virtual-machines/windows/breadcrumb/toc.json)

2. Plan for each VM

    **Checklist**:

    - What does the VM communication with?
    - Which ports are open?
    - Which OS is used?
    - How much disk space is in use?
    - What kind of data does this use? Are there any restrictions?
    - What sort of CPU, memory, and disk IO load does the VM need?

    **Name the VM**:

    - It’s not trivial to change later.
    - You should choose names that are meaningful and consistent which follow a sort of convention.

    **Decide location of the VM**:

    - Criteria:

        - Distance to users, legal, compliance requirements, etc. 
        - Each region has different hardware available, and some configurations are not available in all regions.
        - Prices could be different between locations.
    - Learn more about Azure paired regions: [here](https://docs.microsoft.com/en-us/azure/best-practices-availability-paired-regions#what-are-paired-regions)

    **Decide size of the VM**:

    Types:

    - **General purpose**: Balanced CPU-to-memory ratio. Ideal for testing & development. Suit for small to medium databases and small to medium traffic web servers.
    - **Compute optimized**: High CPU-to memory ratio. Good for medium traffic web servers, network appliances, batch processes, and application servers.
    - **Memory optimized**: High memory-to-CPU. Great for relational database server, medium to large caches, and in-memory analytics.
    - **Storage optimized**: High disk throughput and IO. Ideas for Big Data, SQL, NoSQL database, data warehousing and large transactional databases.
    - **GPU**: Optimized for graphic rendering, machine learning.
    - **High performance compute**: designed for HPC workloads.

    Be careful about resizing production VMs - they will be rebooted automatically which can cause a temporary outage and change some configuration settings such as the IP address.

    **Storage for the VM**:

    - All VM will have at least two virtual hard disks (VHDs). The first stores the operation system, and the second is used as temporary storage.
    - It’s common to create one or more data disks to manage the security, reliability, and performance of the disk independently.
    - Each VHD is held in Azure Storage as page blobs.
    - Two options for managing the relationship between the storage account and each VHD:

        - **Unmanaged disks**: We create and manage our own storage accounts that are used to hold the VHDs. A single storage account has a fixed-rate limit of 20,000 I/O operations/sec which supporting 40 VHDs at full utilization. We need create more storage accounts if we want to scale out.
        - **Managed disks**: ***Recommended model***. Azure will manage the storage accounts for us. We can specify the size of the disk up to 4 TB. We can easily scale-out without worrying about storage account limits.

    **Select the OS**:

    - Azure bundles the cost of the OS license into the price.
    - Scan Azure Marketplace to get the images for variety of stacks

    **Understand the pricing model**:

    - Compute and storage are charged independently
    - Compute expenses are billed on a per-minute basis when a VM is up. The cost includes OS license cost. Linux-based VMs are cheaper because there is no license charge.
    - Storage costs are billed even when a VM is stop or deallocated.

3. Basic commands to provision a VM

    - Create a new VM:

      ```powershell
      New-AzVm `
          -ResourceGroupName "myResourceGroupVM" `
          -Name "myVM" `
          -Location "EastUS" `
          -VirtualNetworkName "myVnet" `
          -SubnetName "mySubnet" `
          -SecurityGroupName "myNetworkSecurityGroup" `
          -PublicIpAddressName "myPublicIpAddress" `
      ```

    - Get Public Ip Address

      ```powershell
      Get-AzPublicIpAddress `
        -ResourceGroupName "myResourceGroupVM"  | Select IpAddress
      ```

#### Configure VM for remote access

***

1. Configure remote access to Linux VM

    There are two approaches we can use to authenticate an SSH connection: username and password, or an SSH key pair.
    - Using username and password to access VM leaves the VM vulnerable to brute-force attacks of passwords. The more secured method is using public-private key pair.
    - But if we need to be able to access the Linux VM from a variety of locations, a username and password combination might be a better approach.

    To connect to the VM via SSH key pair, we need:
    - the public IP address of the VM (*Note*: it could be static or dynamic)
    - the username of the local account on the VM
    - a public key configured in that account
    - access to the corresponding private key
    - port 22 open on the VM

2. Configure remote access to Window VM

    To connect to an Azure VM with an RDP client, we will need:
    - The public IP address of the VM (or private if the VM is configured to connect to your network).
    - Port 3389 open on the VM.

3. Enable just-in-time (JIT) access to VM

    JIT is a feature of Azure Defender. It blocks down inbound traffic to a specific port such as 22 (SSH) and 3389 (RDP), etc to reduce the attach surface of a VM. Nevertheless, it still provides the access to legitimate users when needed.

    By using JIT, we do not need to set rules for NSG or Firewall to access VM that apparently lets some ports be being opened all the time.

    How to enable JIT for VM access:
    1. Enable [Azure Defender for servers](https://docs.microsoft.com/en-us/azure/security-center/defender-for-servers-introduction) on the subscription in Security Center.
    2. Ensure all rules which are used for VM access such as 22 (SSH) and 3389 (RDP) are removed from NSG and Firewall. Otherwise, they will bypass JIT rules and let the ports are opened.
    3. Enable JIT for the VM in specific ports with Azure Portal or Azure Powershell. By that, Security Center will add a rule "deny all inbound traffic" to the selected ports in NSG and Firewall.

    How does a user get access to the VM which has JIT applied:
    1. When a user request access to a VM, Security Center checks that the user has RBAC permissions for that VM.
    2. Security Center then configures NSG and Firewall to allow inbound traffic to the port and from the relevant IP (or range), for an amount of time.
    3. After the time has expired, Security Center restores NSG and Firewall to the previous state. **Connections that are already established are not interrupted**.

    All setup steps could be found [here](https://docs.microsoft.com/en-us/azure/security-center/security-center-just-in-time?tabs=jit-config-asc%2Cjit-request-asc)

#### Manage the availability of your Azure VMs

***

VMs are about to down and reboot because of serveral reasons: physical servers are down or outage, software and hardward update.

If this happens, Azure will move the VM to a healthy host server automatically. However, this self-healing migration could take several minutes, during which, the application(s) hosted on that VM will not be available.

**Fault domain**: a logical group of hardware in Azure that shares a common power source and network switch.

**Update domain**: a logical group of hardware that can undergo maintenance or be rebooted at the same time.

**Avalability set**: a logical group of VMs so that they aren't all subject to a single point of failure and not all upgraded at the same time during a host operating system upgrade in the datacenter. VMs in an avaiabilty set should also have the same funtionalities and have the same software installed.

Avalability set guarantees to spread VMs across Fault Domains and Update Domains.

#### VMs data backup and recovery

***

Azure Backup is a backup as a service offering that protects physical or virtual machines no matter where they reside: on-premises or in the cloud.

Azure Backup is native support for Azure Virtual Machines, both Windows, and Linux.

#### Azure Resource Manager templates

***

Resource Manager templates are JSON files that define the resources you need to deploy for your solution. ARM template features include **declarative syntax** and **repeatable results**.

**Azure VM extensions** are small applications that allow you to configure and automate tasks on Azure VMs after initial deployment. You bundle extensions with a new VM deployment or run them against an existing system.

#### Create, publish and deploy container images for solutions

***

1. Terms

    - **Azure Container Registry**: is a managed, private Docker registry service based on the open-source Docker Registry 2.0. Create and maintain Azure container registries to store and manage your private Docker container images and related artifacts.

    - **Azure Container Regisitry Tasks**: Build and push a single container image to a container registry on-demand, in Azure, without needing a local Docker Engine installation. Think `docker build`, `docker push` in the cloud.

    - **Azure Container Instances**: offers the fastest and simplest way to run a container in Azure, without having to manage any virtual machines and without having to adopt a higher-level service.

2. Create and publish image locally

    Create new image:

    ``` bash
    docker build <path_to_Dockerfile> <name_of_image>
    ```

    Login in Azure Container Registry:

    ``` bash
    # Azure CLI
    az acr login --name <acr_name>
    ```

    Before publishing the image, we need to tag it:

     - Query Login Server of the ACR:

    ``` bash
    # Azure CLI
    az acr show --name <acr_name> --query loginServer --output table

    # Output
    Result
    ------------------------
    <acr_name>.azurecr.io
    ```

    - Tag the image with prefix is the login server:

    ``` bash
    docker tag <name_of_image> <loginServer>/<name_of_image>:<version>

    # Example
    docker tag myimage myacr.azurecr.io/myimage:v1
    ```

    Publish the image to Azure Container Registry:

    ``` bash
    docker push <loginServer>/<name_of_image>:<version> 
    ```

    You then can list images in Azure Container Registry:

    ``` bash
    # Azure CLI
    az acr repository list --name <acr_name> --output table

    # Output
    Result
    ----------------
    myimage 
    ```

    To see the tags for a specific image:

    ``` bash
    az acr repository show-tags --name <acr_name> --repository <name_of_image> --output table
    ```

3. Create and publish container image with ACR Tasks

    ACR Tasks will build our image in Azure and then publish it to ACR for us with one command:

    ``` bash
    # Azure CLI
    az acr build --registry <acr_name> --image <name_of_image>:<version> <path_to_Dockerfile>
    ```

4. Deploy image to Azure Container Instance

    Configure registry authentication with service principal:

    ``` bash
    ACR_NAME=<acr_name>

    # SERVICE_PRINCIPAL_NAME: Must be unique within AD tenant
    SERVICE_PRINCIPAL_NAME=<acr_name>-service-principal

    # Obtain the full registry ID
    ACR_REGISTRY_ID=$(az acr show --name $ACR_NAME --query id --output tsv)

    # Create the service principal with rights scoped to the registry.
    # Default permissions are for docker pull access. Modify the '--role'
    # argument value as desired:
    # acrpull:     pull only
    # acrpush:     push and pull
    # owner:       push, pull, and assign roles
    SP_PASSWD=$(az ad sp create-for-rbac --name http://$SERVICE_PRINCIPAL_NAME --scopes $ACR_REGISTRY_ID --role acrpull --query password --output tsv)

    SP_APP_ID=$(az ad sp show --id http://$SERVICE_PRINCIPAL_NAME --query appId --output tsv)
    ```

    Deploy image:

    ``` bash
    az container create `
      --resource-group <myResourceGroup> `
      --name <container_name> `
      --image <loginServer>/<name_of_image>:<version> `
      -- cpu 1 --memory 1 `
      --registry-login-server <loginServer> `
      --registry-username $SP_APP_ID `
      --registry--password $SP_PASSWD `
      --dns-name-label <dns_name> `
      --ports 80
    ```

    Verify deployment progress:

    ``` bash
    az container show `
      --resource-group <myResourceGroup> `
      --name <container_name> `
      --query instanceView.state
    ```

### Create Azure App Service Web Apps

***

**Azure App Service**: fully managed web application hosting platform. This platform as a service (PaaS) offered by Azure allows you to focus on designing and building your app while Azure takes care of the infrastructure to run and scale your applications.

**Limitation with Linux**:

- App Service on Linux is not supported on Shared pricing tier.
- You can't mix Windows and Linux apps in the same App Service plan.
- Within the same resource group, you can't mix Windows and Linux apps in the same region.

**Work with Azure App Service**:

1. Create a web app with Azure CLI

    ``` bash
    # Create a App Service Plan
    az appservice plan create --name <my_plan> --sku <SKU>

    # Create a Web App
    az webapp create --name <my_web_name> --plan <my_plan> --run-time <RUN_TIME>
    ```

2. Deploy code to web app

    - Use ZIP or WAR file deployment: We can use use the broswer to upload Zip file (for Windows), or Azure CLI, or REST APIs, Powershell.

    ``` bash
    # Azure CLI
    az webapp deployment source config-zip --name <my_web_name> --src <path_to_zip_file>
    ```

    - What happens to my app during deployment?

        All the officially supported deployment methods make changes to the files in the `/home/site/wwwroot` folder of your app.

        Thus, the deployment can failed because of locked files. Or web app might behave unpredictably during the deployment because not all the files updated at the same time.

    - How can prevent?

      - [Run your app from the ZIP package directly without unpacking it.](https://docs.microsoft.com/en-us/azure/app-service/deploy-run-package)
      - Stop your app or enable offline mode for your app during deployment.
      - Deploy to a staging slot with auto swap enabled.

3. Run from package: **Require the app restart**.

    - To enable: Set `WEBSITE_RUN_FROM_PACKAGE` to `1`. Then when we use Zip deployment, it will not unpack all files to `wwwroot` anymore. Instead, it mounts the zip file as read-only directory in `wwwroot`.

    - We can run from the package at external URL by set `WEBSITE_RUN_FROM_PACKAGE` to the `External url` instead.

4. Enable diagnostics logging

    Azure provides built-in diagnostics to assist with debugging an App Service App. There are 5 types of built-in logging and tracing.

    - Application logging: Logs message generated by application code.
    - Web server logging: Logs HTTP request data.
    - Detailed Error Messages
    - Failed request tracing
    - Deployment logging

    To enable application log on Windows:
    - We can use either **File System** or **Blob Storage** option to store logs.
    - However, **File System** is for temporary debugging purposes, and turns itself off in 12 hours.
    - The **Blob** option is for long-term logging. But we can only use storage accounts in the same region as the App Service.
    - We can set also the **Retention Period (Days)** for the logs.

    Stream logs:

    ``` bash
    az webapp log tail --name <appname> --resource-group <myResourceGroup>
    ```

    Alternatives to app diagnostics:
    - Azure Application Insights is a site extension that provides additional performance monitoring features, such as detailed usage and performance data, and is designed for production app deployments as well as being a potentially useful development too.

    - You can also view **Metrics** for your app. You can view CPU, memory, network, and file system usage, and set up alerts when a counter hits a particular threshold.

5. Move an app to another App Service Plan
    - We can move an app to another AppService Plan as long as that plan is in the same resource group and the same geographical region with the current plan.

    > For more information, Azure deploys each App Service Plan to a deployment unit, called a **webspace**. Each region can have many webspaces, but your app can only move between plans that are created in the same webspace. All plans created with the same resource group and region combination are deployed into the same webspace.

    > If you're moving an app from a higher-tiered plan to a lower-tiered plan, such as from D1 to F1, the app may lose certain capabilities in the target plan. For example, if your app uses TLS/SSL certificates.

### Implement Azure Function

***

1. Triggers and bindings concepts

    - **Trigger** defines how a function is invoked and a function must have exactly one trigger. Triggers have assciated data, which is often provided as the payload for the function.
    - **Binding** to a function is a way of declaratively connecting another resource to the function. Bindings may be connected as *output binding*, *input binding*, or both. Data from bindings is provided to the function as parameters. Bindings are optional, a function might have one, or multiple bindings.

2. Handle Azure Functions binding errors

    - Use structured error handling: The top-most level of any function code should include a try/catch block. In the catch block, you can capture and log errors.

    - Retry policies (preview):  

        - A retry policy is evaluated whenever an execution results in an uncaught exception. As a best practice, you should catch all exceptions in your code and rethrow any errors that should result in a retry.

        - Retry policy can be configured on the Host level or Function level.

        - The current retry count is stored in memory of the instance. It's possible that an instance has a failure between retry attempts, and the retry count is lost.

        - Using retry support on top of trigger resilience. For instance, if you used the default Service Bus delivery count of 10, and defined a function retry policy of 5. If the excution of one message failed, the function will retry 5 times before it requeues the message from Queue, and increments the delivery count to 2.

3. Durable Function

    Durable Functions is an extension of Azure Functions that lets you write stateful functions in a serverless compute environment.

    Orchestrator functions use event sourcing to ensure reliable execution and to maintain local variable state. The [replay behavior](https://docs.microsoft.com/en-us/azure/azure-functions/durable/durable-functions-orchestrations?tabs=csharp#reliability) of orchestrator code creates constraints on the type of code that you can write in an orchestrator function.

    **Constraint**: Orchestrator functions must be deterministic: an orchestrator function will be replayed multiple times, and it must produce the same result each time.

    - For instance, `DateTime.Now` in C# is not deterministic and cannot be used in orchsestrators.
    - Bindings are also nondeterministic.
    - [More constraints could be found here](https://docs.microsoft.com/en-us/azure/azure-functions/durable/durable-functions-code-constraints).

    **Versoning**: Any code updates made to Durable Functions apps that affect unfinished orchestrations might break the orchestrations' replay behavior.

    **Common patterns**:

    - Function chaining:

        In the function chaining pattern, a sequence of functions executes in a specific order. In this pattern, the output of one function is applied to the input of another function.

        ![image](https://docs.microsoft.com/en-us/azure/azure-functions/durable/media/durable-functions-concepts/function-chaining.png)

    - Fan out/fan in:

        In the fan out/fan in pattern, you execute multiple functions in parallel and then wait for all functions to finish. Often, some aggregation work is done on the results that are returned from the functions.

        ![image](https://docs.microsoft.com/en-us/azure/azure-functions/durable/media/durable-functions-concepts/fan-out-fan-in.png)

    - Async HTTP APIs:

        The async HTTP API pattern addresses the problem of coordinating the state of long-running operations with external clients. A common way to implement this pattern is by having an HTTP endpoint trigger the long-running action. Then, redirect the client to a status endpoint that the client polls to learn when the operation is finished.

        ![image](https://docs.microsoft.com/en-us/azure/azure-functions/durable/media/durable-functions-concepts/async-http-api.png)

    - Monitor:

        You can use Durable Functions to create flexible recurrence intervals, manage task lifetimes, and create multiple monitor processes from a single orchestration.

        ![image](https://docs.microsoft.com/en-us/azure/azure-functions/durable/media/durable-functions-concepts/monitor.png)

    - Human interaction:

        You can implement the pattern in this example by using an orchestrator function. The orchestrator uses a durable timer to request approval. The orchestrator escalates if timeout occurs. The orchestrator waits for an external event, such as a notification that's generated by a human interaction.

        ![image](https://docs.microsoft.com/en-us/azure/azure-functions/durable/media/durable-functions-concepts/approval.png)

    - Aggregator (stateful entities):

        **In** this pattern, the data being aggregated may come from multiple sources, may be delivered in batches, or may be scattered over long-periods of time. The aggregator might need to take action on event data as it arrives, and external clients may need to query the aggregated data.

        ![image](https://docs.microsoft.com/en-us/azure/azure-functions/durable/media/durable-functions-concepts/aggregator.png)


## Develop Azure Storage (10-15%)

### Develop solutions that use Cosmos DB storage

***

1. Select the appropriate API for your solution

    | | Core (SQL) | MongoDB | Cassandra  | Azure Table | Gremlin |
    | --- | :---: | :---: | :---: | :---: | :---: |
    | New project being created from scratch | x | | | | |
    | Existing MongoDB, Cassandra, Azure Table, or Gremlin data | | x | x | x | x |
    | Analysis of the relationships between data | | | | | x |
    | All other scenarios | x | | | | |

    **Ask the question**: Are there existing databases or applications that use any of the supported APIs?

    - If there is, then you might want to consider using the current API with Azure Cosmos DB, as that choice will reduce your migration tasks, and make the best use of previous experience in your team.

    - If there isn't, then there are a few questions that you can ask in order to help you define the scenario where the database is going to be used:

        - Does the schema change a lot?

            A traditional document database is a good fit in these scenarios, making Core (SQL) a good choice.

        - Is there important data about the relationships between items in the database?

            Relationships that require metadata to be stored for them are best represented in a graph database.

        - Does the data consist of simple key-value pairs?

            Before Azure Cosmos DB existed, Redis or the Table API might have been a good fit for this kind of data; however, Core (SQL) API is now the better choice, as it offers a richer query experience, with improved indexing over the Table API.

2. Implement partitioning schemes

    Azure Cosmos DB uses partitioning to horizontal scale individual containers in a database to meet the performance needs of an application.

    **Logical partitions**:
    - A logical partition consists of a set of items that have the same partition key.
    - A logical partition also defines the scope of database transactions.
    - Each logical partition can store up to 20GB of data.

    **Physical partitions**:
    - Internally, one or more logical partitions are mapped to a single physical partition.
    - The number of physical partitions in a container depends on the following configuration:
        - The number of throughput provisioned (each individual physical partition can provide a throughput of up to 10,000 request units per second).
        - The total data storage (each individual physical partition can store up to 50GB data).
    - Throughput provisioned for a container is divided evenly among physical partitions. A partition key design that doesn't distribute requests evenly might result in too many requests directed to a small subset of partitions that become "hot".

    **Replica sets**:
    - Each physical partition consists of a set of replicas, also referred to as a replica set. A replica set makes the data stored within the physical partition durable, highly available, and consistent.
    - A single physical partition have at least **4** replicas.

    **Choosing a partition key**: Selecting a partition key in a container is a simple but important design decision. Once we select the partition key, it cannot be changed later. If we must to change it, we will have to move all the data to a new container with the new desire partition key. For all container, a partition key should:

    - Be a property that has value which does not change. If a property is set as partition key, we cannot update its value.
    - The property should have a wide range of possible values.
    - Appear frequently in the predicate clause of queries.
    - Spread request unit (RU) consumption and data storage evenly across all logical partitions.

3. Set the appropriate consistency level for operations

    Distributed databases that rely on replication for high availability, low latency, or both, must make a fundamental tradeoff between the read consistency, availability, latency, and throughput.

    Azure Cosmos DB offers five well-defined levels. From strongest to weakest, the levels are:
    - **Strong**: Users are always guaranteed to read the latest committed write.
    - **Bounded staleness**: 
      - The reads might return stale data by at most "K" versions (that is, "updates") of an item or by "T" time interval, whichever is reached first.
      - It still honors the consistent-prefix guarantee.
      - We can trade consistency for availability, latency and throughput or vice versa by adjusting the "K" or "T" variable.
    - **Session**: Within a single client session reads are guaranteed to honor the consistent-prefix, monotonic reads, monotonic writes, read-your-writes, and write-follows-reads guarantees.
    - **Consistent prefix**: Consistent prefix consistency level guarantees that reads never see out-of-order writes.
    - **Eventual**: There's no ordering guarantee for reads. In the absence of any further writes, the replicas eventually converge.

    Each level provides availability and performance tradeoffs. The following image shows the different consistency levels as a spectrum:

    ![image](https://docs.microsoft.com/en-us/azure/cosmos-db/media/consistency-levels/five-consistency-levels.png)

    > **Note**
    >
    > There are no write latency benefits on using strong consistency with multiple write regions because a write to any region must be replicated and committed to all configured regions within the account. This results in the same write latency as a single write region account.

4. Implement server-side programming including stored procedures, triggers, and change feed notifications

    **Stored procedure**: is a piece of application logic written in JavaScript that is registered and executed against a collection as a single transaction.

    **Trigger**: is a piece of application logic that can be executed before (pre-triggers) and after (post-triggers) creation, deletion, and replacement of a document. Triggers are written in JavaScript.

    **Change feed**: we can use it for sort of de-normalization and duplicating data to a different container with different partition key for optimizing query. 

### Develop solutions that use blob storage

***

1. Choose a tool and strategy for copying blobs

    **Azure CLI**:
    - Using `az storage blob copy command`.
    - This command runs asynchronously and uses the Azure Storage service to manage the copy process.
    - This means we don't have to download and upload blobs via local storage to migrate them between accounts.
    - We can put ETag and date condition option to specify which blobs will be copied from the source such as `--source-if-match`, `--source-if-modified-since`, etc or which blobs will be overwrite at the destination such as `--destination-if-match`, `--destination-if-modified-since`, etc.
    - **UseCase**: You want to quickly upload the data in a collection of small files held in a local folder to blob storage. This is a one-off request. You don't want to overwrite blobs that have been modified in the last two days.

    **AZCopy**:
    - The AzCopy utility was written specifically for transferring data into, out of, and between Azure Storage accounts.
    - A key strength of AzCopy over the Azure CLI is that they're recoverable. The AzCopy command tracks the progress of copy operations, and if an operation fails, it can be restarted close to the point of failure.
    - The AzCopy command lacks the ability to select blobs based on their modification dates.
    - AzCopy does provide comprehensive support for hierarchical containers and blob selection by pattern matching (two features not available with the Azure CLI).
    - **UseCase**: You want to transfer a series of large files to blob storage. It may take several hours to upload each file, and you're concerned that if a transfer fails it shouldn't have to restart from the beginning

    **.NET Storage Client library**:
    - The .NET Storage Client library requires a development investment, and may not be suitable for quick, one-off jobs.
    - If we have a complex task that is repeated often, and needs certain of customization and flexibility, then this investment could be worthwhile.
    - **UseCase**: You want to move a set of blobs in Azure storage from one storage account to another. You want to organize the blobs in the destination account in different folders, according to the month in which each blob was last updated. You'll be performing this task at regular intervals.

2. Manage container properties and metadata with .NET

    **About properties and metadata**:

    - **System properties**: System properties exist on each Blob storage resource. Some of them can be read or set, while others are read-only.
    - **User-defined metadata**: User-defined metadata consists of one or more name-value pairs that you specify for a Blob storage resource.

    **Retrieve container properties**:
    - GetProperties
    - GetPropertiesAsync

    **Set metadata**: You can specify metadata as one or more name-value pairs on a blob or container resource. To set metadata, add name-value pairs to an IDictionary object, and then call one of the following methods to write the values:

    - SetMetadata
    - SetMetadataAsync

3. Implement data archiving and retention

    **Store business-critical blob data with immutable storage**:
    - For the duration of the retention interval, blobs can be created and read, but cannot be modified or deleted.
    - Time-based retention policy support: Users can set policies to store data for a specified interval. After the retention period has expired, blobs can be deleted but not overwritten.
    - Legal hold policy support: If the retention interval is not known, users can set legal holds to store immutable data until the legal hold is cleared.
    - Container and storage account deletion are also not permitted if there are any blobs in a container that are protected by a legal hold or a locked time-based policy.
    - Locked time-based policies will protect against container deletion only if at least one blob exists within the container.

    **Access tiers for Azure Blob Storage - hot, cool, and archive**:
    - **Hot** - Optimized for storing data that is accessed frequently.
    - **Cool** - Optimized for storing data that is infrequently accessed and stored for at least 30 days. It can be used for Short-term backup and disaster recovery datasets. It still provides a need to be available immediately when accessed but the cost will higher than **Hot** tier.
    - **Archive** - Optimized for storing data that is rarely accessed and stored for at least 180 days with flexible latency requirements (on the order of hours). It can be used for Long-term backup, secondary backup, and archival datasets

    - Notes:
        - Only the hot and cool access tiers can be set at the account level. The archive access tier isn't available at the account level.
        - Hot, cool, and archive tiers can be set at the blob level during upload or after upload.
        - While a blob is in archive storage, the blob data is offline and can't be read, overwritten, or modified. To read or download a blob in archive, we must first rehydrate it to an online tier.

    **Rehydrate blob data from the archive tier**:
    - There are two options to retrieve and access data stored in the archive access tier.
        - Rehydrate an archived blob to an online tier - Rehydrate an archive blob to hot or cool by changing its tier using the Set Blob Tier operation.
        - Copy an archived blob to an online tier - Create a new copy of an archive blob by using the Copy Blob operation. Specify a different blob name and a destination tier of hot or cool.

    - The process of rehydration can take hours to complete which depends on rehydrate priority:
        - Standard priority: The rehydration request will be processed in the order it was received and may take up to 15 hours.
        - High priority: The rehydration request will be prioritized over Standard requests and may finish in under 1 hour for objects under ten GB in size.

## Implement Azure Security (15-20%)

TODO

## Monitor, troubleshoot, and optimize Azure solutions (10-15%)

TODO

## Connect to and consume Azure services and third-party services (25-30%)

TODO

## Additional Tips and Resources

TODO
