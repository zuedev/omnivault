# Analysis of Linux.RedMenshenBPFDoor

The analyzed malware is a Linux backdoor that leverages raw socket packet capture, a BPF (Berkeley Packet Filter) to filter specific “magic packets,” and RC4 encryption to secure its communications. Its primary goal is to provide covert shell access while hiding its presence by mimicking legitimate system processes.

## Overview

At a high level, the malware performs the following actions:

- **Packet Sniffing with BPF Filtering:**  
  It opens a raw socket on the network interface and attaches a custom BPF filter. This filter inspects incoming Ethernet frames and looks for specially crafted packets that contain a “magic packet” structure.

- **Magic Packet Processing:**  
  The malware defines a `struct magic_packet` that carries a flag, an IP address, a port, and a password. When such a packet is detected, the malware forks a new process to authenticate the command based on the password.

- **RC4 Encryption:**  
  Both inbound and outbound communications (i.e., shell I/O) are obfuscated using an RC4 stream cipher. The attacker must supply the correct password (either `"justforfun"` or `"socket"`) for the malware to execute its payload functions.

- **Shell Spawning and Network Redirection:**  
  Depending on the provided password, the malware either establishes a bind shell directly or uses iptables commands to redirect network traffic and spawn a shell. This allows remote control over the compromised system.

- **Stealth and Persistence:**  
  The malware attempts to hide its presence by changing its process name to mimic common system daemons (e.g., `/sbin/udevd -d` or `/usr/lib/systemd/systemd-journald`). It also checks for an existing PID file (located at `/var/run/haldrund.pid`) to avoid multiple instances and uses self-copying techniques to reside in `/dev/shm`.

The complete code is available for review citeturn0file0, and the following sections break down its key components.

---

## Packet Capture and BPF Filtering

The malware sets up a raw socket bound to the Ethernet interface and attaches a BPF filter. The purpose of the filter is to isolate network packets that match a specific pattern, thereby reducing noise from normal traffic.

```c
if ((sock = socket(PF_PACKET, SOCK_RAW, htons(ETH_P_IP))) < 1)
    return;
if (setsockopt(sock, SOL_SOCKET, SO_ATTACH_FILTER, &filter, sizeof(filter)) == -1) {
    return;
}
```

The filter’s bytecode checks for a unique signature (for example, the constant `0x00007255` found within the filter code) that identifies the magic packet. Once a matching packet is received, the malware extracts the payload and verifies the embedded password.

---

## RC4 Encryption for Secure Communication

The malware implements its own version of the RC4 stream cipher. Two contexts are initialized—one for encrypting outgoing data and one for decrypting incoming data. This ensures that even if network traffic is captured, the shell communication remains obfuscated.

```c
void rc4_init(uchar *key, int len, rc4_ctx *ctx) {
    // Initialization of state array...
}

void rc4(uchar *data, int len, rc4_ctx *ctx) {
    // Encrypts or decrypts data in-place...
}
```

Both `cwrite` and `cread` functions wrap network I/O operations by applying RC4 on the transmitted data. This encryption layer is crucial for evading detection and complicating traffic analysis.

---

## Backdoor Activation and Shell Spawning

When a valid magic packet is detected, the malware forks a new process. Based on the password supplied in the packet (compared in the `logon` function), different actions are triggered:

- **Direct Shell Connection:**  
  If the password matches the first preset (`"justforfun"`), the malware attempts to connect to a provided IP and port. A successful connection results in launching a shell directly over the established socket.

- **iptables Redirection and Bind Shell:**  
  If the password matches the second preset (`"socket"`), the malware calls the `getshell` function. Here, it uses iptables commands to redirect incoming traffic to an ephemeral port, allowing the attacker to connect to a bind shell.

```c
int getshell(char *ip, int fromport) {
    // Establishes iptables rules for NAT redirection and spawns a shell
    snprintf(cmd, sizeof(cmd), cmdfmt, ip, fromport, toport);
    system(cmd);
    // Waits for connection and then executes the shell
}
```

This dual approach gives the attacker flexibility in how they connect to the compromised system.

---

## Process Hiding and Persistence

To avoid easy detection, the malware uses several techniques:

- **Process Name Spoofing:**  
  The `set_proc_name` function alters the process name in both `argv[0]` and via the `prctl` system call. It randomly selects a name from a list of common system processes (e.g., `/sbin/udevd -d`, `/usr/lib/systemd/systemd-journald`) to blend in with legitimate activity.

- **PID File Check:**  
  Before executing, the malware checks for the existence of a PID file (`/var/run/haldrund.pid`). If the file is present, the malware terminates immediately, preventing duplicate instances.

- **Self-Copying:**  
  The `to_open` function copies the binary to a temporary location (typically within `/dev/shm`) and adjusts its permissions. This can help the malware persist across reboots or evade simple file-based detections.

---

## Conclusion

Linux.RedMenshenBPFDoor is a sophisticated example of a Linux backdoor. Its use of raw sockets combined with BPF filtering for detecting magic packets, RC4 encryption for securing communications, and clever process-hiding techniques shows a level of operational security aimed at evading detection by traditional means. By employing iptables for network redirection and spawning a shell, the malware provides a flexible entry point for remote attackers.

Security professionals should be aware of such techniques—especially the use of raw socket-based packet inspection and process masquerading—as indicators of compromise in Linux environments.
