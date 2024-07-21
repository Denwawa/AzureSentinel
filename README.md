# Microsoft Azure Sentinel Attack Map

### Summary
This is a home lab project I worked on in ordder to better understand Microsoft's Azure Sentinel SIEM. I included all of the steps taken to complete this lab and a high level overview of the project can see in the video linked below!

### The following skills were utilized:
- Configuration & Deployment of Azure resources such as virtual machines, Log Analytics Workspaces, and Azure Sentinel
- Hands-on experience and working knowledge of a SIEM Log Management Tool (Microsoft's Azure Sentinel)
- Understand Windows Security Event logs
- Utilization of KQL to query logs
- Display attack data on a dashboard with Workbooks (World Map)

### Tools Utilized:

1. Microsoft Azure
2. Azure Sentinel
3. Kusto Query Language (KQL - Used to build world map)
4. Network Security Groups (Layer 4/3 Firewall in Azure)
5. Remote Desktop Protocol (RDP)
6. 3rd Party API: [ipgeolocation.io](https://ipgeolocation.io/)
7. Custom [Powershell Script](https://github.com/joshmadakor1/Sentinel-Lab/blob/main/Custom_Security_Log_Exporter.ps1) written by Josh Madakor



### Overview:
![](images/SIEM%20Lab.png)

## Step 1: Created a Honeypot Virtual Machine
- Signed into portal.azure.com
- Searched for "virtual machines" 
- Selected Create > Azure virtual machine

![](images/create_azure_vm.png)

### Virtual Machine Details
The following information is what I used to setup the initial virtual machine

![](images/vm1.png)

### Networking
#### Network interface
I edited the NIC network security group to allow all incoming traffick to into the virtual machine. 
- NIC network security group: Advanced > Create new
- I Removed the Default Inbound rules. 
- The Destination port ranges needed to be changed to, "*", as this is a wildcard allowing ALL ports to be opened.
- Also set the Priority: 100 (low)
![](images/network_sec_grp.png)

> Configuring the firewall to allow traffic from anywhere will make the VM easily discoverable.

## Step 3: Created a Log Analytics Workspace
- Searching for "Log analytics workspaces" and created the Log Analytics Workspace and ensured it had the same resource group as VM (honeypotlab)
![](images/log_an_wrk.png)

> The Windows Event Viewer logs will be ingested into Log Analytics workspaces in addition to custom logs with geographic data to map attacker locations.

## Step 4: Configured Microsoft Defender for Cloud
- In "Microsoft Defender for Cloud", you will need to change the settings in order to allow the server to ingest logs into the Log ANalytics Workspace.
- Scroll dow "Environment settings" > subscription name > log analytics workspace name (log-honeypot)

![](images/mcrsft_dfndr.png)

#### Settings | Defender plans
- You will need to turn on Servers in the Defender Plan, and leave SQL servers ON machines OFF 

![](images/defender_plans.png)

#### Settings | Configured Data collection
- Select "All Events" to grab all events from EventViewer on the Virtual Machine

## Step 5: Connected Log Analytics Workspace to Virtual Machine
- Navigated back to Log Analytics workspaces and under Virtual Machines, selected the virtual machine and connected.

![](images/log_an_vm_connect.png)

## Step 6: Configured Microsoft Sentinel
- Searched for "Microsoft Sentinel" and created the enviornment. Also linked the Log Analytics workspace to Microsoft Sentinel

![](images/sentinel_log.png)

## Step 7: Disabled the Firewall in Virtual Machine
- Connected to the Virtual Machine using the public IP address and on my desktop through Windows RDP. Signed in using the credentials created and disabled Windows Firewall. Ensured it was disabled by using the 'ping -t' command on my desktop, ensuring I was able to reach the external IP address of the VM.

![](images/defender_off.png)

## Step 8: Scripting the Security Log Exporter
- I needed to link the ipgeolocation api to the desktop in order to create a log file I could ingest into Azure Sentinel. I was able to find a script that allowed me to do so. I changed the API Key in the powershell script and saved it to the VM's desktop. After saved, I was able to run the script.

![](images/powershell_script.png)

> The script will export data from the Windows Event Viewer to then import into the IP Geolocation service. It will then extract the latitude and longitude and then create a new log called failed_rdp.log in the following location: C:\ProgramData\failed_rdp.log

## Step 9: Created Custom Log in Log Analytics Workspace
In order to map the geolocation of the attackers, I needed to create a custom log to import the data from the IP Geolocation service into Azure Sentinel.
- First I would need sample logs on the VM in order to create these custom logs. From the script, they could be found on C:\ProgramData and I was able to grab the logs.
- Back in Azure, in Log Analytics workspaces. Under Tables, I created a new MMA-based custom log
- 
![](images/MMA_Based_Logs.JPG)


![](images/custom_log.png)


## Step 10: Query the Custom Log
- Ensured the logs were showing in Log Analytics through a query.

![](images/failed_rdp_with_geo.png)

## Step 11: Extract Fields from Custom Log 
> The RawData within a log contains information such as latitude, longitude, destinationhost, etc. Data will have to be extracted to create separate fields for the different types of data
- Right click any of the log results
- Select **Extract fields from 'FAILED_RDP_WITH_GEO_CL'**
- Highlight ONLY the value after the ":" 
- Name the **Field Title** the name of the field of the value
- Under **Field Type** select the appropriate data type
- Hit **Extract**
- If the search results data looks good click the **Save extraction** button
- Do this for ALL available fields in RawData
> NOTE: If one of the search results is not correct select **Modify this highlight** (upper right corner of result) and highlight the correct value. Otherwise go to **Custom logs > Custom fields** Accept warning of unsaved edits and delete field. Redo extraction for deleted field.

![](images/data_extraction.png)

## Step 12: Map Data in Microsoft Sentinel
- Go to Microsoft Sentinel to see the Overview page and available events
- Click on **Workbooks** and **Add workbook** then click **Edit**
- Remove default widgets (Three dots > Remove)
- Click **Add > Add query** 
- Copy/Paste the following query into the query window and **Run Query**

```KQL
FAILED_RDP_WITH_GEO_CL | summarize event_count=count() by sourcehost_CF, latitude_CF, longitude_CF, country_CF, label_CF, destinationhost_CF
| where destinationhost_CF != "samplehost"
| where sourcehost_CF != ""
```
> Kusto Query Language (KQL) - Azure Monitor Logs is based on Azure Data Explorer. The language is designed to be easy to read and use with some practice writing queries and basic guidance.

- Once results come up click the **Visualization** dropdown menu and select **Map**
- Select **Map Settings** for additional configuration
#### Layout Settings
- **Location info using** > Latitude/Longitude
- **Latitude** > latitude_CF
- **Longitude** > longitude_CF
- **Size by** > event_count
#### Color Settings
- **Coloring Type:** Heatmap 
- **Color by** > event_count
- **Aggregation for color** > Sum of values
- **Color palette** > Green to Red
#### Metric Settings
- **Metric Label** > label_CF
- **Metric Value** > event_count
- Select **Apply** button and **Save and Close**
- Save as "Failed RDP World Map" in the same region and under the resource group (honeypotlab)
- Continue to refresh map to display additional incoming failed RDP attacks
> NOTE: The map will only display Event Viewer's failed RDP attempts and not all the other attacks the VM may be receiving.

![](images/failed_rdp_map.png)

> Event Viewer Displaying Failed RDP logon attemps. EventID 4625

![](images/event_viewer.png)

> Custom Powershell script parsing data from 3rd party API

![](images/rdp_script.png)
