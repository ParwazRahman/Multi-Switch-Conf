# ⚡ Cisco Zero-Touch Provisioning (ZTP) Architecture

A fully automated, native Plug-and-Play (PNP) provisioning server for Cisco IOS switches. No Python scripts, no console cables, and no manual intervention required.

## 🛑 The Problem
Provisioning factory-fresh switches is a traditional bottleneck in network administration. The standard operating procedure involves connecting a physical console cable, manually bypassing the initial configuration dialog, and copy-pasting standard security hardening commands. This is slow, unscalable, and prone to human error.

## 💡 The Solution
This project leverages Cisco's native **AutoInstall** feature to create a True Zero-Touch Provisioning environment. By combining DHCP Option 150, a local TFTP server, and an Embedded Event Manager (EEM) applet, factory-fresh switches can completely configure and harden themselves simply by being plugged into the network and powered on.

### 🧠 How It Works (The Flow)
1. **The Trigger:** A brand-new switch boots with an empty NVRAM (`startup-config`), automatically broadcasting for DHCP on VLAN 1.
2. **The Brains:** A provisional switch (acting as the DHCP server) assigns an IP and uses **Option 150** to point the new switch to a TFTP server.
3. **The Payload:** The new switch downloads `network-confg` from the TFTP server, immediately applying a secure baseline (disabling vulnerable services, configuring SSHv2, securing VTY lines).
4. **The Auto-Save:** An embedded EEM applet waits 60 seconds for the configuration to settle, executes a `write memory` to save the config to NVRAM permanently, and then gracefully deletes itself from the running configuration.

---

## 🛠️ The Configurations

### 1. The TFTP Payload (`network-confg`)
This file is hosted on the TFTP server (e.g., a desktop running Tftpd64). The switch will automatically request this exact filename.

```text
! --- Baseline Security Hardening ---
no service tcp-small-servers
no service udp-small-servers
no service finger
no service pad
service password-encryption
no cdp run
no ip http server
no ip http secure-server
no ip source-route
no ip domain-lookup
!
! --- Access Control & SSH ---
enable secret Net0ps@Kol123
username admin privilege 15 secret Net0ps@Kol123
ip domain-name netops.local
crypto key generate rsa modulus 2048
ip ssh version 2
ip ssh time-out 60
ip ssh authentication-retries 3
!
! --- Line Security ---
line con 0
 exec-timeout 5 0
 logging synchronous
 login local
line vty 0 15
 exec-timeout 5 0
 transport input ssh
 login local
!
! --- Interface Prep ---
interface range FastEthernet0/1 - 24
 shutdown
 no shutdown
!
banner motd ^C
  *** AUTHORISED ACCESS ONLY ***
  Unauthorised access is strictly prohibited.
^C
ntp server 192.168.1.1
logging buffered 16384
logging console critical
!
! --- ZTP Auto-Save Macro ---
event manager applet AUTO_SAVE_ZTP
 event timer countdown time 60
 action 1.0 cli command "enable"
 action 2.0 cli command "configure terminal"
 action 3.0 cli command "no event manager applet AUTO_SAVE_ZTP"
 action 4.0 cli command "end"
 action 5.0 cli command "write memory"
 action 6.0 syslog msg "ZTP Auto-Save complete. Configuration written to NVRAM."
!
end