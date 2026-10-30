# Lab 3: Splunk SIEM and Log Analysis

 
![Splunk](https://img.shields.io/badge/Splunk-Enterprise-000000?logo=splunk&logoColor=white)
![Azure](https://img.shields.io/badge/Microsoft%20Azure-Free%20Account-0078D4?logo=microsoftazure&logoColor=white)
![Ubuntu](https://img.shields.io/badge/Ubuntu-22.04%20LTS-E95420?logo=ubuntu&logoColor=white)
![Windows Server](https://img.shields.io/badge/Windows%20Server-2025-0078D6)
![PowerShell](https://img.shields.io/badge/PowerShell-Scripting-5391FE?logo=powershell&logoColor=white)
![Security+](https://img.shields.io/badge/CompTIA-Security%2B-C8202F)
![CySA+](https://img.shields.io/badge/CompTIA-CySA%2B-C8202F)
![Cost](https://img.shields.io/badge/Cost-%240-brightgreen)
![Time](https://img.shields.io/badge/Time-4--6%20hours-blue)


## 🎥 Walkthrough

Turn raw logs into powerful security insights and learn how to detect suspicious activity with Splunk!

[Loom Video: Splunk SIEM and Log Analysis Lab](https://www.loom.com/share/73ad5c648e6b4cf5b21741b42d2a606f)

## 📋 Overview

In this lab, the Windows Server virtual machine (DC01), configured as the domain controller, forwards its event logs to a separate Ubuntu virtual machine running Splunk Enterprise. Splunk indexes the incoming events and makes them searchable through its web interface. Using this interface, I developed search queries, created a monitoring dashboard, and configured an automated alert to detect relevant events.

| Skill | Real-world application |
|---|---|
| Deploy Splunk and configure a data input | Every Splunk deployment starts with getting data in. The Universal Forwarder is how most enterprises feed logs to Splunk. |
| Navigate the Splunk interface | Search, dashboards, alerts, and reports are table stakes for any SOC role. |
| Write SPL searches | SPL is how you ask Splunk questions, and it separates analysts who find threats from analysts who stare at dashboards. |
| Build security dashboards | Visualize login failures over time, top source IPs, and failed authentication by user at a glance. |
| Identify failed login attempts | One of the most common investigations: telling normal user error from a brute force attack. |
| Build an automated alert | Splunk fires an alert when your conditions are met instead of waiting for a human to notice. |
| Search for account lockout events | A trail of lockouts can indicate a password spray attack in progress. |

> Forward Windows Event Logs from an Active Directory domain controller into Splunk, then search, visualize, and alert on them the way a SOC analyst would.
 
## 🧩 Architecture
 
<img width="995" height="1024" alt="Splunk SIEM   Log Architecture" src="https://github.com/user-attachments/assets/b30e0508-5df8-4136-9f07-91250de0f2c5" />

> **NSG rules:** port 22 (SSH) and port 8000 (web UI) open to my IP only; port 9997 (forwarder) open to the VNet only.
 

 
## 🎯 Business Context

A medium-sized organization generates millions of log events every day: Windows Event Logs from workstations, authentication logs from Active Directory, firewall logs, web server access logs, and cloud resource logs. Without a SIEM, those logs sit in separate systems and nobody can search across them, correlate events, or identify patterns that indicate an attack.
 
The SIEM is the security operations center's primary tool. When an alert fires, the SOC analyst opens the SIEM to understand what happened, when, from where, and what was affected. Splunk is the most widely deployed commercial SIEM, so learning it in a free lab gives a concrete, demonstrable skill that appears on job descriptions for almost every security operations role.
 
| Role | How this lab applies |
|---|---|
| SOC Analyst Tier 1 | Monitoring dashboards for alerts, searching logs for suspicious activity, escalating findings |
| SOC Analyst Tier 2-3 | Building detection rules, correlating events across data sources, threat hunting |
| Cloud Security Engineer | Microsoft Sentinel and AWS Security Hub use the same SIEM concepts, and this lab teaches the mental model |
| Incident Responder | Searching logs during an active incident, building a timeline of events, identifying scope of compromise |
 
## ⚙️ Prerequisites/Configurations

<details>
<summary><b>1. Deploy the Windows VM</b></summary>

This is the same Windows Server VM built in Lab 1, which becomes DC01 and the log source for this lab.

1. Go to [azure.microsoft.com/free](https://azure.microsoft.com/free) and create a free account
2. Sign in to [portal.azure.com](https://portal.azure.com)
3. Search **Virtual machines** and click **Create**
4. Use the settings below
5. Click **Review + Create**, then **Create**

| Setting | Value |
|---|---|
| Region | East US |
| Image | Windows Server 2025 Datacenter, Gen2 |
| Size | Standard_B2s (2 vCPU, 4 GB RAM) |
| Authentication | Password |
| Public inbound ports | Allow RDP (3389) |
| OS disk | Standard SSD |

> **Tip:** Stop the VM when not in use. A B2s VM costs roughly $0.05/hour, and stopping (not deleting) pauses compute billing so the $200 free credit lasts much longer.

**Enable copy and paste over RDP.** Download the RDP file from the Azure portal (**Connect → Download RDP File**) and open it with the native Remote Desktop app. Before connecting, click **Show Options → Local Resources** and make sure **Clipboard** is checked. The browser-based console has very limited clipboard sharing.

<img width="1256" height="648" alt="Windows VM Deployed" src="https://github.com/user-attachments/assets/cb1bd509-be8d-4491-9624-58af291a65d7" />

>Windows test VM deployment complete. Deployed CreateVirtualMachine-testVM into the WindowsVM resource group. This Windows machine acts as the monitored endpoint, generating the event logs that are forwarded to the Splunk instance for ingestion, searching and alerting.

</details>

<details>
<summary><b>2. Deploy the Ubuntu VM</b></summary>

In the Azure portal, search **Virtual machines**, click **Create**, and use these settings:

| Setting | Value |
|---|---|
| OS | Ubuntu 22.04 LTS (free tier eligible) |
| Size | Standard_B2s (2 vCPU, 4 GB RAM minimum; Splunk requires at least 4 GB) |
| Disk | 30 GB minimum |
| Inbound NSG ports | 8000 (Splunk web UI), 9997 (forwarder input), 22 (SSH) 

<img width="1402" height="596" alt="Ubuntu VM Deployed" src="https://github.com/user-attachments/assets/eb17f9c6-e6d9-4623-a358-b35b16920f4f" />

>Splunk VM deployment complete. Provisioned the CreateVirtualMachine-SplunkVM deployment in the rg-splunk resource group through the Azure portal. This VM hosts the Splunk Enterprise instance used as the SIEM for the log monitoring lab. A dedicated resource group keeps the SIEM infrastructure isolated and easy to manage or tear down after the lab.


**🔒 Inbound Network Security Group (NSG) Rules**

<img width="1424" height="721" alt="Inbound NSG Rules" src="https://github.com/user-attachments/assets/b745fe60-0e8e-402b-908b-2b0858d19959" />

> 
> 
> Allow-SplunkWebUI-MyIP (TCP/8000): Restricts access to the Splunk Web management interface strictly to my public IP address, preventing unauthorized external scanning and brute-force attacks while allowing dashboard management and SPL query execution.
> 
> Allow-SplunkForwarder-WindowsVM (TCP/9997): Permits secure log transmission from the Windows VM subnet to the Splunk receiver, overriding Azure's default deny-all policy to ensure continuous data ingestion.


</details>

<details>
<summary><b>3. Establish VNet Peering between Ubuntu VM and Windows VM</b></summary>

- a. Go to portal.azure.com → **Virtual Networks** → select the Splunk VM's VNet
- b. Click **Peerings → Add**
- c. Name the links: this VNet link `splunk-vnet-to-ad-vnet`, remote VNet link `ad-vnet-to-splunk-vnet`
- d. Select “Allow virtual network access” is enabled in both directions, so the Windows machines can reach the Splunk server over private IPs.
- e. Under **Remote virtual network**, select the Windows VM's VNet, leave other settings default, and click **Add**
- f. Azure creates both directions; wait until both show **Connected**
 
<img width="545" height="777" alt="VNet Peering Configuration" src="https://github.com/user-attachments/assets/e7631635-9d28-4640-a9d3-f4b81fc56014" />

>Configuring VNet peering between the Splunk and AD networks. On the Add peering page for vnet-eastus-2, I created a bidirectional peering to vnet-eastus-1 (the WindowsVM resource group) in the same subscription. Both peering links are configured in a single step:

<img width="1421" height="710" alt="Vnet Peering" src="https://github.com/user-attachments/assets/3fa2828d-9a23-474c-b9ca-d76256eddc45" />

> VNet peering verified. The Peerings page for vnet-eastus-2 (the Splunk network) shows the link ad-vnet-to-splunk-vnet pointing to the remote network vnet-eastus-1 (the AD/Windows network).

<img width="1024" height="710" alt="VNet Peering" src="https://github.com/user-attachments/assets/30788453-7c61-4b86-a5df-d9dfc9e57123" />

> VNet peering: The two VNets use non-overlapping address spaces, which peering requires. Traffic flows in both directions over private IP connectivity.

</details>

<details>
<summary><b>4. SSH into the Linux VM</b></summary>

**Mac or Linux** (SSH is built in). Open Terminal.

Fix the key file permissions first, or SSH refuses to connect with an `UNPROTECTED PRIVATE KEY FILE` warning:

```bash
cd ~/Downloads
chmod 400 yourkey.pem
```

Then connect:

```bash
ssh -i yourkey.pem <admin-username>@<YOUR_VM_PUBLIC_IP>
```

Replace `<admin-username>` with the admin username set when creating the VM and `<YOUR_VM_PUBLIC_IP>` with the public IP from the Azure portal. Type `yes` at the host fingerprint prompt.

</details>

<details>
<summary><b>5. Install and start Splunk (Commands 1-4)</b></summary
                                                           
Go to the [Splunk Enterprise Download page](https://www.splunk.com/en_us/download/splunk-enterprise.html) to get the latest version. You’ll need to create a free Splunk account (username and password) to download.

Run these inside the SSH session, in order.

**Command 1: Download the installer.** 

- Select your operating system (Linux) and click Copy wget link for the file type you prefer (.tgz or .deb).

- The wget link is for Linux because we connected to the Ubuntu VM over SSH, so the download runs directly on the server instead of on your local machine.

wget is a command-line download tool that comes preinstalled on most Ubuntu images. The -O flag sets the output filename, and the quoted URL is the direct download link from Splunk. This command downloads the Splunk .deb installer from Splunk’s servers and saves it in your current directory.

`wget` downloads a file, and `-O` sets the output filename

```bash
wget -O splunk-10.6.1.1-b2ae2962645c-linux-amd64.deb "https://download.splunk.com/products/splunk/releases/10.6.1.1/linux/splunk-10.6.1.1-b2ae2962645c-linux-amd64.deb"
```

> Note: If this returns a 404 error, Splunk has released a newer version. Log in to splunk.com, go to Free Trials and Downloads, select Linux `.deb`, and copy the `wget` command shown on the download page.

**Command 2: Install the package.** 

`dpkg` the Debian/Ubuntu package manager, and the -i flag means install. sudo runs the command as root because installing software system-wide requires elevated permissions. This command unpacks the .deb file and installs Splunk into /opt/splunk/. You’ll see output for each installation step.

`dpkg -i` installs the package into `/opt/splunk/`. A warning about a missing Python 3.7 path is expected and harmless on Ubuntu 22.04.

```bash
sudo dpkg -i splunk-10.6.1.1-b2ae2962645c-linux-amd64.deb
```

> Note: You may see a warning about a missing Python 3.7 path. This is expected on Ubuntu 22.04 and harmless, because Splunk 10.x ships with its own Python.

**Command 3: Start Splunk and accept the license.** 

`--run-as-root` is required on newer Splunk versions when running as root. You will be prompted to create an admin username and password for the web UI. Save these, since you’ll use them to log in.

```bash
sudo /opt/splunk/bin/splunk start --accept-license --run-as-root
```

**Command 4: Enable start on boot.** 

This command registers Splunk as a system service so it starts automatically whenever the VM reboots. Without it, you would need to SSH into the VM and run the start command manually after every restart.

```bash
sudo /opt/splunk/bin/splunk enable boot-start
```

</details>

<details>
<summary><b>6. Access the web UI</b></summary>

Open this in a browser on the local machine, using the public IP from the VM's **Overview** page in the Azure portal:

```text
http://<YOUR_VM_PUBLIC_IP>:8000
```
 
#### If the browser shows "This site can't be reached"
 
A timeout on port 8000 is almost always a missing NSG rule. Work through these checks in order.
 
**Check 1: Add the port 8000 NSG rule (most common cause).**
 
1. Go to portal.azure.com → your Splunk VM → **Networking → Inbound port rules**
2. If no rule allows port 8000, click **Add inbound port rule** and set:
   
| Field | Value |
|---|---|
| Name | Allow-SplunkWebUI-MyIP |
| Source | My IP address |
| Destination port | 8000 |
| Protocol | TCP |
| Action | Allow |
| Priority | 310 |
 
3. Save the rule and try the browser again
**Check 2: Confirm Splunk is running.**
 
```bash
sudo /opt/splunk/bin/splunk status
# Expected: splunkd is running
```
 
If it says stopped, run the start command again:
 
```bash
sudo /opt/splunk/bin/splunk start --accept-license --run-as-root
```
 
**Check 3: Confirm Splunk is listening on port 8000.**
 
```bash
sudo ss -tlnp | grep 8000
# Expected: a line showing 0.0.0.0:8000 with splunkd in the process column
```
 
If nothing returns, Splunk is not running. If the line shows, Splunk is up and the issue is the NSG rule from Check 1.
 
**Check 4: Confirm the correct public IP.** Azure reassigns public IPs when a VM is stopped and restarted, so always get the current IP from **VM → Overview**. To stop it changing, go to **VM → Public IP address → Configuration** and set **Assignment** from Dynamic to **Static**.
 
**Check 5: Add the port 9997 inbound rule for the forwarder.** If the VMs are in different VNets or subnets, forwarder traffic is blocked by default. Find the Windows VM's private IP (10.x.x.x) on its Overview page, then on the Splunk VM go to **Networking → Inbound port rules → Add inbound port rule**:
 
| Field | Value |
|---|---|
| Name | Allow-SplunkForwarder-WindowsVM |
| Source | IP Addresses |
| Source IP | Windows VM private IP |
| Destination port | 9997 |
| Protocol | TCP |
| Action | Allow |
| Priority | 320 |
 
**Check 6: Set up VNet Peering if the VMs are in different VNets.** Even with NSG rules, VMs in separate VNets cannot talk without peering.
 
1. Go to portal.azure.com → **Virtual Networks** → the Splunk VM's VNet
2. Click **Peerings → Add**
3. Name the links: this VNet link `splunk-vnet-to-ad-vnet`, remote VNet link `ad-vnet-to-splunk-vnet`
4. Under **Remote virtual network**, select the Windows VM's VNet, leave other settings default, and click **Add**
5. Azure creates both directions; wait until both show **Connected**
6. On the Windows VM, restart the forwarder in an Administrator PowerShell:
```powershell
Restart-Service SplunkForwarder
```
 
> **Best practice:** Put all lab VMs in one shared VNet from the start (for example `lab-vnet`, 10.0.0.0/16) with a subnet per environment (10.0.1.0/24 for AD, 10.0.2.0/24 for Splunk). The VMs then communicate freely, NSG rules are managed at the subnet level, and no peering is needed.
 

</details>

<details>
<summary><b>7. Configure Data Inputs</b></summary>

<br>

Splunk is only as useful as the data it receives. The first data source is the Windows Server VM.

<details>
<summary><b>Part A: Enable receiving in Splunk</b></summary>

<br>

Do this in the Splunk web UI (`http://<YOUR_VM_PUBLIC_IP>:8000`) before installing the forwarder.

1. Log into the Splunk web UI
2. Click ⚙ **Settings** → Search **Forwarding and Receiving**
3. Click **Configure Receiving → New Receiving Port**, enter `9997`, and click **Save**
4. Click ⚙ **Settings** → Search **Indexes** → Click **New Index**, name it `windows_logs`, and click **Save**

<img width="1409" height="694" alt="Splunk Forwarding and Receiving" src="https://github.com/user-attachments/assets/63b8c013-0f5c-4ff6-b449-d6de6966c4b0" />

>Forward data (for sending data to other Splunk instances) and Receive data (for accepting data from forwarders). Because this Splunk server is the central SIEM and only needs to accept logs from the Windows VM, I use the Receive data section. Click + Add new next to Configure receiving to set up a listening port.

<img width="1411" height="693" alt="Configure Receiving" src="https://github.com/user-attachments/assets/48bb140d-0d5a-4a6a-b44d-6d996c15cbb4" />

> Enabling the receiving port. On the Configure receiving page, I entered 9997 in the Listen on this port field and clicked Save. Port 9997 is the standard TCP port Splunk uses for forwarder traffic. Once saved, Splunk listens on it, so the Universal Forwarder on the Windows VM can send its logs over the VNet peering to this server.

<img width="1408" height="687" alt="Splunk Index Configuration" src="https://github.com/user-attachments/assets/77fdad8e-d02a-4a02-bd0f-0e26fdf2ea44" />

> Creating the windows_logs events index to store logs from the Windows VM.

</details>

<details>
<summary><b>Part B: Install the Universal Forwarder on the Windows Server VM</b></summary>

<br>

Do this on the **Windows Server VM**, not the Ubuntu VM.

1. In the Windows VM, open a browser and go to [Splunk Universal Forwarder page](https://www.splunk.com/en_us/download/universal-forwarder.html), and log in with the same Splunk account used for the Enterprise download
2. Download the **Windows 64-bit** `.msi` installer (the page also lists 32-bit and ARM versions)
3. Run the installer and work through the screens:

   | Screen | What to enter |
   |---|---|
   | Default Installation Options | Check the license box and select **An on-premises Splunk Enterprise instance**. Do **not** select Splunk Cloud, because that skips the Receiving Indexer screen and the forwarder has nowhere to send data. |
   | Administrator credentials | Username `admin`, uncheck **Generate random password**, set a password and save it for CLI commands later |
   | Deployment Server | **Leave completely blank.** Entering the Splunk VM IP here makes the forwarder phone home to the wrong address and no data flows. |
   | Receiving Indexer | The Splunk VM's **private IP** and port **9997** (for example `10.2.0.4:9997`) |

4. Click **Install** (about 30 seconds), then **Finish**

</details>

<details>
<summary><b>Part C: Configure <code>inputs.conf</code></b></summary>

<br>

`inputs.conf` tells the forwarder which Windows Event Logs to collect. The path must be exact:

Open file explorer and navigate to the file path below: 

```text
C:\Program Files\SplunkUniversalForwarder\etc\system\local
```

Create the `local` folder in Windows Explorer first if it does not exist.

> Where the Universal Forwarder keeps its local configuration files on the Windows VM.
> `outputs.conf` where to send it (the Ubuntu VM’s private IP on port 9997)
> Create `inputs.conf` in this folder but first download Visual Studio Code

Visual Studio Code is the easiest editor for this. Install it on the Windows VM from [Visual Studio Code page](https://code.visualstudio.com), then right-click the shortcut and choose **Run as Administrator** so it can save into Program Files. 

`code .` is a command that opens Visual Studio Code with the current folder as the workspace. If you run it from a terminal that is in C:\Program Files\SplunkUniversalForwarder\etc\system\local, VS Code opens with that folder in the file explorer, so you can edit inputs.conf and outputs.conf there.

<img width="1139" height="609" alt="Open Visual Code Splunk Forwarder local file path " src="https://github.com/user-attachments/assets/65a80514-2f11-4a66-9292-cf2ee914189c" />

> Opening the Universal Forwarder local configuration folder in VS Code using `code .` from File Explorer.



 open it in VS Code with **File → Open Folder**, create a file named `inputs.conf`, paste the configuration below, and save with `Ctrl+S`.



```ini
# Each section in square brackets defines one log source to collect

[WinEventLog://Security]
# Authentication events: logins, failures, lockouts
disabled = 0
# Collect historical events, not just new ones
start_from = oldest
current_only = 0
# Resolve Active Directory object names so usernames appear instead of SIDs
evt_resolve_ad_obj = 1
# Required: without this line data goes to the main index, not windows_logs
index = windows_logs

[WinEventLog://System]
# OS-level events: service starts and stops, driver failures
disabled = 0
index = windows_logs

[WinEventLog://Application]
# Events from installed applications
disabled = 0
index = windows_logs
```

Restart the forwarder to apply the changes (Administrator PowerShell on the Windows VM):

```powershell
Restart-Service SplunkForwarder
```

> Splunk also ships with built-in sample data (**Search → Data Summary**), which can be used for most search exercises while Lab 1 is still being built.

</details>

<details>
<summary><b>Part D: Generate test log data</b></summary>

<br>

A fresh Windows Server VM has mostly empty logs, so generate activity before running the searches. The script creates a temporary local user (`labtest.user`), generates failed and successful logins, service restarts, application events, and an account lockout, then deletes the user. It only creates events in the Windows Event Log; nothing is changed permanently.

Open **PowerShell as Administrator** on the Windows VM and run the whole script (about 60-90 seconds).

<details>
<summary><b>Click to expand: log generator script</b></summary>

```powershell
# Lab 3 Log Generator - run as Administrator on the Windows VM
# Output saved to C:\lab3-log-output.txt
# Lab-only throwaway credentials; the test account is deleted at the end

$logFile = 'C:\lab3-log-output.txt'
$timestamp = Get-Date -Format 'yyyy-MM-dd HH:mm:ss'

function Log($message, $color = 'White') {
    Write-Host $message -ForegroundColor $color
    Add-Content -Path $logFile -Value "[$timestamp] $message"
}

# Clear any previous run output
if (Test-Path $logFile) { Remove-Item $logFile }
Add-Content -Path $logFile -Value "Lab 3 Log Generator - Run started at $timestamp"
Add-Content -Path $logFile -Value '============================================='
Log 'Starting log generation...' Green

# Create a temporary test user account
$testUser = 'labtest.user'
$testPass = ConvertTo-SecureString 'TempPass123!' -AsPlainText -Force
New-LocalUser -Name $testUser -Password $testPass -Description 'Splunk lab test account' -ErrorAction SilentlyContinue
Log "Created test user: $testUser" Gray

# Failed logon attempts (Security log activity)
Log 'Generating failed logon attempts (Security log activity)...' Yellow
$wrongPass = ConvertTo-SecureString 'WrongPassword!' -AsPlainText -Force
1..15 | ForEach-Object {
    $cred = New-Object System.Management.Automation.PSCredential($testUser, $wrongPass)
    Start-Process -FilePath 'cmd.exe' -Credential $cred -ArgumentList '/c exit' -ErrorAction SilentlyContinue
    Start-Sleep -Milliseconds 500
}
Log '  Generated 15 failed logon attempts' Gray

# Successful login (Event ID 4624)
Log 'Generating successful login event (4624)...' Yellow
$correctPass = ConvertTo-SecureString 'TempPass123!' -AsPlainText -Force
$cred = New-Object System.Management.Automation.PSCredential($testUser, $correctPass)
Start-Process -FilePath 'cmd.exe' -Credential $cred -ArgumentList '/c whoami' -Wait -ErrorAction SilentlyContinue
Log '  Generated successful login (Event ID 4624)' Gray

# Service start/stop events (Event ID 7036)
Log 'Generating service events (7036)...' Yellow
$services = @('Spooler', 'Schedule', 'Netlogon')
$services | ForEach-Object {
    Stop-Service -Name $_ -Force -ErrorAction SilentlyContinue
    Start-Sleep -Seconds 2
    Start-Service -Name $_ -ErrorAction SilentlyContinue
    Start-Sleep -Seconds 1
    Log "  Stopped and restarted service: $_" Gray
}

# Application log entries
Log 'Generating application log events...' Yellow
$eventSource = 'SplunkLabTest'
if (-not [System.Diagnostics.EventLog]::SourceExists($eventSource)) {
    New-EventLog -LogName Application -Source $eventSource -ErrorAction SilentlyContinue
}
1..5 | ForEach-Object {
    Write-EventLog -LogName Application -Source $eventSource -EventId 1001 `
        -EntryType Warning -Message 'Splunk lab test event - application warning'
    Start-Sleep -Milliseconds 300
}
Log '  Generated 5 application log entries (Event ID 1001)' Gray

# Lock out the test account (Event ID 4740)
Log 'Generating account lockout event (4740)...' Yellow
$badCred = ConvertTo-SecureString 'BadPass!' -AsPlainText -Force
1..20 | ForEach-Object {
    $cred = New-Object System.Management.Automation.PSCredential($testUser, $badCred)
    Start-Process -FilePath 'cmd.exe' -Credential $cred -ArgumentList '/c exit' -ErrorAction SilentlyContinue
    Start-Sleep -Milliseconds 200
}
Log '  Account lockout triggered (Event ID 4740)' Gray

# Clean up the test account
Start-Sleep -Seconds 3
Remove-LocalUser -Name $testUser -ErrorAction SilentlyContinue
Log "Removed test user: $testUser" Gray

# Restart the forwarder to ship events to Splunk
Log 'Restarting forwarder to ship events to Splunk...' Yellow
Restart-Service SplunkForwarder -ErrorAction SilentlyContinue
Log '  SplunkForwarder restarted' Gray

$endTime = Get-Date -Format 'yyyy-MM-dd HH:mm:ss'
Add-Content -Path $logFile -Value '============================================='
Add-Content -Path $logFile -Value "Run completed at $endTime"
Add-Content -Path $logFile -Value 'Wait 60 seconds then run: index=windows_logs | head 100'

Write-Host ''
Write-Host '=============================================' -ForegroundColor Green
Write-Host 'Done. Output saved to C:\lab3-log-output.txt' -ForegroundColor Green
Write-Host 'Wait 60 seconds then search Splunk.' -ForegroundColor Green
Write-Host '=============================================' -ForegroundColor Green
```

</details>

Wait 60 seconds for the forwarder to ship the events. Then set the Splunk time range to **All Time** and run:

```text
index=windows_logs | head 100
```

Events from the Security, System, and Application logs mean the lab is ready for Step 2.

</details>

</details>

 ## 🛠️ Hands-On Steps
 

<details>
<summary><b>Step 1: Essential SPL Searches</b></summary>

 
Type searches into the **Search & Reporting** bar and choose a time range with the time picker on the right.
 
**a. Confirm Data Flow**
 
```spl
index=windows_logs | head 100
```
 
**What it does:** Filters events strictly within the target index and limits the return to the first 100 results.

 **Why it is important:**

   Pipeline Health Check: This is the ultimate "smoke test" for the log ingestion pipeline. It proves that the Windows Universal Forwarder is capturing logs, transmitting them over TCP port 9997, and that the Ubuntu Splunk receiver is actively parsing and writing them to disk.

   Troubleshooting: If this query returns zero results, it immediately indicates a breakdown in the chain—such as a stopped SplunkForwarder service, a blocked firewall port, or an incorrect index name configuration.

**b. Find Successful Logins**
```spl
index=windows_logs sourcetype=WinEventLog:Security EventCode=4624
| stats count by Account_Name
| sort -count
```

**What it does:** Isolates Windows Security Event ID 4624 (successful logons), aggregates them by user account, and sorts them from highest to lowest volume.

**Why it is important:**

Baseline Behavior Analysis: In a SOC role, you must first understand what "normal" looks like before you can spot anomalies. This query gives an immediate overview of which user accounts and service accounts are actively authenticating across your systems.

Noise Reduction & Auditing: It helps differentiate between noisy background service accounts and standard user logins, allowing analysts to quickly spot unexpected accounts authenticating into the environment.
 
**c. Detect After-Hours Logins**
 
```spl
index=windows_logs sourcetype=WinEventLog:Security EventCode=4624
| eval hour=strftime(_time, "%H")
| where hour < 7 OR hour > 19
| table _time, Account_Name, Account_Domain, ComputerName
| sort -_time
```
 
**What it does:** Filters for successful logons (4624), extracts the hour of the event using time evaluation functions (eval), and filters for events occurring outside of standard business hours (before 7:00 AM or after 7:00 PM).

**Why it is important:**

Threat Detection & IOCs: Unauthorized access outside of standard operational windows is a classic Indicator of Compromise (IOC). Attackers often attempt to move laterally or access systems at night when administrative oversight is lower.

SOC Alert Foundation: This type of logic forms the basis of automated security alerts and correlation rules, helping analysts catch anomalous user activity or potential insider threats early.
 
</details>

<details>
<summary><b>Step 2: Build a Security Dashboard</b></summary>

 
Dashboards give a permanent view of security posture without rerunning searches.
 
a. In Splunk, click **Dashboards** in the top navigation
b. Click **Create New Dashboard** (top right)
c. Set **Dashboard Title** to `Windows Security Overview`, leave **Description** blank, leave **Permissions** as Private, select **Classic Dashboards**, then click **Create** and **Create Dashboard**
d. Click **Add Panel** for each panel below, choose **New Search**, paste the search, pick the visualization, and save
| Panel | Visualization |
|---|---|
| Account Activity (last 24h) | Bar chart |
| Top Processes (last 24h) | Events list |
| Login Activity Over Time | Line chart |
| After-Hours Logins | Events list |
 
**Account Activity**
 
```spl
index=windows_logs sourcetype=WinEventLog:Security EventCode=4624
| stats count by Account_Name
| sort -count
```
 
**Top Processes**
 
```spl
index=windows_logs sourcetype=WinEventLog:Security EventCode=4688
| stats count by Creator_Process_Name
| sort -count
| head 20
```
 
**Login Activity Over Time**
 
```spl
index=windows_logs sourcetype=WinEventLog:Security EventCode=4624
| timechart count
```
 
**After-Hours Logins**
 
```spl
index=windows_logs sourcetype=WinEventLog:Security EventCode=4624
| eval hour=strftime(_time, "%H")
| where hour < 7 OR hour > 19
| table _time, Account_Name, Account_Domain, ComputerName
| sort -_time
```
 
> Panels depend on the event types present in the index, so some (such as process creation, 4688) can be empty on a fresh VM.
 
<!-- Add screenshot: Windows Security Overview dashboard with populated panels -->
 
</details>

<details>
<summary><b>Step 3: Create an Automated Alert</b></summary>

 
Alerts automate detection: Splunk runs a search on a schedule and notifies when conditions are met. First run this search to confirm it works:
 
```spl
index=windows_logs sourcetype=WinEventLog:Security EventCode=4672
| stats count as privilege_logons by Account_Name, ComputerName
| where privilege_logons > 50
```
 
This finds any account with more than 50 privileged logon events. EventCode 4672 fires when admin-level rights are assigned at logon. Then:
 
a. Click **Save As → Alert**
b. **Name:** `High Privileged Logon Count`
c. **Alert type:** Scheduled
d. **Run every:** 15 minutes, using **Cron Schedule** with `*/15 * * * *`
e. **Trigger condition:** Number of Results is greater than 0
f. **Trigger actions:** Add to Triggered Alerts. This logs each firing, with a timestamp and result count, to **Activity → Triggered Alerts**, and needs no email or ticketing integration.
g. Click **Save**
> Alert quality drives monitoring quality. An alert that fires too broadly causes alert fatigue, and one that is too narrow misses real threats. The threshold of 50 is a starting point; in a real environment it would be tuned over time based on false positives.
 
<!-- Add screenshot: alert configuration page -->
 
</details>

---

## ✅ Validation
 
| Check | How to verify |
|---|---|
| Data is flowing into Splunk | Run `index=windows_logs \| head 10`, which should return recent events |
| Login activity search works | Run the EventCode 4624 search, which should return results immediately because the Windows VM generates logon events continuously |
| Dashboard displays data | The Windows Security Overview dashboard shows populated panels with charts |
| Alert is active | **Settings → Searches, Reports, and Alerts** lists the alert as Enabled |
 
## 🛠️ Troubleshooting
 
| Problem | Fix |
|---|---|
| Web UI shows "This site can't be reached" | Work through the six checks under [Prerequisites](#5-access-the-web-ui): NSG rule for port 8000, Splunk status, listening port, current public IP, port 9997 rule, VNet peering |
| Public IP changed after a VM restart | Get the current IP from **VM → Overview**, and set the public IP assignment to **Static** to prevent it |
| `index=windows_logs` returns nothing | Set the time range to **All Time** and confirm the `SplunkForwarder` service is running on the Windows VM. Restart it with `Restart-Service SplunkForwarder`. |
| Forwarder is installed but no data arrives | Confirm receiving is enabled on port 9997, the **Deployment Server** field was left blank, the **Receiving Indexer** is the Splunk VM's private IP and port 9997, and the installer used the on-premises option |
| Events land in `main` instead of `windows_logs` | `inputs.conf` is missing `index = windows_logs` under each stanza. Add it and restart the forwarder. |
| Cannot save `inputs.conf` | Run VS Code as Administrator so it can write into Program Files |
| Forwarder cannot reach the indexer across VNets | Add the port 9997 NSG rule and VNet peering (Checks 5 and 6), then restart the forwarder |
| SSH fails with `UNPROTECTED PRIVATE KEY FILE` | Run `chmod 400 yourkey.pem` before connecting |
| `splunk start` fails with a deprecation error | Add `--run-as-root` to the start command |
| `wget` returns 404 | Splunk released a newer version; copy the current `wget` command from the Splunk download page |
| Cannot paste commands into the Windows VM | Use the downloaded RDP file in the native Remote Desktop app and enable Clipboard under **Local Resources** |
 

## 📖 Key Terms

| Term | Definition |
| --- | --- |
| **SIEM (Security Information and Event Management)** | A platform that collects log data from across an entire environment and makes it searchable in one place, providing event correlation and automated alerting.|
| **SPL (Splunk Processing Language)** | The query language used in Splunk that works as a pipeline, allowing analysts to search, filter, transform, and visualise log data. |
| **Index** | A named storage bucket in Splunk where events are kept, functioning similarly to a database table (e.g., `windows_logs`). |
| **Universal Forwarder** | A lightweight agent installed on source machines (such as a Windows Server VM) to monitor, encrypt, and forward logs to a Splunk indexer over port 9997. |
| **Windows Event IDs** | Unique numbers assigned by Windows to specific system activities, such as successful logons (`4624`), process creations (`4688`), or account lockouts (`4740`). |
| **inputs.conf** | A configuration file used by the Universal Forwarder to specify which log sources and Windows Event Logs to collect. |
| **Dashboard** | A permanent, visual display view in Splunk consisting of multiple panels (such as charts and event lists) based on saved SPL queries. |
| **Alert** | An automated detection mechanism in Splunk that executes a search on a regular schedule and records or notifies when specific criteria are met. |
| **NSG** (Network Security Group) | Azure firewall rules that control inbound and outbound traffic to a VM |
| **VNet Peering** | An Azure networking feature that creates a direct private connection between separate Virtual Networks, enabling communication between VMs (such as the Forwarder and Indexer). |
 
## 🔍 Windows Event IDs
 
| EventCode | Meaning | Why it matters |
|---|---|---|
| **4624** | Successful logon (logon type shows interactive, remote, or service) | Baseline authentication activity and after-hours detection |
| **4625** | Failed logon | Brute force and password spray detection |
| **4672** | Special privileges assigned to a new logon | Privileged account activity |
| **4688** | New process created | Threat hunting for malicious execution |
| **4740** | Account locked out | A trail of lockouts can indicate a password spray |
| **7036** | Service entered a running or stopped state | Service start and stop activity |

## 🛠️ Key Skills Demonstrated 

**Splunk Deployment & Cloud Configuration:** Setting up and running Splunk Enterprise on an Azure Ubuntu virtual machine and configuring local firewall and networking rules.   

**Universal Forwarder Installation:** and configuring the Splunk Universal Forwarder on a Windows Server VM to securely ship logs over port 9997.   

**Data Input & Configuration Management:** Editing inputs.conf to target critical Windows Event Logs (Security, System, Application) and directing them into custom indexes (windows_logs).   

**Splunk Processing Language (SPL) Authoring:** Writing and executing SPL queries to search, filter, transform, and analyze log data efficiently.  

**Security Dashboard Construction:** Designing and building visual security dashboards featuring bar charts, line charts, and event lists to monitor authentication activity and system processes.   

**Automated Alerting & Monitoring:** Creating scheduled alerts based on specific threshold conditions (such as high privileged logon counts) and configuring automated event logging.   

**Security Incident Investigation:** Investigating failed login attempts, account lockouts (Event ID 4740), after-hours logons, and process creation events to spot potential brute-force or malicious activity. 

## 💡 Lessons Learned
 
- **A missing NSG rule is the first suspect for a connection timeout.** Port 8000 for the web UI and port 9997 for the forwarder both need explicit rules.
- **Scope network exposure tightly.** SSH and the web UI are limited to my IP, and the forwarder port is limited to the VNet.
- **Azure public IPs change when a VM is stopped and restarted.** Setting the assignment to Static avoids chasing a new IP every session.
- **Put all lab VMs in one VNet with a subnet per environment.** It avoids VNet peering and keeps NSG management simple.
- **Leave the Deployment Server blank in the Universal Forwarder installer.** Pointing it at the Splunk VM sends the forwarder to the wrong place and no data flows.
- **Every `inputs.conf` stanza needs its `index` setting.** Without it, events go to the default `main` index and searches against `windows_logs` come back empty.
- **A fresh VM has little to detect.** Generating test activity (failed logons, a lockout, service restarts) is what makes searches, dashboards, and alerts demonstrable.
- **Alert thresholds need tuning.** Too broad causes alert fatigue, too narrow misses threats, so start with a baseline and adjust on false positives.


