# Recover a multi-tenant SaaS application using geo-restore from database backups

In this tutorial, you explore a full disaster recovery scenario for a multi-tenant SaaS application implemented using the database-per-tenant model. You use [_geo-restore_](https://docs.microsoft.com/en-us/azure/sql-database/sql-database-recovery-using-backups) to recover the catalog and tenant databases from automatically maintained geo-redundant backups into an alternate recovery region. After the outage is resolved, you use [_geo-replication_](https://docs.microsoft.com/en-us/azure/sql-database/sql-database-geo-replication-overview) to repatriate changed databases to their original production region.

Geo-restore is the lowest-cost disaster recovery solution.  However, restoring from geo-redundant backups can result in data loss and can take a considerable time, depending on the size of the databases. **To recover applications with the lowest possible RPO and RTO, use geo-replication instead of geo-restore**.

To reduce recovery time when restoring large numbers of databases in elastic pools, restore databases in parallel into multiple pools at the same time. To repatriate databases once the outage is resolved, use geo-replication, which incurs no data loss and minimizes disruption. 

For multi-tenant applications with a catalog database that identifies where the active copy of each tenant's database is located, the catalog database is also restored and repatriated. Then as each tenant database is restored or repatriated, the catalog is updated with the new database location.

This tutorial explores both restore and repatriation workflows. You'll learn how to:

* Sync database and elastic pool configuration info into the tenant catalog
* Set up a mirror image recovery environment in a 'recovery' region, comprising application, servers, and pools    
* Recover catalog and tenant databases using _geo-restore_
* Repatriate the tenant catalog and changed tenant databases using _geo-replication_ after the outage is resolved
* Update the catalog as each database is restored or repatriated to track the current location of the active copy of each tenant's database
* Redirect all application requests to ensure they are handled by the application instance deployed in the same region as the active database  
 

Before starting this tutorial, make sure the following prerequisites are completed:
* The Wingtip Tickets SaaS database per tenant app is deployed. To deploy in less than five minutes, see [Deploy and explore the Wingtip Tickets SaaS database per tenant application](saas-dbpertenant-get-started-deploy.md)  
* Azure PowerShell is installed. For details, see [Getting started with Azure PowerShell](https://docs.microsoft.com/powershell/azure/get-started-azureps)

## Introduction to the geo-restore recovery pattern

Disaster recovery (DR) is an important consideration for many applications, whether for compliance reasons or business continuity.  Should there be a prolonged service outage, a well-prepared DR plan can minimize business disruption.

![Recovery Architecture](TutorialMedia/recoveryarchitecture.png)
 
For a SaaS application implemented with a database-per-tenant model, recovery and repatriation must be carefully orchestrated.

In this tutorial, you first recover the Wingtip Tickets application and its databases to a different region. Catalog and tenant databases are restored from geo-redundant copies of backups. When complete, the application is fully functional in the recovery region.

Later, in a separate repatriation step, you use geo-replication to copy the catalog and tenant databases changed after recovery to the original region. The application and databases stay online and available throughout.  When complete, the application is fully functional in the original region.

In a final step, you clean up the resources created in the recovery region.  

> Note that the application is recovered into the _paired region_ of the region in which the application is deployed. For more information, see [Azure paired regions](https://docs.microsoft.com/en-us/azure/best-practices-availability-paired-regions).   

Recovering a SaaS app with its many components into another region needs careful orchestration. You need to: 
* Provision servers and pools in the recovery region to reserve capacity for existing tenants and allow new tenants to be provisioned  
* Deploy the app in the recovery region and enable it to continue provisioning new tenants
* Restore tenant databases across all elastic pools in parallel to ensure maximum throughput
* Submit restore requests in batches to avoid service throttling limits
* Restore tenants in priority order to minimize impact on key customers 
* Submit restore requests asynchronously - the service maintains a queue of requests for each elastic pool to prevent the pool from being overloaded  
* Reactivate tenants in the catalog as each database is restored 
* Enable the app to connect to recovered databases without changing connection strings
* Allow the restore process to be canceled in mid-flight.  Canceling requires you revert all databases not yet recovered, or recovered but not yet updated, to the copy in the original region.  Reverting to the original database prevents data loss and reduces the number of databases that need repatriation
* Repatriate recovered databases that have been modified to the original region without data loss. 
* Ensure the app and tenant database are always colocated to minimize latency. 

In this tutorial, these challenges are addressed using features of Azure SQL Database and the Azure platform:

* Resource Management templates, to provision a mirror image of the production servers and elastic pools in the recovery region, and create a separate server and pool for provisioning new tenants. 
* [Geo-restore](https://docs.microsoft.com/azure/sql-database/sql-database-disaster-recovery), to recover the catalog and tenant databases from automatically maintained geo-redundant backups. 
* DNS aliases, to allow connecting to recovered and repatriated databases without changing or reconfiguring the app. Switching the active database requires only changing its alias.    
* Asynchronously submitted restore requests, to allow SQL Database to queue requests for each pool, and process them in batches to prevent overloading the pool.
* _Geo-replication_, to repatriate databases to the original region after the outage. Using geo-replication ensures there is no data loss and minimal impact on the tenant.    

## Get the disaster recovery management scripts 

The recovery scripts used in this tutorial are available in the [Wingtip Tickets SaaS database per tenant GitHub repository](https://github.com/Microsoft/WingtipTicketsSaaS-DbPerTenant/tree/feature-DR-georestore). Check out the [general guidance](saas-tenancy-wingtip-app-guidance-tips.md) for steps to download and unblock the Wingtip Tickets management scripts.

## Review the healthy state of the application
Before you start the recovery process, review the healthy start state of the application.
1. In your web browser, open the Wingtip Tickets Events Hub (http://events.wingtip-dpt.&lt;user&gt;.trafficmanager.net - substitute &lt;user&gt; with your deployment's user value).
	* Notice the catalog server name in the footer
1. Click on the Contoso Concert Hall tenant and open its event page.
	* Notice the tenants server name in the footer
1. In the [Azure portal](https://portal.azure.com), open the resource group in which the app is deployed
	* Notice the region in which the servers are deployed. 

## Sync tenant configuration into catalog

In this task, you start a process to sync the configuration of the servers, elastic pools, and databases into the tenant catalog.  This information is used later to configure a mirror image environment in the recovery region.

> Note: the sync process is implemented as a local Powershell job. In a production scenario, this process should be implemented as a reliable Azure service of some kind.

1. In the _PowerShell ISE_, open the ...\Learning Modules\UserConfig.psm1 file. Replace `<resourcegroup>` and `<user>` on lines 10 and 11  with the value used when you deployed the app.  Save the file!

2. In the *PowerShell ISE*, open the ...\Learning Modules\Business Continuity and Disaster Recovery\DR-RestoreFromBackup\Demo-RestoreFromBackup.ps1 script and set the following values:
	* **$DemoScenario = 1, Start a background job that syncs tenant server, and pool configuration info into the catalog**

3. Press **F5** to run the sync script. A new PowerShell session is opened to sync the configuration of tenant resources.
![Sync process](TutorialMedia/syncprocess.png)

Leave the PowerShell window running in the background and continue with the rest of the tutorial. 

> Note: the catalog sync process connects to the active catalog via a DNS alias that is modified during restore and repatriation. This ensures that changes made to database and pool configuration in the recovery region are propagated to the original region during repatriation.

## Restore tenant resources into the recovery region

The restore process reserves the required capacity, enables new tenants to be provisioned, and restores tenant databases from backups as fast as possible. This process does the following:

1. Disables the Traffic Manager endpoint for the web app in the original region. Disabling the endpoint prevents users from connecting to the app in an invalid state should the region come online during recovery.

1. Provisions a recovery catalog server in the recovery region and then geo-restores the catalog database and updates the catalog alias to point to the restored database.  
	* The catalog alias is used by the catalog sync process

1. Marks all existing tenants in the recovery catalog as offline to prevent access to tenant databases before they are restored.

1. Provisions an instance of the app in the recovery region and configures it to use the restored catalog in that region.

1. Provisions a server and elastic pool in which new tenants will be provisioned. 
	* To keep app-to-database latency to a minimum, the sample app is designed so that it only connects to a tenant database in the same region.  If the app in one region detects that the active copy of a tenant database is in another region, it will redirect to an instance of the app in the other region. This is important during repatriation.
		
1. Provisions the recovery server and elastic pools required for restoring the existing tenant databases. The servers and pools are a mirror image of servers and pools in the original region, with additional server and pool for new tenants.  Provisioning pools up-front is important to reserve all the capacity needed.
	* An outage in one region may place significant pressure on the resources available in the paired region.  Reserving resources quickly is recommended. Consider using geo-replication if it is critical that an application must be recovered in a specific region. 

1. Enables the Traffic Manager endpoint for the Web app in the recovery region, which allows the application to provision new tenants.   

1. Submits batches of requests to restore databases across all pools in priority order. 
	* Batches are organized so that databases are restored in parallel across all pools.  
	* Restore requests are submitted asynchronously so they are submitted quickly and queued for execution.
	* As restore requests are processed in parallel across all pools, it is better to distribute important tenants across many pools rather than concentrating them in a few pools. 

1. Polls the database service to determine when databases are restored.  Once a tenant database is restored, updates the catalog to record the database rowversion and mark the tenant as online. 
	* Tenant databases can be accessed by the application as soon as they're marked online in the catalog. 
	* Recording the rowversion allows the repatriation process to determine if the database has been updated in the recovery region.   	 

## Run the recovery script

> IMPORTANT This tutorial restores databases from geo-redundant backups. These backups may not be available for 10-20 minutes after initial database creation. Wait for 20 mins from installation of the app before running this script.

Now run the recovery script which automates the restore steps previously described:

1. In the *PowerShell ISE*, open the ...\Learning Modules\Business Continuity and Disaster Recovery\DR-RestoreFromBackup\Demo-RestoreFromBackup.ps1 script and set the following values:
	* **$DemoScenario = 2, Recover the SaaS app into a recovery region by restoring from geo-redundant backups**

2. Press **F5** to run the script.  
	* The script starts a series of PowerShell jobs that run in parallel which restore servers, pools and databases to the recovery region. 
	* The recovery region is the paired region associated with the region in which you deployed the application. For more information, see [Azure paired regions](https://docs.microsoft.com/en-us/azure/best-practices-availability-paired-regions). 

	Monitor the status of the recovery process in the console section of the PowerShell window.

**insert screenshot of powershell window with code running <<<**

>To explore the code for the recovery jobs, review the PowerShell scripts in the ...\Learning Modules\Business Continuity and Disaster Recovery\DR-RestoreFromBackup\RecoveryJobs folder.

## Review the application state during recovery
While the tenant databases are being restored, the tenants are marked offline in the catalog.  Once the application has been deployed to the recovery region and activated, attempts to connect to individual tenants will be unsuccessful until their databases are restored and marked online.  It's important to design your application to handle a tenant being marked offline.

1. Before the restore process completes, refresh the Wingtip Tickets Events Hub in your web browser (http://events.wingtip-dpt.&lt;user&gt;.trafficmanager.net - substitute &lt;user&gt; with your deployment's user value). 
	* Notice that tenants that are not yet restored are marked as offline, and that opening an offline tenant events page displays a tenant offline notification. 

## Provision a new tenant in the recovery region
Even before the tenants are recovered you can provision new tenants, which now occurs in the recovery region. Provisioning uses the new-tenant recovery server and pool that were created during the earlier restore process in the recovery region. Provisioning a new tenant impacts the repatriation process in two ways:
1. As the catalog in the recovery region is changed by this action, the catalog will be repatriated to the origin region.
1. The new-tenant server, its pool and any databases there are repatriated to the origin region. 

## Review the recovered state of the application

When the recovery process completes, the application is fully functional in the recovery region. 

Once the application is fully recovered, review how it behaves.

1. In your web browser, refresh the Wingtip Tickets Events Hub  (http://events.wingtip-dpt.&lt;user&gt;.trafficmanager.net - substitute &lt;user&gt; with your deployment's user value).
	* Notice the value reported for the catalog server in the footer is the catalog recovery server in the recovery region.

	![Recovered tenants list](TutorialMedia/recoveredcatalogserver.png)

	> ** UPDATE THIS IMAGE**

1. In the Events Hub, click on Contoso Concert Hall and open its events browse page, which is now available. Notice that the tenants recovery server referenced in the footer is the recovery server in the recovery region.

1. In the [Azure portal](https://portal.azure.com), inspect the recovery resource group.  Notice that the application and recovery servers are deployed in the paired region of the ordinal deployment of the app.

## Change tenant data 
In this task you will modify some data in one of the restored tenant databases.  This simulates activity by a tenant that occurs after the outage. The repatriation process to be run shortly will copy any restored databases that have been changed to the original region. 

1. In your browser, find the events list for the Contoso Concert Hall and note the last event name.
1. In the *PowerShell ISE*, in the ...\Learning Modules\Business Continuity and Disaster Recovery\DR-RestoreFromBackup\Demo-RestoreFromBackup.ps1 script, set the following:
	* **$DemoScenario = 3** (Delete last event)
1. Press **F5** to execute the script
1. Refresh the Contoso Concert Hall events page (http://events.wingtip-dpt.&lt;user&gt;.trafficmanager.net/contosoconcerthall - substitute &lt;user&gt; with your deployment's user value) and notice that the last event has been deleted.

## Repatriate the application to its original production region

In this task you repatriate the application to its original region.  in the case of a real outage, repatriation would only be triggered once you're satisfied the outage is resolved. Note that an outage may be resolved before the restore has completed. For this reason, starting repatriation cancels any ongoing restore activity.

The scope of repatriation varies depending on circumstances. Databases that weren't restored, or were restored but unchanged, can be reactivated immediately in the original region - these databases won't have suffered any data loss. If new tenants were added in the recovery region during the outage, or any pool or database configuration was changed, then the catalog is repatriated.  New tenant databases and restored tenant databases that have been updated post-restore are repatriated. 

The repatriation process does the following:
1. Reactivates tenant databases in the original region that have not been restored to the recovery region, or if restored, have not been changed there. These databases will be exactly as last accessed by their tenants. once reactivated, these tenants are  immediately available to the application.
1. Causes new tenant onboarding to occur in the original region so no further tenant databases are created in the recovery region.
1. Cancels any outstanding or in-flight database restore requests.
1. Copies all restored databases _that have been changed post-restore_ to the original region. This includes the catalog database if it has changed.
1. Cleans up resources created in the recovery region during the restore process.

It's important that steps 1-3 are done promptly to limit the number of tenant databases that need to be repatriated.  

It's important that step 4 causes no further disruption to tenants and no data loss. To achieve this goal, you use _geo-replication_ to 'move' changed databases to the original region. If the app is working well in the recovery region, there may not be any great urgency to move databases back to the production region. 

Once each database to be repatriated has been replicated to the original region it is failed over.  Failing the database over to the replica effectively moves the database to the original region. Failing over a database causes any open connections to be dropped briefly and the database to be unavailable for a few seconds (99th percentile is under five seconds).  Applications should be written with retry logic to ensure they connect again when this happens.  Although this brief disconnect is often not noticed, you may choose to repatriate databases out of business hours. 

Once a database is failed over to its replica in the production region, the restored database in the recovery region can be deleted. The database in the production region then relies on geo-restore for DR protection again. 

In step 5, resources in the recovery region, including the recovery servers and pools, are deleted.      

## Run the repatriation script
Now let's assume the outage is resolved and run the repatriation script.  This script reverts tenants you didn't modify to their original databases.  It then copies the databases you updated earlier to the production region, replacing the corresponding databases there.    
  
1. In the *PowerShell ISE*, open the ...\Learning Modules\Business Continuity and Disaster Recovery\DR-RestoreFromBackup\Demo-RestoreFromBackup.ps1 script and set the following values:
	* **$DemoScenario = 2, Recover the SaaS app into a recovery region by restoring from geo-redundant backups**

1. Press **F5** to run the recovery script. The repatriation of the changed databases will take several minutes.
1. While the script is running, refresh the Events Hub page (http://events.wingtip-dpt.&lt;user&gt;.trafficmanager.net - substitute &lt;user&gt; with your deployment's user value)
	* Notice that all the tenants are online and accessible throughout this process.
1. Click on the Fabrikam Jazz Club, and if you did not modify this tenant, notice from the footer that the server is already reverted to the original production server.
1. Open or refresh the Contoso Concert Hall events page and notice from the footer that the database is on the _-recovery_ server initially.  
1. Refresh the Contoso Concert Hall events page when the repatriation process completes and notice that the server is now the original server.

## Clean up recovery region resources after repatriation
Once repatriation completes, it's safe to delete the resources in the recovery region.  The restore process creates all the recovery resources in a recovery resource group.  Using a separate resource group allows them to be deleted together with a single action.
1. Open the [Azure portal](https://portal.azure.com) and delete the **ADD NAME OF RG HERE** resource group.
	* Deleting these resources promptly is recommended as it stops billing for them.


## Next steps

In this tutorial you learned how to:

* Sync tenant configuration data into the tenant catalog database
* Use tenant aliases to ensure no application changes are required during the recovery process 
* Restore Azure SQL servers, elastic pools, and Azure SQL databases into a recovery region
* Repatriate recovered databases that have been updated to the original production region

<!--Now, try the [Recover a multi-tenant SaaS application using geo-replication]() to learn how to geo-replication can dramatically reduce the time needed to recover a large-scale multi-tenant application.
-->
## Additional resources

* [Additional tutorials that build upon the Wingtip SaaS application](https://docs.microsoft.com/en-us/azure/sql-database/sql-database-wtp-overview#sql-database-wingtip-saas-tutorials)
