* VMs are part of Infrastructure as a Service (IaaS)

# 1. Deploy workloads on Azure virtual machines
* VM Depot - collection of VMs images from community

## Creating a VM
* You can upload a VM that you have build on-premises (installed from customer's in-house server and computing infrastructure) or instantiate pre-built images available in Marketplace
* You can create clusters of VMs from the Marketplace

# 2. Create and manage a VM image or virtual hard disk
* VHDs are file format that running VM is using to store it's data
* These VHDs needs to be packed into disks to be attached to VM
* When you create a VM you can create an image that captures all details about VM, it's disks, VHDs and creates a template to use for new VM instances
* Specialized instance of VM is a golden copy
* Generalized instance is more like a template - every instance has got eg it's own machine identity
* VM consist of a set of VHDs - operating system disk and zero or more data disks. Each of those disk VHDs are created from VHD image
	
## Uploading VHDs to Azure 
```
 Add-AzureAccount
 Set-AzureSubscription <SubscriptionName> -CurrentStorageAccount <StorageAccountName>
 Add-AzureVhd -Destination <Destination> -LocalFilePath <LocalFilePathToVhd>
```

## Creating a disk
* Only available in existing portal
```
 Add-AzureDisk -DiskName <DiskName> -MediaLocation <URLToVhd> -Label <Label> -OS <OsName> (for data disk without -OS)
```

## Creating a VM using existing disks
* Only available in existing portal

## Attaching data disks to a VM
* Both existing and preview portal
``` 
Get-AzureVM <CloudServiceName> -Name <VmName> | Add-AzureDiskData -Import -DiskName <DiskName> -LUN <Identifier> | Update-AzureVM
```

* Generalizing Windows - sysprep
* Generalizing Linux - VM agent needs to be installed. waagent -deprovision

## Creating or capturing a VM image
* If VM contains only a single VHD, the operating system disk, you can create image from uploaded disk from the existing portal 
* You can capture VM image from VM in existing portal but make sure it was generalized before and shut down
```
Save-AzureVMImage -ServiceName <ServiceName> -Name <VmName> -OSState <GeneralizedOrSpecialized> -NewImageName <NewImageName> -NewImageLabel <NewImageLabel> - after this the target VM is deleted.
```

## Instantiating a VM instance from a VM image
* Can be done from existing portal using My Images

```
New-AzureQuickVM -Windows (or Linux) -Location <LocationName> -ServiceName <ServiceName> -Name <Name> -InstanceSize <InstanceSize> -ImageName <ImageName> -AdminUsername <Username> -Password <Password> -WaitForBoot
```

* To copy image from one location (storage account) to different one use AzCopy:
```
AzCopy /Source: <Source> /Destination:<Destination> /Sourcekey:<SourceStorageKey> /Destkey:<DestinationStorageKey> /Pattern:<ImageFileName>
```

# 3. Perform configuration management
* VM Agent is lightweight process for bootstrapping additional tools on the VM by way of installing, configuring and managing VM extensions
* Popular extensions:
	* DSC
	* Custom Script Extensions
	* Visual Studio Release Manager
	* Octopus Deploy
	* Docker extension
	* Puppet Enterprise agent
	* Chef client

## Custom Script Extension
* Allows to automatically run PowerShell script when VM starts
* Available in both portals
* It cannot ensure to match configuration with desired state

## DSC
* Implemented using PowerShell
* Used for configuring server/set of servers declaratively proving the description of desired state
* Allows to self-provision VM during deployment to desired state and automatically update when there is a configuration drift
* DSC script can describe following intentions:
	* Manage server roles and Windows features
	* Manage registry keys
	* Copy files and folders
	* Deploy software
	* Run PowerShell scripts
	* Manage users
* Custom resources are extensions for DSC. Includes a MOF schema, script module and module manifest
* On each target node runs Local Configuration Manager that:
	* Push configuration to bootstrap target module
	* Push configuration from a specified location to bootstrap or update a target node
	* Apply configuration defined in MOF file to target node either during bootstrapping or configuration drift detection
* To configure VM using DSC you need to first create a PowerShell file with desired state configuration and then:
	* Create a configuration script including collection of resources, collection of nodes in it
	* Prepare configuration package using PowerShell command: Publish-AzureVMDSCConfiguration <FileName> -ConfigurationArchivePath <File> - package will not contain any dependencies required for script to run

## Remote Debugging
* Installs remote debugging extensions

# 4. Configure VM networking
* DNS name during VM provisioning can be used to access the VM - resolves to public Virtual IP address
* DNS cannot be changed after VM provisioning

## Configuring endpoints with instance-level public IP addresses
* Public endpoints use port forwarding to expose single port on the VIP and map that port into private IP and port available on single VM
* If you want to have all ports open, you need to create PIP (Public IP Address) on instance-level
* Advantages of having PIP are:
	* You can use passive FTP that rely on choosing port
	* You can rely on PIP to identify your VM on outgoing requests to external services that have ACL or firewalls to add exceptions
* PIP requires VM to be in VNET (existing portal only)
* PIP can be configured only in preview portal
* PIP is assigned to the Cloud Service to which VM belongs, not to VM itself
* When all VMs in this Cloud Service are stopped or de-allocated PIP address is released
* Reserved IP address is fixed IP address which is not released
* To use a reserved IP address it must be requested during creating both Cloud Service and VM

## Access Control List
* Allows to restrict access to VM to specific ranges of IP addresses by defining a list of permit or deny rules
* ACL are not applied on internal traffic and cannot be applied to a VNET or to a subnet within a VNET
* Without ACL all traffic is allowed
* ACL is applied in priority order - lowest order, highest priority. First rule to match is applied and no evaluated later
* You can restrict access to a single host instead of range with specifying remote subnet /32

## Load Balancing Endpoints
* Available only in Standard mode
* Azure Load Balancing service - all VMs listening on the same port, distribute randomly traffic from the internet (external)
* Azure Internal Load Balancing service - VMs not exposed to public, distribute randomly internal traffic
* Requests from the same IP/source port goes to the same destination IP/destination port 
* Load balancer polls VMs to validate its availability
* To create load balancer set for public endpoints in existing portal (preview allows public and private balancer):
	* Create load balancer set as a part of creating first load balanced endpoint and for each next VM add new endpoint and add it to balancer set
	* Configure HTTP or TCP health probes to be used by LB to determine the availability of a VM

## Direct Server Return
* Allows VM to reply directly to the customer skipping the load balancer while sending a response

## Keep-alives
* Keep-alives are intended to keep the TCP connection with VM open even of absence of communication
* It accomplish this by sending periodically keep-alive package from the client to server application
* Intention was to ensure that Load Balancer will not close connection prematurely
	
## Idle timeout for an endpoint
* Only during endpoint creation and using PowerShell
```
Get-AzureVM -ServiceName <ServiceName> -name <VmName> | 
Add-AzureEndpoint -Name <Name> -Protocol <Protocol> -PublicPort <PublicPort> -LocalPort <LocalPort> -IdleTimeoutInMinutes <Timeout> |
Update-AzureVM
```

## Idle timeout for a load balanced endpoint set
* Only after creation and using PowerShell:
```
 Set-AzureLoadBalancedEndpoint -ServiceName <ServiceName> -LBSetName <SetName> -Protocol <Protocol> -LocalPort <LocalPort> -ProbeProtocolTCP -ProbePort <Port> -IdleTimeoutInMinutes <Timeout>
```

## Name resolution within a cloud service
* VM can access other VM in cloud service using it's hostname. It can be changed manually using native system mechanism

# 5. Scale VM
* Unlike Websites, VMs needs to be pre-provisioned in order for auto-scale
* You can scale VM size up and down by adjusting:
	* The number of VHD disks attached (total IOPS capacity)
	* Size of local temp disk
	* Number of CPU cores
	* Amount of RAM memory available
	* Network performance
```
Get-AzureVM -ServiceName <ServiceName> -Name <Name> | Set-AzureVM -InstanceSize <InstanceSize> | Update-AzureVM
```

## Availability sets
* Allow you to identify group of VMs that should never be brought down simultaneously during updates and should be physically separated
* For multi-tier application it's best to place all VMs belonging to one tier into separate availability set and to have availability for each tier

* In PowerShell you can create availability set only during provisioning of VM:
```
New-AzureVMConfig -Name <Name> -InstanceSize <InstanceSize> -ImageName <ImageName> -AvailabilitySetName <SetName> |
Add-AzureProvisioningConfig -Windows -Password <Password> -AdminUsername <UserName> |
New-AzureVM -ServiceName <ServiceName>
	
Start-AzureVM -ServiceName <ServiceName> -Name <Name>
```
	
* To change availability set of existing VM:
```
Get-AzureVM -ServiceName <ServiceName> -Name <Name> |
Set-AvailabiltySet -AvailabilitySetName <SetName> |
Update-AzureVM
```

## Update domains
* Constraints Azure how to perform update on underlying host machine that is running your VM
* 5 update domains in Azure where updates are performed in round-robin process
* Azure will never bring more than one update domain at the time

## Fault domains
* Fault domains concentrate on isolation in terms of power and network
* VMs will never share the same power source or be connected to the same switch - they will never be on the same host machine or even rack

## Auto-scale
* Available only in existing portal
* Automatically adjust number of VM within a availability set based on load or schedule or queue depth (Azure storage queue, or Service bus queue)
* Achieved by turning on/off additional VMs (pre-provisioned)
* Configured at the level of availability set within a cloud service
* Requirements to all VMs:
	* Must be Standard tier
	* Must belong to the same availability set
	* Must exist in the same cloud service
	* Must be the same instance size

# 6. Design and implement VM storage
* Temp drive uses local disk provided by the host machine and this disk might be shared among all VMs running on the host and thus might be subject of noisy neightbours competing for IOPS or bandwidth
* Each disk is stored as blob in Blob storage
* To increase performance or disk space in Azure VM you need to use multiple disks stripped into single volume
* General rule is that you can attach twice the number of CPU cores (max 16)

## Storage pool
* Enables you to group together a set of disks and then create a volume from the aggregated capacity
* You need to attach all empty disks to VM, create storage pool from them, create storage space and from storage space mount a volume you can access with a drive letter
* D-seriels provide D: temp SSD drive, which should not be used for keeping persistent data

## Disk caching
* Every disk has a host cache preference setting for managing local cache used for read or read/write operations
* Cache does not live in VM, it is external and resides on the host
* Cache options:
	* None - no cache provided (not available for operating disks)
	* Read only - assuming empty cache or no desired data found in local cache, red from Azure Storage and cache in local cache. Write goes directly to Azure Storage
	* Read/Write - the same as above, but write goes to local cache and some later point to Azure Storage
* Default cache for operating system disks is Read/Write and for data disk is none because Azure Storage can provide higher rate of random I/O than local cache
* Operating system operations are great example of sequential I/O workloads and this is the reason why it's enabled

## Geo-replication
* Geo-replication is not synchronized across blob files and therefore VHD disks which means writes for a file that is spread across multiple disks (in storage pool) can be replicated out of order. As the result they get corrupted
* To avoid that use locally redundant replication which does not add additional availability and reduces costs comparing to geo-replication

## Azure File storage
* Azure storage enables VM to access a share located within the same region as your VM regardless storage account or even subscription
* Azure provide support for most of SMB 2.1 scenarios:
	* Supporting applications that rely on file shares for access to data
	* Providing access to share application settings
	* Centralizing storage of logs, metrics and crash dumps
	* Storing commin tools and utilities for development, administration or setup
* Azure File storage is build upon Azure Storage which means it inherit the same availability, durability and scalability characteristics
* Azure File storage requires Azure Storage account
* Access is controlled with the storage account name and key, VMs can access share using storage credentials
* Share cannot be created below another share
* Each file in share or any directory below it can be max of 1 terabyte
* Max capacity of share is 5 terabyte
* It can double capacity of IOPS comparing to a single file in Blob storage
* You can manage shares (create or delete, list shares, get Etag, Last Modified property, user-defined share metadata key and value pair) and share content using REST APIs and through SMB
* In contrast to Azure Storage, Azure File storage allows you to use REST APIs to only manage files (API will respect all SMB file locks)
* Azure File storage can be provisioned only using PowerShell
```
$ctx = New-AzureStorageContext <StorageName> <StorageKey>
	New-AzureStorageShare <ShareName> -Context $ctx
```
	
* Add Storage account credentials to credentials manager to have them available after restarts
```
cmdkey /add:<StorageAccountName>.file.core.windows.net /user:<StorageAccountName> /pass:<StorageAccountPassword>
```
* Add file share using stored credentials:
```
net use z: \\<StorageAccountName>.file.core.windows.net\<ShareName> /Persistent: YES (or NO)
```
	
## Accessing files within the share
* RDP - you can mount the drives from local machine so that they are available in remote session
* AzCopy - you can upload files to share from local machine to remote share
* Azure PowerShell
* Storage Client Library
* Rest API

# 7. Monitor VMs
* Windows and Linux can collect out of box (after Azure VM Agent is installed):
	* Disk read, disk write
	* CPU percentage
	* Network in, network out
	* On Windows agent also installs IaaSDiagnostics extension for collecting monitoring and diagnostics data
	* IaaSDiagnostics can collect data from following groups of performance counter data:
		* Basic metrics
		* Network and web metrics
		* .NET metrics
	* They can also include data collected from logs - collected to tables in selected Azure Storage account
		* Windows event logs: system, security, application logs
		* Diagnostics infrastructure logs
		* IIS logs
	* Metrics are persisted in WADPerformanceCountersTable with aggregates to minute to WADMetricsPT1M, PT1H for hour
	* Event log are in WADWindowsEventLogsTable (Channel column to specify which log is it)
	* Diagnostics infrastructure logs are in WADDiagnosticsInfrastructureLogsTable
	* Application logs in WADLogsTable
	* IIS are writen as blogs to Blob storage under the wad-iis-log-files container

## Endpoint monitoring
* In addition to data collected within a VM by an extension, you can enable outside-in monitoring of HTTP or HTTPS endpoints (ping from Azure to get latency and availability around the world)