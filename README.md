<div align="center">

# 🛡️ Week 01 — Cybersecurity Lab Setup

**VirtualBox + Kali Linux | NAT Network | `10.0.0.0/24`**

A documented, repeatable base environment for cybersecurity and penetration-testing practice.

<br>

<img src="https://img.shields.io/badge/VirtualBox-7.2.16-183A61?style=for-the-badge&logo=virtualbox&logoColor=white">
<img src="https://img.shields.io/badge/Kali%20Linux-2026.2-557C94?style=for-the-badge&logo=kalilinux&logoColor=white">
<img src="https://img.shields.io/badge/Network-10.0.0.0%2F24-0B7285?style=for-the-badge">
<img src="https://img.shields.io/badge/Status-Complete-2E8B57?style=for-the-badge">

</div>

---

## 📌 What I Built

Week 1 was about building the foundation for the rest of my cybersecurity lab.

The target was simple on paper: get Kali Linux running in VirtualBox, put it on the required private network, give it a fixed address, verify Internet access, configure host/guest integration, and create a recovery point.

In practice, I treated it like a small infrastructure build.

I checked what the machine was actually doing, verified each layer with commands, and worked through the problems instead of reinstalling the VM whenever something went wrong.

The final result is a working Kali attacker environment on `10.0.0.2/24` with verified gateway, Internet, and DNS connectivity, matching Guest Additions, shared storage, and a snapshot-based recovery strategy.

---

## 🎯 Week 01 Requirements

The lab was built against these requirements:

| Requirement | Target | Result |
|---|---|:---:|
| Hypervisor | VirtualBox | ✅ |
| VirtualBox version | `7.2.16` | ✅ |
| Attacker VM | Kali Linux `2026.2` | ✅ |
| Network type | NAT Network | ✅ |
| Network | `10.0.0.0/24` | ✅ |
| Kali address | `10.0.0.2/24` | ✅ |
| Gateway | `10.0.0.1` | ✅ |
| Internet access | Required | ✅ |
| DNS resolution | Working | ✅ |
| Shared Clipboard | Bidirectional | ✅ |
| Drag & Drop | Enabled | ✅ |
| Host shared folder | `downloads` | ✅ |
| VM snapshot | Required baseline | ✅ |

> The supplied Week 1 brief specifies VirtualBox, Kali Linux, a `10.0.0.0/24` NAT Network, Kali at `10.0.0.2/24`, clipboard/Drag & Drop, a host `downloads` share, Internet access, and a VM snapshot. fileciteturn3file0L3-L12

---

## 🗺️ Lab Topology

```text
                         ┌──────────────────┐
                         │     INTERNET     │
                         └────────┬─────────┘
                                  │
                           VirtualBox NAT
                                  │
                         ┌────────▼────────┐
                         │   NatNetwork    │
                         │   10.0.0.0/24   │
                         │   Gateway .1    │
                         └────────┬────────┘
                                  │
                         ┌────────▼────────┐
                         │   Kali Linux    │
                         │      eth0       │
                         │   10.0.0.2/24   │
                         │    Attacker     │
                         └─────────────────┘

             Future Windows / Linux / vulnerable VMs
                    can be added to 10.0.0.0/24
```

### Why this design?

I kept the design deliberately straightforward.

- **NAT Network** gives me a private VM network that can later contain multiple machines.
- **`10.0.0.0/24`** keeps the address plan easy to understand and document.
- **`10.0.0.2`** gives the Kali attacker a predictable address for later scripts, notes, scans, and lab exercises.
- **Snapshots** give me a known-good point to return to before risky work.

The goal is a lab that I can break on purpose later without making the host machine part of the experiment.

---

# ⚙️ Environment

| Component | Final configuration |
|---|---|
| Host | Windows |
| Hypervisor | VirtualBox `7.2.16` |
| Guest | Kali Linux `2026.2` |
| Desktop | XFCE |
| Session | X11 |
| Kali CPU | 2 vCPU |
| Kali RAM | 2048 MB |
| Adapter | NAT Network |
| NAT Network | `NatNetwork` |
| Subnet | `10.0.0.0/24` |
| Kali IP | `10.0.0.2/24` |
| Gateway | `10.0.0.1` |
| Guest Additions | `7.2.16r174877` |
| Shared folder | `downloads` |
| Promiscuous mode | Allow All |

---

# 🪜 Setup Process

## 1. Create the NAT Network

In VirtualBox:

```text
Tools
→ Network
→ NAT Networks
→ Create
```

Configuration:

```text
Name:          NatNetwork
IPv4 Prefix:   10.0.0.0/24
DHCP:          Enabled
IPv6:          Disabled
```

The important part here is that the VirtualBox NAT Network and the lab subnet match.

I verified the network in the VirtualBox GUI before changing anything inside Kali.

---

## 2. Register the Kali VM

The Kali VirtualBox image was extracted and the existing `.vbox` machine definition was registered in VirtualBox.

I used the existing VM definition instead of manually rebuilding the machine because the supplied Kali image already contained the guest configuration and virtual disk.

The VM was kept at:

```text
2 CPUs
2048 MB RAM
```

and Adapter 1 was attached to:

```text
NAT Network
└── NatNetwork
```

---

## 3. Configure Host/Guest Integration

### Clipboard

```text
Shared Clipboard: Bidirectional
```

### Drag & Drop

```text
Drag & Drop: Bidirectional
```

### Shared folder

The host `downloads` directory was configured as a VirtualBox shared folder with auto-mount enabled.

This gives the lab a predictable place for moving scripts, wordlists, captures, reports, and other files later.

---

## 4. First Kali Boot — What I Actually Found

After the first boot, I did not assume the IP address was already correct.

I checked:

```bash
ip a
```

The VM initially received:

```text
10.0.0.3/24
```

That was actually useful information.

DHCP was clearly working, the VM was on the correct subnet, and the problem was simply that the address did not match the required fixed address of `10.0.0.2/24`.

I then checked the route:

```bash
ip route
```

and found:

```text
default via 10.0.0.1 dev eth0
10.0.0.0/24 dev eth0
```

At that point I knew the VirtualBox network itself was working, so I focused on the guest-side IP configuration instead of changing the NAT Network again.

---

# 🌐 Network Configuration

## 5. Identify the Active NetworkManager Profile

```bash
nmcli connection show
```

The active Ethernet connection was:

```text
Wired connection 1
```

That meant I could modify the existing profile rather than creating another connection.

---

## 6. Configure Kali with a Static IP

```bash
sudo nmcli connection modify "Wired connection 1" \
  ipv4.method manual \
  ipv4.addresses 10.0.0.2/24 \
  ipv4.gateway 10.0.0.1 \
  ipv4.dns "10.0.0.1 8.8.8.8"
```

### Why I used this

The assignment required Kali to use a predictable fixed address.

NetworkManager was already managing `eth0`, so modifying the existing profile was the cleanest approach and avoids relying on a temporary `ip addr` change.

---

## 7. Static IP Reconnection Issue

The first static configuration did not reconnect cleanly.

Instead of changing several settings at once, I checked the state and applied the troubleshooting adjustment from the lab material:

```bash
sudo nmcli connection modify "Wired connection 1" \
  ipv4.dad-timeout 0
```

Then I restarted the connection:

```bash
sudo nmcli connection down "Wired connection 1"
sudo nmcli connection up "Wired connection 1"
```

I verified the result:

```bash
ip -br addr
```

Final result:

```text
eth0   UP   10.0.0.2/24
```

### Why I took that approach

The VirtualBox network had already proven itself.

DHCP had already provided a working address, and the route was correct. So rebuilding the VM network would have been solving the wrong problem.

I kept the known-good lower layer intact and worked only on the NetworkManager profile.

---

# ✅ Connectivity Verification

I verified connectivity in layers instead of treating "Internet works" as one test.

## 8. Verify the Gateway

```bash
ping -c 4 10.0.0.1
```

Result:

```text
4 packets transmitted
4 received
0% packet loss
```

This proves Kali can reach the VirtualBox NAT gateway.

---

## 9. Verify Internet Connectivity

```bash
ping -c 4 8.8.8.8
```

Result:

```text
4 packets transmitted
4 received
0% packet loss
```

This proves the VM can reach the Internet by IP address without depending on DNS.

---

## 10. Verify DNS

```bash
getent hosts google.com
```

A valid address for `google.com` was returned.

This is a separate check because:

```text
Internet connectivity ≠ DNS resolution
```

The completed verification chain was:

```text
eth0
  ↓
10.0.0.2/24
  ↓
10.0.0.1        ✅ gateway
  ↓
8.8.8.8         ✅ Internet
  ↓
google.com      ✅ DNS
```

---

# 🧩 Guest Additions

## 11. Detect the Version Mismatch

The first Guest Additions version reported by Kali was:

```bash
VBoxClient --version
```

Result:

```text
7.2.8_Debianr173730
```

The Windows host was running:

```text
VirtualBox 7.2.16
```

So there was a version mismatch.

Rather than assuming Guest Additions were completely broken, I checked what was already running.

---

## 12. Check VBoxClient Processes

```bash
ps aux | grep -i VBoxClient
```

Important processes included:

```text
VBoxClient --clipboard
VBoxClient --seamless
VBoxClient --draganddrop
VBoxClient --vmsvga-session
```

This told me that Guest Additions were already installed and active.

That changed the troubleshooting direction: the problem was not simply "Guest Additions are missing."

---

## 13. Attach and Mount the Matching ISO

I obtained the matching:

```text
VBoxGuestAdditions_7.2.16.iso
```

After attaching it to the VM, I checked the optical device:

```bash
lsblk
```

The CD/DVD device appeared as:

```text
sr0
```

Kali did not automatically create the expected `/media/kali/` mount, so I created a predictable mount point:

```bash
sudo mkdir -p /mnt/vboxga
```

Then:

```bash
sudo mount /dev/sr0 /mnt/vboxga
```

The ISO was mounted read-only, which is expected.

I verified its contents:

```bash
ls -la /mnt/vboxga
```

and confirmed:

```text
VBoxLinuxAdditions.run
```

was present.

---

# 🧱 Kernel and Header Investigation

## 14. Check the Running Kernel

```bash
uname -r
```

The original kernel was:

```text
6.19.14+kali-amd64
```

I checked for the matching kernel build directory:

```bash
ls -ld /lib/modules/$(uname -r)/build
```

and:

```bash
ls -ld /usr/src/linux-headers-$(uname -r)
```

Both showed that the matching header/build environment was missing.

That meant the Guest Additions kernel modules could not be cleanly built yet.

---

## 15. Attempt to Keep the Existing Kernel

I first tried to stay on the existing kernel and obtain its matching header packages.

```bash
cd ~/Downloads
```

```bash
wget https://http.kali.org/pool/main/l/linux/linux-headers-6.19.14+kali-common_6.19.14-1+kali1_all.deb
```

```bash
wget https://http.kali.org/pool/main/l/linux/linux-headers-6.19.14+kali-amd64_6.19.14-1+kali1_amd64.deb
```

Both downloads completed.

I then tried:

```bash
sudo apt install ./linux-headers-6.19.14+kali-common_6.19.14-1+kali1_all.deb \
                 ./linux-headers-6.19.14+kali-amd64_6.19.14-1+kali1_amd64.deb
```

APT reported unsatisfied dependencies, including:

```text
gcc-15-for-host
linux-kbuild-6.19.14+kali
```

I checked them separately:

```bash
apt-cache policy gcc-15-for-host
```

The compiler package was available.

Then:

```bash
apt-cache policy linux-kbuild-6.19.14+kali
```

and:

```bash
apt-cache search linux-kbuild-6.19.14
```

The matching `linux-kbuild` package was not available from the configured repository.

---

## 16. Why I Did Not Force the Package Installation

This was one of the more useful decisions during the setup.

I could have tried to force the package installation, but that would have created an incomplete kernel build environment.

Instead, I followed a simpler rule:

> Use a complete, internally consistent kernel/header/build stack rather than forcing one missing dependency into place.

That made the next step more predictable.

---

# 🐧 Kernel Compatibility Resolution

## 17. Install the Available Matching Kernel/Header Pair

I checked what Kali currently offered:

```bash
apt-cache policy linux-image-amd64 linux-headers-amd64
```

A complete newer kernel/header pair was available.

I installed it with:

```bash
sudo apt install -y linux-image-amd64 linux-headers-amd64
```

This installed the matching:

```text
linux-image-7.1.5+kali-amd64
linux-headers-7.1.5+kali-amd64
linux-kbuild-7.1.5+kali
```

The original `6.19.14` kernel was left installed.

That gave me a clean fallback option if I needed it.

---

## 18. Verify the New Kernel

After reboot:

```bash
uname -r
```

Result:

```text
7.1.5+kali-amd64
```

Then:

```bash
ls -ld /lib/modules/$(uname -r)/build
```

The build link pointed to:

```text
/usr/src/linux-headers-7.1.5+kali-amd64
```

Now the running kernel and headers matched.

---

# 🔧 Guest Additions Installation

## 19. Install Guest Additions 7.2.16

With the ISO mounted and the matching kernel/header environment available:

```bash
cd /mnt/vboxga
sudo sh ./VBoxLinuxAdditions.run
```

The installation completed.

I verified the version:

```bash
VBoxClient --version
```

Final result:

```text
7.2.16r174877
```

So the host and guest versions now match:

```text
VirtualBox Host:    7.2.16
Guest Additions:    7.2.16
```

---

# 🖥️ Session Verification

I checked the actual Kali session:

```bash
echo $XDG_SESSION_TYPE
```

Result:

```text
x11
```

And:

```bash
echo $XDG_CURRENT_DESKTOP
```

Result:

```text
XFCE
```

So the final desktop/session combination is:

```text
XFCE + X11
```

I kept this in the documentation because graphical/session details are useful when troubleshooting VM integration later.

---

# 📂 Shared Folder

The host `downloads` directory was added as a VirtualBox shared folder and configured for auto-mounting.

The intended workflow is:

```text
Windows Host
     │
     │ shared folder
     ▼
downloads
     │
     ▼
Kali Linux
```

I plan to use this for:

- scripts
- wordlists
- scan results
- PCAP files
- reports
- lab evidence
- controlled test files

---

# 📸 Snapshots and Recovery

Snapshots were taken during the setup so that troubleshooting remained reversible.

The first baseline was created after reaching a known-good network state.

A later baseline was created after the Guest Additions/kernel work.

The principle is simple:

```text
Known-good configuration
          ↓
       Snapshot
          ↓
     Experiment
          ↓
   Something breaks?
          ↓
       Roll back
```

That is important for a cybersecurity lab because future exercises may intentionally change services, packages, or system configuration.

---

# 🧠 Why I Worked This Way

I wanted the setup process to teach me something beyond just clicking through VirtualBox menus.

### Verify before changing

Commands such as:

```bash
ip a
ip route
nmcli connection show
uname -r
```

told me what the machine was actually doing.

### Troubleshoot the correct layer

For example:

```text
VirtualBox network works
        ↓
DHCP works
        ↓
Gateway route exists
        ↓
IP is wrong
        ↓
Fix NetworkManager
```

There was no reason to rebuild the VirtualBox network.

The same idea applied to Guest Additions:

```text
VBoxClient exists
        ↓
Guest Additions exists
        ↓
Version mismatch found
        ↓
Kernel build environment missing
        ↓
Fix kernel/header layer
        ↓
Install matching Guest Additions
```

### Prefer reversible changes

Snapshots and keeping the old kernel installed gave me room to experiment without turning the lab into a fragile one-way setup.

---

# 🧰 Command Reference

These are the commands I want to keep as a reusable Week 1 reference.

## Network

```bash
ip a
```

```bash
ip -br addr
```

```bash
ip route
```

```bash
nmcli connection show
```

```bash
nmcli device status
```

## Connectivity

```bash
ping -c 4 10.0.0.1
```

```bash
ping -c 4 8.8.8.8
```

```bash
getent hosts google.com
```

## Static IP

```bash
sudo nmcli connection modify "Wired connection 1" \
  ipv4.method manual \
  ipv4.addresses 10.0.0.2/24 \
  ipv4.gateway 10.0.0.1 \
  ipv4.dns "10.0.0.1 8.8.8.8"
```

```bash
sudo nmcli connection modify "Wired connection 1" \
  ipv4.dad-timeout 0
```

```bash
sudo nmcli connection down "Wired connection 1"
```

```bash
sudo nmcli connection up "Wired connection 1"
```

## Guest Additions

```bash
VBoxClient --version
```

```bash
ps aux | grep -i VBoxClient
```

```bash
lsblk
```

```bash
sudo mkdir -p /mnt/vboxga
```

```bash
sudo mount /dev/sr0 /mnt/vboxga
```

```bash
ls -la /mnt/vboxga
```

```bash
cd /mnt/vboxga
sudo sh ./VBoxLinuxAdditions.run
```

## Kernel / Headers

```bash
uname -r
```

```bash
ls -ld /lib/modules/$(uname -r)/build
```

```bash
ls -ld /usr/src/linux-headers-$(uname -r)
```

```bash
apt-cache policy linux-image-amd64 linux-headers-amd64
```

```bash
apt-cache policy linux-kbuild-6.19.14+kali
```

```bash
apt-cache search linux-kbuild-6.19.14
```

```bash
sudo apt install -y linux-image-amd64 linux-headers-amd64
```

## Desktop/session

```bash
echo $XDG_SESSION_TYPE
```

```bash
echo $XDG_CURRENT_DESKTOP
```

---

# 🧪 Verification Matrix

| Test | Command / Check | Result |
|---|---|:---:|
| Kali interface | `ip -br addr` | ✅ |
| Kali IP | `10.0.0.2/24` | ✅ |
| Routing | `ip route` | ✅ |
| Gateway | `ping -c 4 10.0.0.1` | ✅ |
| Internet | `ping -c 4 8.8.8.8` | ✅ |
| DNS | `getent hosts google.com` | ✅ |
| NetworkManager | `nmcli connection show` | ✅ |
| Guest Additions | `VBoxClient --version` | ✅ |
| Guest Additions version | `7.2.16r174877` | ✅ |
| Kernel | `7.1.5+kali-amd64` | ✅ |
| Kernel build directory | `/lib/modules/.../build` | ✅ |
| Desktop | XFCE | ✅ |
| Session | X11 | ✅ |
| Clipboard | Bidirectional | ✅ |
| Drag & Drop | Bidirectional | ✅ |
| Shared folder | `downloads` | ✅ |
| Snapshot | Baseline created | ✅ |

---

# 📦 Final State

```text
VirtualBox 7.2.16
│
└── Kali Linux 2026.2
    │
    ├── XFCE / X11
    │
    ├── eth0
    │   └── 10.0.0.2/24
    │
    ├── Gateway
    │   └── 10.0.0.1
    │
    ├── Network
    │   └── 10.0.0.0/24
    │
    ├── Internet
    │   └── Verified
    │
    ├── DNS
    │   └── Verified
    │
    ├── Guest Additions
    │   └── 7.2.16
    │
    ├── Clipboard
    │   └── Bidirectional
    │
    ├── Drag & Drop
    │   └── Bidirectional
    │
    ├── Shared Folder
    │   └── downloads
    │
    └── Snapshot
        └── Week 1 baseline
```

---

# 🔐 Lab Safety

This environment is intended for systems that I own or have explicit permission to test.

Future scanning, exploitation, packet analysis, and vulnerability-testing work will be performed against machines intentionally added to this isolated lab.

The purpose of the NAT Network is to keep those experiments inside a controlled virtual environment rather than treating the physical network as the test environment.

---

# 🚀 Next Steps

With the base Kali attacker VM established, the next stage can build the target side of the lab.

Planned additions include:

```text
Kali Linux
10.0.0.2
   │
   ├── Windows target
   ├── Linux target
   ├── Vulnerable web application
   ├── Metasploitable / CTF target
   └── Additional test machines
```

The same verification pattern will be used for future machines:

```text
Configure
   ↓
Inspect
   ↓
Test
   ↓
Document
   ↓
Snapshot
```

---

# 🏁 Conclusion

Week 1 established the foundation for the cybersecurity lab.

The important part was not just getting Kali to boot. I wanted to understand what each layer was doing and be able to explain why the configuration looks the way it does.

The environment now has a predictable attacker address, a dedicated private NAT Network, working Internet and DNS, matching Guest Additions, host/guest integration, shared storage, and recovery snapshots.

The troubleshooting was part of the learning process. I started with a DHCP-assigned `10.0.0.3`, moved to the required static `10.0.0.2`, dealt with a NetworkManager reconnection issue, investigated a Guest Additions version mismatch, found that the original kernel did not have the required build environment, avoided forcing an incomplete dependency chain, and moved to a complete matching kernel/header stack before installing Guest Additions.

That gave me a much better understanding of the lab than simply following a checklist.

**Week 01 — LAB FOUNDATION COMPLETE ✅**

