**1.Problem**
The organization hosts its central identity platform (Active Directory + DNS) in AWS on a Windows Server.

Linux workload servers are deployed on-premises (local infrastructure).

There is **no private connectivity** between:

 on-prem Linux servers
 cloud-based domain controllers

Exposing Active Directory services directly to the public internet is not acceptable from a security standpoint.

The Linux workloads must be able to:

 resolve domain DNS
 authenticate users
 Later, join the domain

over a private and encrypted channel

**2. Design decision**
We implemented a lightweight site tunnel using WireGuard between:

 The Linux workload server (linux-app-01)
 The cloud domain controller (DC-CLOUD-01)

Why WireGuard:
 free and open source
 small attack surface
 easy to deploy on both Linux and Windows
 suitable for small and medium environments
 no dependency on cloud VPN appliances

Design principle:
**On-prem systems must communicate with the domain controller only through the VPN interface.**

The cloud instance’s physical network interface is not exposed to on-prem systems.

**3. Architecture**
linux-app-01 (on-prem, UTM on Mac)
  wg0: 10.50.0.2
        |
        |  Encrypted WireGuard tunnel (UDP 51820)
        |
DC-CLOUD-01 (AWS)
  wg-hybrid: 10.50.0.1
  Windows Server 2022
  Active Directory + DNS

**4. Implementation steps**
Components
 linux-app-01
   Ubuntu Server 22.04
   workload server
 DC-CLOUD-01
   Windows Server 2022
   Active Directory Domain Services
   DNS

**4.2 Network design**
Three independent networks exist:
AWS VPC
172.31.0.0/16
**Used only inside AWS.**

Hybrid tunnel network
10.50.0.0/24
**Created for hybrid connectivity.**

Assignments:
 DC-CLOUD-01 → 10.50.0.1
 linux-app-01 → 10.50.0.2

Local Lab access network(Linux)
192.168.64.0/24
**Used only between the Mac host and the Linux VM.**

**4.3 WireGuard on DC-CLOUD-01 (Windows)**
 WireGuard for Windows is installed
 Tunnel created: wg-hybrid
 Interface configuration:

Address = 10.50.0.1/24
ListenPort = 51820

**4.4 WireGuard on linux-app-01**
WireGuard installed.

Tunnel configuration:
 Address = 10.50.0.2/24
 Endpoint = <public IP of DC-CLOUD-01>:51820
 AllowedIPs = 10.50.0.0/24

**4.5 Security controls**
 AWS security group:
    Inbound UDP 51820 from administrator's public IP
 Windows firewall:
    WireGuard listening allowed

**4.6 DNS integration**
Linux must use Active Directory DNS.

DNS is bound to the WireGuard interface:
 10.50.0.1 

The DNS server is intentionally the **VPN interface** of the domain controller, not its AWS private NIC.
Manual binding is applied using:
 resolvectl dns wg0 10.50.0.1
 resolvectl domain wg0 corp.lab

This ensures:
 All domain lookups use the VPN path
 Name resolution does not leak to local/home DNS

**4.7 Important design rule**
On-prem servers must never use:
 172.31.x.x
to reach the domain controller.
They must always use:
 10.50.0.1
This avoids Windows multi-NIC and firewall host-model problems and matches standard site-to-site VPN design.

**5. Validation**
All validation is performed from linux-app-01.

**5.1 Tunnel health**
wg show

Expected:
latest handshake: <recent>

**5.2 Connectivity to domain controller over tunnel**
ping 10.50.0.1

**5.3 DNS resolution path**
resolvectl status

Expected:
wg0 → DNS Servers: 10.50.0.1

**5.4 Domain name resolution**
nslookup corp.lab

Expected DNS server:
10.50.0.1

5.5 Directory services ports
LDAP:
 nc -zv 10.50.0.1 389

Kerberos:
 nc -zv 10.50.0.1 88

Both must succeed.

**6. Failure cases**
Failure	                                          Cause
No WireGuard handshake	                          wrong public IP, wrong peer key, blocked UDP 51820
Handshake works but no DNS	                  DNS still pointing to local NAT DNS
Tunnel stops after EC2 stop/start	          public IP of EC2 changed
Domain resolution fails	                          wg0 DNS not configured
Connectivity unstable	                          endpoint IP mismatch or firewall rule removed

**7. Rollback**
To remove the hybrid connectivity:

**On linux-app-01**
sudo wg-quick down wg0

Remove:
 /etc/wireguard/wg0.conf

Remove DNS bindings:
 resolvectl revert wg0

**On DC-CLOUD-01**
deactivate WireGuard tunnel
remove peer configuration
remove Windows firewall rule
remove AWS security group UDP rule

**8. Operational note**
The EC2 instance uses a dynamic public IP.
If the instance is stopped and started, the Linux WireGuard endpoint must be updated.
For production usage, an Elastic IP must be attached.
