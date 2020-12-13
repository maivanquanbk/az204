# AZ-204 Revise Note

- [AZ-204 Revise Note](#az-204-revise-note)
  - [Develop Azure compute solution (25-30%)](#develop-azure-compute-solution-25-30)
    - [Implement IaaS Solutions](#implement-iaas-solutions)
      - [Provision VMs](#provision-vms)
      - [Configure VM for remote access](#configure-vm-for-remote-access)
      - [Manage the availability of your Azure VMs](#manage-the-availability-of-your-azure-vms)
      - [VMs data backup and recovery](#vms-data-backup-and-recovery)
      - [Azure Resource Manager templates](#azure-resource-manager-templates)
    - [Create Azure App Service Web Apps](#create-azure-app-service-web-apps)
    - [Implement Azure Function](#implement-azure-function)
  - [Develop Azure Storage (10-15%)](#develop-azure-storage-10-15)
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

### Create Azure App Service Web Apps

***
TODO

### Implement Azure Function

***
TODO

## Develop Azure Storage (10-15%)

TODO

## Implement Azure Security (15-20%)

TODO

## Monitor, troubleshoot, and optimize Azure solutions (10-15%)

TODO

## Connect to and consume Azure services and third-party services (25-30%)

TODO

## Additional Tips and Resources

TODO
