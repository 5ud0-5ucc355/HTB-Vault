# Hack The Box Sherlock: LogJammer (Easy)

<p align="center">
  <img src="images/logjammer-badge.png" width="220" alt="LogJammer Sherlock badge: a jar labeled Log Flavour inside a green shield">
</p>

A beginner-friendly walkthrough of the Hack The Box Sherlock **LogJammer**, an investigation of Windows event logs from one computer (`DESKTOP-887GK2L`) where a user named **CyberJunkie** tampered with the system. No security background needed: every step explains what we looked for, how we found it, and why that was the right place to look.

> **Spoilers ahead.** The answers are hidden in collapsible sections. Try each question yourself first.

## Contents

- [Overview](#overview)
- [Background: the five ideas you need](#background-the-five-ideas-you-need)
- [Getting set up](#getting-set-up)
- [Part 1: The login (Question 1)](#part-1-the-login-question-1)
- [Part 2: The firewall rule (Questions 2 and 3)](#part-2-the-firewall-rule-questions-2-and-3)
- [Part 3: The audit policy and the scheduled task (Questions 4 to 7)](#part-3-the-audit-policy-and-the-scheduled-task-questions-4-to-7)
- [Part 4: What the antivirus found and did (Questions 8 to 10)](#part-4-what-the-antivirus-found-and-did-questions-8-to-10)
- [Part 5: The PowerShell command and the cleared logs (Questions 11 and 12)](#part-5-the-powershell-command-and-the-cleared-logs-questions-11-and-12)
- [Timeline of events (UTC)](#timeline-of-events-utc)
- [Glossary](#glossary)
- [Reference: command cheatsheet](#reference-command-cheatsheet)

## Overview

The case is solved in 12 questions, using five Windows event logs converted to JSON and searched with `jq`. All times are in **UTC**, the same clock the logs use.

<details>
<summary><b>Show all 12 answers at a glance</b></summary>

| # | Question | Answer | Where it was found |
| --- | --- | --- | --- |
| 1 | When did cyberjunkie successfully log in (UTC)? | 2023-03-27 14:37:09 UTC (HTB format: `27/03/2023 14:37:09`) | Security log, Event 4624 |
| 2 | Name of the firewall rule that was added | `Metasploit C2 Bypass` | Firewall log, Event 2004 |
| 3 | Direction of that rule | Outbound | Firewall log, Event 2004 |
| 4 | Subcategory of the changed audit policy | Other Object Access Events | Security log, Event 4719 |
| 5 | Name of the scheduled task | `HTB-AUTOMATION` | Security log, Event 4698 |
| 6 | Full path of the file the task runs | `C:\Users\CyberJunkie\Desktop\Automation-HTB.ps1` | Event 4698, task XML |
| 7 | Arguments of the task's command | `-A cyberjunkie@hackthebox.eu` | Event 4698, task XML |
| 8 | Tool the antivirus flagged as malware | SharpHound | Defender log, Events 1116 and 1117 |
| 9 | Full path of that malware | `C:\Users\CyberJunkie\Downloads\SharpHound-v1.1.0.zip` | Defender log, Event 1116 |
| 10 | Action the antivirus took | Quarantine | Defender log, Event 1117 |
| 11 | PowerShell command the user ran | `Get-FileHash -Algorithm md5 .\Desktop\Automation-HTB.ps1` | PowerShell log, Event 4104 |
| 12 | Which event log file was cleared | `Microsoft-Windows-Windows Firewall With Advanced Security/Firewall` (see Part 5 for the other cleared logs) | System and Security logs, Events 104 and 1102 |

</details>

## Background: the five ideas you need

**1. Event logs are the computer's diary.** Windows writes a record, called an event, whenever something notable happens: someone logs in, a program is blocked, a setting changes. Events are grouped into separate logs, such as Security, System, Windows Defender and PowerShell. Each log is saved as an `.evtx` file. Investigators read these files to reconstruct who did what, and when.

**2. Every event has an Event ID.** The Event ID is a number that says what kind of event it is, like a category code. For example, `4624` always means "someone logged on successfully" and `1102` always means "the Security log was cleared". Knowing a handful of IDs lets you jump straight to the interesting records instead of reading thousands of lines.

**3. Attackers leave traces and try to hide them.** Real attackers create accounts, add firewall exceptions, schedule tasks and run tools. Smart ones also clear logs or switch off auditing to cover their tracks. This case contains all of that, which is why the logs are worth reading closely.

**4. `.evtx` is a binary format, so we convert it.** You can't open an `.evtx` file in a text editor or search it on Linux. A converter turns each event into JSON, a plain-text format where every piece of data has a label. Here is a trimmed event so you can see the shape:

```json
{"Event": {"System": {"EventID": 4624, "TimeCreated": {"#attributes": {"SystemTime": "2023-01-15T09:12:30Z"}}},
           "EventData": {"TargetUserName": "CyberJunkie", "LogonType": 2}}}
```

The `{ }` brackets nest data inside labeled boxes. To read a value you walk down the labels: `Event`, then `System`, then `EventID`. (The timestamp above is only an example of the format.)

**5. `jq` is a search tool for JSON.** Given a file of JSON events, `jq` lets you say "show me only events where EventID is 4624 and the user is CyberJunkie, and print just the time". That is the core skill of this walkthrough.

One more detail: timestamps in the logs end in `Z`, which means UTC (the world-standard time zone). Using one clock lets you line up events from different logs.

**The files you are given:** five logs, each recording a different part of the system.

| File | What it records |
| --- | --- |
| `Security.evtx` | Logins, audit policy changes, scheduled tasks, log clearing |
| `Windows Firewall-Firewall.evtx` | Changes to Windows Firewall rules |
| `Windows Defender-Operational.evtx` | Antivirus detections and the actions taken |
| `Powershell-Operational.evtx` | Commands run in PowerShell |
| `System.evtx` | System-wide events, including some log clearing |

## Getting set up

I worked from the Hack The Box Pwnbox (a Linux machine in the browser), so nothing needs installing on your own computer. The setup has four steps.

### Step 1: Download the Sherlock

The download link from the Sherlock page contains `&` characters. In a Linux shell, an unquoted `&` means "run the previous command in the background", which cuts the link short and downloads nothing (you see `[1] 9586` job numbers and 0 bytes). The fix is to put the whole link in quotes and name the output file:

```bash
curl -L -o logjammer.zip "https://labs.hackthebox.com/api/v4/challenges/557/cdn/redirect?auth_user_id=102070&expires=1795143316&signature=e68f05b32ae77c4d9c271ae3fe68e00896bd73e994a24178a6eded12fa54d066"
file logjammer.zip
```

These links are signed and expire after a couple of hours. If you get a 403 or an error page instead of a zip, go back to the Sherlock page and copy a fresh link. Hack The Box Sherlock zips are normally password-protected with `hacktheblue`:

```bash
mkdir ~/Event-Logs && cd ~/Event-Logs
unzip -P hacktheblue ../logjammer.zip
```

### Step 2: Pick a converter that outputs JSON

There are several evtx tools, and they are easy to confuse:

| Tool | Output | Used here? |
| --- | --- | --- |
| `evtxexport` (from the `libevtx-utils` package) | Plain text or XML only | No: it has no JSON mode, and the text is hard to filter |
| `evtx_dump` (from the Rust `evtx` project) | JSON, one event per line | **Yes** |
| Chainsaw | JSON, plus detection rules | A good alternative, a single downloaded binary |

### Step 3: Install evtx_dump

`evtx_dump` is built with Rust's package manager, cargo. Install cargo first (`sudo apt install cargo` if it's missing). Then try the newest release:

```bash
cargo install evtx
```

On the Pwnbox this fails with "rustc 1.85.1 is not supported", because the latest version of evtx needs a newer Rust compiler than the box has. Adding `--locked` does not help. The fix is to install an older release of evtx that still builds on the older compiler. I used an older release (`0.8.4` is an example; if it fails to build for you, try an even older one):

```bash
cargo install evtx --version 0.8.4 --locked
export PATH="$HOME/.cargo/bin:$PATH"
which evtx_dump
```

The `export PATH` line tells your shell to look in cargo's install folder, which is why `evtx_dump: command not found` appears without it. If that version fails with another compiler error, try an even older release.

### Step 4: Convert every log to JSON

Run the converter once per file. `-o jsonl` means "JSON Lines": one event per line, which suits `jq`. Quote the names that contain spaces.

```bash
evtx_dump -o jsonl -f security.json   Security.evtx
evtx_dump -o jsonl -f firewall.json   "Windows Firewall-Firewall.evtx"
evtx_dump -o jsonl -f defender.json   "Windows Defender-Operational.evtx"
evtx_dump -o jsonl -f powershell.json "Powershell-Operational.evtx"
evtx_dump -o jsonl -f system.json     System.evtx
```

Check that it worked with `wc -l security.json` (thousands of lines, not 0) and look at the first event with `head -n1 security.json | jq .`. Make sure `jq` is installed (`which jq`); the Pwnbox has it.

<!-- SCREENSHOT: terminal showing the converted .json files
![Converted JSON files in the Event-Logs folder](images/01-converted-files.png)
-->

## Part 1: The login (Question 1)

**Question:** When did the user cyberjunkie successfully log in to the system (UTC)?

**Why the Security log.** Logins are security events, so they live in `Security.evtx`. Windows writes **Event ID 4624** each time an account logs on successfully, and **4625** when a login fails. Because the question says "successfully", we want 4624.

**Our first attempt found nothing.** We searched for the exact name `cyberjunkie`. The log stores the name as `CyberJunkie` (capital C and J), and `jq` treats uppercase and lowercase as different. That is a very common reason for an empty result. The fix is to lowercase the field before comparing:

```bash
jq -r 'select(.Event.System.EventID==4624 and (.Event.EventData.TargetUserName|ascii_downcase)=="cyberjunkie")
  | [.Event.System.TimeCreated["#attributes"].SystemTime, .Event.EventData.LogonType, .Event.EventData.IpAddress, .Event.EventData.TargetLogonId] | @tsv' security.json
```

How to read that command:

- `select(...)` keeps only events that match every condition inside it.
- `.Event.System.EventID==4624` keeps successful logons.
- `.Event.EventData.TargetUserName|ascii_downcase` takes the account that logged on and lowercases it, so `CyberJunkie` equals `cyberjunkie`.
- The part after the second `|` chooses what to print: the time, the logon type, the source IP address and the logon ID, separated by tabs (`@tsv`).
- `-r` prints plain text without quotation marks.

**How to read the result.** The first column is the time in UTC. The logon type says how the user logged in: 2 is typing at the keyboard, 3 is over the network, 10 is Remote Desktop. A user can have several 4624 events, because Windows logs one for each session it starts, so decide which one the question means (the first logon is the usual choice) and copy its time exactly as shown.

<details>
<summary><b>Answer to Question 1</b></summary>

`2023-03-27T14:37:09Z` in the log, entered on Hack The Box as `27/03/2023 14:37:09`

</details>

**A useful side discovery.** When we first searched the whole file for the username, the first hit was not a login. It was Event 1102, "The audit log was cleared", written by CyberJunkie at 2023-03-27 14:36:45 UTC. The lesson: always filter on the Event ID as well as the username, because the name appears in many kinds of events. That clearing event came 24 seconds before the logon time above (14:36:45 versus 14:37:09). We come back to it in Part 5.

## Part 2: The firewall rule (Questions 2 and 3)

**Questions:** What is the name of the firewall rule the user added, and what is its direction?

**Background.** A firewall decides which network traffic is allowed in or out of a computer. An attacker who controls a machine often adds a rule to let their own traffic through. Metasploit is a well-known attack toolkit, and "C2" stands for command and control, the channel an attacker uses to send orders to a compromised machine. A rule named "Metasploit C2 Bypass" is therefore a strong hint of what the user was doing.

**Where to look.** Firewall changes are in their own log, `Windows Firewall-Firewall.evtx`. **Event ID 2004** means "a rule has been added" (2005 is modified, 2006 is deleted).

**Narrowing down.** Our first query returned many events, and the first one we opened was a normal Windows rule (an app called CBSPreview) created on 2023-02-01, long before anything suspicious. Windows and its apps create lots of rules on their own, so we needed two filters: a time window (after the suspicious activity began on 2023-03-27) and the field that says who made the rule.

```bash
jq -r 'select(.Event.System.EventID==2004)
  | [.Event.System.TimeCreated["#attributes"].SystemTime, .Event.EventData.RuleName, .Event.EventData.Direction, .Event.EventData.ApplicationPath, .Event.EventData.ModifyingUser, .Event.EventData.ModifyingApplication] | @tsv' firewall.json | sort
```

One row stood out:

```
2023-03-27T14:44:43Z   Metasploit C2 Bypass   2   (no app path)   S-1-5-21-3393683511-3463148672-371912004-1001   C:\Windows\System32\mmc.exe
```

Why we trust it is the right one:

- **The name** is custom and obviously not a Windows default.
- **The account** is the SID ending in `-1001`, the same SID that appears for CyberJunkie in the log-cleared event from Part 1. A SID is the unique ID Windows uses for an account.
- **The program** is `mmc.exe`, the Microsoft Management Console. The user created the rule by hand in the Windows Defender Firewall settings window, not with a script.
- **The date** is the same day as the other suspicious activity.

<details>
<summary><b>Answer to Question 2</b></summary>

`Metasploit C2 Bypass`

</details>

**Direction (Question 3).** The log stores the direction as a number, and it does not say what the number means. In Windows firewall events, `1` means inbound and `2` means outbound ([reference](https://psmths.gitbook.io/windows-forensics/artifacts-by-type/event-log-artifacts/microsoft-windows-windows-firewall-with-advanced-security/evtx-2004-firewall)). We also checked it against our own data: a Microsoft Edge rule named `Microsoft Edge (mDNS-In)` has Direction `1`, and the "-In" in its name means inbound. The Metasploit rule has Direction `2`.

<details>
<summary><b>Answer to Question 3</b></summary>

Outbound. This matches the purpose of the rule: letting the infected computer send traffic out to the attacker.

</details>

<!-- SCREENSHOT: the Metasploit C2 Bypass row in the jq output
![jq output showing the Metasploit C2 Bypass firewall rule](images/02-firewall-rule.png)
-->

## Part 3: The audit policy and the scheduled task (Questions 4 to 7)

### Question 4: Which audit policy subcategory was changed?

**Background.** Windows only records some kinds of events if an audit policy switches that recording on. Changing the policy changes what gets logged. Attackers may switch auditing off to hide, or on to capture something. Windows always records the change itself as **Event ID 4719** ("System audit policy was changed") in the Security log.

```bash
jq -r 'select(.Event.System.EventID==4719)
  | [.Event.System.TimeCreated["#attributes"].SystemTime, .Event.EventData.SubjectUserName, .Event.EventData.SubcategoryGuid, .Event.EventData.AuditPolicyChanges] | @tsv' security.json
```

There was exactly one 4719 event, at `2023-03-27T14:50:03Z`. The log did not give us a name, only a GUID (a long unique ID): `0CCE9227-69AE-11D9-BED3-505054503030`. We looked that GUID up in a published list of audit subcategory GUIDs ([Tenable's audit reference](https://www.tenable.com/audits/items/MSCT_Windows_Server_2004_DC_v1.0.0.audit:0660bec3f6a5698b4d1c797a9dffe954)), and it belongs to **Other Object Access Events**, under the Object Access category. On a Windows machine you can also see the GUIDs with `auditpol /list /subcategory:* /v`.

Note that the account on this event is the computer account `DESKTOP-887GK2L$`, not CyberJunkie. That usually means the change ran with system-level rights, so don't expect to see the user's name here.

<details>
<summary><b>Answer to Question 4</b></summary>

Other Object Access Events

</details>

### Questions 5 to 7: The scheduled task

**Background.** A scheduled task makes Windows run a program automatically, for example every morning. Attackers use tasks for persistence, meaning their code keeps running even after the computer restarts. Creating a task is logged as **Event ID 4698**, and the object access auditing we just saw is what makes that logging possible. The task was created about 78 seconds after the policy change, so the order fits.

```bash
jq -r 'select(.Event.System.EventID==4698)
  | [.Event.System.TimeCreated["#attributes"].SystemTime, .Event.EventData.SubjectUserName, .Event.EventData.TaskName] | @tsv' security.json
```

Result: `2023-03-27T14:51:21Z`, user `CyberJunkie`, task `\HTB-AUTOMATION`. The leading backslash is just Task Scheduler's top-level folder, so the name is **`HTB-AUTOMATION`**.

**What the task runs.** The event also stores the task's full settings as XML in a field called `TaskContent`. Printing it shows everything the task does:

```bash
jq -r 'select(.Event.System.EventID==4698) | .Event.EventData.TaskContent' security.json
```

The important lines are the `<Actions>` section:

```xml
<Exec>
  <Command>C:\Users\CyberJunkie\Desktop\Automation-HTB.ps1</Command>
  <Arguments>-A cyberjunkie@hackthebox.eu</Arguments>
</Exec>
```

- **Question 6, the full path of the file the task runs:** `C:\Users\CyberJunkie\Desktop\Automation-HTB.ps1`. It is a PowerShell script (`.ps1`) on the user's Desktop.
- **Question 7, the arguments of the command:** `-A cyberjunkie@hackthebox.eu`.

The XML also tells us the task runs daily from 2023-03-27 09:00 (local time on the computer), under the CyberJunkie account, with normal (non-administrator) rights, and the description is just "practice". Times inside the XML are local time, while the event time at the top is UTC, which is why the XML says 07:51 and the event says 14:51.

<details>
<summary><b>Answers to Questions 5, 6 and 7</b></summary>

- **Q5, task name:** `HTB-AUTOMATION`
- **Q6, file path:** `C:\Users\CyberJunkie\Desktop\Automation-HTB.ps1`
- **Q7, arguments:** `-A cyberjunkie@hackthebox.eu`

</details>

A tip for pasting answers: the JSON output shows backslashes twice (`C:\\Users\\...`). That is just how JSON writes a backslash. The real Windows path has single backslashes.

## Part 4: What the antivirus found and did (Questions 8 to 10)

**Questions:** Which tool did the antivirus flag as malware, what is its full path, and what action did the antivirus take?

**Where to look.** Windows Defender is the built-in antivirus, and it keeps its own log: `Windows Defender-Operational.evtx`. Two Event IDs matter:

- **1116**: malware was detected.
- **1117**: Defender took action to protect the system.

Defender's field names contain spaces (for example `Threat Name`), so in `jq` they need square brackets and quotes:

```bash
jq -r 'select(.Event.System.EventID==1116 or .Event.System.EventID==1117)
  | [.Event.System.TimeCreated["#attributes"].SystemTime, .Event.System.EventID, .Event.EventData["Threat Name"], .Event.EventData.Path] | @tsv' defender.json
```

**Two separate detections.** The output was long, with two distinct stories in it:

| When (UTC) | Threat name | What it is |
| --- | --- | --- |
| 2023-03-10, about 02:11 to 02:46 | `Trojan:Win64/Meterpreter.B` | `msf.dll`, a Metasploit payload. Defender caught it again and again, hundreds of rows, because the same file kept being recreated. |
| 2023-03-27 14:42 | `HackTool:PowerShell/SharpHound.B` and `HackTool:MSIL/SharpHound!MSR` | SharpHound, found inside a downloaded zip file |

The question asks for a "tool". SharpHound is labeled `HackTool` by Defender. It is the data collector for BloodHound, a program attackers (and penetration testers) use to map a Windows network and find paths to powerful accounts. The Meterpreter file is a malicious payload rather than a tool, so we chose SharpHound.

**Question 9, the full path.** The `Path` field on the 1116 event reads `containerfile:_C:\Users\CyberJunkie\Downloads\SharpHound-v1.1.0.zip; file:...->SharpHound.exe`. The pieces:

- `containerfile` means the threat was found inside an archive.
- `file:...->SharpHound.exe` names the files inside the zip that Defender flagged (`SharpHound.exe` and `SharpHound.ps1`).
- The location on disk is the zip file in the user's Downloads folder. The `webfile` part shows it came from a GitHub release download.

The `file:_` prefix and doubled backslashes are formatting, not part of the path.

**Question 10, the action.** Detection (1116) and action (1117) are separate events, and the action is stored in its own field. Our first output only proved that Defender acted, so we asked for the action directly:

```bash
jq -r 'select(.Event.System.EventID==1117 and (.Event.EventData["Threat Name"]|test("SharpHound")))
  | [.Event.System.TimeCreated["#attributes"].SystemTime, .Event.EventData["Threat Name"], .Event.EventData["Action Name"], .Event.EventData["Action ID"]] | @tsv' defender.json
```

Both rows came back with `Action Name` = `Quarantine` (Action ID 2), at `2023-03-27T14:42:48Z`, 14 seconds after the detection. Quarantine means Defender locked the file away so it can no longer run.

<details>
<summary><b>Answers to Questions 8, 9 and 10</b></summary>

- **Q8, tool flagged as malware:** SharpHound
- **Q9, full path:** `C:\Users\CyberJunkie\Downloads\SharpHound-v1.1.0.zip`
- **Q10, action taken:** Quarantine

</details>

<!-- SCREENSHOT: Defender output showing the SharpHound detection and Quarantine action
![Defender events for SharpHound](images/03-defender-sharphound.png)
-->

## Part 5: The PowerShell command and the cleared logs (Questions 11 and 12)

### Question 11: What PowerShell command did the user run?

**Background.** PowerShell is Windows' built-in command-line and scripting tool, popular with both administrators and attackers. When script block logging is on, Windows records the text of everything run in PowerShell as **Event ID 4104**, in `Powershell-Operational.evtx`. The command text is in a field called `ScriptBlockText`.

PowerShell and Windows also run many internal scripts, so an unfiltered query is noisy. We kept only events from the day of the attack, after 14:30 UTC:

```bash
jq -r 'select(.Event.System.EventID==4104 and .Event.System.TimeCreated["#attributes"].SystemTime >= "2023-03-27T14:30")
  | [.Event.System.TimeCreated["#attributes"].SystemTime, .Event.EventData.ScriptBlockText] | @tsv' powershell.json
```

The whole result was three lines:

```
2023-03-27T14:58:07Z   prompt
2023-03-27T14:58:33Z   Get-FileHash -Algorithm md5 .\Desktop\Automation-HTB.ps1
2023-03-27T14:58:33Z   prompt
```

The two `prompt` lines are PowerShell redrawing its input prompt, not something the user typed. The real command is `Get-FileHash`, which calculates a file's "fingerprint" (a hash). The user hashed the same `Automation-HTB.ps1` script that the scheduled task runs, probably to record or verify it.

<details>
<summary><b>Answer to Question 11</b></summary>

`Get-FileHash -Algorithm md5 .\Desktop\Automation-HTB.ps1`

</details>

### Question 12: Which event log file was cleared?

**Background.** Clearing a log erases its history, and the act of clearing is itself recorded. There are two events to know:

- **1102** in the Security log: the Security log itself was cleared.
- **104** in the System log: some other log was cleared, and a `Channel` field names which one.

We searched for both. We already had a 1102 from Part 1. Here is the query for 104 events:

```bash
jq -r 'select(.Event.System.EventID==104)
  | [.Event.System.TimeCreated["#attributes"].SystemTime, .Event.UserData.LogFileCleared.SubjectUserName, .Event.UserData.LogFileCleared.Channel] | @tsv' system.json
```

Every clearing event was done by CyberJunkie:

| When (UTC) | Log cleared | Event |
| --- | --- | --- |
| 2023-03-24, 14 times between 19:42 and 21:31 | `Microsoft-Windows-Sysmon/Operational` | 104 |
| 2023-03-27 14:36:45 | Security | 1102 |
| 2023-03-27 15:01:56 | `Microsoft-Windows-Windows Firewall With Advanced Security/Firewall` | 104 |

So more than one log was cleared, and the question has to be answered by judgment. We chose the firewall log. It is the only one cleared during the attack window after the activity we traced: it happened at 15:01:56, a few minutes after the `Get-FileHash` command at 14:58, and it is the log that holds the `Metasploit C2 Bypass` rule. That fits an attacker covering their tracks. The Sysmon clears are three days earlier and repeat 14 times, which looks more like testing. Hack The Box accepts the firewall log as the answer.

<details>
<summary><b>Answer to Question 12</b></summary>

`Microsoft-Windows-Windows Firewall With Advanced Security/Firewall`

</details>

## Timeline of events (UTC)

Placing every finding on one clock shows the shape of the activity: tools arrive, the firewall and audit settings change, a scheduled task is planted, and logs are cleared. The order below is oldest first.

| When (UTC) | What happened | Log and Event ID |
| --- | --- | --- |
| 2023-03-10, 02:11 to 02:46 | Defender detects the Meterpreter payload `msf.dll` over and over (first in a VMware drag-and-drop folder, then in `Temp`) | Defender, 1116 and 1117 |
| 2023-03-24, 19:42 to 21:31 | The Sysmon log is cleared 14 times | System, 104 |
| 2023-03-27, 14:36:45 | The Security log is cleared | Security, 1102 |
| 2023-03-27, 14:37:09 | CyberJunkie logs in (Question 1) | Security, 4624 |
| 2023-03-27, 14:42:34 | Defender detects SharpHound in the downloaded `SharpHound-v1.1.0.zip` | Defender, 1116 |
| 2023-03-27, 14:42:48 | Defender quarantines it | Defender, 1117 |
| 2023-03-27, 14:44:43 | Outbound firewall rule `Metasploit C2 Bypass` is created through `mmc.exe` | Firewall, 2004 |
| 2023-03-27, 14:50:03 | Audit policy for Other Object Access Events is changed | Security, 4719 |
| 2023-03-27, 14:51:21 | Scheduled task `HTB-AUTOMATION` is created to run `Automation-HTB.ps1` daily | Security, 4698 |
| 2023-03-27, 14:58:33 | `Get-FileHash -Algorithm md5 .\Desktop\Automation-HTB.ps1` is run in PowerShell | PowerShell, 4104 |
| 2023-03-27, 15:01:56 | The Windows Firewall log is cleared | System, 104 |

Read as a story, this is a plausible pattern for an intruder or a practice session (the task description literally says "practice"): bring in an attack tool, open a path out to a command server, set up persistence, then erase the evidence. The logs alone can't prove intent, only what happened and in what order.

## Glossary

| Term | Plain-language meaning |
| --- | --- |
| Audit policy | Settings that decide which kinds of events Windows records |
| C2 (command and control) | The channel an attacker uses to send orders to a compromised computer |
| Cargo | Rust's package installer, used here to install `evtx_dump` |
| DFIR | Digital forensics and incident response: investigating what happened after a security incident |
| Event ID | A number that identifies the kind of event, such as 4624 for a successful logon |
| `.evtx` | The binary file format Windows uses to store an event log |
| GUID | A long unique identifier, such as `0CCE9227-69AE-11D9-BED3-505054503030` |
| Hash | A fingerprint of a file's contents; the same file always gives the same hash |
| JSON / JSON Lines | A text format that labels every piece of data; JSON Lines puts one event on each line |
| `jq` | A command-line tool for searching and filtering JSON |
| Logon type | How a login happened: at the keyboard (2), over the network (3), or by Remote Desktop (10) |
| Metasploit / Meterpreter | A widely used attack toolkit and its remote-control payload |
| Persistence | A way for an attacker's code to keep running after a restart, such as a scheduled task |
| Quarantine | Antivirus action that isolates a file so it can't run |
| Scheduled task | A job Windows runs automatically at set times |
| Script block logging | A PowerShell feature that records the text of the commands that were run |
| SharpHound | A tool that collects Active Directory data for BloodHound; flagged as a HackTool by Defender |
| SID | Security identifier, the unique ID Windows gives each account |
| Sherlock | A Hack The Box challenge that teaches investigation and forensics |
| UTC | The world-standard time zone, written with a trailing `Z` in the logs |

## Reference: command cheatsheet

A compact version of the commands used above, for reuse on other Sherlocks.

### 1. Setup: get evtx_dump running

```bash
sudo apt install cargo                         # if missing
cargo install evtx --version 0.8.4 --locked    # an older release; the newest fails on rustc 1.85.1 (try older if 0.8.4 won't build)
export PATH="$HOME/.cargo/bin:$PATH"
which evtx_dump
```

`libevtx-utils` is a different package: it provides `evtxexport` (text or XML only, no JSON). Prebuilt alternatives with no compiling: the `evtx_dump` binary from the omerbenamram/evtx GitHub releases, or Chainsaw.

### 2. Convert .evtx to JSON

```bash
cd ~/Event-Logs
evtx_dump -o jsonl -f security.json Security.evtx
evtx_dump -o jsonl -f firewall.json "Windows Firewall-Firewall.evtx"
evtx_dump -o jsonl -f defender.json "Windows Defender-Operational.evtx"
evtx_dump -o jsonl -f powershell.json "Powershell-Operational.evtx"
evtx_dump -o jsonl -f system.json System.evtx

wc -l security.json                            # should be thousands of lines, not 0
head -n1 security.json | jq .                  # see the field names for this version
grep -m1 -i cyberjunkie security.json | jq .   # first event mentioning a user
```

### 3. jq cheatsheet: when did a user log in successfully?

```bash
jq -r 'select(.Event.System.EventID==4624 and (.Event.EventData.TargetUserName|ascii_downcase)=="cyberjunkie")
  | [.Event.System.TimeCreated["#attributes"].SystemTime, .Event.EventData.LogonType, .Event.EventData.IpAddress, .Event.EventData.TargetLogonId] | @tsv' security.json
```

`SystemTime` ends in `Z`, so it is already UTC. Add `| sort | head -1` to get the earliest match.

| Piece | What it does |
| --- | --- |
| `-r` | Raw output: no quotes around strings |
| `select(...)` | Keep only events where the condition is true |
| `.Event.System.EventID==4624` | Successful logon (EventID is a plain number in this JSON) |
| `.Event.EventData.TargetUserName` | The account that logged on |
| `ascii_downcase` | Lowercases the name so `CyberJunkie` matches `cyberjunkie` |
| `.Event.System.TimeCreated["#attributes"].SystemTime` | Event timestamp in UTC |
| `[a, b, c] \| @tsv` | Print the chosen fields as one tab-separated row |

Reusable variants:

```bash
# Every 4624 logon, with user, type and source IP
jq -r 'select(.Event.System.EventID==4624) | [.Event.System.TimeCreated["#attributes"].SystemTime, .Event.EventData.TargetUserName, .Event.EventData.LogonType, .Event.EventData.IpAddress] | @tsv' security.json

# Failed logons (4625) for a user
jq -r 'select(.Event.System.EventID==4625 and (.Event.EventData.TargetUserName|ascii_downcase)=="cyberjunkie") | .Event.System.TimeCreated["#attributes"].SystemTime' security.json

# Security log cleared (1102): the user is under UserData, not EventData
jq -r 'select(.Event.System.EventID==1102) | [.Event.System.TimeCreated["#attributes"].SystemTime, .Event.UserData.LogFileCleared.SubjectUserName] | @tsv' security.json

# New firewall rule (2004) in the firewall log
jq 'select(.Event.System.EventID==2004) | .Event.EventData' firewall.json

# How many events of each ID, most common first
jq -r '.Event.System.EventID' security.json | sort | uniq -c | sort -rn | head

# Pretty-print one matching event in full
grep -m1 -i cyberjunkie security.json | jq .
```

### 4. Windows Event ID quick reference

| Event ID | Log | Meaning |
| --- | --- | --- |
| 4624 | Security | Successful logon |
| 4625 | Security | Failed logon |
| 4634 / 4647 | Security | Logoff / user-initiated logoff |
| 4648 | Security | Logon with explicit credentials (runas) |
| 4672 | Security | Special privileges assigned to a new logon (admin-level) |
| 4768 / 4769 | Security (DC) | Kerberos TGT / service ticket requested |
| 4698 | Security | Scheduled task created |
| 4719 | Security | System audit policy changed |
| 1102 | Security | Audit log cleared |
| 104 | System | A log other than Security was cleared |
| 2004 | Windows Firewall-Firewall | Firewall rule added |
| 2005 / 2006 | Windows Firewall-Firewall | Firewall rule modified / deleted |
| 4946 / 4947 / 4948 | Security | Firewall rule added / modified / deleted (needs MPSSVC audit policy) |
| 1116 / 1117 | Windows Defender-Operational | Malware detected / action taken |
| 4104 | Powershell-Operational | Script block logged (command text) |

Logon types in a 4624 event:

| LogonType | Meaning |
| --- | --- |
| 2 | Interactive (local) |
| 3 | Network (SMB, etc.) |
| 4 | Batch |
| 5 | Service |
| 7 | Unlock |
| 10 | RemoteInteractive (RDP) |
| 11 | Cached interactive |

Useful 4624 fields: `TargetUserName`, `LogonType`, `IpAddress`, `WorkstationName`, `TargetLogonId` (ties a session to later events). In a 2004 firewall event look at `RuleName`, `Direction`, `Action`, `ApplicationPath`, and `ModifyingUser` / `ModifyingApplication`.

### 5. Troubleshooting

When jq prints nothing even though the text log shows the event:

- **Case mismatch.** The username was stored as `CyberJunkie`, so `=="cyberjunkie"` matched nothing. Lowercase the field with `ascii_downcase`, or test with `test("cyberjunkie";"i")`.
- **Wrong field location.** Most events keep the user in `.Event.EventData`, but some, like 1102, use `.Event.UserData`. Look at one event first: `grep -m1 -i <name> security.json | jq .`
- **EventID shape.** In this build it is a plain number. In other versions it can be an object, so test it loosely: `(.Event.System.EventID|tostring|test("4624"))`, or check `.Event.System.EventID."#text"`.
- **Empty or truncated file.** Check `wc -l security.json`. Re-run `evtx_dump` and look for errors.
- **Wrong log file.** Firewall rule events are in the Firewall evtx, not Security.
- **A first grep hit is not always the event you want.** Searching for the username found a 1102 log-cleared event before any 4624. Always filter on the Event ID too.

When `cargo install evtx` fails with "rustc ... is not supported": use an older release (`--version`), update Rust with rustup, or download a prebuilt binary.
