# Home Cybersecurity Lab

**Infrastructure, Virtualization, Network Segmentation, and Security Monitoring**

Technical documentation for a segmented enterprise-style homelab focused on virtualization, systems administration, network defense, SIEM/XDR, network security monitoring, vulnerability management, and incident response.

| **Document Owner**         | Daniel Miller                                                 |
|----------------------------|---------------------------------------------------------------|
| **Version**                | 1.2                                                           |
| **Date**                   | August 15, 2026                                               |
| Primary Security Platforms | Wazuh SIEM / XDR + Security Onion NSM + Elastic / ELK log analytics |

*Purpose: Maintain an accurate, version-controlled record of the homelab architecture, security controls, systems, and operating procedures.*

## Table of Contents
- [1. Document Purpose and Scope](#1-document-purpose-and-scope)
- [2. Current Environment Summary](#2-current-environment-summary)
- [3. Physical Network Architecture](#3-physical-network-architecture)
- [4. Logical Network and VLAN Design](#4-logical-network-and-vlan-design)
- [5. Firewall and Access-Control Model](#5-firewall-and-access-control-model)
- [6. UniFi Port Profiles](#6-unifi-port-profiles)
- [7. Physical Server and Virtualization Platform](#7-physical-server-and-virtualization-platform)
- [8. Core Virtual Workloads](#8-core-virtual-workloads)
- [9. Wazuh SIEM and XDR](#9-wazuh-siem-and-xdr)
- [10. Security Onion Network Security Monitoring](#10-security-onion-network-security-monitoring)
- [11. Elastic / ELK Stack](#11-elastic--elk-stack)
- [12. Identity and Windows Enterprise Services](#12-identity-and-windows-enterprise-services)
- [13. Security Tooling and Network Analysis](#13-security-tooling-and-network-analysis)
- [14. Vulnerability Management Workflow](#14-vulnerability-management-workflow)
- [15. Security Testing and Blue-Team Validation](#15-security-testing-and-blue-team-validation)
- [16. Incident Response Procedure](#16-incident-response-procedure)
- [17. Backup and Recovery](#17-backup-and-recovery)
- [18. Administrative Security Standards](#18-administrative-security-standards)
- [19. Patch and Change Management](#19-patch-and-change-management)
- [20. Naming and IP Addressing Standards](#20-naming-and-ip-addressing-standards)
- [21. Documentation and Recordkeeping](#21-documentation-and-recordkeeping)
- [22. Future Development Roadmap](#22-future-development-roadmap)
- [23. Security Design Principles](#23-security-design-principles)
- [Appendix A. Build and Maintenance Checklist](#appendix-a-build-and-maintenance-checklist)
- [Appendix B. Document Change Log](#appendix-b-document-change-log)

# 1. Document Purpose and Scope

This document describes the architecture and operating model of a home cybersecurity lab built to support hands-on learning in systems administration, virtualization, network defense, vulnerability management, security monitoring, and incident response. The lab is intended to resemble a small enterprise environment while preserving strong isolation between trusted infrastructure and higher-risk testing systems.
- Document the physical and logical network architecture.
- Record the role of the UniFi gateway, switch, physical servers, and Proxmox virtualization layer.
- Define segmentation and access-control expectations for management, server, workstation, security-lab, and isolated-lab networks.
- Document Wazuh as the primary SIEM/XDR platform, Security Onion as the complementary network-security monitoring and threat-hunting platform, and Elastic/ELK as a standalone centralized logging and analytics environment.
- Provide repeatable operating procedures for vulnerability management, incident response, logging, backups, and change management.
- Create a baseline that can be updated as the lab expands.

# 2. Current Environment Summary

| **Component**               | **Current Role**                                                                                                                                                           |
|-----------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Google Fiber                | Upstream Internet service.                                                                                                                                                 |
| Ubiquiti Cloud Gateway Max  | Primary router, firewall, inter-VLAN gateway, and UniFi management platform.                                                                                               |
| UniFi Standard 16 PoE       | Primary managed switch for physical systems and VLAN connectivity.                                                                                                         |
| Four physical servers       | Compute platform for virtualization and security services.                                                                                                                 |
| Proxmox VE                  | Primary hypervisor and virtualization platform.                                                                                                                            |
| Wazuh                       | Primary SIEM and XDR platform for endpoint telemetry, log analysis, alerting, vulnerability visibility, and response workflows.                                            |
| Security Onion              | Network visibility, intrusion detection, threat hunting, packet/protocol analysis, log management, and case management; complements Wazuh endpoint and SIEM/XDR telemetry. |
| Elastic / ELK Stack         | Standalone centralized logging and analytics environment for ingestion pipelines, indexed search, dashboards, visualization, and custom log-engineering exercises. |
| Windows and Linux systems   | Endpoints and servers used for administration, security monitoring, and testing.                                                                                           |
| Kali Linux / security tools | Authorized security testing, traffic analysis, scanning, and defensive validation.                                                                                         |

Note: VLAN IDs, IP ranges, and some workloads in this document are presented as a recommended or  design when exact production values have not yet been fixed. Those entries should be updated when the final addressing plan is implemented.

# 3. Physical Network Architecture

The lab receives Internet access from Google Fiber. The UniFi Cloud Gateway Max is the edge routing and security device. The UniFi Standard 16 PoE switch provides wired connectivity to the physical servers and administrative systems. No wireless access point is required for the core lab design.

```text
                         Internet
                            |
                       Google Fiber
                            |
                +-----------+-----------+
                | UniFi Cloud Gateway  |
                |         Max           |
                |  Router / Firewall    |
                +-----------+-----------+
                            |
                +-----------+-----------+
                | UniFi Standard 16 PoE |
                |     Managed Switch    |
                +-----+-----------+-----+
                      |           |
                      |           +---- Administrative / Lab Workstations
                      |
                      +---------------- Physical Proxmox Servers (4)
```

## 3.1 UniFi Cloud Gateway Max

The Cloud Gateway Max forms the principal trust boundary between the lab and the Internet. It is also responsible for routing traffic between VLANs and enforcing network-access policy.
- Internet routing and NAT
- Stateful firewall enforcement
- VLAN routing and gateway services
- DHCP and DNS forwarding as configured
- UniFi device management
- Traffic visibility and security monitoring
- VPN capability for approved remote administration
- IDS/IPS features where enabled

## 3.2 UniFi Standard 16 PoE

The managed switch provides wired connectivity and VLAN assignment for servers and workstations. Switch ports should be assigned according to the trust level and function of the connected device.
- Access ports for single-VLAN devices
- 802.1Q trunking to Proxmox hosts where multiple VLANs are required
- Centralized port configuration through UniFi
- Future PoE support for approved devices

# 4. Logical Network and VLAN Design

Network segmentation is used to reduce lateral movement, separate administrative traffic from user traffic, and isolate intentionally vulnerable or untrusted systems. The following table is a recommended logical design and can be adjusted to match the final addressing plan.

| **VLAN / Zone** | ** ID** | ** Subnet** | **Purpose**                                                                    |
|-----------------|----------------|--------------------|--------------------------------------------------------------------------------|
| Management      | 10             | 192.168.10.0/24    | Network devices, Proxmox management, iLO, and other administrative interfaces. |
| Servers         | 20             | 192.168.20.0/24    | Physical and virtual servers providing infrastructure and security services.   |
| Computers       | 30             | 192.168.30.0/24    | Trusted administrative and end-user workstations.                              |
| Security Lab    | 40             | 192.168.40.0/24    | Kali, scanning, packet analysis, IDS testing, and defensive tooling.           |
| Isolated Lab    | 50             | 192.168.50.0/24    | Intentionally vulnerable hosts, CTF targets, and higher-risk experiments.      |

## 4.1 Management VLAN

The Management VLAN contains infrastructure interfaces such as the UniFi gateway, switch management, Proxmox management interfaces, HPE iLO, and other privileged administration endpoints. Access should be restricted to specifically authorized administrative systems.

## 4.2 Server VLAN

The Server VLAN hosts physical and virtual infrastructure services. Typical workloads include Active Directory, DNS, Windows and Linux servers, Wazuh components, Security Onion management/search roles where appropriate, Elastic/ELK components, monitoring services, and future backup or automation systems.

## 4.3 Computer VLAN

The Computer VLAN contains trusted administrative workstations. These systems may access approved services on the Server VLAN but should not receive unrestricted access to the Management VLAN.

## 4.4 Security Lab VLAN

The Security Lab VLAN is used for security tooling, traffic analysis, scanning, defensive validation, and authorized testing. It may contain Kali Linux, packet-capture systems, scanning tools, or other security-focused workloads.

## 4.5 Isolated Lab VLAN

The Isolated Lab VLAN is intended for intentionally vulnerable or untrusted systems. It should be denied access to trusted networks by default and granted Internet access only when an exercise requires it.

# 5. Firewall and Access-Control Model

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

## 5.1  Rule Order

**1.** Allow established and related sessions.

**2.** Drop invalid session states.

**3.** Allow authorized management systems to infrastructure interfaces.

**4.** Permit required workstation-to-server services.

**5.** Block Computers from the Management VLAN unless explicitly approved.

**6.** Block Security Lab systems from the Management VLAN.

**7.** Block Isolated Lab systems from trusted private networks.

**8.** Apply a final deny rule for otherwise unauthorized inter-VLAN traffic.

## 5.2 Common Service Ports

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

# 6. UniFi Port Profiles

Port profiles should clearly express the intended trust zone. The exact profile names may be changed, but a consistent naming standard is recommended.

| **Profile**     | **Native VLAN** | **Tagged VLANs**                             | **Use**                                            |
|-----------------|-----------------|----------------------------------------------|----------------------------------------------------|
| MGMT-ACCESS     | Management      | None                                         | Dedicated management interfaces.                   |
| SERVER-ACCESS   | Server          | None                                         | Servers that require only the Server VLAN.         |
| COMPUTER-ACCESS | Computer        | None                                         | Trusted workstations.                              |
| LAB-ACCESS      | Security Lab    | None                                         | Security testing workstations.                     |
| PROXMOX-TRUNK   | Management      | Server, Computer, Security Lab, Isolated Lab | Proxmox host uplink carrying multiple guest VLANs. |

# 7. Physical Server and Virtualization Platform

The lab contains four physical servers that provide compute resources for virtualization. Proxmox VE is the primary hypervisor platform and can support a multi-node cluster as the environment matures.
- Virtual machines and Linux containers
- Centralized web-based management
- Virtual networking and VLAN tagging
- Snapshots and backups
- Storage management
- Clustering and future high-availability experimentation

## 7.1 Recommended Host Naming

| **Host** | **Role**       |
|----------|----------------|
| PVE01    | Proxmox node 1 |
| PVE02    | Proxmox node 2 |
| PVE03    | Proxmox node 3 |
| PVE04    | Proxmox node 4 |

## 7.2 Proxmox Network Model

A VLAN-aware Linux bridge, such as vmbr0, can carry multiple logical networks from the UniFi switch to each Proxmox host. Individual virtual machines are assigned VLAN tags according to their function and trust level.

| ** VLAN Tag** | **Assigned Workloads**                           |
|----------------------|--------------------------------------------------|
| 10                   | Management interfaces where appropriate          |
| 20                   | Server workloads                                 |
| 30                   | Windows or user-workstation test VMs             |
| 40                   | Security tools and monitoring systems            |
| 50                   | Isolated or intentionally vulnerable lab targets |

## 7.3 Storage Considerations

One server configuration under consideration uses eight 1 TB drives. RAID 6 can provide dual-drive fault tolerance and approximately 6 TB of raw usable capacity before filesystem and RAID overhead. RAID improves availability after disk failure but is not a backup strategy.

# 8. Core Virtual Workloads

The environment can host a mixture of Windows and Linux systems. The following names provide a consistent convention and can be adjusted as actual systems are deployed.

| **System** | ** Role**                                                                                             |
|------------|--------------------------------------------------------------------------------------------------------------|
| DC01       | Primary Windows Server domain controller and DNS.                                                            |
| DC02       | Secondary domain controller / DNS redundancy.                                                                |
| WIN11-01   | Domain-joined Windows 11 client.                                                                             |
| WIN11-02   | Secondary Windows client for policy or security testing.                                                     |
| LNX-SRV01  | General-purpose Linux server.                                                                                |
| KALI01     | Authorized security-testing workstation.                                                                     |
| WAZUH01    | Wazuh SIEM/XDR platform or primary Wazuh server role.                                                        |
| SO01       | Security Onion network-security monitoring, threat-hunting, and intrusion-detection platform or sensor role. |
| ELK01      | Standalone Elastic/ELK logging, search, dashboard, and log-pipeline platform.                              |
| SCAN01     | Vulnerability-scanning workload.                                                                             |

# 9. Wazuh SIEM and XDR

Wazuh is the primary Security Information and Event Management (SIEM) and Extended Detection and Response (XDR) platform for the homelab. It serves as the central blue-team monitoring and investigation platform for supported Windows, Linux, and other integrated systems.
- Centralized security-event and endpoint-telemetry collection.
- Windows and Linux security monitoring.
- Authentication and account-activity visibility.
- File integrity monitoring.
- Vulnerability visibility and security-configuration assessment.
- Alert generation, correlation, dashboards, and investigation workflows.
- Threat hunting and incident triage.
- Active-response workflows where safely configured.

## 9.1 Wazuh Components

| **Component**   | **Function**                                                                                           |
|-----------------|--------------------------------------------------------------------------------------------------------|
| Wazuh Server    | Receives and analyzes telemetry, evaluates rules, and generates security alerts.                       |
| Wazuh Indexer   | Stores and indexes alert and security-event data for search and analysis.                              |
| Wazuh Dashboard | Provides the web interface for monitoring, investigations, dashboards, and administration.             |
| Wazuh Agents    | Collect endpoint telemetry from monitored Windows and Linux systems and provide host-level visibility. |

## 9.2 Homelab Wazuh Architecture

```text
Windows Endpoints ----\
Linux Servers ---------+----> Wazuh Server ----> Wazuh Indexer ----> Wazuh Dashboard
Security Systems ------/            |
                                     +----> Alerts / Investigation / Response
```

## 9.3  Detection and Investigation Use Cases
- Repeated failed authentication attempts and brute-force behavior.
- Unexpected privileged-account activity.
- Suspicious PowerShell or command-line execution.
- Unauthorized file changes or persistence activity.
- Endpoint vulnerability findings and configuration weaknesses.
- Suspicious administrative changes.
- Potential malware indicators or lateral-movement evidence.
- Correlating host telemetry with firewall and network evidence during an investigation.

**Reference:** [Wazuh](https://wazuh.com/)

# 10. Security Onion Network Security Monitoring

Security Onion complements Wazuh by providing network-centric visibility and investigation capabilities. Security Onion Solutions describes the platform as a free and open security platform built for defenders, with network visibility, host visibility, intrusion detection, honeypots, log management, and case management. In this homelab, its primary role is network security monitoring (NSM), network intrusion detection, packet/protocol analysis, and threat hunting.

Primary Security Onion capabilities used or planned for the lab include:
- Network visibility and network-security monitoring.
- Network intrusion detection using Suricata.
- Protocol analysis and network metadata generation using Zeek.
- Packet capture and evidence review where the deployment model supports it.
- Threat hunting through the Security Onion Console (SOC) and associated search interfaces.
- Centralized log analysis and correlation of network and host data sources.
- Case management for documenting alerts, investigations, evidence, and analyst actions.
- Honeypot and intrusion-detection capabilities for controlled defensive exercises.

## 10.1 Relationship Between Wazuh and Security Onion

The two platforms are intentionally used for different but complementary visibility layers. Wazuh remains the primary SIEM/XDR platform for endpoint telemetry, file-integrity monitoring, security-configuration assessment, vulnerability visibility, alerting, and endpoint-focused response workflows. Security Onion adds deeper network visibility through traffic inspection, NIDS alerts, protocol metadata, packet evidence, and network-focused threat hunting. During an investigation, findings from both platforms can be correlated to reconstruct activity across hosts and the network.

## 10.2 Core Security Onion Technologies
- Suricata - network intrusion detection and alert generation from monitored traffic.
- Zeek - protocol analysis and rich network metadata for investigative searches and threat hunting.
- Elastic components - indexing, searching, and analysis used within the Security Onion architecture; these are distinct from the standalone ELK lab documented in Section 11.
- Security Onion Console (SOC) - analyst interface for alerts, hunting, dashboards, case management, and related workflows.

## 10.3 Homelab Network Visibility Architecture

Security Onion should receive a copy of selected network traffic through an authorized monitoring method such as a switch mirror/SPAN configuration or network TAP. A dedicated monitoring interface can observe traffic while a separate management interface is used for administration. Only VLANs and traffic flows required for the lab exercise should be mirrored to the sensor.

```text
Selected VLAN Traffic
        |
        v
Switch Mirror / SPAN / TAP
        |
        v
Security Onion Sensor
        |
        +----> Suricata ----> NIDS Alerts
        +----> Zeek --------> Network Metadata
        +----> Packet Evidence / Logs
        |
        v
Security Onion SOC
Hunt / Cases / Analysis
```

## 10.4  Security Onion Use Cases
- Detecting reconnaissance and network scanning generated during authorized lab exercises.
- Investigating suspicious DNS, HTTP, TLS, SMB, or other protocol activity.
- Reviewing Suricata alerts and validating detections against packet or flow evidence.
- Using Zeek metadata to identify unusual connections, lateral movement, beaconing, or unexpected services.
- Correlating network evidence with Wazuh endpoint alerts and Windows/Linux event data.
- Building repeatable threat-hunting exercises and documenting findings in cases.
- Tuning signatures and detections to reduce false positives while preserving useful coverage.

**References:** [Security Onion Solutions – Software](https://securityonionsolutions.com/software) and [Security Onion Documentation](https://docs.securityonion.net/).

# 11. Elastic / ELK Stack

The homelab includes a standalone **Elastic/ELK Stack** environment for centralized logging, log engineering, indexed search, dashboards, visualization, and general-purpose security analytics. This environment is intentionally documented separately from Wazuh and Security Onion so each platform has a clear role.

Historically, **ELK** refers to **Elasticsearch, Logstash, and Kibana**. The broader Elastic Stack can also use Elastic Agent or Beats for data collection. In this lab, ELK is primarily used to practice building ingestion pipelines, parsing and normalizing logs, creating dashboards, and performing ad hoc searches across Windows, Linux, application, and selected network data.

## 11.1 Core Elastic / ELK Components

| **Component** | **Homelab Role** |
|---------------|------------------|
| Elasticsearch | Stores, indexes, and searches structured and unstructured log and event data. |
| Logstash | Ingests, parses, transforms, enriches, and routes log data into Elasticsearch or other approved destinations. |
| Kibana | Provides dashboards, visualizations, search, saved queries, and interactive analysis of indexed data. |
| Elastic Agent / Beats | Optional collectors used to forward host, service, or application telemetry into the ELK pipeline. |

## 11.2 Homelab ELK Architecture

```text
Windows / Linux / Applications / Selected Network Logs
                         |
                         v
              Elastic Agent / Beats / Syslog
                         |
                         v
                    Logstash
              Parse / Normalize / Enrich
                         |
                         v
                  Elasticsearch
                Index / Store / Search
                         |
                         v
                      Kibana
             Dashboards / Hunt / Analysis
```

## 11.3  ELK Use Cases

- Centralize Windows, Linux, application, and infrastructure logs for search and analysis.
- Build Logstash pipelines to parse raw events and normalize fields.
- Create Kibana dashboards for authentication, system activity, network events, and lab health.
- Practice structured and full-text searches across large event sets.
- Compare host activity with firewall, application, or selected network telemetry.
- Experiment with index design, mappings, retention, and lifecycle-management concepts.
- Develop reusable saved searches and visualizations for recurring investigations.
- Create controlled datasets for security analytics and detection-engineering exercises.

## 11.4 Relationship to Wazuh and Security Onion

The three platforms serve complementary purposes:

- **Wazuh** remains the primary SIEM/XDR platform for endpoint-focused detection, file-integrity monitoring, security-configuration assessment, vulnerability visibility, alerting, and response workflows.
- **Security Onion** provides network-centric security monitoring, Suricata alerts, Zeek metadata, packet evidence, threat hunting, and case-management workflows.
- **Elastic/ELK** provides a separate general-purpose environment for centralized log ingestion, transformation, search, visualization, and custom analytics.

Selected data sources may be sent to more than one platform when a lab exercise requires correlation, but each platform is maintained as an independent workload. The standalone ELK environment should not be confused with the Wazuh Indexer or with Elastic components used internally by Security Onion.

# 12. Identity and Windows Enterprise Services

The lab will include a Windows Active Directory environment to practice enterprise identity administration and defensive monitoring. Typical exercises include user and group provisioning, Organizational Units, Group Policy, Kerberos, DNS, service accounts, password policy, account lockout, least privilege, and security auditing.

Microsoft cloud services may be integrated for hybrid identity or endpoint-management exercises, including Microsoft Entra ID, Microsoft 365, Exchange Online, Azure, Intune, Conditional Access, and multi-factor authentication where available.

# 13. Security Tooling and Network Analysis

| **Tool / Platform** | **Lab Use**                                                                                                                                           |
|---------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------|
| Wazuh               | Primary SIEM/XDR for endpoint telemetry, file integrity monitoring, vulnerability visibility, alerting, investigation, and endpoint-focused response. |
| Security Onion      | Network security monitoring, network intrusion detection, threat hunting, packet/protocol analysis, log management, and case management.              |
| Elastic / ELK Stack | Standalone centralized log ingestion, Logstash parsing/enrichment, Elasticsearch search/indexing, and Kibana dashboards/visualization.                 |
| Wireshark           | Interactive packet capture and protocol analysis.                                                                                                     |
| tcpdump             | Command-line packet capture on Linux.                                                                                                                 |
| Nmap                | Authorized host discovery, port scanning, and service enumeration.                                                                                    |
| RITA                | Network-traffic analysis and beaconing investigation where applicable.                                                                                |
| Zeek                | Network protocol analysis and metadata generation as part of the Security Onion monitoring stack.                                                     |
| Suricata            | Network intrusion detection and signature-based alerting through Security Onion.                                                                      |
| Python              | Custom defensive utilities such as IDS prototypes and port-scanning tools.                                                                            |
| PowerShell          | Windows administration, automation, event collection, and security analysis.                                                                          |

# 14. Vulnerability Management Workflow

The lab is designed to support a repeatable vulnerability-management lifecycle rather than one-time scanning. Findings should be validated and prioritized according to technical severity and actual exposure.

**1.** Identify and inventory the asset.

**2.** Perform an authorized vulnerability scan.

**3.** Validate significant findings.

**4.** Research the associated CVE or weakness.

**5.** Assess exploitability, exposure, asset criticality, and potential impact.

**6.** Prioritize remediation based on risk rather than CVSS alone.

**7.** Patch, reconfigure, remove, or otherwise mitigate the weakness.

**8.** Rescan the asset.

**9.** Document the remediation and verification result.

# 15. Security Testing and Blue Team Validation

Security testing should remain limited to systems owned by or explicitly authorized for the homelab. The preferred workflow uses controlled offensive activity to validate defensive visibility and response.

| **Phase**         | **Activity**                                                                                                                  |
|-------------------|-------------------------------------------------------------------------------------------------------------------------------|
| Deploy            | Create or restore a test target and place it on the correct VLAN.                                                             |
| Baseline          | Record IP address, operating system, services, open ports, and expected behavior.                                             |
| Generate Activity | Perform an authorized scan, authentication test, or simulated attack.                                                         |
| Detect            | Review Wazuh endpoint alerts, Security Onion network alerts/metadata, Elastic/ELK searches and dashboards, Windows/Linux logs, firewall logs, and packet evidence. |
| Investigate       | Identify source, destination, time, technique, affected systems, host telemetry, network evidence, and indicators.            |
| Respond           | Isolate hosts, block sources, disable accounts, patch weaknesses, or remove persistence as appropriate.                       |
| Document          | Record the timeline, findings, response actions, and lessons learned.                                                         |

# 16. Incident Response Procedure

## 16.1 Preparation

Maintain logging, monitoring, backups, administrator access, known-good baselines, and tested response tools.

## 16.2 Detection and Analysis

Review alerts and supporting telemetry to determine whether malicious or unauthorized activity occurred.

## 16.3 Containment

Use VLAN isolation, firewall blocks, account disabling, or VM network disconnection to limit impact.

## 16.4 Eradication

Remove malicious files, unauthorized accounts, persistence mechanisms, and vulnerable configurations.

## 16.5 Recovery

Restore secure services, verify normal operation, and monitor for recurrence.

## 16.6 Lessons Learned

Document the incident timeline, indicators, root cause, control gaps, actions performed, and recommended improvements.

# 17. Backup and Recovery

Critical configuration and workloads should be backed up outside the primary storage array. RAID protects against some disk failures but does not protect against deletion, corruption, ransomware, or administrative error.
- Proxmox configuration and critical virtual machines
- UniFi configuration backups
- Wazuh configuration and detection content
- Security Onion configuration, detection content, case data, and deployment documentation as appropriate
- Elastic/ELK configuration, Logstash pipelines, Kibana saved objects, index templates, and deployment documentation as appropriate
- Active Directory or other infrastructure configuration
- Scripts, automation, and custom security tools
- Network diagrams and documentation
- Critical vulnerability and incident-response records

# 18. Administrative Security Standards
- Use separate administrative accounts for privileged tasks when practical.
- Use strong, unique passwords and MFA wherever supported.
- Restrict management interfaces to the Management VLAN or other explicitly trusted sources.
- Use SSH keys for Linux administration where appropriate.
- Apply least privilege and remove unnecessary accounts or permissions.
- Monitor privileged logons and administrative changes.
- Do not expose management interfaces directly to the Internet without a justified and secured design.

# 19. Patch and Change Management

Routine updates should include Proxmox hosts, Windows servers and clients, Linux systems, UniFi devices, Wazuh components, Security Onion components, Elastic/ELK components, and other security applications. Significant configuration changes should be recorded so that the environment can be troubleshot and restored consistently.

| **Change Record Field** | **Description**                                 |
|-------------------------|-------------------------------------------------|
| Date                    | Date the change was made.                       |
| System                  | Device, service, VLAN, or workload affected.    |
| Change                  | What was modified.                              |
| Reason                  | Why the modification was required.              |
| Result                  | Successful, partial, or failed.                 |
| Rollback                | How to restore the prior state if necessary.    |
| Notes                   | Additional validation or follow-up information. |

# 20. Naming Standards

Consistent naming and address allocation simplify troubleshooting, monitoring, and documentation. Infrastructure systems should use static addresses or DHCP reservations where appropriate.

## 20.1 Naming s

| **Category**           | **Names**                            |
|------------------------|--------------------------------------|
| Proxmox Hosts          | PVE01, PVE02, PVE03, PVE04           |
| Domain Controllers     | DC01, DC02                           |
| Windows Clients        | WIN11-01, WIN11-02                   |
| Linux Servers          | LNX-SRV01                            |
| Security Systems       | KALI01, WAZUH01, SO01, ELK01, IDS01, SCAN01 |
| Network Infrastructure | UCG-MAX01, USW-16-01                 |

# 21. Documentation and Recordkeeping
- Physical network diagram
- Logical network diagram
- VLAN and subnet table
- Device inventory
- IP address inventory
- Virtual-machine inventory
- Firewall-rule documentation
- Switch-port assignments
- Backup and recovery procedures
- Security-tool configurations, including Wazuh, Security Onion, and Elastic/ELK
- Vulnerability reports
- Incident-response reports
- Change log

Passwords, private keys, API tokens, recovery codes, and other secrets should not be stored in general-purpose lab documentation. Use a dedicated secrets-management or password-management solution instead.

# 22. Future Development Roadmap
- Complete and document the four-node Proxmox cluster.
- Deploy centralized Proxmox backup infrastructure.
- Expand Wazuh coverage to all supported Windows and Linux workloads.
- Develop custom Wazuh detection rules and response playbooks.
- Operationalize Security Onion network visibility for selected VLANs and validate Suricata and Zeek telemetry.
- Develop Security Onion threat-hunting queries, Suricata detection tuning, and investigation/case workflows.
- Build and tune Elastic/ELK ingestion pipelines, dashboards, saved searches, and retention policies for selected lab data sources.
- Build an Active Directory attack-and-defense range.
- Implement dedicated red-team and blue-team lab segments if needed.
- Automate host deployment and configuration with PowerShell, Python, or Ansible.
- Implement internal PKI and certificate-based services.
- Add UPS monitoring and graceful shutdown procedures.
- Map selected detection exercises to MITRE ATT&CK techniques.

# 23. Security Design Principles

| **Principle**    | **Application in the Homelab**                                                                                                             |
|------------------|--------------------------------------------------------------------------------------------------------------------------------------------|
| Segmentation     | Separate systems by function and trust level using VLANs and firewall policy.                                                              |
| Least Privilege  | Grant only the network and administrative access required for a task.                                                                      |
| Default Deny     | Block inter-zone communication unless a documented requirement exists.                                                                     |
| Defense in Depth | Combine network, identity, endpoint, logging, vulnerability, and administrative controls.                                                  |
| Visibility       | Collect endpoint telemetry with Wazuh, network telemetry with Security Onion, and centralized searchable logs in Elastic/ELK so activity can be correlated, detected, and investigated. |
| Repeatability    | Use naming standards, documentation, change records, and repeatable test procedures.                                                       |

# Appendix A. Build and Maintenance Checklist
- [ ] UniFi gateway configuration backed up.
- [ ] UniFi switch configuration backed up.
- [ ] VLANs and subnets documented.
- [ ] Firewall rules reviewed for least privilege.
- [ ] Management interfaces restricted to trusted sources.
- [ ] Proxmox host names and management addresses documented.
- [ ] Virtual machines inventoried.
- [ ] Wazuh agents deployed to supported endpoints.
- [ ] Wazuh alerting and dashboard access verified.
- [ ] Security Onion management access restricted to trusted administrative sources.
- [ ] Security Onion monitoring interface receiving the intended mirrored/TAP traffic.
- [ ] Suricata alerts and Zeek network metadata visible in Security Onion SOC.
- [ ] Wazuh and Security Onion evidence can be correlated during a test investigation.
- [ ] ELK01 receives logs from at least one Windows or Linux source.
- [ ] Logstash or direct ingestion parsing is validated for selected data sources.
- [ ] Kibana dashboards and saved searches are backed up or exportable.
- [ ] Elasticsearch storage and retention settings are documented.
- [ ] Vulnerability scan schedule or manual procedure documented.
- [ ] Critical systems included in backup plan.
- [ ] Restore procedure tested for at least one workload.
- [ ] Security exercises kept within authorized lab boundaries.
- [ ] Change log updated after significant modifications.

# Appendix B. Document Change Log

| **Version** | **Date**        | **Change**                                                                                                                                                                                                              |
|-------------|-----------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| 1.0         | August 14, 2026 | Initial consolidated homelab documentation. Wazuh identified as the primary SIEM and XDR platform.                                                                                                                      |
| 1.1         | August 15, 2026 | Added Security Onion as the complementary network-security monitoring and threat-hunting platform; documented Suricata, Zeek, traffic-monitoring architecture, investigation use cases, and updated tooling/checklists. |
| 1.2         | August 15, 2026 | Added a standalone Elastic/ELK section covering Elasticsearch, Logstash, Kibana, ingestion architecture, log analytics use cases, integrations, backups, naming, and maintenance checks. |
