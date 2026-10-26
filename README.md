# 🔐 Cybersecurity Lab Environment Setup

**Building a private virtual lab for penetration testing and ethical hacking practice using VirtualBox and Kali Linux**

---

## 📌 Project Overview

This project documents how I built a **virtual cybersecurity and penetration-testing lab** using VirtualBox and Kali Linux.

The lab gives me a controlled environment where I can safely and repeatedly practise network scanning, reconnaissance, vulnerability assessment and other security-testing activities.

It runs on a **private virtual NAT Network** (`10.0.0.0/24`) with outbound-only internet access, so more machines (targets, a Splunk server, and so on) can be added later and tested against.

---

## 🎯 Objectives

- Install and configure VirtualBox.
- Import Kali Linux as a virtual machine.
- Create a private **NAT Network** for the lab.
- Assign a consistent static IP address to the Kali VM.
- Verify network connectivity and DNS resolution.
- Take a clean VM snapshot for recovery.
- Document the full process, including the problems I hit.
- Prepare the environment for future projects.

---

## 🛡️ Purpose of the Lab

The lab can be used for:

- Network reconnaissance and port scanning
- Vulnerability assessment
- Packet analysis
- Web security testing
- Exploitation practice
- Security-tool experimentation

⚠️ **Important:** Only use this lab against systems you own or have explicit permission to test.

---

## 🏗️ Lab Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Host Machine (Windows 10/11)             │
│                                                             │
│   ┌─────────────────────────────────────────────────────┐   │
│   │              VirtualBox Hypervisor                  │   │
│   │                                                     │   │
│   │   ┌──────────────┐         ┌──────────────────┐    │   │
│   │   │  Kali Linux  │◄───────►│  Future Target   │    │   │
│   │   │  10.0.0.10   │   NAT   │  VMs (10.0.0.x)  │    │   │
│   │   │              │ Network │                  │    │   │
│   │   └──────────────┘         └──────────────────┘    │   │
│   │              │                                      │   │
│   └──────────────┼──────────────────────────────────────┘   │
│                  │ NAT (outbound only)                      │
│                  ▼                                          │
│            Internet Access                                  │
└─────────────────────────────────────────────────────────────┘
```

---

## ⚙️ Lab Configuration

| Component          | Configuration                         |
|--------------------|---------------------------------------|
| Host OS            | Windows 10 / 11                       |
| Host RAM           | 8 GB+                                 |
| Hypervisor         | VirtualBox 7.x                        |
| Security OS        | Kali Linux (version: `<add version>`) |
| Kali RAM / CPU     | 2048 MB / `<add cores>` cores         |
| Virtual Network    | NAT Network (`NatNetwork`)            |
| Network Address    | 10.0.0.0/24                           |
| Kali IP Address    | 10.0.0.10/24 (static)                 |
| Default Gateway    | 10.0.0.1                              |
| DNS Server         | 8.8.8.8                               |
| Static IP Range    | 10.0.0.10 – 10.0.0.99 (by convention) |

> **Note:** In a VirtualBox NAT Network, `10.0.0.1` is the gateway and the built-in DHCP server also occupies a low address. Keeping my static IPs at `.10` and above avoids clashing with these.

---

## 🪜 Lab Setup Procedure

### Step 1. Install 7-Zip

7-Zip is used to extract the Kali VM package, which is distributed as a `.7z` archive.

**Tool:** [7-Zip](https://7-zip.org/download.html)

### Step 2. Install VirtualBox

**Tool:** [VirtualBox](https://www.virtualbox.org/wiki/Downloads)

### Step 3. Create the NAT Network

In VirtualBox: **File → Tools → Network Manager → NAT Networks → Create**.

```
Network Name: NatNetwork
IPv4 Prefix:  10.0.0.0/24
DHCP:         Enabled
IPv6:         Disabled
```

I chose a **NAT Network** because VMs attached to it can talk to each other and still have outbound internet access. A plain NAT adapter isolates each VM from the others.

> 📸 `[Add screenshot: NAT Network properties]`

### Step 4. Import Kali Linux

I downloaded the pre-built Kali VM from the official site and imported it into VirtualBox.

```
Adapter 1
Attached to:  NAT Network
Name:         NatNetwork
Adapter Type: Intel PRO/1000 MT Desktop
RAM:          2048 MB
```

A shared folder was also set up to move files between the host and the VM (this needs VirtualBox Guest Additions, which are included in Kali's pre-built VM image).

### Step 5. Configure a Static IP on Kali

First find the real connection name, since it differs between systems:

```bash
nmcli connection show
```

Then set the static configuration:

```bash
sudo nmcli connection modify "Wired connection 1" \
  ipv4.method manual \
  ipv4.addresses 10.0.0.10/24 \
  ipv4.gateway 10.0.0.1 \
  ipv4.dns 8.8.8.8
sudo nmcli connection up "Wired connection 1"
```

> 📸 `[Add screenshot: ip a output]`

### Step 6. Create a Clean VM Snapshot

After the configuration was working, I took a snapshot named:

```
Clean Kali - Network Setup
```

This is my known-good baseline. If a later exercise breaks the VM, I can restore it.

> 📸 `[Add screenshot: snapshot list]`

---

## 🔎 Lab Verification

| Test                  | Command                     | Expected Result                |
|-----------------------|-----------------------------|--------------------------------|
| Check IP address      | `ip a`                      | `10.0.0.10/24` shown           |
| Check default route   | `ip route`                  | Default via `10.0.0.1`         |
| Test gateway          | `ping -c 4 10.0.0.1`        | Successful replies             |
| Test internet         | `ping -c 4 8.8.8.8`         | Successful replies             |
| Test DNS              | `nslookup networkwalks.com` | Domain resolves                |
| Verify Nmap           | `nmap --version`            | Version displayed              |
| Verify snapshot       | Restore snapshot, then `ip a` | Baseline config restored     |

> 📸 `[Add screenshot: ping and nslookup results]`

---

## 🐞 Problems Encountered & Solutions

### Problem 1. No internet after setting a static IP

After I set the IPv4 address manually, internet connectivity failed.

**What I checked:**

- The gateway and DNS were actually set (`ip route`, `resolvectl status` or `cat /etc/resolv.conf`).
- The connection was brought down and up again after editing.

**Workaround I used during the lab:**

```bash
sudo nmcli connection modify "Wired connection 1" ipv4.dad-timeout 0
```

This disables the duplicate-address-detection delay in NetworkManager. After restarting the connection, connectivity worked. Connection names differ between systems, so check yours with `nmcli connection show` first.

### Problem 2. VirtualBox VT-x / virtualization error

The VM failed to start because hardware virtualization was disabled in the BIOS/UEFI.

**Fix:**

1. Restart the computer and enter BIOS/UEFI settings.
2. Enable Intel VT-x (or AMD-V) / hardware virtualization.
3. Save and restart.
4. Start the Kali VM again.

---

## 💡 What I Learned

1. **NAT vs NAT Network:** plain NAT isolates each VM; a NAT Network lets VMs on the same network talk to each other while sharing outbound internet access. The host cannot reach the VMs directly unless port forwarding or a Host-Only adapter is added.
2. **VM networking:** how VirtualBox adapter modes change which machines can communicate.
3. **Static IP configuration:** setting and verifying the address, mask, gateway and DNS in Kali with `nmcli`.
4. **Snapshots:** take a clean snapshot *before* risky or experimental work.
5. **Documentation:** recording commands, configuration, screenshots and fixes is a core part of a professional security project.

---

## 🚀 Next Steps

- Add **Metasploitable2** as a vulnerable target on the same NAT Network.
- Add a **Splunk** VM and forward logs from the targets.
- Build the SOC home-lab project on top of this base (threat detection, log analysis, MITRE ATT&CK mapping).

---

## 🔐 Security & Ethical Use

This lab is for educational purposes only.

- Only test systems you own or have explicit written permission to test.
- Never use this environment to attack real-world systems without authorization.
- Follow all applicable laws and the ethical guidelines of the profession.

---

## 🔗 Tools & Resources

- **7-Zip:** https://7-zip.org/download.html
- **VirtualBox:** https://www.virtualbox.org/wiki/Downloads
- **Kali Linux:** https://www.kali.org/get-kali/

---

## 👤 Author

**Vipin P P**

Aspiring cybersecurity analyst (SOC / VAPT)

- LinkedIn: [linkedin.com/in/vipin-p-p-92b440393](https://www.linkedin.com/in/vipin-p-p-92b440393)
- GitHub: [github.com/croissant911](https://github.com/croissant911)

---

## 📌 Project Information

| Field          | Details                              |
|----------------|--------------------------------------|
| **Program**    | Cybersecurity at Networkwalks        |
| **Week**       | 01                                   |
| **Project**    | Cybersecurity & Pentesting Lab Setup |
| **Status**     | Completed                            |
