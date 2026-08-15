# HOME CYBERSECURITY LAB
**Infrastructure, Virtualization, Network Segmentation, and Security Monitoring**

**Technical Documentation**

| **Document Owner**            | Daniel Miller    |
|-------------------------------|------------------|
| **Version**                   | 1.55             |
| **Date**                      | August 14, 2026  |
| **Primary Security Platform** | Wazuh SIEM / XDR |

*Purpose: Maintain an accurate, printable record of the homelab architecture, security controls, systems, and operating procedures.*

## 1. Document Purpose and Scope
This document describes the architecture and operating model of a home cybersecurity lab built to support hands-on learning in systems administration, virtualization, network defense, vulnerability management, security monitoring, and incident response. The lab is intended to resemble a small enterprise environment while preserving strong isolation between trusted infrastructure and higher-risk testing systems.

- Document the physical and logical network architecture.
- Record the role of the UniFi gateway, switch, physical servers, and Proxmox virtualization layer.
- Define segmentation and access-control expectations for management, server, workstation, security-lab, and isolated-lab networks.
- Document Wazuh as the primary SIEM and XDR platform.
- Provide repeatable operating procedures for vulnerability management, incident response, logging, backups, and change management.
- Create a baseline that can be updated as the lab expands.

## 2. Current Environment Summary
| **Component**              | **Current Role**                                                                                                                |
|----------------------------|---------------------------------------------------------------------------------------------------------------------------------|
| Google Fiber               | Upstream Internet service.                                                                                                      |
| Ubiquiti Cloud Gateway Max | Primary router, firewall, inter-VLAN gateway, and UniFi management platform.                                                    |
| UniFi Standard 16 PoE      | Primary managed switch for physical systems and VLAN connectivity.                                                              |
| Four physical servers      | Compute platform for virtualization and security services.                                                                      |
| Proxmox VE                 | Primary hypervisor and virtualization platform.                                                                                 |
| Wazuh                      | Primary SIEM and XDR platform for endpoint telemetry, log analysis, alerting, vulnerability visibility, and response workflows. |
| Windows and Linux systems  | Endpoints and servers used for administration, security monitoring, and testing.                                                |
| Kali Linux workstation     | Authorized security testing, traffic analysis, scanning, and defensive validation.                                              |

## 3. Physical Network Architecture
The lab receives Internet access from Google Fiber. The UniFi Cloud Gateway Max is the edge routing and security device. The UniFi Standard 16 PoE switch provides wired connectivity to the physical servers and administrative systems. No wireless access point is required for the core lab design.

```text
Internet / Google Fiber
|
v
+--------------------------+
| UniFi Cloud Gateway Max |
| Router / Firewall |
+------------+-------------+
|
v
+--------------------------+
| UniFi Standard 16 PoE |
| Managed Switch |
+---+-----------+----------+
| |
| +---- Administrative / Lab Workstations
|
+---------------- Physical Proxmox Servers (4)
```

### 3.1 UniFi Cloud Gateway Max
The Cloud Gateway Max forms the principal trust boundary between the lab and the Internet. It is also responsible for routing traffic between VLANs and enforcing network-access policy.

- Internet routing and NAT
- Stateful firewall enforcement
- VLAN routing and gateway services
- DHCP and DNS forwarding as configured
- UniFi device management
- Traffic visibility and security monitoring
- VPN capability for approved remote administration
- IDS/IPS features where enabled

### 3.2 UniFi Standard 16 PoE
The managed switch provides wired connectivity and VLAN assignment for servers and workstations. Switch ports should be assigned according to the trust level and function of the connected device.

- Access ports for single-VLAN devices
- 802.1Q trunking to Proxmox hosts where multiple VLANs are required
- Centralized port configuration through UniFi
- Future PoE support for approved devices

## 4. Logical Network and VLAN Design
Network segmentation is used to reduce lateral movement, separate administrative traffic from user traffic, and isolate intentionally vulnerable or untrusted systems. The following table is a recommended logical design and can be adjusted to match the final addressing plan.

| **VLAN / Zone** | **Vlan ID** | **Subnet**      | **Purpose**                                                                    |
|-----------------|-------------|-----------------|--------------------------------------------------------------------------------|
| Management      | 10          | 192.168.10.0/24 | Network devices, Proxmox management, iLO, and other administrative interfaces. |
| Servers         | 20          | 192.168.20.0/24 | Physical and virtual servers providing infrastructure and security services.   |
| Computers       | 30          | 192.168.30.0/24 | Trusted administrative and end-user workstations.                              |
| Security Lab    | 40          | 192.168.40.0/24 | Kali, scanning, packet analysis, IDS testing, and defensive tooling.           |
| Isolated Lab    | 50          | 192.168.50.0/24 | Intentionally vulnerable hosts, CTF targets, and higher-risk experiments.      |

### 4.1 Management VLAN
The Management VLAN contains infrastructure interfaces such as the UniFi gateway, switch management, Proxmox management interfaces, HPE iLO, and other privileged administration endpoints. Access should be restricted to specifically authorized administrative systems.

### 4.2 Server VLAN
The Server VLAN hosts physical and virtual infrastructure services. Typical workloads include Active Directory, DNS, Windows and Linux servers, Wazuh components, monitoring services, and future backup or automation systems.

### 4.3 Computer VLAN
The Computer VLAN contains trusted administrative workstations. These systems may access approved services on the Server VLAN but should not receive unrestricted access to the Management VLAN.

### 4.4 Security Lab VLAN
The Security Lab VLAN is used for security tooling, traffic analysis, scanning, defensive validation, and authorized testing. It may contain Kali Linux, packet-capture systems, scanning tools, or other security-focused workloads.

### 4.5 Isolated Lab VLAN
The Isolated Lab VLAN is intended for intentionally vulnerable or untrusted systems. It should be denied access to trusted networks by default and granted Internet access only when an exercise requires it.

## 5. Firewall and Access-Control Model
The preferred network-security model is default-deny for inter-VLAN traffic. Access is granted only when a documented administrative or application requirement exists. Stateful return traffic for approved sessions should be permitted, while invalid traffic should be dropped.

| **Source**   | **Destination**          | **Default Action** | **Rationale**                                                         |
|--------------|--------------------------|--------------------|-----------------------------------------------------------------------|
| Management   | Infrastructure / Servers | Allow as required  | Supports administration from trusted systems.                         |
| Computers    | Internet                 | Allow              | Normal user and administrative Internet access.                       |
| Computers    | Servers                  | Limited allow      | Only approved services such as DNS, HTTPS, AD, RDP, or SSH as needed. |
| Computers    | Management               | Deny               | Protects privileged infrastructure interfaces.                        |
| Security Lab | Management               | Deny               | Prevents testing systems from reaching management planes.             |
| Security Lab | Servers                  | Limited allow      | Only when a test or monitoring workflow requires it.                  |
| Isolated Lab | Trusted RFC1918 networks | Deny               | Contains vulnerable or compromised systems.                           |
| Isolated Lab | Internet                 | Deny by default    | Enable temporarily only when required by an exercise.                 |

### 5.1 Rule Order
1. Allow established and related sessions.
2. Drop invalid session states.
3. Allow authorized management systems to infrastructure interfaces.
4. Permit required workstation-to-server services.
5. Block Computers from the Management VLAN unless explicitly approved.
6. Block Security Lab systems from the Management VLAN.
7. Block Isolated Lab systems from trusted private networks.
8. Apply a final deny rule for otherwise unauthorized inter-VLAN traffic.

### 5.2 Common Service Ports
| **Service** | **Port / Protocol** | **Typical Use**                      |
|-------------|---------------------|--------------------------------------|
| DNS         | 53 TCP/UDP          | Name resolution                      |
| Kerberos    | 88 TCP/UDP          | Active Directory authentication      |
| LDAP        | 389 TCP/UDP         | Directory access                     |
| LDAPS       | 636 TCP             | Encrypted directory access           |
| SMB         | 445 TCP             | Windows file and domain services     |
| HTTPS       | 443 TCP             | Web administration and applications  |
| SSH         | 22 TCP              | Linux administration                 |
| RDP         | 3389 TCP            | Windows administration when required |

## 6. UniFi Port Profiles
Port profiles should clearly express the intended trust zone. The exact profile names may be changed, but a consistent naming standard is recommended.

| **Profile**     | **Native VLAN** | **Tagged VLANs**                             | **Use**                                            |
|-----------------|-----------------|----------------------------------------------|----------------------------------------------------|
| MGMT-ACCESS     | Management      | None                                         | Dedicated management interfaces.                   |
| SERVER-ACCESS   | Server          | None                                         | Servers that require only the Server VLAN.         |
| COMPUTER-ACCESS | Computer        | None                                         | Trusted workstations.                              |
| LAB-ACCESS      | Security Lab    | None                                         | Security testing workstations.                     |
| PROXMOX-TRUNK   | Management      | Server, Computer, Security Lab, Isolated Lab | Proxmox host uplink carrying multiple guest VLANs. |

## 7. Physical Server and Virtualization Platform
The lab contains four physical servers that provide compute resources for virtualization. Proxmox VE is the primary hypervisor platform and can support a multi-node cluster as the environment matures.

- Virtual machines and Linux containers
- Centralized web-based management
- Virtual networking and VLAN tagging
- Snapshots and backups
- Storage management
- Clustering and future high-availability experimentation

### 7.1 Recommended Host Naming
| **Host** | **Role**       |
|----------|----------------|
| PVE01    | Proxmox node 1 |
| PVE02    | Proxmox node 2 |
| PVE03    | Proxmox node 3 |
| PVE04    | Proxmox node 4 |

### 7.2 Proxmox Network Model
A VLAN-aware Linux bridge, such as vmbr0, can carry multiple logical networks from the UniFi switch to each Proxmox host. Individual virtual machines are assigned VLAN tags according to their function and trust level.

| **Example VLAN Tag** | **Assigned Workloads**                           |
|----------------------|--------------------------------------------------|
| 10                   | Management interfaces where appropriate          |
| 20                   | Server workloads                                 |
| 30                   | Windows or user-workstation test VMs             |
| 40                   | Security tools and monitoring systems            |
| 50                   | Isolated or intentionally vulnerable lab targets |

### 7.3 Storage Considerations
One server configuration under consideration uses eight 1 TB drives. RAID 6 can provide dual-drive fault tolerance and approximately 6 TB of raw usable capacity before filesystem and RAID overhead. RAID improves availability after disk failure but is not a backup strategy.

## 8. Core Virtual Workloads
The environment can host a mixture of Windows and Linux systems. The following names provide a consistent convention and can be adjusted as actual systems are deployed.

| **System** | **Example Role**                                         |
|------------|----------------------------------------------------------|
| DC01       | Primary Windows Server domain controller and DNS.        |
| DC02       | Secondary domain controller / DNS redundancy.            |
| WIN11-01   | Domain-joined Windows 11 client.                         |
| WIN11-02   | Secondary Windows client for policy or security testing. |
| LNX-SRV01  | General-purpose Linux server.                            |
| KALI01     | Authorized security-testing workstation.                 |
| WAZUH01    | Wazuh SIEM/XDR platform or primary Wazuh server role.    |
| SCAN01     | Vulnerability-scanning workload.                         |

## 9. Wazuh SIEM and XDR
Wazuh is the primary Security Information and Event Management (SIEM) and Extended Detection and Response (XDR) platform for the homelab. It serves as the central blue-team monitoring and investigation platform for supported Windows, Linux, and other integrated systems.

- Centralized security-event and endpoint-telemetry collection.
- Windows and Linux security monitoring.
- Authentication and account-activity visibility.
- File integrity monitoring.
- Vulnerability visibility and security-configuration assessment.
- Alert generation, correlation, dashboards, and investigation workflows.
- Threat hunting and incident triage.
- Active-response workflows where safely configured.

### 9.1 Wazuh Components
| **Component**   | **Function**                                                                                           |
|-----------------|--------------------------------------------------------------------------------------------------------|
| Wazuh Server    | Receives and analyzes telemetry, evaluates rules, and generates security alerts.                       |
| Wazuh Indexer   | Stores and indexes alert and security-event data for search and analysis.                              |
| Wazuh Dashboard | Provides the web interface for monitoring, investigations, dashboards, and administration.             |
| Wazuh Agents    | Collect endpoint telemetry from monitored Windows and Linux systems and provide host-level visibility. |

### 9.2 Homelab Wazuh Architecture
```text
Windows Endpoints ----\
Linux Servers ---------+--> Wazuh Server --> Wazuh Indexer --> Wazuh Dashboard
Security Systems ------/ |
+--> Alerts / Investigation / Response
```

### 9.3 Example Detection and Investigation Use Cases
- Repeated failed authentication attempts and brute-force behavior.
- Unexpected privileged-account activity.
- Suspicious PowerShell or command-line execution.
- Unauthorized file changes or persistence activity.
- Endpoint vulnerability findings and configuration weaknesses.
- Suspicious administrative changes.
- Potential malware indicators or lateral-movement evidence.
- Correlating host telemetry with firewall and network evidence during an investigation.

## 10. Identity and Windows Enterprise Services
The lab can include a Windows Active Directory environment to practice enterprise identity administration and defensive monitoring. Typical exercises include user and group provisioning, Organizational Units, Group Policy, Kerberos, DNS, service accounts, password policy, account lockout, least privilege, and security auditing.

Microsoft cloud services may be integrated for hybrid identity or endpoint-management exercises, including Microsoft Entra ID, Microsoft 365, Exchange Online, Azure, Intune, Conditional Access, and multi-factor authentication where available.

## 11. Security Tooling and Network Analysis
| **Tool / Platform** | **Lab Use**                                                                      |
|---------------------|----------------------------------------------------------------------------------|
| Wazuh               | SIEM/XDR, endpoint telemetry, alerting, investigation, vulnerability visibility. |
| Wireshark           | Interactive packet capture and protocol analysis.                                |
| tcpdump             | Command-line packet capture on Linux.                                            |
| Nmap                | Authorized host discovery, port scanning, and service enumeration.               |
| RITA                | Network-traffic analysis and beaconing investigation where applicable.           |
| Zeek                | Potential future network-security monitoring and metadata generation.            |
| Suricata / Snort    | Potential IDS testing and signature-based detection.                             |
| Python              | Custom defensive utilities such as IDS prototypes and port-scanning tools.       |
| PowerShell          | Windows administration, automation, event collection, and security analysis.     |

## 12. Vulnerability Management Workflow
The lab is designed to support a repeatable vulnerability-management lifecycle rather than one-time scanning. Findings should be validated and prioritized according to technical severity and actual exposure.

1. Identify and inventory the asset.
2. Perform an authorized vulnerability scan.
3. Validate significant findings.
4. Research the associated CVE or weakness.
5. Assess exploitability, exposure, asset criticality, and potential impact.
6. Prioritize remediation based on risk rather than CVSS alone.
7. Patch, reconfigure, remove, or otherwise mitigate the weakness.
8. Rescan the asset.
9. Document the remediation and verification result.

## 13. Security Testing and Blue Team Validation
Security testing should remain limited to systems owned by or explicitly authorized for the homelab. The preferred workflow uses controlled offensive activity to validate defensive visibility and response.

| **Phase**         | **Activity**                                                                                            |
|-------------------|---------------------------------------------------------------------------------------------------------|
| Deploy            | Create or restore a test target and place it on the correct VLAN.                                       |
| Baseline          | Record IP address, operating system, services, open ports, and expected behavior.                       |
| Generate Activity | Perform an authorized scan, authentication test, or simulated attack.                                   |
| Detect            | Review Wazuh alerts, Windows/Linux logs, firewall logs, and packet captures.                            |
| Investigate       | Identify source, destination, time, technique, affected systems, and indicators.                        |
| Respond           | Isolate hosts, block sources, disable accounts, patch weaknesses, or remove persistence as appropriate. |
| Document          | Record the timeline, findings, response actions, and lessons learned.                                   |

## 14. Incident Response Procedure
### 14.1 Preparation
Maintain logging, monitoring, backups, administrator access, known-good baselines, and tested response tools.

### 14.2 Detection and Analysis
Review alerts and supporting telemetry to determine whether malicious or unauthorized activity occurred.

### 14.3 Containment
Use VLAN isolation, firewall blocks, account disabling, or VM network disconnection to limit impact.

### 14.4 Eradication
Remove malicious files, unauthorized accounts, persistence mechanisms, and vulnerable configurations.

### 14.5 Recovery
Restore secure services, verify normal operation, and monitor for recurrence.

### 14.6 Lessons Learned
Document the incident timeline, indicators, root cause, control gaps, actions performed, and recommended improvements.

## 15. Backup and Recovery
Critical configuration and workloads should be backed up outside the primary storage array. Back ups system in the works.

- Proxmox configuration and critical virtual machines
- UniFi configuration backups
- Wazuh configuration and detection content
- Active Directory or other infrastructure configuration
- Scripts, automation, and custom security tools
- Network diagrams and documentation
- Critical vulnerability and incident-response records

## 16. Administrative Security Standards
- Use separate administrative accounts for privileged tasks when practical.
- Use strong, unique passwords and MFA wherever supported.
- Restrict management interfaces to the Management VLAN or other explicitly trusted sources.
- Use SSH keys for Linux administration where appropriate.
- Apply least privilege and remove unnecessary accounts or permissions.
- Monitor privileged logons and administrative changes.
- Do not expose management interfaces directly to the Internet without a justified and secured design.

## 17. Patch and Change Management
Routine updates should include Proxmox hosts, Windows servers and clients, Linux systems, UniFi devices, Wazuh components, and security applications. Significant configuration changes should be recorded so that the environment can be troubleshot and restored consistently.

| **Change Record Field** | **Description**                                 |
|-------------------------|-------------------------------------------------|
| Date                    | Date the change was made.                       |
| System                  | Device, service, VLAN, or workload affected.    |
| Change                  | What was modified.                              |
| Reason                  | Why the modification was required.              |
| Result                  | Successful, partial, or failed.                 |
| Rollback                | How to restore the prior state if necessary.    |
| Notes                   | Additional validation or follow-up information. |

## 18. Naming and IP Addressing Standards
### 18.1 Naming
| **Category**           | **Examples**                   |
|------------------------|--------------------------------|
| Proxmox Hosts          | PVE01, PVE02, PVE03, PVE04     |
| Domain Controllers     | DC01, DC02                     |
| Windows Clients        | WIN11-01, WIN11-02             |
| Linux Servers          | LNX-SRV01                      |
| Security Systems       | KALI01, WAZUH01, IDS01, SCAN01 |
| Network Infrastructure | UCG-MAX01, USW-16-01           |

## 19. Documentation and Recordkeeping
- Physical network diagram
- Logical network diagram
- VLAN and subnet table
- Device inventory
- IP address inventory
- Virtual-machine inventory
- Firewall-rule documentation
- Switch-port assignments
- Backup and recovery procedures
- Security-tool configurations
- Vulnerability reports
- Incident-response reports
- Change log

Passwords, private keys, API tokens, recovery codes, and other secrets should not be stored in documentation. Check Password Manager

## 20. Future Development Roadmap
- Complete and document the four-node Proxmox cluster.
- Deploy centralized Proxmox backup infrastructure.
- Expand Wazuh coverage to all supported Windows and Linux workloads.
- Develop custom Wazuh detection rules and response playbooks.
- Add centralized network telemetry using Zeek and/or Suricata.
- Build an Active Directory attack-and-defense range.
- Implement dedicated red-team and blue-team lab segments if needed.
- Automate host deployment and configuration with PowerShell, Python, or Ansible.
- Implement internal PKI and certificate-based services.
- Add UPS monitoring and graceful shutdown procedures.
- Map selected detection exercises to MITRE ATT&CK techniques.

## 21. Security Design Principles
| **Principle**    | **Application in the Homelab**                                                              |
|------------------|---------------------------------------------------------------------------------------------|
| Segmentation     | Separate systems by function and trust level using VLANs and firewall policy.               |
| Least Privilege  | Grant only the network and administrative access required for a task.                       |
| Default Deny     | Block inter-zone communication unless a documented requirement exists.                      |
| Defense in Depth | Combine network, identity, endpoint, logging, vulnerability, and administrative controls.   |
| Visibility       | Collect endpoint and infrastructure telemetry so activity can be detected and investigated. |
| Repeatability    | Use naming standards, documentation, change records, and repeatable test procedures.        |

## Appendix A. Build and Maintenance Checklist
- [ ] UniFi gateway configuration backed up.
- [ ] UniFi switch configuration backed up.
- [ ] VLANs and subnets documented.
- [ ] Firewall rules reviewed for least privilege.
- [ ] Management interfaces restricted to trusted sources.
- [ ] Proxmox host names and management addresses documented.
- [ ] Virtual machines inventoried.
- [ ] Wazuh agents deployed to supported endpoints.
- [ ] Wazuh alerting and dashboard access verified.
- [ ] Vulnerability scan schedule or manual procedure documented.
- [ ] Critical systems included in backup plan.
- [ ] Restore procedure tested for at least one workload.
- [ ] Security exercises kept within authorized lab boundaries.
- [ ] Change log updated after significant modifications.

## Appendix B. Document Change Log
| **Version** | **Date**     | **Change**                                                                                                 |
|-------------|--------------|------------------------------------------------------------------------------------------------------------|
| 1.0         | July 5, 2026 | Initial consolidated homelab documentation. Wazuh identified as the primary SIEM and XDR platform for lab. |
