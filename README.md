# Sophos-Central-Endpoint-Protection-Lab
A lab covering agent deployment and real-time threat detection using an industry-standard test file

**Platform:** Sophos Central (Free Trial)  
**Environment:** Windows 10 Home VM (VMware Workstation)  
**Device:** DESKTOP-79QNLKS  
**Date:** October 2026

--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

## Overview

This lab demonstrates hands-on use of Sophos Central Endpoint Protection — a commercial endpoint security platform used in enterprise environments. It covers agent deployment, real-time threat detection using an industry-standard test file, and live endpoint querying using osquery SQL through Sophos Live Discover.

**Skills demonstrated:**
- Endpoint agent deployment and console verification
- Real-time malware detection and alert analysis
- Live endpoint investigation using SQL queries (osquery)
- Identifying running processes, open ports, and user sessions

--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------


## Lab Setup

- **Sophos Central free trial** - cloud-hosted console at fusion.sophos.com
- **Windows 10 Home VM** running in VMware Workstation on an isolated Internal Network
- **Sophos Endpoint agent** installed directly on the Windows VM

--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------


## Part 1 - Agent Installation and Verification

Downloaded the Complete Windows Installer from:  
**My Products → Endpoint → Installers → Download Complete Windows Installer**

The installer ran silently on the Windows 10 VM with no configuration required. After approximately 2 minutes, the device appeared in the Sophos Central console under **Endpoint → Computers**.

![Agent installed and showing Protected status](images/image1.png)

The device **DESKTOP-79QNLKS** registered with the following status:
- **Online** — active connection to the Sophos cloud console
- **Tamper protection: On** — prevents unauthorised removal of the agent
- **Encryption: ✓** — device encryption policy applied
- **DNS & ZTNA: ✓** — Zero Trust Network Access policy applied
- **Agent mode: XDR** — extended detection and response tier active

--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------


## Part 2 - EICAR Threat Detection Test

The EICAR test file is an industry-standard, harmless string that every antivirus product is required to detect. It is not real malware — it exists specifically to verify that endpoint protection is functioning.

**Source:** https://www.eicar.org/download-anti-malware-testfile/

The `eicar.com` file was downloaded and executed on the protected Windows VM. Sophos detected and logged each attempt in real time.

**Result:** 4 detections logged in **Threat Analysis Center → Threat Graphs**

![EICAR detections in Threat Graphs](images/image2.png)

Each detection shows:
- **Name:** EICAR-AV-Test
- **Status:** New
- **Priority:** Low
- **Device:** DESKTOP-79QNLKS
- **Time:** Oct 26, 2026 — between 4:10 PM and 4:15 PM

This confirms that Sophos real-time protection is active and correctly identifying and logging threat events on the endpoint.

--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------


## Part 3 - Live Discover Queries (osquery)

Live Discover is Sophos's live endpoint query tool, found under **Threat Analysis Center → Live Discover**. It uses osquery — an open-source framework that exposes the operating system as a SQL database, allowing analysts to query running processes, open ports, user sessions, and more in real time.

**Mode used:** Designer Mode (custom SQL)  
**Source:** Live Endpoint  
**Device targeted:** DESKTOP-79QNLKS (1/1 Devices completed on each query)

--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------


### Query 1 - All Running Processes

```sql
SELECT name, path FROM processes;
```

**Purpose:** Returns a full list of every process currently running on the endpoint, along with the file path of its executable. This is the first check in any malware investigation — establishing a baseline of what is running before narrowing down suspicious activity.

![All running processes query results](images/image3.png)

**Result:** Multiple processes returned including `[System Process]`, `System`, `Registry`, `smss.exe` (`C:\Windows\System32\smss.exe`), `csrss.exe` (`C:\Windows\System32\csrss.exe`), and `wininit.exe` (`C:\Windows\System32\wininit.exe`) — all standard Windows system processes running from expected paths.

--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------


### Query 2 - Find a Specific Process

```sql
SELECT name, path FROM processes WHERE name = 'explorer.exe';
```

**Purpose:** Uses a WHERE clause to filter results to a single named process. In a real investigation, once a suspicious process is identified, this query confirms whether it is running and, critically, where it is running from. Malware often impersonates legitimate process names while running from unexpected directories.

![explorer.exe query result](images/image4.png)

**Result:** One row returned - `explorer.exe` running from `C:\Windows\explorer.exe`. The path is the expected system location, confirming this is the legitimate Windows shell process and not an impersonator.

--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------


### Query 3 - Logged-In Users

```sql
SELECT username, tty, host FROM logged_in_users;
```

**Purpose:** Returns active user sessions on the endpoint - useful when investigating unauthorised access or confirming who was logged in at the time of an incident.

![Logged in users query — no results](images/image5.png)

**Result:** No items returned. This is expected behaviour on Windows. The `logged_in_users` osquery table has limited support on Windows endpoints and does not reliably report interactive console sessions. The query completed successfully (1/1 Devices completed) - there was simply no data to return from this table on a Windows host. This table functions correctly on Linux and macOS endpoints.

--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

### Query 4 - Open Network Ports

```sql
SELECT pid, port, family FROM listening_ports;
```

**Purpose:** Lists every port the machine is currently listening on, with the process ID responsible and the address family (IPv4/IPv6). If malware opens a backdoor, it will appear here as an unexpected listening port. This query is a standard check during network-based threat investigation.

![Open network ports query results](images/image6.png)

**Result:** Multiple ports returned including 135, 445, 5040, 7680, 49064, 49065, and 60000 - all standard Windows system ports. Port 135 is RPC, port 445 is SMB (Windows file sharing). No unexpected or suspicious listening ports were identified.

--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------


## Summary

|      Step        |                       Task                       |                Result                    |
|------------------|--------------------------------------------------|------------------------------------------|
| Agent deployment | Sophos Endpoint agent installed on Windows 10 VM | ✅ Device showing Online and Protected   |
| Threat detection | EICAR test file executed on protected endpoint   | ✅ 4 detections logged in Threat Graphs  |
| Live query 1     | `SELECT name, path FROM processes`               | ✅ All running processes returned        |
| Live query 2     | `WHERE name = 'explorer.exe'`                    | ✅ 1 result - path confirmed as expected |
| Live query 3     | `SELECT username, tty, host FROM logged_in_users`| ⚠️ No data - limited Windows support     |
| Live query 4     | `SELECT pid, port, family FROM listening_ports`  | ✅ All listening ports returned          |

--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------


## Key Takeaways

- Sophos Central provides a cloud-hosted console that registers endpoints within minutes of agent installation
- The EICAR test confirms real-time on-access scanning is active - the agent caught every execution attempt
- Live Discover bridges endpoint detection and investigation - analysts can run SQL directly against a live machine without needing remote access
- osquery's `processes` table is the standard starting point for malware triage; the `listening_ports` table is the equivalent for network-based threats
- The `logged_in_users` table behaviour differs by OS - results on Windows should be treated as unreliable without supplementary telemetry
