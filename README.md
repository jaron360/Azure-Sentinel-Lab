

Azure sentinel homelab guide · MD
# Setting Up Azure Sentinel in Your Cybersecurity Homelab
 
A step-by-step guide to standing up Microsoft Sentinel as a SIEM for your homelab, connecting Linux endpoints via Azure Arc, and collecting logs for analysis.
 
## Prerequisites
 
- A Microsoft/Outlook account for Azure sign-up
- A credit card (required for identity verification, even on the free tier)
- At least one Linux (or Windows) machine in your homelab to act as a monitored endpoint
- Basic comfort with the Linux terminal and `sudo`
---
 
## 1. Sign Up for Azure Free Tier
 
1. Go to [azure.microsoft.com/free](https://azure.microsoft.com/free).
2. Sign up with a Microsoft account (or create one).
3. Complete identity verification (phone + card).
4. You'll receive **$200 in free credit for 30 days**, plus a set of "always free" services for 12 months.
> **Tip:** Set a spending/budget alert in **Cost Management + Billing** right away so your homelab doesn't quietly burn through the credit.
 
---
 
## 2. Create a Log Analytics Workspace
 
Sentinel needs a Log Analytics Workspace (LAW) to store and query data.
 
1. In the [Azure Portal](https://portal.azure.com), search for **Log Analytics workspaces**.
2. Click **+ Create**.
3. Fill in:
   - **Subscription**: your free trial subscription
   - **Resource Group**: create a new one, e.g. `rg-homelab-sentinel`
   - **Name**: e.g. `law-homelab-sentinel`
   - **Region**: pick one close to you (keep it consistent for all resources)
4. Click **Review + create**, then **Create**.
5. Wait for deployment to finish, then click **Go to resource**.
---
 
## 3. Add Microsoft Sentinel to the Workspace
 
1. In the Azure Portal search bar, search for **Microsoft Sentinel**.
2. Click **+ Create**.
3. Select the Log Analytics workspace you just created (`law-homelab-sentinel`).
4. Click **Add**.
5. Sentinel is now layered on top of your workspace and will appear in your Sentinel resource list.
---
 
## 4. Connect Endpoints via Azure Arc
 
Azure Arc lets you onboard on-prem/homelab machines as if they were Azure resources, which is required to apply Data Collection Rules to them.
 
1. In the Azure Portal, search for **Azure Arc**.
2. Go to **Servers** → **+ Add/Create** → **Add a machine**.
3. Choose **Add a single server** (or multiple, if scripting bulk onboarding).
4. Select:
   - Subscription and the same resource group (`rg-homelab-sentinel`)
   - Region
   - Operating system of your endpoint (Linux/Windows)
5. Azure will generate an onboarding script.
6. **Copy the script** and run it on your endpoint via SSH/terminal.
7. Once the script finishes, verify the connection on the endpoint:
```bash
sudo azcmagent show
```
 
You should see output confirming:
- **Agent Status**: Connected
- **Resource Name / Resource Group** matching what you configured in Arc
---
 
## 5. Create a Data Collection Rule (DCR)
 
A DCR tells Azure Monitor Agent what logs to collect and where to send them.
 
1. Go to the **Azure Portal**.
2. Search for **Monitor** in the top search bar, then select **Data Collection Rules** under **Settings**.
3. Click **+ Create**:
   - **Rule Name**: e.g. `dcr-linux-syslog`
   - **Platform Type**: Linux
4. Under the **Resources** tab:
   - Click **+ Add resources** and select your Azure Arc-enabled Linux server.
   - > **Note:** Attaching your Arc server here automatically installs the **Azure Monitor Agent (AMA)** extension onto your Linux machine.
5. Under the **Collect and deliver** tab:
   - Click **Add data source** → select **Linux Syslog** (or **Performance Counters** / **Custom Text Logs**).
   - Choose your desired log levels — e.g. `LOG_INFO`, `LOG_WARN`, `LOG_ERR` for facilities like `auth`, `daemon`, `syslog`, etc.
   - Under **Destination**, choose **Azure Monitor Logs** and select your Log Analytics Workspace (`law-homelab-sentinel`).
6. Click **Review + create**, then **Create**.
### Re-checking / Re-associating a Resource on the DCR
 
If an endpoint (e.g. a `wazuh-server`) isn't picking up the DCR, or you need to force a refresh of the association, do the following:
 
1. Open the **Azure Portal** and search for **Data Collection Rules**.
2. Click your Syslog DCR (e.g., `dcr-linux-syslog`).
3. Select **Resources** under **Configuration**:
   - Check if `wazuh-server` is listed.
   - **If it IS listed:** Uncheck/Remove `wazuh-server`, click **Save**, wait 10 seconds, then click **+ Add**, re-select `wazuh-server`, and click **Save** again.
   - **If it is NOT listed:** Click **+ Add**, select `wazuh-server`, and click **Save**.
> This remove-and-re-add cycle forces Azure Monitor Agent to re-pull its configuration, which is useful if logs stop flowing after a config change or extension update.
 
---
 
## 6. Verify Agent Status on the Endpoint
 
Back on your Linux machine, confirm the Azure Monitor Agent installed correctly and is running:
 
```bash
systemctl status azuremonitoragent
```
 
Look for `active (running)` in the output. If it's not running, check:
 
```bash
sudo journalctl -u azuremonitoragent -n 100 --no-pager
```
 
---
 
## 7. Validate Workspace Ingestion
 
Allow **5–10 minutes** post-sync, then confirm ingestion directly in the portal:
 
1. Go to the **Azure Portal** and search for **Log Analytics workspaces**.
2. Click your workspace (e.g. `law-homelab-sentinel`).
3. In the left-hand menu, under **General**, click **Logs**.
   - If a "Queries" pop-up appears, click **X** to dismiss it and go straight to the query editor.
4. Paste in and run the following queries (click **Run** or press `Shift + Enter`):
```kql
// Verify Agent Heartbeat
Heartbeat
| where Category == "Azure Monitor Agent"
| summarize arg_max(TimeGenerated, *) by Computer
```
 
```kql
// Verify Syslog Ingestion
Syslog
| order by TimeGenerated desc
```
 
- The **Heartbeat** query confirms the agent itself is checking in with the workspace, even before any syslog data arrives.
- The **Syslog** query confirms actual log records are being ingested.
If rows appear in both, your endpoint is successfully reporting into Sentinel.
 
---
 
## Azure Arc & AMA Syslog Ingestion Troubleshooting
 
If logs still aren't showing up after the steps above, work through this deeper diagnostic chain in order.
 
### 1. Verify Azure Arc Agent Health
 
Ensure the Arc control plane and underlying daemons are connected and running:
 
```bash
sudo azcmagent show
```
 
- Confirm **Agent Status** reads `Connected`.
- Verify the dependent daemons (`himdsd`, `arcproxyd`, `extd`, `gcad`) show as `active`.
### 2. Verify AMA Extension Status
 
Confirm that the Azure Monitor Agent extension is installed and healthy on the Arc host:
 
```bash
sudo azcmagent extension list
```
 
- Ensure `AzureMonitorLinuxAgent` is listed with state `Succeeded`.
### 3. Check Data Collection Rule (DCR) Download Status
 
Verify if the agent has pulled the DCR JSON payload from the Azure Monitor Configuration Service (AMCS):
 
```bash
sudo ls -la /etc/opt/microsoft/azuremonitoragent/config-cache/configchunks/
```
 
- **Empty directory:** The DCR is not assigned, or the AMCS sync has not fired.
- **`.json` file present:** The DCR has successfully synchronized to the machine.
### 4. Force Service & Extension Sync
 
If `configchunks` is empty or stale, restart the Arc extension manager and AMA daemons:
 
```bash
# Restart Arc Extension Manager & AMA Services
sudo systemctl restart extd
sudo systemctl restart azuremonitoragent
```
 
**Azure Portal re-association** (if `configchunks` remains empty):
 
1. Navigate to **Azure Portal → Monitor → Data Collection Rules**.
2. Select your Linux Syslog DCR → **Resources**.
3. Remove the target Arc host, save the rule, re-add the host, and save again to force an AMCS sync event.
### 5. Confirm Syslog Forwarding & Socket Generation
 
Once the DCR JSON populates in `configchunks`, verify that AMA generated its `rsyslog` drop-in rule and listening socket:
 
```bash
# Check for generated rsyslog rule
ls -l /etc/rsyslog.d/10-azuremonitoragent*.conf
 
# Check for active Unix socket
ls -la /run/azuremonitoragent/default_syslog.socket
```
 
If the rules exist but logs still aren't flowing, restart syslog:
 
```bash
sudo systemctl restart rsyslog
```
 
### 6. Validate Workspace Ingestion
 
Allow 5–10 minutes post-sync, then run these KQL queries in the linked Log Analytics Workspace:
 
```kql
// Verify Agent Heartbeat
Heartbeat
| where Category == "Azure Monitor Agent"
| summarize arg_max(TimeGenerated, *) by Computer
```
 
```kql
// Verify Syslog Ingestion
Syslog
| order by TimeGenerated desc
```
 
---
 
## 8. Create Detection Rules (Analytics Rules)
 
Analytics Rules are what turn raw ingested logs into actionable **incidents** in Sentinel.
 
1. Go to your **Microsoft Sentinel** resource in the Azure Portal.
2. In the left-hand menu, under **Configuration**, click **Analytics**.
3. Click **+ Create** → **Scheduled query rule** (most common rule type for homelab use).
4. **General tab:**
   - **Name**: e.g. `Detect Failed SSH Logins`
   - **Description**: brief summary of what the rule detects
   - **Tactics/Techniques**: optionally map to MITRE ATT&CK (e.g. `Credential Access`, `T1110`)
   - **Severity**: Low / Medium / High / Informational
5. **Set rule logic tab:**
   - Write a KQL query that defines the detection. Example — repeated failed SSH logins:
```kql
     Syslog
     | where Facility == "auth" and SyslogMessage has "Failed password"
     | summarize FailedAttempts = count() by Computer, HostName, bin(TimeGenerated, 5m)
     | where FailedAttempts > 5
```
   - Set **Query scheduling**: how often the rule runs (e.g. every 5 minutes) and the lookback period (e.g. last 5 minutes).
   - Set **Alert threshold**: e.g. trigger when results are "greater than 0".
6. **Incident settings tab:**
   - Leave **Create incidents from alerts** enabled.
   - Optionally enable **alert grouping** to combine related alerts into a single incident.
7. **Automated response tab (optional):**
   - Attach a **Playbook** (Logic App) here if you want automated actions (e.g. notify via email/Teams, disable an account).
8. Click **Review + create**, then **Create**.
> **Tip:** Start with a few high-signal rules (failed logins, sudo usage, new user creation) rather than importing every built-in template — homelab log volume is low, so overly broad rules will just create noise.
 
You can also browse Microsoft's **built-in rule templates** under the **Rule templates** tab in Analytics — many can be cloned and adapted to your Syslog schema instead of writing detections from scratch. To add one:
 
1. In **Microsoft Sentinel** → **Analytics**, click the **Rule templates** tab.
2. Use the search bar or **Data source** filter to find a template relevant to your data (e.g. filter by **Syslog** or search "SSH").
3. Click a template to preview its description, logic, and required data sources.
4. Click **Create rule** in the details pane.
5. Walk through the same tabs as a custom rule (**General**, **Set rule logic**, **Incident settings**, **Automated response**) — the fields will be pre-populated from the template, but you can edit the KQL query and thresholds to fit your homelab.
6. Click **Review + create**, then **Create** to add the rule to your active Analytics rules list.
---
 
## 9. Triage Incidents
 
Once an analytics rule fires, it produces an **incident** for you to investigate.
 
1. In Microsoft Sentinel, go to **Incidents** under **Threat management**.
2. Click an incident to open its details pane. You'll see:
   - **Severity** and **status** (New / Active / Closed)
   - **Alerts** that triggered it, and the **entities** involved (host, account, IP)
   - A timeline of related events
3. Click **Investigate** to open the investigation graph, which visually maps related entities (accounts, hosts, IPs) so you can see how they connect.
4. Work the incident:
   - Click into the underlying **alerts** to review the raw log entries that triggered the detection.
   - Use **Logs** (or the entity's **Related events** tab) to pivot and run follow-up KQL queries — e.g. checking what that account/IP did before and after the alert.
5. Assign an **owner** (yourself, in a homelab) and set **status** to **Active** while investigating.
6. Add **comments** to the incident documenting what you found — good practice for building an audit trail, even solo.
7. Once resolved, set **status** to **Closed** and select a **classification reason**:
   - `True Positive`, `Benign Positive`, or `False Positive`
   - Add a short justification — this trains your judgment and gives you a record to tune future rules against.
> **Homelab tip:** Deliberately trigger some of your own detections (e.g. SSH brute-force a test account from another VM) so you can practice the full triage workflow end-to-end before relying on it for real detections.
 
---
 
## Troubleshooting Tips
 
| Symptom | Likely Cause | Fix |
|---|---|---|
| `azcmagent show` shows "Disconnected" | Firewall/proxy blocking outbound HTTPS | Ensure ports 443 outbound to `*.his.arc.azure.com` and related endpoints are open |
| No logs in `Syslog` table | DCR not associated with endpoint, or AMA extension not installed | Recheck DCR **Resources** tab; confirm extension in Arc server's **Extensions** blade |
| AMA service not running | Extension install failed | Reinstall via Arc server → Extensions → remove/re-add Azure Monitor Agent |
| Free credit draining fast | Log Analytics ingestion/retention costs | Set daily cap on workspace under **Usage and estimated costs** |
 
---
 
## Next Steps
 
- Enable **Analytics Rules** in Sentinel to generate incidents from your ingested logs.
- Explore built-in **Workbooks** for visualization.
- Add Windows endpoints using the same Arc + DCR pattern (swap Syslog for **Windows Event Logs**).
- Consider onboarding a honeypot or vulnerable VM to generate interesting log data for detection practice.
 
