## 📂 Section: Organization & Methodology
> [!INFO] **Professional Directory Structure**
> HTB Academy emphasizes that a clean workspace equals a clean mind. This structure separates your targets by OS and your progress by the phase of the attack (Enumeration -> Exploitation -> Post-Exploitation).

### Organizing your data
This is the goal for every new engagement:

```text
.
└── Penetration-Testing
    ├── Pre-Engagement
    ├── Linux
    │   ├── Information-Gathering
    │   ├── Vulnerability-Assessment
    │   ├── Exploitation
    │   ├── Post-Exploitation
    │   └── Lateral-Movement
    ├── Windows
    │   ├── Information-Gathering
    │   ├── Vulnerability-Assessment
    │   ├── Exploitation
    │   ├── Post-Exploitation
    │   └── Lateral-Movement
    ├── Reporting
    └── Results

Here's another version to structure

```text
Cry0l1t3@htb[/htb]$ tree ..

└── Penetration-Testing
    │
    ├── Pre-Engagement
    │       └── ...
    ├── Network-Pentesting
    │       ├── Linux
    │       │   ├── Information-Gathering
    │       │   │   └── ...
    │       │   ├── Vulnerability-Assessment
    │       │   │   └── ...
    │       │   └── ...
    │       │       └── ...
    │       ├── Windows
    │       │   ├── Information-Gathering
    │       │   │   └── ...
    │       │   └── ...
    │       └── ...
    ├── WebApp-Pentesting
    │       └── ...
    ├── Social-Engineering
    │       └── ...
    ├── .......
    │       └── ...
    ├── Reporting
    │   └── ...
    └── Results
        └── ...
