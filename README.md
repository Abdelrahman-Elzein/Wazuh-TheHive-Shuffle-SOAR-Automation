# 🛡️ Wazuh SIEM, TheHive & Shuffle SOAR Automated Incident Response Lab

[![SOC Automation](https://img.shields.io/badge/SOC-Automation-blue.svg)](https://github.com/Abdelrahman-Elzein/Wazuh-TheHive-Shuffle-SOAR-Automation)
[![SIEM](https://img.shields.io/badge/SIEM-Wazuh%20v4.12-00a9e0.svg)](https://wazuh.com/)
[![SOAR](https://img.shields.io/badge/SOAR-Shuffle%20Cloud%20%26%20Orborus-orange.svg)](https://shuffler.io/)
[![Case Management](https://img.shields.io/badge/Case%20Management-TheHive-purple.svg)](https://thehive-project.org/)
[![Threat Intel](https://img.shields.io/badge/Threat%20Intel-VirusTotal-394eff.svg)](https://www.virustotal.com/)

An end-to-end hybrid Security Orchestration, Automation, and Response (SOAR) lab integrating **Wazuh SIEM**, **Shuffle Cloud & On-Premises Orborus Worker**, **VirusTotal Threat Intelligence**, and **TheHive Case Management** to automatically detect credential dumping (`mimikatz.exe`), enrich file hashes, create incident alerts, notify SOC analysts, and isolate compromised endpoints via Wazuh Active Response.

---

## 🏗️ Topology & Architecture

1. I started by creating the diagram/topology of the SOAR automation workflow using "draw.io".
2. In this workflow, the credential-dumping malware `mimikatz.exe` executes on a Windows endpoint. The local Wazuh agent detects the malicious activity and forwards the telemetry to the Wazuh Manager, generating a high-severity alert on the SIEM dashboard.
3. Wazuh automatically forwards the alert to Shuffle to initiate the automated response workflow.
4. Shuffle extracts the file hash from the alert and queries VirusTotal to verify the malware's reputation.
5. Shuffle then sends an email containing the detection details.
6. Shuffle automatically generates a new security alert in TheHive for incident tracking and triage.
7. Shuffle triggers Wazuh Active Response to execute a Windows Firewall script, automatically isolating the compromised host from the network.

### Architecture Diagram (Draw.io)
![SOAR Workflow Diagram Draw.io](Screenshots/00%20Diagram-(Draw.io).png)

### Shuffle SOAR Workflows Overview
| Cloud Main Workflow (`Wazuh-TheHive-Shuffle-SOAR-Automation`) | On-Premises Subflow (`TheHive-Local-SOC-Lab-Location`) |
| :---: | :---: |
| ![Shuffle Workflow 1](Screenshots/00%20Diagram-Shuffle-1.png) | ![Shuffle Workflow 2](Screenshots/00%20Diagram-Shuffle-2.png) |

---

## 🌐 Lab Architecture & Network Setup

This lab uses a hybrid cloud and on-premises architecture hosted locally on Oracle VirtualBox. Shuffle Cloud (`shuffler.io`) acts as the central SOAR orchestrator, while a local Shuffle Orborus worker (deployed via Docker on TheHive VM) bridges the cloud playbook to the self-hosted on-premises infrastructure without exposing internal services to the public internet.

### VirtualBox VM & Network Configuration

* **Windows 10 Endpoint (Victim Host)**
  * **Interface 1 (NAT):** Provides outbound internet access for initial setup and updates without exposing the VM to the host LAN.
  * **Interface 2 (Internal Network — `private-net1`):** Assigned static IP `10.10.10.10/24`. Used to communicate directly with the Wazuh Manager while isolating the compromised endpoint (where `mimikatz.exe` executes) from the host network—avoiding the risks of a Bridged Adapter.

* **Wazuh Server (SIEM & Manager)**
  * **Interface 1 (Bridged Adapter):** Assigned `192.168.8.100/24` to communicate with Shuffle Cloud, external APIs, and TheHive.
  * **Interface 2 (Internal Network — `private-net1`):** Assigned `10.10.10.5/24` to securely receive agent telemetry and execute Active Response commands on the isolated Windows 10 endpoint.

* **TheHive & Shuffle Orborus (Case Management & Local Worker)**
  * **Interface 1 (Bridged Adapter):** Assigned `192.168.8.101/24` to receive orchestrated actions from Shuffle Cloud via the local Orborus Docker worker and integrate with the Wazuh Manager API.

---

## 📚 Base Setup & Reference Documentation

To keep this guide focused on the custom detections, hybrid networking, and SOAR automation, standard base installations were performed following the official vendor documentation:
* **Wazuh (v4.12):** Deployed using the all-in-one installation assistant via the Wazuh Quickstart Guide (`https://documentation.wazuh.com/4.12/quickstart.html`).
* **TheHive:** Deployed on a standalone Linux server following the TheHive Software Requirements and TheHive Standalone Server Installation Guide.

---

## ⚙️ Implementation Steps

### 1. Sysmon Installation
1. I've installed Sysmon on the Windows 10 endpoint.

---

### 2. Deploy Wazuh Agent
2. Deploy Wazuh Agent: In the Wazuh web dashboard Overview page, I clicked "Deploy new agent", selected "Windows" as the operating system, and entered the Wazuh Manager's internal IP address (`10.10.10.5`). I then copied the generated PowerShell installation command and executed it in an elevated PowerShell terminal on the Windows 10 endpoint to install and enroll the agent.

| Wazuh Overview Page | Deploy New Agent - Step 1 |
| :---: | :---: |
| ![Wazuh Overview Page](Screenshots/01%20Wazuh-Overview-Page.png) | ![Wazuh Deploy New Agent 1](Screenshots/02%20Wazuh-Deploy-New-Agent-1.png) |

| Deploy New Agent - Step 2 | PowerShell Installation Execution |
| :---: | :---: |
| ![Wazuh Deploy New Agent 2](Screenshots/03%20Wazuh-Deploy-New-Agent-2.png) | ![PowerShell Command](Screenshots/04%20Wazuh-Deploy-New-Agent-2-(Powershell-command).png) |

---

### 3. Configure Sysmon-Only Log Forwarding
3. Configure Sysmon-Only Log Forwarding: On the Windows 10 endpoint, I edited `C:\Program Files (x86)\ossec-agent\ossec.conf` under the Log analysis section—removing the default Windows event channels (Application, Security, System) so the Wazuh Agent forwards only Sysmon logs for noise reduction—and restarted the Wazuh agent service for the changes to take effect:

```xml
<!-- Log analysis -->
<localfile>
  <location>Microsoft-Windows-Sysmon/Operational</location>
  <log_format>eventchannel</log_format>
</localfile>
```

| Sysmon-Only `ossec.conf` Configuration | Wazuh Agent Service Restart |
| :---: | :---: |
| ![Win10 Config Sysmon Only](Screenshots/05%20Win10-Endpoint-Config-Wazuh-Agent-Send-Sysmon-Only.png) | ![Wazuh Agent Service Restart](Screenshots/06%20Wazuh-Agent-Service-Restart-After-Previous-Config.png) |

---

### 4–6. Enable Raw Event Archiving & Create Archives Index Pattern
4. Enable Raw Event Archiving: On the Wazuh Manager, I modified `/var/ossec/etc/ossec.conf` to enable `logall` and `logall_json`, and updated `/etc/filebeat/filebeat.yml` to enable archives (`archives: enabled: true`). This instructs Wazuh to store every raw event received from agents in `/var/ossec/logs/archives/`—even when no detection rule is triggered—making it possible to inspect raw telemetry for custom rule development and troubleshooting:

```xml
<logall>yes</logall>
<logall_json>yes</logall_json>
```

![Enabled Logall on Wazuh Manager](Screenshots/07%20Enabled-Logall-On-Wazuh-Manager.png)

5. Restart Wazuh & Filebeat Services: I restarted both services using `systemctl restart wazuh-manager.service` and `systemctl restart filebeat.service` for the configuration changes to take effect.

6. Create the Archives Index Pattern: In the Wazuh web dashboard, I navigated to `Dashboards Management -> Index Patterns -> Create Index Pattern`, named the pattern `wazuh-archives-*`, and selected `timestamp` as the "Time field" so all archived events could be searched and viewed directly inside the Discover tab.

| Creating Index Pattern - 1 | Creating Index Pattern - 2 | Creating Index Pattern - 3 |
| :---: | :---: | :---: |
| ![Index Pattern 1](Screenshots/08%20Creating-wazuh-archives-Index-Pattern-1.png) | ![Index Pattern 2](Screenshots/09%20Creating-wazuh-archives-Index-Pattern-2.png) | ![Index Pattern 3](Screenshots/10%20Creating-wazuh-archives-Index-Pattern-3.png) |

| Verify Index Pattern Creation - 1 | Verify Index Pattern Creation - 2 |
| :---: | :---: |
| ![Verify Index Pattern 1](Screenshots/10%20Verify-wazuh-archives-Index-Pattern-Creation-1.png) | ![Verify Index Pattern 2](Screenshots/11%20Verify-wazuh-archives-Index-Pattern-Creation-2.png) |

---

### 7. Baseline Telemetry Capture
7. Baseline Telemetry Capture: On the Windows 10 endpoint, I executed `mimikatz.exe` via PowerShell and verified that the raw Sysmon logs appeared in Wazuh Discover under the `wazuh-archives-*` index. I also confirmed the ingestion directly on the Wazuh Manager CLI using `cat /var/ossec/logs/archives/archives.log | grep -i mimikatz`.

| Win10 Mimikatz Baseline Execution | Wazuh Archives Raw Telemetry Logs |
| :---: | :---: |
| ![Win10 Mimikatz Execution](Screenshots/12%20Win10-mimikatz-Baseline-Execution.png) | ![Wazuh Archives Raw Logs](Screenshots/13%20Wazuh-Archives-mimikatz-Raw-Logs.png) |

---

### 8. Custom Mimikatz Detection Rule
8. Custom Mimikatz Detection Rule: In the Wazuh web dashboard, I navigated to `Server management -> Rules -> Custom rules -> local_rules.xml` and added custom rule `100002` to detect Mimikatz execution via its PE `originalFileName` metadata (ensuring detection even if the binary is renamed) and map it to MITRE ATT&CK technique `T1003`:

```xml
<rule id="100002" level="15">
  <if_group>sysmon_event1</if_group>
  <field name="win.eventdata.originalFileName" type="pcre2">(?i)mimikatz\.exe</field>
  <description>Mimikatz Usage Detected</description>
  <mitre>
    <id>T1003</id>
  </mitre>
</rule>
```

| Adding Custom Rule - 1 | Adding Custom Rule - 2 | Custom Rule Added & Verified |
| :---: | :---: | :---: |
| ![Adding Custom Rule 1](Screenshots/14%20Wazuh-Adding-New-Custom-Rule-1.png) | ![Adding Custom Rule 2](Screenshots/15%20Wazuh-Adding-New-Custom-Rule-2.png) | ![Mimikatz Rule Added](Screenshots/16%20Wazuh-Mimikatz-Usage-Detected-Rule-Added.png) |

---

### 9. Shuffle Workflow & Wazuh Webhook Integration
9. Shuffle Workflow & Wazuh Webhook Integration: Logged into Shuffle Cloud (`shuffler.io`), created a new workflow named `Wazuh-TheHive-Shuffle-SOAR-Automation`, and added a webhook trigger node named `Wazuh Alerts`. Next, I accessed the Wazuh Manager VM and added the following integration block to `/var/ossec/etc/ossec.conf` so Wazuh forwards JSON alert payloads exclusively when custom rule `100002` triggers:

```xml
<integration>
  <name>shuffle</name>
  <hook_url><MY-WEBHOOK-URL></hook_url>
  <rule_id>100002</rule_id>
  <alert_format>json</alert_format>
</integration>
```

After restarting the Wazuh Manager (`sudo systemctl restart wazuh-manager.service`) and starting the webhook listener in Shuffle, I verified that incoming alerts were successfully ingested.

| Created Shuffle Cloud Workflow | Added Webhook (`Wazuh_Alerts`) |
| :---: | :---: |
| ![Created Shuffle Workflow](Screenshots/17%20Created-My-First-Shuffle-SOAR-Cloud-Workflow.png) | ![Added Webhook](Screenshots/18%20Added-Webhook-Named-Wazuh_Alerts.png) |

| Added Shuffle Webhook to Wazuh `ossec.conf` | Verified Webhook Receiving Wazuh Alerts |
| :---: | :---: |
| ![Adding Shuffle Webhook to Wazuh](Screenshots/19%20Adding-Shuffle-Webhook_To_Wazuh.png) | ![Verified Webhook Alerts](Screenshots/20%20Verified-Webhook-Receiving-Alerts-From-Wazuh.png) |

---

### 10–12. SHA256 Hash Extraction & VirusTotal Threat Intelligence Enrichment
10. SHA256 Hash Extraction (Regex Node): Next, I added a Shuffle Tools node (renamed `SHA256-Hash`) and set its action to `Regex capture group`. Because Sysmon bundles multiple hash algorithms (SHA1, SHA256, MD5, and IMPHASH) into a single field, I mapped `$exec.text.win.eventdata.hashes` into the Input data field using Shuffle's Autocomplete feature and applied the regex pattern `SHA256=([0-9A-Fa-f]{64})` to isolate only the 64-character SHA256 hash for VirusTotal lookup.

![SHA256 Regex Node](Screenshots/21%20Added-Get-File-Meta-Node-For-Capturing-SHA256-Hash.png)

11. Threat Intelligence Enrichment (VirusTotal Node): I retrieved my API key from VirusTotal (`Profile Icon -> API Key`), added a VirusTotal node to the Shuffle canvas, and set its action to `Get a hash report`. Under the Setup tab, I created a new authentication profile (`VT-Auth`) using the API key. Finally, under the Configuration tab, I mapped the extracted hash variable (`$sha256-hash.group_0.#`) into the Id field via Shuffle's Autocomplete menu.

| VirusTotal Node Configuration - 1 | VirusTotal Node Configuration - 2 |
| :---: | :---: |
| ![VirusTotal Node 1](Screenshots/22%20Added-VirusTotal-Node-That-Uses-Previous-SHA256-Hash-1.png) | ![VirusTotal Node 2](Screenshots/23%20Added-VirusTotal-Node-That-Uses-Previous-SHA256-Hash-2.png) |

12. Verify Hash Extraction & Threat Intelligence Enrichment: Next, I tested the execution and verified that both the `SHA256-Hash` node (`Regex capture group` action) and the `VirusTotal` node (`get_a_hash_report` action) worked as expected. Because Sysmon bundles multiple hash algorithms together (SHA1, MD5, SHA256, IMPHASH) in a single field, the regex node cleanly isolated the SHA256 hash—confirmed by the VirusTotal response showing `"status": 200`, the extracted hash under `"id"`, and a `"malicious"` verdict.

![Verified VirusTotal and SHA256 Regex](Screenshots/24%20Verified-VirusTotal-Node-And-Previous-SHA256-Regex-Works.png)

---

### 13–14. TheHive Backend Services, Organization & API User Setup
13. TheHive & Backend Services Configuration: Next, I configured Apache Cassandra, Elasticsearch, and TheHive on the TheHive VM (`192.168.8.101`):

* **Apache Cassandra (`/etc/cassandra/cassandra.yaml`):** Updated the cluster name and bound the listen, RPC, and seed addresses to the VM's bridged IP:

```yaml
cluster_name: 'Abdelrahman-Elzein'
listen_address: 192.168.8.101
rpc_address: 192.168.8.101

seed_provider:
  - class_name: org.apache.cassandra.locator.SimpleSeedProvider
    parameters:
      - seeds: "192.168.8.101:7000"
```

* Set the supported Java version, reset the data directory permissions, and restarted Cassandra:

```bash
# Ensure the supported Java version is selected for Cassandra & TheHive
sudo update-alternatives --config java

sudo chown -R cassandra:cassandra /var/lib/cassandra
sudo systemctl stop cassandra.service
sudo rm -rf /var/lib/cassandra/*
sudo systemctl start cassandra.service
sudo systemctl status cassandra.service
```

* **Elasticsearch (`/etc/elasticsearch/elasticsearch.yml`):** Configured the single-node cluster and bound it to `192.168.8.101` *(Note: Check the bottom of the file first—Elasticsearch auto-configuration often appends a duplicate `cluster.initial_master_nodes` line that will cause startup errors if not removed)*:

```yaml
cluster.name: Abdelrahman-Elzein
node.name: node-1
network.host: 192.168.8.101
http.port: 9200
cluster.initial_master_nodes: ["node-1"]

xpack.security.enabled: false
xpack.security.http.ssl:
  enabled: false
```

* Started Elasticsearch and set ownership on TheHive's file storage directory:

```bash
sudo systemctl start elasticsearch.service
sudo systemctl status elasticsearch.service
sudo chown -R thehive:thehive /opt/thp
```

* **TheHive (`/etc/thehive/application.conf`):** Connected TheHive to the local Cassandra and Elasticsearch instances and set the web interface URL:

```hocon
db.janusgraph {
  storage {
    backend = cql
    hostname = ["192.168.8.101"]

    cql {
      cluster-name = Abdelrahman-Elzein
    }
  }
  index.search {
    backend = elasticsearch
    hostname = ["192.168.8.101"]
    index-name = thehive
  }
}

application.baseUrl = "http://192.168.8.101:9000"
```

* Enabled and started TheHive service:

```bash
sudo systemctl enable --now thehive.service
sudo systemctl status thehive.service
```

14. TheHive Organization & API User Setup: In TheHive's web dashboard, I created a new organization named `Abdelrahman-Elzein`. Inside that organization, I created two users with `analyst` profile: a Normal user named `Abdelrahman Elzein` for manual triage, and a Service user named `SOAR-Automation` for API integration, copying its generated API key for Shuffle.

| TheHive Web GUI Config - 1 | TheHive Web GUI Config - 2 | TheHive Web GUI Config - 3 |
| :---: | :---: | :---: |
| ![TheHive Config 1](Screenshots/25%20TheHive-Web-GUI-Config-1.png) | ![TheHive Config 2](Screenshots/26%20TheHive-Web-GUI-Config-2.png) | ![TheHive Config 3](Screenshots/27%20TheHive-Web-GUI-Config-3.png) |

---

### 15–18. Local Shuffle Orborus Worker & TheHive Subflow Integration
15. Docker & Local Shuffle Orborus Worker Setup: Following the Docker Engine Ubuntu installation guide (within the URL `https://docs.docker.com/engine/install/ubuntu/`), I installed Docker on the TheHive Ubuntu VM so it could host an on-premises Shuffle worker capable of communicating with my local VMs over private IPs. Next, in Shuffle Cloud (`shuffler.io`), I navigated to `Admin -> Locations`—where only the default Cloud location existed—and created a new on-premises location named `Local-SOC-Lab`. Selecting the new location generated a `docker run` command for `shuffle-orborus`, which I executed on the TheHive VM. After about a minute, refreshing Shuffle's Locations page confirmed the status of `Local-SOC-Lab` changed from `Stopped` to `Running`.

| Adding On-Prem Location - 1 | Adding On-Prem Location - 2 | Adding On-Prem Location - 3 |
| :---: | :---: | :---: |
| ![On-Prem Location 1](Screenshots/28%20Shuffle-Adding-New-Onprem-Location-1.png) | ![On-Prem Location 2](Screenshots/29%20Shuffle-Adding-New-Onprem-Location-2.png) | ![On-Prem Location 3](Screenshots/30%20Shuffle-Adding-New-Onprem-Location-3.png) |

| Adding On-Prem Location - 4 | On-Prem Location Running (`Local-SOC-Lab`) |
| :---: | :---: |
| ![On-Prem Location 4](Screenshots/31%20Shuffle-Adding-New-Onprem-Location-4.png) | ![On-Prem Location 5](Screenshots/32%20Shuffle-Adding-New-Onprem-Location-5.png) |

16. Create Local Runtime Subflow: In Shuffle, I created a new workflow named `TheHive-Local-SOC-Lab-Location` and assigned its runtime location to the on-premises `Local-SOC-Lab` worker so its actions could reach my private lab network.

![Created On-Premises Workflow](Screenshots/33%20Shuffle-Created-A-New-Workflow-For-The-New-Onprem-Location.png)

17. Configure TheHive Alert Creation Node: Inside the `TheHive-Local-SOC-Lab-Location` workflow, I added a TheHive node, named it `TheHive_1`, and set its action to `(POST) Create alert`. Under the `Setup` tab, I created an authentication profile named `TheHive-Auth` using the `SOAR-Automation` service account's API key and set the target URL to `192.168.8.101:9000`. Under the `Configuration` tab, I switched from `Simple` to `Advanced` view, set `Ssl verify` and `To file` to `False`, and configured the JSON `Body` using Shuffle's Autocomplete variables to structure the incoming Wazuh telemetry into a rich alert:

```json
{
  "type": "wazuh_alert",
  "source": "$exec.pretext",
  "sourceRef": "$exec.id",
  "title": "$exec.title",
  "description": "Mimikatz activity detected on host: $exec.all_fields.agent.name (IP: $exec.all_fields.agent.ip) | Rule ID: $exec.rule_id",
  "summary": "ALERT SUMMARY: $exec.title (Rule $exec.rule_id) detected on $exec.text.win.system.computer ($exec.all_fields.agent.ip) at $exec.text.win.eventdata.utcTime | User: $exec.text.win.eventdata.user | Original File: $exec.text.win.eventdata.originalFileName | Image Path: $exec.text.win.eventdata.image | Working Dir: $exec.text.win.eventdata.currentDirectory | Parent Process: $exec.text.win.eventdata.parentImage | Hashes: $exec.text.win.eventdata.hashes",
  "severity": $exec.severity,
  "tlp": 2,
  "pap": 2,
  "flag": false,
  "tags": [
    "Mimikatz",
    "T1003"
  ],
  "status": "New"
}
```

| Configuring TheHive Node - 1 | Configuring TheHive Node - 2 | Configuring TheHive Node - 3 |
| :---: | :---: | :---: |
| ![TheHive Node 1](Screenshots/34%20Shuffle-Adding-And-Configuring-TheHive-Node-1.png) | ![TheHive Node 2](Screenshots/35%20Shuffle-Adding-And-Configuring-TheHive-Node-2.png) | ![TheHive Node 3](Screenshots/36%20Shuffle-Adding-And-Configuring-TheHive-Node-3.png) |

18. Connect TheHive Subflow: Back in the main workflow (`Wazuh-TheHive-Shuffle-SOAR-Automation`), I added a Subflow node named `TheHive`, selected `TheHive-Local-SOC-Lab-Location` as the target workflow with `TheHive_1` as the start node, and passed `$exec` as the Runtime Argument so the full alert payload is forwarded to my on-premises Orborus worker.

![Connected TheHive Subflow](Screenshots/37%20Shuffle-Added-TheHive-Subflow-And-Connected-It-To-TheHive-Node.png)

---

### 19. Configure Analyst Email Notification
19. Configure Analyst Email Notification: Next, I added an Email node named `Send_Email` to the main workflow, selected the `Send email shuffle` action, entered the SOC recipient email address, set the Subject to `[Alert] $exec.title`, and structured the email Body using Shuffle's Autocomplete variables to include key incident telemetry and the isolated SHA256 file hash:

```text
Title: $exec.title
Time: $exec.text.win.eventdata.utcTime
Username: $exec.text.win.eventdata.user
Hostname: $exec.text.win.system.computer
SourceIP: $exec.all_fields.agent.ip
File Path: $exec.text.win.eventdata.currentDirectory
Command Line: $exec.text.win.eventdata.commandLine
Agent ID: $exec.all_fields.agent.id
File Hash: $sha256-hash.group_0.#
```

![Added Send Email Action](Screenshots/38%20Shuffle-Added-A-Send-Email-Action.png)

---

### 20–21. Endpoint Isolation Scripts & Wazuh Active Response Registration
20. Create Endpoint Isolation Scripts: On the Windows 10 endpoint, I placed three custom scripts inside `C:\Program Files (x86)\ossec-agent\active-response\bin` to handle automated network containment and recovery via Windows Firewall:

* **`isolate-host.cmd`:** Acts as the batch wrapper invoked by Wazuh Active Response, launching the PowerShell isolation script with execution policy bypassed:

```bat
@powershell.exe -NoProfile -ExecutionPolicy Bypass -File "C:\Program Files (x86)\ossec-agent\active-response\bin\isolate-host.ps1"
```

* **`isolate-host.ps1`:** Parses Wazuh's stateful JSON input to automatically isolate or restore the host. When receiving the `add` command, it blocks all inbound and outbound Windows Firewall traffic except TCP ports `1514` and `1515` to the Wazuh Manager (`10.10.10.5`)—preserving agent management connectivity while stopping lateral movement. When receiving the `delete` command, it removes those isolation rules and restores normal outbound network access:

```powershell
$InputJson = [Console]::In.ReadLine()
if (-not $InputJson) { exit 0 }

$Data = $InputJson | ConvertFrom-Json
$Command = $Data.command
$WazuhIP = "10.10.10.5"

if ($Command -eq "add") {
    netsh advfirewall set allprofiles firewallpolicy blockinbound,blockoutbound
    netsh advfirewall firewall add rule name="Wazuh-Isolation-Allow-Out" dir=out action=allow protocol=TCP remoteip=$WazuhIP remoteport=1514,1515
    netsh advfirewall firewall add rule name="Wazuh-Isolation-Allow-In" dir=in action=allow protocol=TCP remoteip=$WazuhIP remoteport=1514,1515
}
elseif ($Command -eq "delete") {
    netsh advfirewall firewall delete rule name="Wazuh-Isolation-Allow-Out"
    netsh advfirewall firewall delete rule name="Wazuh-Isolation-Allow-In"
    netsh advfirewall set allprofiles firewallpolicy blockinbound,allowoutbound
}
```

* **`unisolate-host.cmd`:** Provides a standalone local batch script for administrators to manually remove the isolation firewall rules and restore network access directly on the endpoint if needed:

```bat
@echo off
netsh advfirewall firewall delete rule name="Wazuh-Isolation-Allow-Out"
netsh advfirewall firewall delete rule name="Wazuh-Isolation-Allow-In"
netsh advfirewall set allprofiles firewallpolicy blockinbound,allowoutbound
```

![Created Endpoint Isolation Scripts](Screenshots/39%20Create-Endpoint-Isolation-Scripts.png)

21. Register Active Response Commands on Wazuh Manager: On the Wazuh Manager VM, I modified `/var/ossec/etc/ossec.conf` to register both custom Windows executables as Active Response commands. By omitting a rule or level trigger inside the `<active-response>` blocks, Wazuh leaves containment disabled by default until explicitly invoked on demand by Shuffle via the Wazuh API:

```xml
<command>
  <name>isolate-host</name>
  <executable>isolate-host.cmd</executable>
  <timeout_allowed>yes</timeout_allowed>
</command>

<active-response>
  <command>isolate-host</command>
  <location>local</location>
  <timeout>0</timeout>
</active-response>

<command>
  <name>unisolate-host</name>
  <executable>unisolate-host.cmd</executable>
</command>

<active-response>
  <command>unisolate-host</command>
  <location>local</location>
</active-response>
```

After restarting the Wazuh Manager (`sudo systemctl restart wazuh-manager.service`) to apply the configuration, I ran `sudo grep isolate /var/ossec/etc/shared/ar.conf` to retrieve the exact internal command names generated by Wazuh (`isolate-host0` and `unisolate-host0`), which are required when configuring the Wazuh API action in Shuffle.

| Registering Active Response - 1 | Registering Active Response - 2 |
| :---: | :---: |
| ![Active Response 1](Screenshots/40%20Registering_Active-Response-Commands-On-Wazuh-Manager-1.png) | ![Active Response 2](Screenshots/41%20Registering_Active-Response-Commands-On-Wazuh-Manager-2.png) |

---

### 22–24. Automated Endpoint Isolation via Wazuh API & Shuffle Subflow
22. Authenticate with Wazuh API (HTTP Curl Node): Inside the on-premises workflow (`TheHive-Local-SOC-Lab-Location`), I added an HTTP node named `Get-API_1`, set its action to `CURL`, and entered the following command in the Statement field to retrieve a temporary JWT authentication token from the Wazuh Manager API:

```bash
curl -u wazuh:<WAZUH_API_PASSWORD> -k -X GET "https://192.168.8.100:55000/security/user/authenticate?raw=true"
```

![HTTP CURL Node Configuration](Screenshots/42%20Shuffle-Added-And-Configured-An-HTTP-CURL-Node.png)

23. Configure Automated Endpoint Isolation (Wazuh Node): Still inside the on-premises workflow (`TheHive-Local-SOC-Lab-Location`), I connected the `Get-API_1` node to a new Wazuh node named `Wazuh_Isolate_Host` and set its action to `PUT Run command`. Leaving the standard Authentication profile blank, I mapped the dynamic JWT token (`$get-api_1.body`) into the `Apikey` field and set the target URL to `https://192.168.8.100:55000`. Under the configuration options, I set `Command` to `isolate-host0`, dynamically mapped the compromised agent ID (`$exec.all_fields.agent.id`) into `Agents list`, disabled `SSL Verify` and `To File` (`False`), and enabled `Wait for complete` (`true`) so Shuffle verifies the containment command finishes executing.

| Configuring Wazuh Isolate Node - 1 | Configuring Wazuh Isolate Node - 2 | Configuring Wazuh Isolate Node - 3 |
| :---: | :---: | :---: |
| ![Wazuh Node 1](Screenshots/43%20Shuffle-Addition-And-Configuration-Of-Wazuh-Node-1.png) | ![Wazuh Node 2](Screenshots/44%20Shuffle-Addition-And-Configuration-Of-Wazuh-Node-2.png) | ![Wazuh Node 3](Screenshots/45%20Shuffle-Addition-And-Configuration-Of-Wazuh-Node-3.png) |

24. Connect Host Isolation Subflow: Back in the main cloud workflow (`Wazuh-TheHive-Shuffle-SOAR-Automation`), I added a second Subflow node named `Wazuh_Isolate_Host` connected downstream from the `VirusTotal` node. I set its target workflow to `TheHive-Local-SOC-Lab-Location`, selected `Get-API_1` as the start node, and passed `$exec` as the `Runtime Argument` so the on-premises Orborus worker receives the compromised agent's metadata to execute containment.

![Wazuh Isolate Host Subflow](Screenshots/46%20Shuffle-Wazuh-Isolate-Host-Subflow-Addition-And-Configuration.png)

> **Note on Host Recovery (Un-isolation):** Once incident investigation and remediation are complete, the compromised endpoint can be remotely restored to the network from the Wazuh Manager CLI using the following command (where `001` is the target agent ID):
> ```bash
> sudo /var/ossec/bin/agent_control -b 0.0.0.0 -f unisolate-host0 -u 001
> ```

---

## 🧪 End-to-End Validation Test

To validate the complete SOAR workflow without manual analyst intervention, I tested the playbook against a renamed Mimikatz binary:

### 1. Simulated Defense Evasion Execution (`Workflow-Execution 1`)
Executed `Renamed-mimikatz.exe` via PowerShell on the Windows 10 endpoint (7:52 PM) to simulate an attacker renaming a known credential-dumping tool to bypass filename-based detections.

![Workflow-Execution 1](Screenshots/Workflow-Execution%201.png)

---

### 2. SIEM Detection & Alert Generation (`Workflow-Execution 2`)
Verified in Wazuh Discover (`wazuh-alerts-*`) that custom rule `100002` detected the execution via its PE `originalFileName` metadata (`mimikatz.exe`) and generated a high-severity alert.

![Workflow-Execution 2](Screenshots/Workflow-Execution%202.png)

---

### 3. Automated Email Notification (`Workflow-Execution 3`)
Confirmed that Shuffle's `Send_Email` action delivered an alert email containing the extracted SHA256 file hash and endpoint telemetry.

![Workflow-Execution 3](Screenshots/Workflow-Execution%203.png)

---

### 4. Automated Case Creation in TheHive — Alerts List (`Workflow-Execution 4`)
Verified that the on-premises Shuffle subflow created a new High-severity alert in TheHive tagged with `Mimikatz` and `T1003`.

![Workflow-Execution 4](Screenshots/Workflow-Execution%204.png)

---

### 5. Automated Case Creation in TheHive — Alert Details (`Workflow-Execution 5`)
Opened the alert preview in TheHive to confirm all host details, process paths, and file hashes were populated automatically by the `SOAR-Automation` service account.

![Workflow-Execution 5](Screenshots/Workflow-Execution%205.png)

---

### 6. Automated Endpoint Containment (`Workflow-Execution 6`)
Immediately after execution (7:53 PM), Wazuh Active Response applied the Windows Firewall isolation policy, causing the continuous ping (`ping 8.8.8.8 -t`) on the Windows 10 endpoint to fail with `"General failure."`

![Workflow-Execution 6](Screenshots/Workflow-Execution%206.png)

---

### 7. Manual Host Recovery Command (`Workflow-Execution 7`)
Executed `sudo /var/ossec/bin/agent_control -b 0.0.0.0 -f unisolate-host0 -u 001` on the Wazuh Manager CLI to lift the firewall block and restore network access to agent `001`.

![Workflow-Execution 7](Screenshots/Workflow-Execution%207.png)

---

### 8. Network Connectivity Restored (`Workflow-Execution 8`)
Verified on the Windows 10 endpoint (7:58 PM) that the Windows Firewall block was removed and outbound network connectivity returned. *(Note: Pardon my unstable internet connection at the time of testing, which caused several "Request timed out" lines—the immediate disappearance of the "General failure." messages and return of "Reply from 8.8.8.8" confirms that the remote un-isolation command worked successfully.)*

![Workflow-Execution 8](Screenshots/Workflow-Execution%208.png)

---

## 👨‍💻 Author

**Abdelrahman Elzein**  
