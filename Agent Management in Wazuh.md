# Wazuh Agent Management: A Practical Guide for Security Engineers
 
**By Md Jamiul Haque | SOC Analyst & Blue Teamer**
 
---

 <img src="https://imgs.search.brave.com/CjKzIViYGbWoDHhE6UG8s21OMLcQsNf2JocoooLh350/rs:fit:860:0:0:0/g:ce/aHR0cHM6Ly83NzIz/OTk4LmZzMS5odWJz/cG90dXNlcmNvbnRl/bnQtbmExLm5ldC9o/dWJmcy83NzIzOTk4/L3dhenVoLnBuZw">
---
 
If you've spent any time in a SOC managing endpoint visibility, you already know that keeping your monitoring agents organised and up to date isn't just good hygiene — it's operationally critical. The moment your agents fall out of sync, your detection coverage has gaps you might not even be aware of.
 
Wazuh is one of the most capable open-source SIEM and XDR platforms out there, and one of its underappreciated strengths is how much control it gives you over agent management. In this guide, I'll walk you through everything you need to know — from organising agents into groups, to performing remote upgrades, to cleaning up stale agents — all from both the command line and the Dashboard.
 
Whether you prefer clicking through a GUI or living in the terminal, I've got you covered.
 
---
 
## Table of Contents
 
1. Why Agent Management Matters in a SOC
2. Understanding Agent Groups
3. Agent Grouping — CLI Method
4. Agent Grouping — Dashboard (GUI) Method
5. Upgrading Wazuh Agents via CLI
6. Upgrading Wazuh Agents via Dashboard
7. Removing Wazuh Agents
8. Final Thoughts
---
 
## 1. Why Agent Management Matters in a SOC
 
Picture this: you have 200 endpoints reporting into your Wazuh manager. Some are Windows workstations, some are Linux servers, a few are critical infrastructure hosts. They're all dumping logs into the same pipeline — with the same default configuration. No differentiation, no targeted monitoring, no group-specific rules.
 
From a detection standpoint, that's noise. From an operational standpoint, that's a management nightmare.
 
Wazuh solves this through **agent groups** — a way to assign tailored configurations to sets of agents based on their role, OS, or criticality. Pair that with remote upgrade capabilities, and you have a surprisingly powerful fleet management system built right into your SIEM.
 
Let's dig in.
 
---
 
## 2. Understanding Agent Groups
 
Every Wazuh agent, the moment it connects to the manager for the first time, gets automatically assigned to a group called **"default."** This group has its own configuration file and a set of pre-installed compliance check files — CIS benchmarks, rootkit detection rules, and OS-specific audit policies.
 
The real power comes when you create your own groups. A common and practical approach in a SOC environment is to separate by operating system and criticality:
 
- **Windows** — for Windows workstations and servers
- **Linux** — for Linux-based endpoints
- **Critical** — for high-value assets requiring stricter monitoring rules
Each group has its own configuration directory on the Wazuh manager, located at:
 
```
/var/ossec/etc/shared/<GROUP_NAME>/
```
 
The most important file in each group folder is **`agent.conf`** — this is where you define what the agents in that group should monitor, which log files to collect, what FIM paths to watch, and other behavioural settings. When an agent is assigned to a group, it automatically pulls this configuration from the manager.
 
The default group lives at `/var/ossec/etc/shared/default/` and comes pre-loaded with a range of compliance files right out of the box.
 
---
 
## 3. Agent Grouping — CLI Method
 
### 3.1 Exploring the Shared Directory
 
Start by listing the contents of the shared directory to see your current group folders:
 
```bash
ls -lah /var/ossec/etc/shared/
```
 
*[INSERT SCREENSHOT — Output of the shared directory listing showing existing group folders]*
 <img src="">
You'll see a subfolder for each group that exists on your manager. If you haven't created any custom groups yet, you'll only see the `default` folder here.
 
Now take a look inside the `default` group:
 
```bash
ls -lah /var/ossec/etc/shared/default
```
 
*[INSERT SCREENSHOT — Contents of the /var/ossec/etc/shared/default directory showing compliance and config files]*
 
Inside you'll find files like `cis_debian_linux_rcl.txt`, `win_audit_rcl.txt`, `rootkit_files.txt`, and more. These are Wazuh's built-in security and compliance check definitions, distributed automatically to all agents in the default group.
 
Now here's what a custom group looks like — let's use a `windows` group as an example:
 
*[INSERT SCREENSHOT — Contents of the custom "windows" group folder showing agent.conf]*
 
As you can see, a custom group starts lean — just the `agent.conf` file. That's your canvas.
 
Let's look at what an actual `agent.conf` might contain:
 
*[INSERT SCREENSHOT — Contents of agent.conf from the "windows" group showing localfile/log monitoring configuration]*
 
In my lab, the Windows group's `agent.conf` tells agents to monitor specific Windows Event Viewer channels and forward those events to the Wazuh manager. Once received, the manager processes and normalises them, and they appear in the Dashboard. It's a clean, centralised way to push targeted monitoring instructions without touching each endpoint individually.
 
---
 
### 3.2 Creating a New Agent Group
 
Wazuh ships with a built-in CLI tool for group management: `agent_groups`. Start by running it to see your existing groups and the number of agents in each:
 
```bash
/var/ossec/bin/agent_groups
```
 
To create a new group, use the `-a` (add) and `-g` (group name) flags:
 
```bash
/var/ossec/bin/agent_groups -a -g <your_group_name>
```
 
If the operation succeeds, you'll get confirmation that the group was created. Run `agent_groups` again to verify it appears in the list.
 
*[INSERT SCREENSHOT — Terminal showing the creation of a new group and the updated group list confirming its addition]*
 
> **SOC Tip:** Use a naming convention from day one. Something like `win-workstations`, `linux-servers`, or `critical-assets` beats generic names when you're staring at a list of 15 groups at 2am.
 
---
 
### 3.3 Assigning an Agent to a Group
 
Once your group exists, assigning an agent is straightforward. You'll need the agent's ID:
 
```bash
/var/ossec/bin/manage_agents
```
 
Type `L` and hit Enter to list all agents with their IDs and statuses.
 
> **Note:** Agent IDs in Wazuh are unique — you'll never have two agents sharing the same ID. Duplicate ID conflicts will show up as connection errors in your logs, which is worth knowing when troubleshooting.
 
With the agent ID in hand, run:
 
```bash
/var/ossec/bin/agent_groups -a -i <agent_id> -g <group_name>
```
 
Then verify the assignment:
 
```bash
/var/ossec/bin/agent_groups
```
 
Or for a more targeted check:
 
```bash
/var/ossec/bin/agent_groups -l
```
 
*[INSERT SCREENSHOT — Terminal showing the agent assignment command and the subsequent group listing confirming the agent was added]*
 
*[INSERT SCREENSHOT — Zoomed-in view of the group listing with the agent now appearing under the correct group]*
 
---
 
### 3.4 Assigning Agents via the Wazuh API
 
If you prefer automation or work in a cluster environment, the Wazuh RESTful API is the way to go. It's a bit more involved, but extremely powerful once you get the hang of it.
 
**Step 1 — Authenticate and get your JWT token:**
 
```bash
curl -u <USER>:<PASSWORD> -k -X POST "https://<HOST_IP>:55000/security/user/authenticate"
```
 
Default credentials are `wazuh-wui:wazuh-wui`. The response will contain your token — copy the full string.
 
> **Heads up:** JWT tokens issued by the Wazuh API expire after **15 minutes (900 seconds)**. If your subsequent API calls start failing with auth errors, just re-run the authentication command above to get a fresh token.
 
**Step 2 — Assign the agent to a group:**
 
```bash
curl -k -X PUT "https://<WAZUH_MANAGER_IP>:55000/agents/<agent_id>/group/<group_name>?pretty=true" \
  -H "Authorization: Bearer $TOKEN"
```
 
> Replace `$TOKEN` with the actual token string — the variable won't resolve unless you've exported it in your session.
 
*[INSERT SCREENSHOT — Terminal output showing a successful API response after assigning an agent to a group]*
 
**Step 3 — Verify via CLI:**
 
```bash
/var/ossec/bin/agent_groups -l -g <group_name>
```
 
Or via API:
 
```bash
curl -k -X GET "https://<WAZUH_MANAGER_IP>:55000/groups/<group_name>/agents?pretty=true&select=id,name" \
  -H "Authorization: Bearer $TOKEN"
```
 
*[INSERT SCREENSHOT — API response showing the list of agents now assigned to the group]*
 
**One important thing to know:** A single agent can belong to multiple groups simultaneously. It will pull and merge configurations from all assigned groups. This is useful when you want a baseline config from one group and a role-specific config from another.
 
---
 
### 3.5 Removing an Agent from a Group
 
To remove an agent from a specific group without affecting its other group memberships:
 
```bash
/var/ossec/bin/agent_groups -r -i <agent_id> -g <group_name> -q
```
 
Then confirm it's gone:
 
```bash
/var/ossec/bin/agent_groups -s -i <agent_id>
```
 
*[INSERT SCREENSHOT — Terminal output showing the agent removal command followed by the group membership check confirming removal]*
 
*[INSERT SCREENSHOT — Group membership output showing the agent now belongs only to its remaining groups]*
 
---
 
### 3.6 Overwriting an Agent's Group Assignments
 
Sometimes you need to completely reassign an agent — wiping its current group memberships and starting fresh. The `-f` flag (force) handles this:
 
First, check what groups the agent currently belongs to:
 
```bash
/var/ossec/bin/agent_groups -s -i <agent_id>
```
 
*[INSERT SCREENSHOT — Output showing the agent currently belonging to multiple groups (e.g., "windows" and "demo")]*
 
Then overwrite all existing memberships and assign it to a single new group:
 
```bash
/var/ossec/bin/agent_groups -a -f -i <agent_id> -g <new_group_name>
```
 
Confirm the change:
 
```bash
/var/ossec/bin/agent_groups -s -i <agent_id>
```
 
*[INSERT SCREENSHOT — Full terminal sequence showing the force-reassignment and the final group membership confirmation]*
 
---
 
## 4. Agent Grouping — Dashboard (GUI) Method
 
If you prefer working visually — or if you're training a teammate who isn't yet comfortable in the terminal — the Wazuh Dashboard provides a clean interface for all the same operations.
 
### 4.1 Navigating to Group Management
 
Log into the Wazuh Dashboard and head to:
 
**Agent Management → Groups**
 
*[INSERT SCREENSHOT — Wazuh Dashboard navigation showing the path to Agent Management > Groups]*
 
You'll see a list of all your groups, with agent counts and action buttons on the right side of each row.
 
*[INSERT SCREENSHOT — Groups overview page showing all groups, agent counts, and the Actions column with icons]*
 
*[INSERT SCREENSHOT — Closer view of the Actions column showing the eye (view), pencil (edit), and trash (delete) icons]*
 
The three action icons do the following:
- 👁️ **Eye** — View group details and the agents assigned to it
- ✏️ **Pencil** — Open and edit the group's `agent.conf` configuration file
- 🗑️ **Trash** — Delete the group entirely
In the upper right, you have buttons to **Add new group**, refresh the view, or export the group data as a CSV file.
 
---
 
### 4.2 Assigning an Agent to a Group via Dashboard
 
Click the **eye icon** next to the group you want to manage (e.g., the "windows" group).
 
You'll see two tabs: **Agents** (currently assigned agents) and **Files** (the group's config files, including `agent.conf`).
 
*[INSERT SCREENSHOT — Group detail view showing the Agents tab with currently assigned agents]*
 
*[INSERT SCREENSHOT — Group detail view showing the Files tab with agent.conf listed]*
 
To assign a new agent, click **Manage Agents** in the upper right corner.
 
*[INSERT SCREENSHOT — "Manage Agents" button highlighted in the upper right of the group detail view]*
 
A two-panel view appears:
- **Left panel** — All available agents (not yet in this group)
- **Right panel** — Agents currently assigned to this group
*[INSERT SCREENSHOT — Manage Agents panel showing left (available) and right (assigned) columns]*
 
*[INSERT SCREENSHOT — Closer view of the two-panel layout showing agent names in both columns]*
 
Double-click an agent on the left to move it to the right, then click **Apply Changes** to confirm.
 
*[INSERT SCREENSHOT — Post-assignment view showing the agent now visible in the right "assigned" panel]*
 
After saving, the agent will appear in the group list and automatically receive the group's configuration.
 
*[INSERT SCREENSHOT — Updated group view showing the newly assigned agent now listed under the group]*
 
---
 
### 4.3 Removing an Agent from a Group via Dashboard
 
In the **Manage Agents** panel, double-click the agent in the right-hand (assigned) panel to move it back to the left. Alternatively, use the **Remove selected items** option. Hit **Apply Changes** to confirm.
 
*[INSERT SCREENSHOT — Manage Agents panel showing the remove action with the agent being moved back to the available column]*
 
---
 
### 4.4 Editing a Group's Configuration via Dashboard
 
Go back to **Agent Management → Groups**. Click the **pencil icon** next to the group you want to configure.
 
*[INSERT SCREENSHOT — Groups list with the pencil/edit icon highlighted for a specific group]*
 
You'll land directly in the `agent.conf` editor. This is the same file we discussed earlier — where you define log file paths to monitor, FIM directories, syscheck settings, and more. Any changes here are automatically pushed to all agents in that group.
 
*[INSERT SCREENSHOT — agent.conf editor in the Dashboard showing an example Windows Event Log monitoring configuration]*
 
To create a brand new group, just click **Add new group** in the upper right of the Groups page, give it a name, and save.
 
*[INSERT SCREENSHOT — "Add new group" dialog or button visible in the Groups section]*
 
---
 
## 5. Upgrading Wazuh Agents — CLI Method
 
Keeping agents on the same version as your Wazuh manager isn't optional — it's a requirement for full feature parity. When Wazuh releases a new module or detection capability, outdated agents simply won't support it. In a large-scale environment, manually updating each endpoint is impractical. Thankfully, Wazuh supports **remote agent upgrades** right out of the box.
 
> **Important version rules to know:**
> - Agent version must be **equal to or lower** than the manager version
> - An agent running a **higher** version than the manager will fail to connect
> - In a **clustered Wazuh deployment** (Master + Worker nodes), it's recommended to use the RESTful API method for upgrades
 
---
 
### 5.1 Using the `agent_upgrade` Tool
 
Connect to your Wazuh manager via SSH and start by listing agents that are running outdated versions:
 
```bash
/var/ossec/bin/agent_upgrade -l
```
 
*[INSERT SCREENSHOT — Terminal output of agent_upgrade -l showing agents on older versions (e.g., 4.12) that are eligible for upgrade]*
 
You can also run the tool with no flags to see all available options:
 
```bash
/var/ossec/bin/agent_upgrade
```
 
*[INSERT SCREENSHOT — agent_upgrade help output showing all available flags and usage options]*
 
To upgrade a single agent:
 
```bash
/var/ossec/bin/agent_upgrade -a <agent_id>
```
 
To upgrade multiple agents at once:
 
```bash
/var/ossec/bin/agent_upgrade -a 001 002 003 004 005
```
 
> **Prerequisite:** The target agent must be in **Active** status and connected to the manager. Disconnected agents can't be upgraded remotely.
 
*[INSERT SCREENSHOT — Terminal showing the upgrade command being run and the successful upgrade output from version 4.12 to 4.14]*
 
> **Gotcha to watch out for:** Occasionally, the upgrade command returns output saying the agent was updated to the same version it was already on. This is a known quirk — just run the command again and it typically completes successfully on the second attempt.
 
*[INSERT SCREENSHOT — Terminal showing the quirk where the upgrade reports the same version, and then the re-run succeeding]*
 
After the upgrade, verify the agent's current version:
 
```bash
/var/ossec/bin/agent_control -i <agent_id>
```
 
*[INSERT SCREENSHOT — agent_control output confirming the agent is now running Wazuh 4.14]*
 
---
 
### 5.2 Upgrading Agents via RESTful API
 
For clustered environments or automated workflows, the API approach is recommended. Here's the full process:
 
**Step 1 — Authenticate:**
 
```bash
TOKEN=$(curl -u <WAZUH_API_USER>:<WAZUH_API_PASSWORD> -k -X POST \
  "https://localhost:55000/security/user/authenticate?raw=true")
```
 
Default credentials are `wazuh:wazuh`. If you ran the password rotation script during initial setup, retrieve the current API password with:
 
```bash
tar -axf wazuh-install-files.tar wazuh-install-files/wazuh-passwords.txt -O | grep -P "'wazuh'" -A 1
```
 
**Step 2 — Check your token:**
 
```bash
echo $TOKEN
```
 
**Step 3 — Verify API connectivity:**
 
```bash
curl -k -X GET "https://localhost:55000/?pretty=true" \
  -H "Authorization: Bearer $TOKEN"
```
 
*[INSERT SCREENSHOT — Terminal showing successful API authentication and the test connectivity response]*
 
**Step 4 — List agents with outdated versions:**
 
```bash
curl -k -X GET "https://<WAZUH_MANAGER_IP>:55000/agents?pretty=true&select=id,name,version&status=active" \
  -H "Authorization: Bearer $TOKEN"
```
 
**Step 5 — Trigger the upgrade:**
 
```bash
curl -k -X PUT "https://<WAZUH_MANAGER_IP>:55000/agents/upgrade?agents_list=<agent_id>,<agent_id>&pretty=true" \
  -H "Authorization: Bearer $TOKEN"
```
 
*[INSERT SCREENSHOT — API response showing the upgrade being initiated for the specified agents]*
 
**Step 6 — Check upgrade result:**
 
```bash
curl -k -X GET "https://<WAZUH_MANAGER_IP>:55000/agents/upgrade_result?agents_list=<agent_id>&pretty=true" \
  -H "Authorization: Bearer $TOKEN"
```
 
*[INSERT SCREENSHOT — API response showing "status: updated" and "message: success" for the upgraded agent]*
 
**Step 7 — Confirm the new version:**
 
```bash
curl -k -X GET "https://<WAZUH_MANAGER_IP>:55000/agents?agents_list=<agent_id>&pretty=true&select=version" \
  -H "Authorization: Bearer $TOKEN"
```
 
*[INSERT SCREENSHOT — API response confirming the agent is now running version 4.14.0]*
 
---
 
## 6. Upgrading Wazuh Agents — Dashboard Method
 
If you'd rather handle upgrades from the UI — especially useful during a quick operational check — the Dashboard makes it straightforward.
 
Log into the Dashboard and navigate to:
 
**Agent Management → Summary**
 
*[INSERT SCREENSHOT — Dashboard navigation path to Agent Management > Summary]*
 
The agent list shows each agent's name, status, and version. If an agent is running an outdated version, you'll see a **red dot** next to its version number — your visual cue to act.
 
*[INSERT SCREENSHOT — Agent summary list showing agents with red dots next to outdated version numbers]*
 
### Upgrading a Single Agent
 
Find the agent you want to upgrade, click the **three-dot menu (⋮)** in the Actions column, and select **Upgrade**.
 
*[INSERT SCREENSHOT — Three-dot menu expanded showing the "Upgrade" option for a specific agent]*
 
*[INSERT SCREENSHOT — Upgrade confirmation or progress indicator appearing after clicking Upgrade]*
 
The process typically takes around 1–2 minutes. You can monitor the status directly in the Dashboard.
 
*[INSERT SCREENSHOT — Dashboard showing the agent upgrade progress with an "in progress" status indicator]*
 
*[INSERT SCREENSHOT — Dashboard showing the upgrade status marked as "done"]*
 
Once complete, check the version column to confirm the update was successful.
 
*[INSERT SCREENSHOT — Agent list showing the updated agent now displaying version 4.14 with no red dot]*
 
---
 
### Upgrading Multiple Agents at Once
 
Select the agents you want to upgrade using the **checkboxes** on the left side of the agent list.
 
*[INSERT SCREENSHOT — Agent list with multiple agents selected via checkboxes]*
 
Then click **More → Upgrade agents**.
 
*[INSERT SCREENSHOT — "More" dropdown showing the "Upgrade agents" option]*
 
*[INSERT SCREENSHOT — Bulk upgrade confirmation dialog or progress view]*
 
> **Scale consideration:** Bulk upgrades from the Dashboard work well for smaller fleets. For environments with ~3,000+ agents, use the API with appropriate timeout parameters to avoid hitting the API timeout limit.
 
*[INSERT SCREENSHOT — Bulk upgrade progress screen showing multiple agents being updated]*
 
*[INSERT SCREENSHOT — Completion view showing all selected agents successfully upgraded]*
 
Once done, verify the version column for each upgraded agent to confirm everything landed correctly.
 
*[INSERT SCREENSHOT — Final agent list showing all upgraded agents now on version 4.14]*
 
---
 
## 7. Removing Wazuh Agents
 
When an endpoint is decommissioned, repurposed, or simply removed from your monitored scope, you'll want to clean it up from the Wazuh manager too. Stale agents clutter your dashboard, skew your agent count, and can cause confusion during incident response.
 
> **Important distinction:** Removing an agent from the Wazuh manager (via CLI or API) does **not** uninstall the agent software from the endpoint itself. It simply de-registers the agent from the manager — it will no longer appear in the Dashboard or consume a license slot. To fully remove Wazuh from the endpoint, you'll need to uninstall the package separately.
 
---
 
### 7.1 Removing Agents via CLI (`manage_agents`)
 
Connect to the Wazuh manager and run:
 
```bash
/var/ossec/bin/manage_agents
```
 
*[INSERT SCREENSHOT — manage_agents menu showing the available options including "Remove an agent (R)"]*
 
Type `R` and press Enter. You'll see the list of registered agents. Enter the ID of the agent you want to remove and confirm. Then list your agents again to verify it's gone.
 
*[INSERT SCREENSHOT — Full terminal sequence: selecting R, entering the agent ID, confirming deletion, and the updated agent list confirming removal]*
 
---
 
### 7.2 Removing Agents via API
 
Generate your JWT token:
 
```bash
TOKEN=$(curl -u <USER>:<PASSWORD> -k -X GET \
  "https://<WAZUH_MANAGER_IP>:55000/security/user/authenticate?raw=true")
```
 
Delete one or more agents by ID:
 
```bash
curl -k -X DELETE \
  "https://<WAZUH_MANAGER_IP>:55000/agents?pretty=true&older_than=0s&agents_list=<agent_id>,<agent_id>&status=all" \
  -H "Authorization: Bearer $TOKEN"
```
 
You can also clean up agents that have a **"never connected"** status or haven't been seen in a while. For example, to remove all disconnected or never-connected agents that have been inactive for more than 21 days:
 
```bash
curl -k -X DELETE \
  "https://<WAZUH_MANAGER_IP>:55000/agents?pretty=true&older_than=21d&agents_list=all&status=never_connected,disconnected" \
  -H "Authorization: Bearer $TOKEN"
```
 
This is a great command to run as part of a routine hygiene script — especially in dynamic environments where VMs or containers spin up and down frequently.
 
---
 
## 8. Final Thoughts
 
From a SOC perspective, agent management is one of those foundational skills that pays dividends quietly — until it doesn't, and you realise half your agents have been on an outdated version for six months or that your Windows endpoints have been running with the default config all along.
 
Here's what I'd take away from this guide:
 
- **Group your agents from day one.** Default configs are a starting point, not a destination.
- **Use the CLI for scripting and automation; use the Dashboard for quick operational tasks.** Both paths are valid; know when to use each.
- **Keep agents current.** Version mismatches between agents and the manager can silently limit your detection capabilities.
- **Use the API in cluster environments.** The `agent_upgrade` tool is great for standalone setups, but for HA deployments, the API is more reliable.
- **Schedule regular cleanup.** Stale agents are noise. The API's `older_than` filter makes it easy to automate this.
Agent management isn't glamorous work, but it's the kind of thing that separates a well-run Wazuh deployment from one that causes headaches during a real incident.
 
---
 
*If you found this useful, feel free to connect with me on LinkedIn — always happy to talk blue team and Wazuh.*
 
---
 
**Tags:** `Wazuh` `SIEM` `SOC` `Blue Team` `Cybersecurity` `Agent Management` `Open Source Security` `XDR` `Endpoint Security`
