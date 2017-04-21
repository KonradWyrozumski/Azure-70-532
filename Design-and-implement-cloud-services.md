* Part of Platform-as-a-service (PaaS)
* Offers ability to deploy websites and asynchronous work to Azure services without need to manage complex networking and operating system updates
* It's mechanism for packaging, deploying, configurine one or more websites (web role) or async workers (worker role)
* Cloud service allows to group all of required roles (many web roles + many worker roles) into single deployment pack and specify how those roles should be distributed in terms of topology and physical servers
* As a part of Azure SDK you get: 
	* Storage emulator 
	* Compute emulator
	
# 1. Design and develop a cloud service
* To develop cloud service you need to install Azure SDK
* Cloud service sup
* port two types of roles:
	* Web role - used for web server applications hosted in IIS
	* Worker role - used for compute workload, both for executable process or for background worker implementations

## Web role
* During development of web worker role consider putting key configuration settings in the role configuration instead of web.config - this makes it possible to surface these settings to management portal
* If your application has any dependencies that require installation on destination VM, or control over IIS settings, use startup task to provide an unattented deployment for this configuration

## Worker role
* RolePointEntry is the heart of its functionality
* Providing implementation of OnStart, Run, OnEnd is implied

* Service Definition file - definition for the cloud service, list of startup tasks and definition for each web and worker role
* Service Configuration file - one for local configuration and one for cloud with for example connection strings

## Design and implement resiliency
* Following patterns are useful to availability for both worker and web roles:
	* Static content hosting pattern
	* Cache-Aside pattern
	* Health Endpoint Monitoring pattern
	* Compensating Transaction pattern
	* Command and Query Responsibility Segregation pattern
* For Worker roles we can also apply the same patterns as for WebJobs:
	* Competing Customers pattern
	* Priority Queue pattern
	* Queue-Based Load Leveling pattern
	* Scheduler Agent Supervision pattern
## Startup tasks
* Startup tasks are used to perform operations prior to starting a role
* It can be batch script to change registry status, runs MSI or invoke PowerShell
* Process of starting a role follows this steps:
	* Role enters "starting" state, does not receive any traffic,
	* Startup tasks are executed - simple runs in order, background and foreground start asynchronously
	* IIS initialized
	* OnStart() is called in role
	* Role enters "ready" state and receives a traffic
	* Run() method is called
* Execution context can be 
	* Limited ( running without admin rights)
	* Elevated (running with admin rights)
* Task type can be:
	* Simple - running synchronously in order
	* Foreground - asynchronous task running in foreground - the role will not start until those are finished but might be put in ready state. Role cannot be recycled while they run
    * Background - asynchronous task running in background running in parallel with other background tasks. Role can be recycled while they run
* Startup tasks may run more than once on VM - when restarted they are executed again

# 2. Configure cloud services and roles
* Similar to websites and VMs you can scale both in terms of instance size and instance count and you can use auto-scale for instance count
* You can scale instance sizes to adjust:
    * The size of local temp disk
	* The number of CPU cores available
	* The amount of RAM available
	* The network performance
* Unlike websites and VM which instance size can be adjusted through management portal, scale the instance size requires to change cloud service definition and re-deploy package
* Auto scale can be configured based on 
	* Schedule
	* Queue depth of Azure Storage queue
	* Queue depth of Service bus queue

## Configuring cloud service networking
* Input endpoint - controls transport protocol and port used by traffic coming from the Internet to instance of the role. IP address to communicate with this role is VIP of the cloud service and the role instance will see the traffic on configured local port or one assigned by Azure fabric controller if one is not specified
* Instance Input endpoint - defines a specific protocol and port that when used by Internet traffic goes via port forwarding directly to specific role instance. The IP used to communicate is the VIP of the cloud service
* Internal endpoint - configured a protocol and a dynamically allocated public port range used by communication between roles and role instances within a cloud service. The IP address used to communicate is the internal IP address assigned to each role instance and the role instance will see traffic on configured private port

## Configuring HTTPS endpoints
* HTTPS endpoints are used to secure traffic between public Internet and your role instances via TSL by using a certificate that you can generate by yourself or purchase from certificate authority.
* Process has four main tasks:
	* Acquire a certificate
	* Convert certificate to PFX format by exporting it from the key store
	* Upload certificate to certificate store used by your cloud serve
	* Configure endpoint in your role to use HTTPS and certificate to secure traffic

## Access control lists
* ACL are used for restricting access to an endpoint within cloud service but the source requiring access is not a role within the cloud service (it can be a VM)
* ACL is configured only by configuration file and requires redeploy

## Configuring virtual networks
* You can join cloud service to existing virtual network, however you need to edit configuration file and re-deploy
* To join to virtual network you need to:
	* Associate the cloud service with a virtual network
	* Add Address assignments element with one Instance address for each role in cloud service

## Configuring reserved IPs and public IPs
* Reserved IP - keep VIP from changing
* Public IP - enables direct access to instance without specifying port
* Configured in configuration file of a cloud service
* To configure reserved or public IP you first need to:
	* Create reserved/public IP address:
    ```	
    New-AzureReservedIP -Location <RegionName> -ReservedIpName <ReservedIPName>
    ```

    * Associate role with reserved IP address

## Configuring local storage
* Local storage is a temporary disk space for your cloud service
* Used for temporary location for file uploads, storage for generated files, MSI installers
* Local storage will persist between restarts of the VM that hosts your role, but data will be lost once you migrate to different VM host
* You can specify bigger disk space than allowed for your VM, but you will get exception once disk space will be filled up to capacity allotted
* Clean On Role Recycle -> once checked, local storage will be cleared on restart

## Configuring multiple websites in a web role
* Single web role can host multiple websites through differentiating by:
	* Host headers
	* Port 

## Configuring custom domains
* When deploying a cloud service, any roles for which you have configured an input endpoint will be accessible by <cloudServiceName>.cloudapp.net
* To have custom domain you need to configure DNS either with:
	* A record - map custom website to VIP address (@ -> root of the domain)
	* CNAME record - map a subdomain to the DNS name of your cloud service

## Configuring caching
* Role can be used to provide a cache cluster using In-Role cache for Azure Cache
* In-memory cache used to store frequently accessed data so that access is fast and offloads the data retrieval from resources like SQL database
* Configured to run within a single role, single cloud service deployment where the instances of your cache roll make up the cache cluster
* Cache can be accessed only using managed code running within role instances within the same cloud service deployment that contains the cache role
* In-Role cache can be configured in two topologies:
	* Co-located cache - each role instance contributes a certain percentage of memory to the cache and joins in as part of larger cache cluster. Cache is provided as addition to main function of the role
	* Dedicated cache - worker role that is used only for caching
* In-Role Cache stores object in serialized form within the in-memory cache that is distributed over instances. Because of that cached classes must be instances of classed marked with Serialized attribute
* When configuring web role you can have only co-located cache, worker role can have either co-located cache or dedicated cache configured
* Once you do not select storage account for caching purposes, it will maintain it's state only in-memory
* Cache policies:
	* High availability - ensure that single piece of cache if represented by two copies, each on separate instance within cluster
	* Notification - cache clients to poll cluster for notification about data that has changed. Notification surfaced as callbacks that are handled by cache clients
	* Eviction Polity - choose whether to enforce eviction policy when memory runs low (for example least recently used objects are removed first). If None no data is cached when fills up
	* Expiration type - Absolute -remove when duration since adding exceeds TTL setting. Sliding Window - timer will be reset every time object is accessed
	* Time to Live - time used by above expiration type
* Named cache is a logical grouping of cache data
* If using cache in RoleEntryPoint logic, you must first programmatically configure the cache instead of relying on web config
* Always dispose DataCacheFactory object and try to re-use it

# 3. Deploy a cloud service
* Application package contains all application and service model files needed to deploy and run an application in Azure cloud service
* When deploying to Azure using a package you deploy two files:
	* Application package file
	* Service configuration file
* Deployment options:
	* Incremental - Azure will roll out the upgrade while attempting to minimalize the impact on the overall service by upgrading instances one upgrade domain at time. This includes stopping instance, apply update and restart. Only after all instance in upgrade domain are back online, Azure will move to instances in next upgrade domain. It takes a lot of time
	* Simultaneous - Azure stops all instances at once, apply upgrade and brings the instances back online
	* Full Deployment - Azure do not attempt to upgrade instances, it removes existing instances and performs fresh deployment
* Changes requiring full deployment:
	* Change the name of role
	* Change to the upgrade domain count
	* Reduction in the size of local storage resources

## VIP swapping a deployment
* Production and staging environment in which you deploy cloud service
* Production receives <dns-prefix>.cloudapp.net and VIP, staging environment receives <guid>.cloudapp.net and different value for the VIP
* Swap changes URL and VIP values
* Production IP address assigned to the VIP is always maintained

## Affinity groups
* Main reason for introducing affinity groups was to request Azure to physically locate related services next to each other to minimize the impact of network latency
* Since infrastructure changes that eliminates latencies experiences between services there is no need to specify affinity groups
* Affinity groups restricts your ability to select from new resources introduced into the datacenter after you create affinity group
* You can choose affinity group only during provisioning of cloud service, you cannot change it later
	
```
New-AzureAffinityGroup -Location <Location> -Name <Name>
New-AzureService -AffinityGroup <AffinityGroupName> -ServiceName <ServiceName>
```
# 4. Monitor and debug a cloud service
* You configure collection of diagnostic data first by enabling Azure Diagnostics module in service definition and then configuring diagnostics data you want to collect
* Azure Diagnostics monitor will start automatically when the role instance starts
* Diagnostics monitor can be changed once cloud service is deployed with Diagnostics enabled

## Remote debugging
* IntelliTrace is alternative to remote debugging
* Gives you experience similar to debugging without the potential to interrupt the service operation
* You work from a reply of captured events