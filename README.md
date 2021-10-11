# SQL-Managed Instance: Disaster Recovery Reference Architecture 
# Step by Step Implementation Guide

SQL Managed Instance Installation process. - Step by Step to Disaster Recovery - Ready for Massive roll out;

This document provide the Best Practice guidence for the SQL-Managed Instance implementation considering a Disaster Recovery Architecture with a full Failover Group configured.

The entire process can be implemented by Azure Resource Manager configuration, Powershell, ARM Templates or Infrastructure as a Code using Terraform. This Step by Step guide covers the first scenario which is using Azure Resouce Manager.

In order to undertand the SQL Managed Instance Resource Limits, check [here](https://docs.microsoft.com/en-us/azure/sql-database/sql-database-managed-instance-resource-limits) for more information. Few points to higlight. 
* Gen 4 vs Gen 5;
* Managed instance has two service tiers: General Purpose and Business Critical. These tiers provide different capabilities, as described in this [table](https://docs.microsoft.com/en-us/azure/sql-database/sql-database-managed-instance-resource-limits#service-tier-characteristics)..

## Topics that you will work in this guide:
* [End to End Architecture](https://github.com/hcmarque/SQL-Managed-Instance-Disaster-Recovery-Architecture-Design/blob/master/README.md#e2e-architecture)
* [Requirements for the SQL Managed Instances in Disaster Recovery implementation](https://github.com/hcmarque/SQL-Managed-Instance-Disaster-Recovery-Architecture-Design/blob/master/README.md#requirements-to-have-your-disaster-recovery-implementation-working)
* [Create the Resouce Groups](https://github.com/hcmarque/SQL-Managed-Instance-Disaster-Recovery-Architecture-Design/blob/master/README.md#step-1-main-activity-create-two-resources-groups---sql-mi-and-network)
* [DDoS Standard Configuration](https://github.com/hcmarque/SQL-Managed-Instance-Disaster-Recovery-Architecture-Design/blob/master/README.md#step-2-ddos-standard-design)
* [Create the Virtual Network on Both Regions](https://github.com/hcmarque/SQL-Managed-Instance-Disaster-Recovery-Architecture-Design/blob/master/README.md#step-3-create-the-virtual-network-on-both-regions)
* [Create the Virtual Network Gateways and Connections](https://github.com/hcmarque/SQL-Managed-Instance-Disaster-Recovery-Architecture-Design/blob/master/README.md#step-4-create-the-network-gateways-to-connect-the-both-regions-in-this-scenario-east-us2-and-central-us)
* [Create the SQL Managed Instance on the Fisrt Region](https://github.com/hcmarque/SQL-Managed-Instance-Disaster-Recovery-Architecture-Design/blob/master/README.md#step-5-create-the-sql-managed-instance-on-the-first-region)
* [SSMS Connection to First SQL Managed Instance](https://github.com/hcmarque/SQL-Managed-Instance-Disaster-Recovery-Architecture-Design/blob/master/README.md#step-6-ssms-connection-to-your-sql-managed-instance)
* [Create the SQL Managed Instance on the Second Region](https://github.com/hcmarque/SQL-Managed-Instance-Disaster-Recovery-Architecture-Design/blob/master/README.md#step-7-create-a-second-sql-managed-instance)
* [SSMS Connection to Second SQL Managed Instance](https://github.com/hcmarque/SQL-Managed-Instance-Disaster-Recovery-Architecture-Design/blob/master/README.md#step-8-check-the-connectivity-to-the-second-managed-instance-withthe-ssms-installed-in-another-region---e2e-architecture-test)
* [Create the Failover Group](https://github.com/hcmarque/SQL-Managed-Instance-Disaster-Recovery-Architecture-Design/blob/master/README.md#step-9-create-the-failover-group)



## Introduction
**SQL Database Managed Instance** is a deployment option in Azure SQL Database that is highly compatible with SQL Server, providing out-of-the-box support for most SQL Server features and accompanying tools and services. 
Managed Instance also provides native virtual network (VNET) support for an isolated, highly-secure environment to run your applications. Now you can combine the rich SQL Server programming surface area in the cloud with the operational and financial benefits of an intelligent, fully-managed database service, making Managed Instance the best PaaS destination for your SQL Server workloads.


## E2E Architecture
In order to ilustrate what you will have at the end of this deployment, please find here the **end to end Architecture.**
* Included on this deployment - please consider as **Best Practice** based on numbers of big customers deployment:
* 2 Regions (MUST be Pair Regions at this moment - this will garantee the Disaster Recovery desing. Across the region pairs Azure serializes platform updates (planned maintenance), so that only one paired region is updated at a time. In the event of an outage affecting multiple regions, at least one region in each pair will be prioritized for recovery. - For the example below, I chose **EAST-US2 and CENTRAL-US.** 


E2E SQL-Managed Instance Architecture:

<img src="Images/sqlmiarchitecture.png">

## E2E Architecture (visio)
The Visio version can be found [here](https://github.com/hcmarque/SQL-Managed-Instance-Disaster-Recovery-Architecture-Design/blob/master/Images/SQLMI-DR.vsdx)

## Requirements to have your Disaster Recovery Implementation working:
For the First Step, let’s create your Infrastructure, which include the following: 
* **Resource Groups (RG)** - We will create a dedicate RG for Network and another for SQL-MI; **Attention:** Both SQL Instances needs to be created on the same Resource Group, even located in different Regions;
* **Networks in 2 Azure Regions**: (EAST-US2 and CENTRAL-US in this design;
* **VPN Gateways between the Regions;** **Attention:** The virtual networks used by the the managed instances need to be connected through a VPN Gateway or Express Route. When two virtual networks connect through an on-premises network, ensure there is no firewall rule blocking ports 5022, and 11000-11999. Global VNet Peering is **not supported**.
* **Connections betweem the VPN Gateways;**
* Please find [here](https://docs.microsoft.com/en-us/azure/sql-database/sql-database-auto-failover-group#enabling-geo-replication-between-managed-instances-and-their-vnets) more details about the Network Requirements for SQL Managed Instances:  
Check [here](https://docs.microsoft.com/en-us/azure/best-practices-availability-paired-regions#what-are-paired-regions) the Azure Regions orginized in Pairs availables that would help you on your Region definition for your Network Design.
* Related to the Network configuration, please find [here](https://docs.microsoft.com/en-us/azure/sql-database/sql-database-howto-managed-instance#network-configuration) details that needs to be considered during your planning session:
    - Determine size of a managed instance subnet – Managed instance is placed in dedicates subnet that cannot be resized once you add the resources inside. Therefore, you would need to calculate what IP range of addresses would be required for the subnet depending on the number and types of instances that you want to deploy in the subnet.
    - Create new VNet and a DEDICATED subnet for a managed instance – Azure VNet and subnet where you want to deploy your managed instances must be configured according to the network requirements described here. In this guide you can find the easiest way to create your new VNet and subnet properly configured for managed instances.
    - Configure existing VNet and subnet for a managed instance – if you want to configure your existing VNet and subnet to deploy managed instances inside, here you can find the script that checks the network requirements and make configures your subnet according to the requirements.
    - Configure custom DNS – you need to configure custom DNS if you want to access external resources on the custom domains from your managed instance via linked server of db mail profiles.
Sync network configuration - It might happen that although you integrated your app with an Azure Virtual Network, you can't establish connection to a managed instance. One thing you can try is to refresh networking configuration for your service plan.
    - Find management endpoint IP address – Managed instance uses public endpoint for management-purposes. You can determine IP address of the management endpoint using the script described here.
    - Verify built-in firewall protection – Managed instance is protected with built-in firewall that allows the traffic only on necessary ports. You can check and verify the built-in firewall rules using the script described in this guide.
    - Connect applications – Managed instance is placed in your own private Azure VNet with private IP address. Learn about different patterns for connecting the applications to your managed instance.

## Step 1: Main Activity: Create two Resources Groups - SQL-MI and Network
* 1.1 - Create the Resource Group that will be used for both SQL Managed Instances (Primary and Secondary) –  Needs to be under the same Resource Group, otherwise your Failover Group wont work on later step). Click on `Create a Resource Group`

<p align="center">
  <img src="Images/picture02.png" alt="drawing" width="600"/>
</p>

* 1.2 - Select the desired `Subscription`, `Resource group` name and `Region` **Remember** In case of Disaster Recovery design, please consider the Pair regions considerations mentioned E2E Architecture considerations. In this Architecture Desing Example, we will consider EAST-US2 and CENTRAL-US for the Instances.
Now CLick on `Review + Create`.

<p align="center">
  <img src="Images/picture03.png" alt="drawing" width="600"/>
</p>

* 1.3 - As soon as you have the Validation passed green light, click on `Create`

<p align="center">
  <img src="Images/picture04.png" alt="drawing" width="600"/>
</p>

* 1.4 - Create the Resource Group that will be used for the Network Configurations. Click on `Create a Resource Group` and Select the desired `Subscription`, `Resource group` name and `Region`.
Now CLick on `Review + Create`.

<p align="center">
  <img src="Images/picture05.png" alt="drawing" width="600"/>
</p>

* 1.5 - As soon as you have the Validation passed green light, click on `Create`

<p align="center">
  <img src="Images/picture06.png" alt="drawing" width="600"/>
</p>


* 1.6 - Resource Group Summary for the SQL-Managed Instance Disaster Recovery Implementation:

<p align="center">
  <img src="Images/picture07.png" alt="drawing" width="600"/>
</p>

## Step 2: DDoS Standard Design (Optional)

* 2.1 DDoS Standard
On the moment to create your network for the SQL-MI, is recommend to enable the DDoS Standard. 
Azure DDoS basic protection is integrated into the Azure platform by default and at no additional cost. 
Azure DDoS standard protection is a premium paid service that offers enhanced DDoS mitigation capabilities via adaptive tuning, attack notification, and telemetry to protect against the impacts of a DDoS attack for all protected resources within this virtual network.

Search for `DDoS protection plans` and then create a protection plan following the steps below:

<p align="center">
  <img src="Images/picture08.png" alt="drawing" width="600"/>
</p>

* 2.2 DDoS Parameters:
Create the DDoS Standard selecting the `Subscription`, `Resource Group` (The Network Resource Group that you just created on the previous step, Instance Details with the `Name` and `Region` Create the DDoS plan for both Regions, in this step will be to CENTRAL-US. Click in `Review + Create` and then, `Create` after the Validation.

<p align="center">
  <img src="Images/picture09.png" alt="drawing" width="600"/>
</p>


* 2.3 Create now for another region that you will launch your second instance: 

<p align="center">
  <img src="Images/picture10.png" alt="drawing" width="600"/>
</p>

* 2.4 Create the DDoS Standard selecting the `Subscription`, `Resource Group` (The Network Resource Group that you just created on the previous step, Instance Details with the `Name` and `Region` Create the DDoS plan for both Regions, in this step will be to EAST-US2. Click in `Review + Create` and then, `Create` after the Validation.

<p align="center">
  <img src="Images/picture11.png" alt="drawing" width="600"/>
</p>

<p align="center">
  <img src="Images/picture12.png" alt="drawing" width="600"/>
</p>

* 2.5 Back now to the Network Resource Group to see that the DDoS Protection Plan was created and is ready to be attached to a vnet that you will create now.

<p align="center">
  <img src="Images/picture13.png" alt="drawing" width="600"/>
</p>


## Step 3: Create the Virtual Network on both Regions

* 3.1 Inside of the MarketPlace, search for Virtual Netowork and create the Virtual Network clicking in `Create` as demonstrated on the picture below.

<p align="center">
  <img src="Images/picture14.png" alt="drawing" width="600"/>
</p>


* 3.2 The following parameters will be used for this deployment. Use the names according with your Company/Department requirements. 
**Important: Use your own Ip Address Range, and be sure that you are not overlaying your On-Premise IP Address Range Network. The IP Address used in this example was used for this specif use case.**

<p align="center">
  <img src="Images/picture15.png" alt="drawing" width="600"/>
</p>


* 3.3 Back to your Github-Network Resource Group and `+ Add` a new Network

<p align="center">
  <img src="Images/picture16.png" alt="drawing" width="600"/>
</p>

* 3.4 Configure now the new Virtual Network considering another region, in this scenario the Central US was used. The following parameters will be used for this deployment. Use the names according with your Company/Department requirements. 
**Important: Use your own Ip Address Range, and be sure that you are not overlaying your On-Premise IP Address Range Network. The IP Address used in this example was used for this specif use case.**

<p align="center">
  <img src="Images/picture17.png" alt="drawing" width="600"/>
</p>

* 3.5 Check back to the Github-Network Resource Group that you have the following resources deployed at your environment. 

<p align="center">
  <img src="Images/picture18.png" alt="drawing" width="600"/>
</p>


## Step 4: Create the Network Gateways to connect the both Regions (In this scenario, EAST-US2 and CENTRAL-US)

Enabling geo-replication between managed instances and their VNets: (this reference can be found [here](https://docs.microsoft.com/en-us/azure/sql-database/sql-database-auto-failover-group#enabling-geo-rCreeplication-between-managed-instances-and-their-vnets)

When you set up a failover group between primary and secondary managed instances in two different regions, each instance is isolated using an independent virtual network. To allow replication traffic between these VNets ensure these prerequisites are met:
1.	The two managed instances need to be in different Azure regions.
2.	The two managed instances need to be the same service tier, and have the same storage size.
3.	Your secondary managed instance must be empty (no user databases).
4.	The virtual networks used by the the managed instances need to be connected through a VPN Gateway or Express Route. When two virtual networks connect through an on-premises network, ensure there is no firewall rule blocking ports 5022, and 11000-11999. Global VNet Peering is not supported.
5.	The two managed instance VNets cannot have overlapping IP addresses.
6.	You need to set up your Network Security Groups (NSG) such that ports 5022 and the range 11000~12000 are open inbound and outbound for connections from the other managed instanced subnet. This is to allow replication traffic between the instances
Important
Misconfigured NSG security rules leads to stuck database copy operations.
7.	The secondary instance is configured with the correct DNS zone ID. DNS zone is a property of a managed instance and its ID is included in the host name address. The zone ID is generated as a random string when the first managed instance is created in each VNet and the same ID is assigned to all other instances in the same subnet. Once assigned, the DNS zone cannot be modified. Managed instances included in the same failover group must share the DNS zone. You accomplish this by passing the primary instance's zone ID as the value of DnsZonePartner parameter when creating the secondary instance.

* 4.1 In order to do that, let’s go through the process and follow the step by step as you can see on the next process as follow. Search for `Virtual Network Gateway`

<p align="center">
  <img src="Images/picture19.png" alt="drawing" width="600"/>
</p>

* 4.2 Click in `Create Virtual Network Gateway`

<p align="center">
  <img src="Images/picture20.png" alt="drawing" width="600"/>
</p>

* 4.3 Check the informations used on the paramters described on the picture below. Please check this [table](https://docs.microsoft.com/en-us/azure/vpn-gateway/vpn-gateway-about-vpngateways#benchmark) to undertand which VPN Gateway SKU should be used in your enviroment. In this example, the VPN Gateway SKU VpnGw3AZ was used due the Aggregate Throughput requirement for 1.25 Gbps and the Zone-Redudant requirement. After complete the parameters, click in `Review + Create`
**important** please use the IP Address that you assigned to your environment.  **be sure that you dont have any IP Address with your on-premise networking.** `Click on Review + Create`.

<p align="center">
  <img src="Images/picture21.png" alt="drawing" width="600"/>
</p>


* 4.4 As soon as you have the Validation Passed, click in `Create`

<p align="center">
  <img src="Images/picture22.png" alt="drawing" width="600"/>
</p>

* 4.5 This process will take around 30 minutes to be completed, in meanwhile, let’s create the VPN GW for another region, EAST-US2 at this time. 
Back to the Virtual Network Page space and click `+ Add` to create the new VPN GW

<p align="center">
  <img src="Images/picture23.png" alt="drawing" width="600"/>
</p>


* 4.6 Check the informations used on the paramters described on the picture below. Please check this [table](https://docs.microsoft.com/en-us/azure/vpn-gateway/vpn-gateway-about-vpngateways#benchmark) to undertand which VPN Gateway SKU should be used in your enviroment. In this example, the VPN Gateway SKU VpnGw3AZ was used due the Aggregate Throughput requirement for 1.25 Gbps and the Zone-Redudant requirement. After complete the parameters, click in `Review + Create`
**important** please use the IP Address that you assigned to your environment.  **be sure that you dont have any IP Address with your on-premise networking.**

<p align="center">
  <img src="Images/picture24.png" alt="drawing" width="600"/>
</p>

* 4.7 As soon as the validation passed, click in `Create`

<p align="center">
  <img src="Images/picture25.png" alt="drawing" width="600"/>
</p>

* 4.8 The process to create the VPN Gateway will take around 30 minutes to complete. 
As soon as both VPN Gateways were created, back to the Github-Network Resource group (or the name that you used for the Network), select `gateway-centralus` and click on the `VPN Gateway Central`, click on `Connections` and `+ Add` in order to create a new connection between both VPN GWs (CENTRAL-US and EAST-US2) that was just created on the steps above.

<p align="center">
  <img src="Images/picture29.png" alt="drawing" width="600"/>
</p>

<p align="center">
  <img src="Images/picture26.png" alt="drawing" width="600"/>
</p>

* 4.9 Add a connection named `centralus-to-eastus2` with the followng parameters. Chose for the second virtual network gateway and select the vpn gateway available (that was just created). Fill all parameters and click `OK`

<p align="center">
  <img src="Images/picture27.png" alt="drawing" width="600"/>
</p>

* 4.10 back to the Github-Network Resource group (or the name that you used for the Network), select `gateway-eastus2` and click on the `VPN Gateway Central`, click on `Connections` and `+ Add` in order to create a new connection between both VPN GWs (CENTRAL-US and EAST-US2) that was just created on the steps above.

<p align="center">
  <img src="Images/picture29.png" alt="drawing" width="600"/>
</p>

<p align="center">
  <img src="Images/picture30.png" alt="drawing" width="600"/>
</p>

<p align="center">
  <img src="Images/picture31.png" alt="drawing" width="600"/>
</p>


* 4.11 Add a connection named `eastus2-to-centralus` with the followng parameters. Chose for the second virtual network gateway and select the vpn gateway available (that was just created). Fill all parameters and click `OK`

<p align="center">
  <img src="Images/picture32.png" alt="drawing" width="600"/>
</p>


* 4.12 Back to the Network Resource Group, in this example, named `Github-Network` and check that the both Connections were created. At this step you have created the following Network compoments.

<p align="center">
  <img src="Images/picture33.png" alt="drawing" width="600"/>
</p>


## Step 5: Create the SQL Managed Instance on the first region

* 5.1 Click on Resource Groups and select the Github-SQLMI Resource Group. Then, click in `+ Add`.

<p align="center">
  <img src="Images/picture34.png" alt="drawing" width="600"/>
</p>

<p align="center">
  <img src="Images/picture35.png" alt="drawing" width="600"/>
</p>


* 5.2 Search for `SQL Managed Instances` and click to start to deploy this service

<p align="center">
  <img src="Images/picture36.png" alt="drawing" width="600"/>
</p>

* 5.3 Click on Create SQL Managed Instances

<p align="center">
  <img src="Images/picture37.png" alt="drawing" width="600"/>
</p>

* 5.4 Add the SQL-MI parameters as described on the picture below. 

<p align="center">
  <img src="Images/picture38.png" alt="drawing" width="600"/>
</p>


* 5.5 On the Price Tier, chose the options that are aligned with your Business Requirements, once again, check the Business Critical vs General Proposal information mentioned on the steps above. Click on `Apply` and then `Create`.

<p align="center">
  <img src="Images/picture39.png" alt="drawing" width="600"/>
</p>


* 5.6 This process will take around 4 to 6 hours to be completed. As soon as you have the confirmation, click on the resource group and be sure that the Deployments parameters shows now 1 Succeeded.
Click on the SQL managed instance that you just created:

<p align="center">
  <img src="Images/picture40.png" alt="drawing" width="600"/>
</p>


## Step 6: SSMS Connection to your SQL Managed Instance
In this Step, a Windows Jumpbox will be created on the same vnet, separated subnet, so you will be able to access the SQL Managed Instance using SSMS.

* 6.1 Click on the SQL managed instance that you just created:

<p align="center">
  <img src="Images/picture40.png" alt="drawing" width="600"/>
</p>

* 6.2 Click on the Quick start:
Create Virtual Machine with the latest SSMS and attach it to the virtual network
You would run PowerShell code shown on your quick start, either in Azure Cloud Shell or from your computer to automate this step. In this deployment example, run this from the Azure Cloud Shell:

<p align="center">
  <img src="Images/picture41.png" alt="drawing" width="600"/>
</p>

* 6.3 As soon as you click on Cloud Shell, If you don’t have the Storage created yet, select your subscription and the storage will be created automatcliy.

<p align="center">
  <img src="Images/picture42.png" alt="drawing" width="600"/>
</p>

* 6.4 As soon the Storage was created, select Powershell and Paste the code under the step 1 below. The only change that have to be made is the password. Copy and Paste the Step 1 to your Powershell and adjust the Passowrd (as mentioned on the previous step)

<p align="center">
  <img src="Images/picture43.png" alt="drawing" width="600"/>
</p>

<p align="center">
  <img src="Images/picture44.png" alt="drawing" width="600"/>
</p>

* 6.5 Back to the Network Resource Group, in this example, named `Github-Network` and click on the Virtual Machine that was just created.

<p align="center">
  <img src="Images/picture45.png" alt="drawing" width="600"/>
</p>

* 6.6 Click on the Jumpbox Virtual Machine and `connect` the RDP. click on `Download RDP File`.

<p align="center">
  <img src="Images/picture45.png" alt="drawing" width="600"/>
</p>

<p align="center">
  <img src="Images/picture46.png" alt="drawing" width="600"/>
</p>

<p align="center">
  <img src="Images/picture47.png" alt="drawing" width="600"/>
</p>

* 6.7 Enter the Credentials that you used on the previous Steps to access the Virtual Machine and access the SSMS (SQL Server Management Studio). Start SQL Server Management Studio. The first time you run SSMS, the Connect to Server window opens. If it doesn't open, you can open it manually by selecting Object Explorer > Connect > Database Engine.

<p align="center">
  <img src="Images/picture50.png" alt="drawing" width="600"/>
</p>

* 6.8 Connect the SQL-MI from the SSMS using the login that you created, the password and the Server name that can be found here:

<p align="center">
  <img src="Images/picture51.png" alt="drawing" width="600"/>
</p>

<p align="center">
  <img src="Images/picture52.png" alt="drawing" width="600"/>
</p>

<p align="center">
  <img src="Images/picture53.png" alt="drawing" width="600"/>
</p>

* 6.7 The connection below shows thart you are connected to the SQL-MI.

<p align="center">
  <img src="Images/picture54.png" alt="drawing" width="600"/>
</p>

* 6.8 Create a database
Create a database named TutorialDB by following the below steps. This step by step can be found [here](https://docs.microsoft.com/en-us/sql/ssms/tutorials/connect-query-sql-server?view=sql-server-2017) as well.
Right-click your server instance in Object Explorer, and then select New Query:

<p align="center">
  <img src="Images/picture56.png" alt="drawing" width="600"/>
</p>

* 6.9 Into the query window, paste the following T-SQL code snippet:

```
USE master
GO
IF NOT EXISTS (
   SELECT name
   FROM sys.databases
   WHERE name = N'TutorialDB'
)
CREATE DATABASE [TutorialDB]
GO
```
* 6.10 To execute the query, select Execute (or select F5 on your keyboard).

<p align="center">
  <img src="Images/picture58.png" alt="drawing" width="600"/>
</p>

* 6.11 After the query is complete, the new TutorialDB database appears in the list of databases in Object Explorer. If it isn't displayed, right-click the Databases node, and then select Refresh.

<p align="center">
  <img src="Images/picture59.png" alt="drawing" width="600"/>
</p>

* 6.12 Create a table in the new database
In this section, you create a table in the newly created TutorialDB database. Because the query editor is still in the context of the master database, switch the connection context to the TutorialDB database by doing the following steps:
In the database drop-down list, select the database that you want, as shown here:

<p align="center">
  <img src="Images/picture60.png" alt="drawing" width="600"/>
</p>

* 6.13 Paste the following T-SQL code snippet into the query window, select it, and then select Execute (or select F5 on your keyboard).

```
-- Create a new table called 'Customers' in schema 'dbo'
-- Drop the table if it already exists
IF OBJECT_ID('dbo.Customers', 'U') IS NOT NULL
DROP TABLE dbo.Customers
GO
-- Create the table in the specified schema
CREATE TABLE dbo.Customers
(
   CustomerId        INT    NOT NULL   PRIMARY KEY, -- primary key column
   Name      [NVARCHAR](50)  NOT NULL,
   Location  [NVARCHAR](50)  NOT NULL,
   Email     [NVARCHAR](50)  NOT NULL
);
GO
```

* 6.14 You can either replace the existing text in the query window or append it to the end. To execute everything in the query window, select Execute. To execute a portion of the text, highlight that portion, and then select Execute.

<p align="center">
  <img src="Images/picture61.png" alt="drawing" width="600"/>
</p>

* 6.15 After the query is complete, the new Customers table is displayed in the list of tables in Object Explorer. If the table isn't displayed, right-click the TutorialDB > Tables node in Object Explorer, and then select Refresh.

<p align="center">
  <img src="Images/picture62.png" alt="drawing" width="600"/>
</p>

* 6.16 Insert rows into the new table
Insert some rows into the Customers table that you created previously. To do so, paste the following T-SQL code snippet into the query window, and then select Execute:

```
-- Insert rows into table 'Customers'
INSERT INTO dbo.Customers
   ([CustomerId],[Name],[Location],[Email])
VALUES
   ( 1, N'Orlando', N'Australia', N''),
   ( 2, N'Keith', N'India', N'keith0@adventure-works.com'),
   ( 3, N'Donna', N'Germany', N'donna0@adventure-works.com'),
   ( 4, N'Janet', N'United States', N'janet1@adventure-works.com')
GO
```

* 6.17 Query the table and view the results
The results of a query are visible below the query text window. To query the Customers table and view the rows that were previously inserted, follow these steps:
Paste the following T-SQL code snippet into the query window, and then select Execute:

```
-- Select rows from table 'Customers'
SELECT * FROM dbo.Customers;
```

* 6.18 From Github-SQMI Resource Group, you will be able to see that the TutorialDB is deployed at EastUS2 Region.

<p align="center">
  <img src="Images/picture64.png" alt="drawing" width="600"/>
</p>

# Step 7: Create a Second SQL Managed Instance. 

* 7.1 So, after created the first instance, network, gateway and tested creating a first DB using SSMS, it’s time to create the second Instance on the Pair region, in our case, CENTRAL-US. Click `+ Add`

<p align="center">
  <img src="Images/picture65.png" alt="drawing" width="600"/>
</p>

* 7.2 Search for `SQL Managed Instance`

<p align="center">
  <img src="Images/picture66.png" alt="drawing" width="600"/>
</p>

* 7.3 Click in `Create`

<p align="center">
  <img src="Images/picture67.png" alt="drawing" width="600"/>
</p>

* 7.4 It’s **important** that the Collation and Time zone are the same on booth instances. In case of Collation, check with your developers which Collation are they using on the on-premise implementation. During the SQL Managed Instances Requirments, we saw that the same DNS needs to be use. To garantee that you wont have any problem on that space, be sure to click on `use this instance as Failover Group Secondary`

<p align="center">
  <img src="Images/picture68.png" alt="drawing" width="600"/>
</p>

* 7.5 The price Tier that you will use for the Second Instance needs to be exaclty the same as the first one.

<p align="center">
  <img src="Images/picture69.png" alt="drawing" width="600"/>
</p>

7.6 After around 4 to 6 hours, the second Instance will be ready for use. As you can see, at this moment there are 2 Managed Instated created (EAST-US2 and CENTRAL-US)

<p align="center">
  <img src="Images/picture70.png" alt="drawing" width="600"/>
</p>


# Step 8: Check the connectivity to the Second Managed Instance withthe SSMS installed in another Region - E2E Architecture Test

* 8.1 In this Step, we will test the end to end Architecture Copy the address of the Host that you just created. Host for CENTRAL-US Managed Instance.

<p align="center">
  <img src="Images/picture71.png" alt="drawing" width="600"/>
</p>

* 8.2 In this case, Let’s test the end to end connectivity. Basically, now we have to test the connection between the Virtual Machine that we created @ EASTus2 Region towards to the CENTRAL-US Region – using the second Instanced created. Back to Github-Network Resource Group, click on Jumpbox.

<p align="center">
  <img src="Images/picture72.png" alt="drawing" width="600"/>
</p>

* 8.3 Click on Connect and Download the RDP file.

<p align="center">
  <img src="Images/picture73.png" alt="drawing" width="600"/>
</p>

* 8.4 In order to connect to the Second SQL Managed Instance (Located in CENTRAL-US), use the 

<p align="center">
  <img src="Images/picture74.png" alt="drawing" width="600"/>
</p>


* 8.5 As you can see, no DB created yet. 

<p align="center">
  <img src="Images/picture75.png" alt="drawing" width="600"/>
</p>


# Step 9: Create the Failover Group

* 9.1 Creation of Failover Group for the Managed Instance. Back to Resource Group created for the SQL-MI `Github_SQLMI` in this example. And Click on the Primary SQL-MI.

<p align="center">
  <img src="Images/picture76.png" alt="drawing" width="600"/>
</p>

* 9.2 Under Settings, click on `Instance Failover` and Click on `Add group`

<p align="center">
  <img src="Images/picture77.png" alt="drawing" width="600"/>
</p>

* 9.3 You will see that the First Instance was automatic selected and now you have to select the `Secondary Managed Instance` And select the Read/Write failover group as Manual or Automatic (Depending of your Business Requirement) - In this case, we selected Manual so I can chose when to make the Failover between the regions. And then, click on `Create`.

<p align="center">
  <img src="Images/picture78.png" alt="drawing" width="600"/>
</p>

* 9.4 As you can see on the image below, the Failover Group was created between the Regions.

<p align="center">
  <img src="Images/picture79.png" alt="drawing" width="600"/>
</p>

* 9.5 Back to the Resouce Group named `Github-SQLMI` and you will see that the TutorialDB that you created to the First region was replicated to the Secnd Region as well.

<p align="center">
  <img src="Images/picture80.png" alt="drawing" width="600"/>
</p>

