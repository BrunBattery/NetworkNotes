---
title: Azure Basics (AZ-900)
format:
  html:
    toc: true
    toc-depth: 5
    css: styles.css
editor_options:
  markdown:
    mode: gfm
---

#### Azure Acronyms
- **ACU** - Azure compute unit - way to compare CPU performance across different SKUs
- **SKU** - Stock keeping unit - describes particular type of purchasable product (VM SKU)
- **RBAC** - Rule-based access control - authorization control system in Azure
- **RG** - Resource groups - folders in Azure for organizing resources
- **ASG** - Application security group - group of like resources for use with NSG rules
- **ARM** - Azure resource manager - single management plane for all of Azure
- **ACI** - Azure container instance - Useful for small scale use of containers in Azure
- **AKS** - Azure kubernetes service - Used for container orchestration and management
- **ACR** - Azure kubernetes registry - Used to keep track of valid container images
- **UDR** - User-defined routes - made by modifying the route table(s) in Azure
- **NVA** - Network virtual appliance - Network VM such as Cisco CSR, Palo Alto VM-100
- **P2S** - Point to site - Method for connecting individual hosts to Azure via VPN
- **ER** - Express route - Private link connectivity to Azure through an ISP
- **NSG** - Network security group - Azure stateful ACL
- **CDN** - Content delivery network - places copies of cached data at edge nodes in many locations, allowing better latency and performance

---

#### Interacting with Azure

##### Access Methods
- **Azure Portal**
	- Web GUI for managing Azure
	- All aspects of Azure are possible to manage through the portal
- **Azure CLI**
	- Text-only interface for Azure with Bash
	- Allows for automation and greater efficiency than the portal
- **Azure PowerShell**
	- Also possible to interact with Azure through PowerShell
	- PowerShell is also useful for many other windows tasks, which makes skills with PowerShell more transferrable outside of Azure than the CLI or Portal
- **Cloud Shell**
	- Allows access to either a Bash (Azure CLI) or PowerShell terminal through the web or mobile app
	- Has dedicated storage, allowing persistent data between sessions
	- Accessed via the shell icon at the top right corner of the web portal
	- Sometimes bugs out and needs to be refreshed with `clear`
- **Azure Mobile App**
	- Another option for interaction with Azure - can access basic portal functions and the Azure cloud shell via Mobile

##### ARM & ARM Templates
- ARM stands for Azure Resource Manager
- All configuration in Azure is brokered through Azure resource manager control plane - the Azure portal, PowerShell, CLI, REST APIs all interact with Azure through ARM
- Ensures consistency between all forms of configuration by keeping control plane consistent
- All Azure resources actually created with JSON ARM templates which are visible in Azure under 'Automation > Export Template'
- Templates are JSON files that define infrastructure configurations for Azure resources
- Idempotent, declarative and reusable
- Very useful for automation of Azure
- Key sections for ARM templates:
	- **Schema** - Defines ARM template version
	- **Content Version** - Optional, for declaring version of that specific template
	- **Parameters** - Allows parameterization of template
	- **Variables** - Defines reusable variables in the template
	- **Resources** - Specifies Azure resource being deployed/modified, including type, name, location and properties
	- **Outputs** - Defines what returns after ARM template completes, such as resource IDs or strings
- Example ARM template:

```
					{
			  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
			  "contentVersion": "1.0.0.0",
			  "parameters": {
			    "storageAccountType": {
			      "type": "string",
			      "defaultValue": "Standard_LRS",
			      "allowedValues": [
			        "Standard_LRS",
			        "Standard_GRS",
			        "Standard_ZRS"
			      ],
			      "metadata": {
			        "description": "Storage Account type"
			      }
			    }
			  },
			  "variables": {
			    "storageAccountName": "[concat('storage', uniqueString(resourceGroup().id))]"
			  },
			  "resources": [
			    {
			      "type": "Microsoft.Storage/storageAccounts",
			      "apiVersion": "2019-06-01",
			      "name": "[variables('storageAccountName')]",
			      "location": "[resourceGroup().location]",
			      "sku": {
			        "name": "[parameters('storageAccountType')]"
			      },
			      "kind": "StorageV2",
			      "properties": {}
			    }
			  ],
			  "outputs": {
			    "storageAccountName": {
			      "type": "string",
			      "value": "[variables('storageAccountName')]"
			    }
			  }
			}
```

#### Cloud Computing Fundamentals

##### Cloud Computing Vocabulary
- **High Availability** - Cloud computing allows near-instant failover, as you can abstract the applications from the hardware much more than in traditional deployments
- **Reliability/Fault Tolerance/Disaster Recovery** - Resiliency through deployment in multiple locations/regions and avoiding single points of failure
	- Azure is auto-healing - if a node or disk fails, services will migrate
	- Azure always stores 3 copies of any data
	- Azure allows auto-scale to scale capacity
	- When considering reliability, you must design for failure scenarios
		- Designs considering HA and DR should utilize multiple regions, availability zones
- **Scalability** - The ability of servers/services to increase/decrease resources as needed - with Azure you can set up auto-scaling to accomplish this for you
	- Horizontal scaling - 'scaling out', adding additional VMs for increased demands
	- Vertical scaling - 'scaling up', increasing resources (CPU, RAM) on a particular host
- **Predictability** - With cloud deployments predictability can be ensured (via scaling, HA, load balancing etc) to provide a consistent user experience - costs are also predictable thanks to cost tracking analytics and resource usage forecasting
	- Azure has specific SKUs, ACUs that will always perform the same way
	- APIs and templates are consistent between regions
	- Using templates and automation can assist in keeping things predictable
- **Security** - Cloud security is maintained through typical proper security best-practices like regular patching, account management, MFA, etc, however some of this can be offloaded to Microsoft with Azure and Azure Advisor will recommend best-practices as well
- **Governance** - Standardizing your environments, meeting regulatory requirements and maintaining compliance with your policies
- **Manageability** - Refers to both direct management of cloud resources through the portal, CLI, APIs etc and also monitoring and templating your deployments

##### CapEx vs OpEx
- **CapEx** is capital expenditure - paying upfront for hardware
	- One key factor with CapEx is that you need to build out the hardware to account for peak capacity whereas OpEx can scale to load
- **OpEx** is operational expenditure - you pay for what you use
	- Cloud is consumption-based and OpEx

##### IAAS, PAAS & SAAS
- IAAS - infrastructure as a service - VM in the cloud would be an example
	- Azure handles compute, network and storage in their DCs
	- Most flexible as you have access to the OS to change whatever you need
- PAAS - platform as a service - app services, serverless functions and logic apps
- SAAS - software as a service - O365 - pay purely for the application

##### Private, Public & Hybrid Clouds
- **Private cloud**
	- Owned by the enterprise itself
	- CapEx, accessed locally
- **Public cloud**
	- Azure, AWS, GCP
	- OpEx, accessed over the internet, many regions
- **Hybrid cloud**
	- Combination of both public and private cloud
	- Can use Azure Arc for management with Azure platform in private cloud

#### Azure Architecture, Containers & Resources

##### Regions and Availability Zones
- **Regions & Region Pairs**
	- Regions defined by 2ms round-trip latency (e.g. West US, East US, UK South, etc)
	- Choosing regions close to intended users ensures lower latency, better performance
	- Data sovereignty requirements for certain organizations/governments may mean data has to say in certain regions
	- Using multiple regions for DR is a good idea to avoid the possibilities of natural disasters or failures affecting all services
	- Azure has region pairs for replication that are located within the same regional boundaries (e.g. West US, East US & Canada Central, Canada East)
	- Azure will not deploy updates to both members of a region pair at the same time to avoid causing issues in both locations
- **Availability Zones**
	- Availability zones allow redundancy within regions
	- The different availability zones protect against datacenter-level problems affecting services within a region - can choose from 3 different availability zones in a subscription
	- Zonal assets like VMs allow you to choose availability zone
	- Zone redundant assets (usually Azure native services) deploy to all 3 zones by default

##### Resource Groups
- Function like folders for organizing resources in Azure
- Resources all must live in one and only one resource group
- Can have many different resources from different regions in one resource group
- Usually best to group resources based on usage - i.e. all resources for one firewall (VM, disk, public IP, NIC, etc) in one resource group
- Can apply RBAC, policies, budgets based on resource groups

##### Virtual Machines
- Virtualized machine, example of IAAS
- VMs are their own resource in Azure but require other resources to function
- Other resources are data disks, OS disks, vNICs, NSGs, Public IPs, vNETs, etc
- Priced per hour and become more expensive the more CPU/RAM allocated to it

##### Scale Sets
- A pool of identical, load balanced VMs, used for auto-scaling horizontally
- Can be run at a very large scale of potentially thousands of VMs

##### Containers
- Operate like lightweight virtual machines that share the OS kernel but have their own file systems and processes. Highly efficient compared to VMs as the resources used for the OS only required once for multiple containers - also start up quickly which allows them to be effective for dynamic scaling. They also run in a consistent fashion across multiple cloud vendors and on-prem clouds. They can be implemented at a small scale in Azure with ACI (Azure Container Instance), or, if more orchestration is required with AKS (Azure Kubernetes Service).

##### App Services
- PAAS offering used to build web and mobile apps and APIs directly in the cloud without the need for containers or VMs. Infrastructure management is abstracted and managed by Azure. Highly scalable and provides built-in HA and load balancing.
- Can be connected into a VNET via networking tab, requiring a subnet within the VNET in the same region as the app service
- If connecting to a different region, connecting to the VNET requires P2S VPN gateway integration
- **Web Apps** - Website and online apps hosted by Azure - auto-scaling and load balancing
- **Web Apps for Containers** - Similar to Web Apps above but containerized and portable between different environments
- **API Apps** - Used to allow API to your backend for programmatic interfacing

##### Azure Functions/Serverless
- Another infrastructure-abstracted method of running code in Azure, triggering based on certain events or triggers. Azure functions require you to write the code, whereas Azure logicapps are drag and drop and can be done by anyone without coding experience.

#### Azure Storage

##### Storage Accounts
- Different storage service types are queue, table, files and container blobs
- Unique Azure namespace with its own web address
- `companyname.<storagetype>.core.windows.net`, for example
- Exists in particular regions - care should be taken to make sure it exists in the same region as resources using it
- **Performance**
    - Standard - always general purpose V2
    - Premium - options are block blob, page blob and files
        - Block blobs are for unstructured data
        - Page blobs are for VHDs of Azure VMs
        - Azure files is a managed file share service accessed over SMB or NFS
- **Redundancy**
	- All data in Azure has minimum of 3 copies regardless of redundancy tier
    - LRS - Locally Redundant Storage - basic protection against rack failures
    - ZRS - Zone Redundant Storage - protects against DC failures
    - GRS - Geo Redundant Storage - protects against region failures
    - GZRS - Combines GRS & ZRS - protects against all failure scenarios
    - Premium storage types can only use LRS, ZRS

##### Blob Storage
- 'Binary Large Object' - basically any type of files you want
- Block blobs - Store unstructured data, binary files up to 4.7TB
- Append blobs - Optimized for 'append' operations - ideal for logging
- Page blobs - Mainly used for VHDs, can be up to 8TB in size
- 3 different pricing tiers - Hot (for frequently accessed files), Cold (Lower storage cost but higher access times), Archive (Very low cost and high access times)
- Can automate moving blobs from hot to cold, archive or delete based on file age/last modified date

##### Disk storage
- Comes in 4 types: HDD, Standard SSD, Premium SSD and Ultra Disk
- Azure ensures data is redundant, backed up and always accessible

##### File Transfers to Azure
- **AzCopy** - Command-line utility for copying files to Azure
- `azcopy cp "filename.mp4" "https://mystorage.blob.core.windows.net/contain`
- **Storage Explorer** - GUI utility that allows drag & drop like Windows explorer
- **Azure File Sync** - Syncs SMB fileshares between Azure & on-prem automatically
- **Azure Data Box** - Offline data transfer method for large file transfers to Azure, an encrypted data storage solution shipped to your location - files added and moved to Azure once device is shipped back
- **Azure Migrate** - helps to assess and migrate on-prem resources into Azure - will help with compatibility, sizing, estimating cost and migration of resources

#### Azure IAM & Management

##### Entra ID
- IAM (Identity & Access Management) service for Azure that provides an identity repository, including users, groups, policies and security features
- Tenant is a dedicated instance of Entra ID representing an organization, created when signing up for Azure
- Each tenant is distinct - users belong to only one tenant but can be guests on other tenants
- Subscriptions are billing entities that use one specific Entra ID tenant - tenants can have multiple subscriptions under them
- Can set budget, RBAC and policy at subscription level - this will be inherited by RGs within the subscription or overridden at the RG level
- Subscriptions can be separated for different environments (prod & test) or budget control for different cost centers
- Groups can either be assigned manually, dynamically assigned based on user attributes (for example 'Department:HR' assigning a user to an HR group) or dynamic based on device
- Similar to groups, administrative units can be manually or dynamically assigned and are a way of scoping management responsibilities of administrators within a subscription

##### Azure ADDS
- Fully managed PAAS solution for active directory in Azure
- Useful for migration of legacy on-prem applications to cloud that don't support modern entra-id authentication methods (OAuth 2.0)
- Support traditional ADDS features like group policy, LDAP, NTLM and Kerberos
- Needs to act as a completely separate domain, cannot extend existing AD forest to Azure ADDS
- As an alternative, you can host your AD DS in a virtual machine in Azure instead and extend your domain into Azure that way

##### Management Groups
- Allows setting budget, RBAC and policy at a higher level than subscriptions
- Can choose particular management groups for certain subscriptions, RBAC, policies and budget will be inherited by subscriptions and RGs in subscriptions
- Root is always at the top - can create up to 6 levels below this

##### Conditional Access
- Premium feature in Entra ID that uses if/then logic to allow access to defined applications
- Often paired with MFA
- Conditions selected from a policy to determine access, including
	- User/group
	- Application
	- Location (IP)
	- Approved device
	- MFA requirements

##### Azure Governance
- Allows you to control what users can and cannot do through defined policies in Azure, preventing creation/modification of resources that don't comply with the polices
- RBAC (Role-Based Access Control) allows you define user's access to resources (allow access to specific resource groups, services with read or read/write permissions)
- Locks prevent the modification or deletion of subscriptions or resources
- Locks can be ReadOnly locks or CanNotDelete locks and are inherited by lower resources (e.g. applying a lock to a resource group would apply to all resources in group)
- Blueprints are templates for creating standard Azure environments from scratch

##### Azure Policy
- Policy assignment allows enforcing policies at different levels of Azure deployment (management group, subscription, resource group, resource)
- Policy definitions define the evaluation criteria and whether they should be allowed or denied
- Initiative definitions are collections of policies with a specific goal

##### Tags
- Consist of a name and value pair
- Up to 50 tags can be assigned to a resource
- Tags are not inherited from higher resources

##### Azure Bastion
- PaaS Jumphost solution in Azure
- Connect to Bastion host via HTML5 browser, connection secured with TLS
- Allows you to access VMs without allowing any access through NSGs/adding any Public IPs

##### Passwordless Auth
- Replacing password prompt with alternative auth
- MFA from authenticator app is one option
- Another is a hardware key like a Yubikey

##### SSPR
- Self-Service Password Reset - allows users to reset their own Azure passwords
- Can use email, mobile app, mobile auth, phone numbers as reset methods
- By default, administrators will require at least two methods to reset their passwords for security reasons

#### Azure Monitoring & Pricing

##### Azure Monitor
- Dashboard in Azure that gathers telemetry from Azure resources
- Log Analytics stores and queries Azure monitor data (using KQL queries)
- Application insights provides performance insights for web apps
- Monitor alerts notifies you of unexpected events (such as VM unresponsive, high app latency, VM using excessive CPU, etc)

##### Azure Service Health
- Dashboard in the Azure portal that notifies you of any service issues with the Azure platform, what regions the issues are in, and can be set up to alert you of such issues

##### Azure Pricing
- **Azure Cost Management**
	- Dashboard in Azure for management of costs
	- Will offer recommendations to improve your cost
- **Pricing Factors**
	- Resource sizing, resource types, location, and bandwidth in/out of Azure all influence your pricing in Azure
	- To help predict your prices, you can use the Azure pricing calculator
- **Reserved Instances/Capacity**
	- You can reserve instances or capacity in Azure to pay less money by committing to 1 to 3 years instead of monthly usage

#### Azure DR, Backups & Update Management

- **Device Registration**
	- Can register devices with Azure Entra ID tenant - can manage these devices with Microsoft Intune as well

- **Update Management**
	- Azure can evaluate VM compliance for updates on Windows & Linux machines and schedule deployments and reboots for these updates

- **Desired State Configuration (DSC)**
	- Can be used to enforce desired state on VMs through automation
	- VMs are added to the DSC, which will then enforce configuration based on declared desired state via Powershell scripts

- **Azure Backups**
	- Native feature in Azure that allows you to create automated backups of VMs, On-premise machines, SQL servers and restore from backups if required

- **Azure Site Recovery**
	- Allows recovery of Azure resources from one region to another in case of a disaster
	- Can be configured to fail over many resources at once


#### Azure Database Fundamentals

- **Azure SQL**
	- Database-as-a-service - hardware & infrastructure managed by Azure

- **Azure Database for MySQL, PostGreSQL**
	- Open-source MySQL & PostGreSQL implementation in Azure
	- PAAS - service infrastructure managed by Azure
	- PostGres most commonly used in finance and manufacturing, as it offers automated failover functionality

- **Database Migration Services**
	- Azure tool to migrate Microsoft SQL to Azure SQL

- **Cosmos DB**
	- Azure DB that allows global access to database with low latency
	- Allows automatic scaling
	- Supports SQL, MongoDB and Cassandra
	- Downside is it can be very expensive

#### Other Azure Features/Services

- **Azure Marketplace**
	- Microsoft and other partner offerings in Azure, such as VMs, templates and services
	- 3rd party integrations into Azure to make it easier to use their tools (PA VMs, CSRs, etc)

- **Azure Advisor**
	- Azure function that provides recommendations to improve cost, security and availability/reliability
	- It's a good idea to follow the advisor's recommendations unless you have specific reasons not to

- **Microsoft Defender for Cloud**
	- A separate portal in Azure that provides threat alerts, policy and compliance metrics and a security score

- **Azure DDOS Protection**
	- Azure has a default DDOS protection service for all Azure resources exposed to the public internet

- **External Guest Access**
	- Can create a new account for an external user, or invite user as guest to Azure tenant

- **ARC**
	- Allows bringing Azure ARM control plane to other clouds or on-prem clouds

- **Azure Virtual Desktop**
	- Virtualized version of Windows running 100% in the cloud

- **Azure Sentinel**
	- Azure-native SIEM