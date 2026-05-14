# Client Environment Profile — Intake Guide

## Introduction

As part of our co-managed Microsoft Sentinel service we are building a comprehensive
security monitoring workbook that lives directly inside your Sentinel environment. This
workbook serves as a living reference for both your team and ours — giving any engineer
who opens it an immediate, clear picture of your environment, your security coverage,
and the health of your monitoring platform.

To build this accurately we need you to help us document your environment. The
information you provide here will be used by our engineers to configure monitoring,
tune detections, identify coverage gaps, and respond to incidents faster and more
effectively. The better we understand your environment the better we can protect it.

This is a one-time exercise. Once complete we will maintain and update the profile as
your environment evolves.

## How to Use This Document

Work through each section and fill in what you know. You do not need to be exhaustive
— if you are unsure about something leave it blank or add a note and we will follow up.
Short, accurate answers are better than long, uncertain ones.

An example profile filled out with a fictional company called Contoso Medical Group is
included alongside this document. Use it as a reference if you are unsure what we are
looking for in any section.

If you have questions at any point contact your Optiv engineer before submitting.

---

## Environment Summary

### What We Are Looking For

A short paragraph — three to five sentences — that gives our engineers an immediate
technical orientation to your environment. Think of this as what a new security engineer
would need to know in their first 60 seconds working with your environment.

This is free form text. Write it in plain language. You do not need to be technical —
just describe what you have and how it is set up at a high level.

### Why This Matters for Security Monitoring

Understanding your environment at a glance helps our engineers respond faster during
incidents, configure monitoring correctly from the start, and avoid making assumptions
about your environment that could lead to gaps in coverage.

### Considerations — What to Include

Think about the following when writing your summary. You do not need to answer all
of these — use them as prompts to help you think about what is most relevant:

- Is your environment cloud only, hybrid, or primarily on-premises?
- How many users and devices are in your environment roughly?
- How many physical locations or sites do you operate from?
- What is the most critical application or system in your environment — the one that
  if compromised would cause the most damage?
- Are there any compliance or regulatory requirements that apply to your organization
  such as HIPAA, PCI, SOC 2, or similar?
- What is the general maturity of your security program — are basic controls like MFA
  and endpoint protection in place?
- Is there anything unique or unusual about your environment that a security engineer
  should know before working in it?

### Example

Hybrid environment with on-premises Active Directory synced to Entra ID. Approximately
450 users and 600 managed endpoints across 12 clinic locations in the Pacific Northwest.
Primary workloads are Windows 11 endpoints managed via Intune. Critical dependency on
Epic EHR hosted on-premises at primary data center. HIPAA regulated. MFA enforced,
Conditional Access in place, Defender for Endpoint fully deployed.

---

## Network

### What We Are Looking For

Key details about the network devices and services that protect and route traffic in
your environment. You do not need to be deeply technical here — vendor names, product
names, and a yes or no on a few questions is all we need.

### Why This Matters for Security Monitoring

Network devices are some of the most important sources of security data in your
environment. Knowing what you have tells us what data we can collect, how to collect
it, and what blind spots may exist. For example knowing your DNS service tells us
where DNS logs come from and how to ingest them. Knowing your firewall vendor tells
us what connector to use and what format the logs arrive in.

### Considerations

- Firewall Vendor and Model — the brand and model name of your firewall. Examples
  include Palo Alto, Fortinet, Cisco, Check Point, or Meraki
- Number of Firewalls — how many firewalls do you have in total across all locations
- Firewall High Availability — do you have two firewalls configured to back each other
  up if one fails? This is sometimes called an active/passive or active/active pair
- Multiple Locations — if you have more than one office or site, does each location
  have its own firewall or do all locations share one central firewall
- Virtual Private Network Solution — the service your remote users connect through to
  access company resources. Examples include GlobalProtect, Cisco AnyConnect, or
  Zscaler Private Access
- Web Proxy / Filtering Service — a service that controls and monitors what websites
  your users can access. Examples include Zscaler, Cisco Umbrella, or Netskope
- DNS Service — the service that resolves domain names on your network. Examples
  include Windows DNS, Cisco Umbrella, or Zscaler. If you are unsure ask your IT team
- Email Security Service — if you use a third party service to filter and protect your
  email beyond what Microsoft provides. Examples include Mimecast, Proofpoint, or
  Barracuda. If you only use Microsoft Exchange Online with Defender you can leave
  this blank as we already have that from your licensing information
- Network Segmentation — is your network divided into separate zones or segments?
  For example separate networks for servers, workstations, guests, or specialized
  devices. A simple yes or no is fine with a brief description if yes

### Example

Firewall Vendor: Palo Alto Networks
Firewall Model: PA-3260
Number of Firewalls: 13 total — 1 primary data center, 12 branch locations
Firewall High Availability: Yes — Active/Passive pair at primary data center only
Multiple Locations: Yes — each clinic location has its own dedicated firewall
Virtual Private Network Solution: GlobalProtect
Web Proxy / Filtering Service: Zscaler Internet Access
DNS Service: Internal Windows DNS forwarding to Cisco Umbrella for filtering
Email Security Service: Exchange Online with Defender for Office 365 — no third party
Network Segmentation: Yes — clinical, corporate, guest, and medical device networks

Notes:
- Branch location firewalls have no high availability — a single firewall failure at
  any clinic means complete loss of network visibility for that site
- Zscaler handles DNS filtering for all internet bound traffic — DNS logs available
  via Zscaler connector

---

## Identity

### What We Are Looking For

Key details about how your organization manages user identities and access. Identity
is the most frequently targeted area in modern cyberattacks so understanding your
identity environment helps us configure the right monitoring and ensure we have
visibility into the right places.

### Why This Matters for Security Monitoring

Different identity configurations produce different security data. For example if you
use a third party identity provider like Okta your sign-in events may not appear in
Microsoft logs at all — meaning we could have a significant blind spot without knowing
it. Similarly knowing whether Defender for Identity sensors are deployed on your domain
controllers tells us whether we have on-premises identity threat detection in place.

### Considerations

- Active Directory — do you have an on-premises Active Directory environment? This is
  the traditional Microsoft directory service that manages users, computers, and
  policies on your internal network. A simple yes or no is fine
- Number of Domain Controllers — if you have Active Directory, how many domain
  controller servers do you have and where are they located. A domain controller is
  the server that handles authentication for your Active Directory environment
- Entra ID — this is Microsoft's cloud identity service previously known as Azure
  Active Directory. Is your environment cloud only meaning users exist only in Entra
  ID, or hybrid meaning you have both on-premises Active Directory and Entra ID
  synchronized together
- Entra ID Protection — this is a Microsoft feature that automatically detects and
  flags risky sign-ins and potentially compromised accounts. Is it enabled in your
  environment? If you are unsure check with your IT team or look in the Microsoft
  Entra admin center under Protection
- Multi-Factor Authentication Enforced — is multi-factor authentication required for
  all users when they sign in? Examples include Microsoft Authenticator, SMS codes,
  or hardware tokens
- Third Party Identity Provider — do you use any identity service other than Microsoft
  to manage user logins? Examples include Okta, Ping Identity, or Duo Security. If
  your users log into applications through a non-Microsoft login page this likely
  applies to you
- Password Manager — does your organization use a tool to store and manage passwords?
  Examples include 1Password, LastPass, Bitwarden, or KeePass
- Privileged Access Management Solution — do you use a dedicated tool to manage and
  control access to sensitive admin accounts and systems? Examples include CyberArk,
  BeyondTrust, or Thycotic. This is different from a regular password manager and is
  typically used by IT and security teams to manage privileged credentials
- Microsoft Defender for Identity Sensors Deployed — if you have on-premises Active
  Directory, have the Defender for Identity sensors been installed on your domain
  controllers? Your Optiv engineer can confirm this if you are unsure

### Example

Active Directory: Yes — single domain contoso.local
Number of Domain Controllers: 3 — primary data center, secondary data center, DR site
Entra ID: Hybrid — synchronized with on-premises Active Directory
Entra ID Protection Enabled: Yes
Multi-Factor Authentication Enforced: Yes — Microsoft Authenticator for all users
Third Party Identity Provider: No — Microsoft native only
Password Manager: 1Password — company wide deployment
Privileged Access Management Solution: No
Microsoft Defender for Identity Sensors Deployed: Yes — all 3 domain controllers

Additional Notes — include anything else about your identity environment
that may be relevant for security monitoring or logging:
- Duo Security used specifically for Epic EHR authentication — separate from
  Microsoft MFA

---

## Endpoints

### What We Are Looking For

Basic information about the devices in your environment, how they are managed,
and what security software is in place. We only need high level information here.

### Why This Matters for Security Monitoring

Endpoints are one of the most critical sources of security data in your environment.
Knowing your operating systems tells us what agents to deploy. Knowing your endpoint
protection solution tells us what security data is available in Sentinel. The Defender
XDR Data Connector in particular is one of the single most important connections for
our monitoring capability — if it is not configured we may be missing critical threat
detection data entirely.

### Considerations

- Operating System Mix — what operating systems do your endpoints run. Examples
  include Windows 10, Windows 11, macOS, or Linux. A rough breakdown is fine
- Endpoint Management Solution — the tool used to manage and deploy software to
  endpoints. Examples include Microsoft Intune, Microsoft Configuration Manager
  also known as SCCM, or Jamf for Mac environments. If there is no centralized
  management solution please note that
- Endpoint Detection and Response Solution — the advanced security software installed
  on your endpoints that monitors and responds to threats. Examples include Microsoft
  Defender for Endpoint, CrowdStrike Falcon, or SentinelOne. This is different from
  basic antivirus
- Defender XDR Data Connector — if you use Microsoft Defender for Endpoint this
  connector sends your security data into Sentinel. Please refer to the attached
  Defender XDR Data Connector Setup Guide for instructions on how to find and
  configure this connector and how to answer the questions below. Your IT team
  should be able to provide this information
- Third Party Endpoint Detection and Response — if you use a non-Microsoft endpoint
  security solution please describe whether it is connected to Sentinel and how

### Example

Operating System Mix: Primarily Windows 11 — remainder macOS

Endpoint Management Solution: Microsoft Intune — fully deployed

Endpoint Detection and Response Solution: Microsoft Defender for Endpoint Plan 2

If Microsoft Defender for Endpoint is in use:
  Defender XDR Data Connector Installed: Yes
  Incidents and Alerts Connected: Yes
  Defender for Endpoint Events Enabled: Yes
  Defender for Identity Events Enabled: Yes
  Defender for Office 365 Events Enabled: Yes
  Defender for Cloud Apps Events Enabled: No
  — See attached Defender XDR Data Connector Setup Guide for instructions

If a third party Endpoint Detection and Response solution is in use:
  Solution Name: N/A
  Connected to Sentinel: N/A
  Connection Method: N/A

Additional Notes — include anything else about your endpoint environment
that may be relevant for security monitoring or logging:
- Approximately 120 iPads managed via Intune — no Defender for Endpoint on mobile
- Zebra label printers at each clinic location are unmanaged network devices with
  no agent coverage

---

## Cloud

### What We Are Looking For

A simple overview of what cloud platforms and services your organization uses beyond
Microsoft Azure. We already have your Azure and licensing details from the spreadsheet
so we only need the information here that fills in the gaps — specifically what types
of Azure workloads you run and whether you use any other cloud platforms.

### Why This Matters for Security Monitoring

Different cloud workloads and platforms produce different types of security data.
Knowing what Azure workloads you run tells us which additional log sources and
connectors are relevant. Knowing whether you use AWS or GCP tells us whether we
need to configure cloud connectors to pull security data from those environments
into Sentinel. Without this information entire cloud environments could be invisible
to our monitoring.

### Considerations

- Azure Workload Types — what types of resources do you run in Azure. You do not
  need a full inventory — just the types of resources. Examples include Virtual
  Machines, Azure SQL databases, Azure Storage accounts, Azure Key Vault, Azure
  App Services, or Azure Kubernetes Service. If you are unsure ask your IT team
  to describe what your organization hosts in Azure
- AWS — does your organization use Amazon Web Services for any workloads or
  services? If yes please describe what services you use. Examples include EC2
  virtual machines, S3 storage, RDS databases, CloudTrail audit logs, or Route 53
  DNS. Each of these has its own Sentinel connector available
- GCP — does your organization use Google Cloud Platform for any workloads or
  services? If yes please describe what services you use. Examples include Compute
  Engine, Cloud Storage, BigQuery, VPC Flow logs, or Google Kubernetes Engine
- Other Cloud Platforms — do you use any other cloud platforms not mentioned above.
  Examples include Oracle Cloud or IBM Cloud

### Example

Azure Workload Types: Virtual Machines — 8 total, Azure SQL databases — 2,
Azure Storage accounts — 3, Azure Key Vault — 1

AWS Present: No

GCP Present: No

Other Cloud Platforms: No

Additional Notes — include anything else about your cloud environment
that may be relevant for security monitoring or logging:
- Azure Key Vault stores Epic EHR integration credentials — sensitive, monitor
  closely for unauthorized access
- Azure SQL databases contain patient health information — PHI

---

## Virtualization

### What We Are Looking For

Basic information about your virtualization environment. We only need to know what
platforms you use — not a full inventory of virtual machines.

### Why This Matters for Security Monitoring

Different virtualization platforms produce different security data and require different
monitoring approaches. VMware ESXi for example has a dedicated Microsoft Sentinel
solution with built-in detection rules specifically for ESXi log data. Azure Virtual
Desktop requires specific configuration for monitoring user sessions and host activity.
Knowing what you use tells us what solutions to deploy and what log sources to expect.

### Considerations

- Hypervisor Platform — the software that runs your virtual machines. Common examples
  include VMware vSphere or ESXi, Microsoft Hyper-V, or a combination of both. If you
  are unsure ask your IT team what platform your virtual machines run on
- VMware ESXi in Use — if you use VMware, is ESXi specifically part of your
  environment? ESXi is VMware's bare metal hypervisor that runs directly on physical
  servers. A simple yes or no is fine
- Azure Virtual Desktop in Use — do you use Azure Virtual Desktop to provide virtual
  desktops or remote applications to your users? This is sometimes called AVD or
  Windows Virtual Desktop. A simple yes or no is fine

### Example

Hypervisor Platform: VMware vSphere 7.0
VMware ESXi in Use: Yes
Azure Virtual Desktop in Use: No

Additional Notes — include anything else about your virtualization environment
that may be relevant for security monitoring or logging:
- vCenter syslog forwarding configured but not consistently monitored — silent
  log detection recommended
- Approximately 45 virtual machines at primary data center — mix of Windows
  Server and Linux

---

## Containers

### What We Are Looking For

Basic information about whether your organization uses containers and what platforms
you run them on. If you do not use containers simply write No and move on.

### Why This Matters for Security Monitoring

Container environments produce unique security data that requires specific connectors
and monitoring approaches. Microsoft Sentinel has dedicated solutions for platforms
like Azure Kubernetes Service and Amazon EKS with built-in detection rules specifically
for container activity. Knowing what you use tells us which connectors to deploy and
what security data is available. Without this information container environments can
be completely invisible to our monitoring.

### Considerations

- Containers in Use — does your organization use containers for any applications or
  services? Containers are a way of packaging and running applications in isolated
  environments. A simple yes or no is fine. If no you can skip the rest of this section
- Container Platform — what platform do you run your containers on. Examples include
  Azure Kubernetes Service also known as AKS, Amazon Elastic Kubernetes Service also
  known as EKS, Google Kubernetes Engine also known as GKE, or standalone Docker.
  If you are unsure ask your IT or development team
- Container Orchestration — do you use Kubernetes to manage your containers.
  Kubernetes is the most common tool for managing large numbers of containers.
  If yes is it a managed service like AKS or self-managed on your own servers
- Container Registry — where do you store your container images. Examples include
  Azure Container Registry, Docker Hub, or GitHub Container Registry. A container
  registry is like a library of application packages that get deployed as containers

### Example

Containers in Use: No

Additional Notes — include anything else about your container environment
that may be relevant for security monitoring or logging:

---

## SaaS and Applications

### What We Are Looking For

A simple list of the key applications and SaaS services your organization uses.
We want to know what is currently connected to our monitoring platform and what
else exists that may be worth connecting in the future.

### Why This Matters for Security Monitoring

SaaS applications are increasingly targeted by attackers and are a common source
of data breaches. Knowing what applications you use tells us where potential blind
spots exist in our monitoring coverage. Some applications have native connectors
to Microsoft Sentinel or Microsoft Defender for Cloud Apps that allow us to monitor
activity and detect threats directly within those platforms.

Microsoft Defender for Cloud Apps is a security service that monitors activity inside
your cloud applications and reports suspicious behavior to our security team. It can
detect things like unusual file downloads, logins from unexpected locations, and
unauthorized data sharing. It only monitors company managed devices and corporate
network traffic — it does not monitor personal devices.

### Considerations

- Applications Currently Connected — list any applications or SaaS services that
  are currently connected to Microsoft Sentinel or Microsoft Defender for Cloud Apps
  for security monitoring. If you are unsure ask your IT team
- Applications Not Currently Connected — list any other key business applications
  or SaaS services your organization uses that are not currently connected to security
  monitoring. Include things like CRM systems, HR platforms, finance tools,
  collaboration tools, file sharing services, or any other critical business application

### Example

Applications and SaaS Services Currently Connected to Sentinel or
Defender for Cloud Apps:
- Microsoft 365 — Exchange, Teams, SharePoint, OneDrive
- Zoom — connected via Defender for Cloud Apps for shadow IT visibility

Applications and SaaS Services Not Currently Connected but Worth Noting:
- Salesforce — CRM platform, contains customer data
- Workday — HR and payroll platform, contains sensitive employee data
- Epic EHR — on-premises clinical application, no cloud connector available
- Duo Security — used for Epic authentication only

Additional Notes — include anything else about your applications that may be
relevant for security monitoring or logging:

---

## Public Facing Infrastructure

### What We Are Looking For

Basic information about any websites, web applications, or services your organization
exposes to the internet. We only need to know what exists and what protects it — not
a full technical inventory.

### Why This Matters for Security Monitoring

Public facing infrastructure is the most exposed part of your environment and a
primary target for attackers. Web application firewalls in particular are a critical
data source for Sentinel — they log every attack attempt against your web applications
including SQL injection, cross-site scripting, and other common attacks. Microsoft
Sentinel has dedicated connectors and pre-built detection rules specifically for web
application firewall logs. Without this data we have no visibility into attacks
targeting your public facing applications.

### Considerations

- Public Website or Web Application — does your organization have any public facing
  websites or web applications accessible from the internet? A simple yes or no is
  fine. If yes please describe what platform they are hosted on. Examples include
  Azure App Service, on-premises web server, or a third party hosting provider
- Web Application Firewall — do you have a Web Application Firewall protecting your
  public websites or applications? A Web Application Firewall is a security service
  that sits in front of your web applications and blocks malicious traffic before it
  reaches your application. Examples include Azure Web Application Firewall,
  Cloudflare, or Akamai. If you are unsure ask your IT team
- Web Application Firewall Logs Connected to Sentinel — if you have a Web Application
  Firewall are its logs currently being sent to Microsoft Sentinel? This is one of
  the most valuable data sources for detecting web application attacks. Your IT team
  should be able to confirm this
- Content Delivery Network — do you use a Content Delivery Network service to deliver
  your website or application content? Examples include Azure CDN, Cloudflare, or
  Akamai. A Content Delivery Network speeds up your website by serving content from
  servers closer to your users. If you are unsure ask your IT team
- Public Facing Customer Portal — do you have any portals that external users such as
  customers, patients, or partners log into over the internet? Examples include a
  patient portal, client portal, or partner extranet. These are higher value targets
  because they involve external authentication
- External Email Presence — does your organization use Microsoft Exchange Online for
  email with public MX records? This is already captured in your licensing spreadsheet
  so only note here if you use a different email platform that is publicly accessible

### Example

Public Website or Web Application: Yes — contosomed.com hosted on Azure App Service
Web Application Firewall: Azure Web Application Firewall — Standard tier
Web Application Firewall Logs Connected to Sentinel: Yes — via Diagnostic Settings
Content Delivery Network: Azure CDN
Public Facing Customer Portal: Yes — MyChart patient portal hosted and managed
by Epic externally — no direct log access available
External Email Presence: Exchange Online — MX records public facing

Additional Notes — include anything else about your public facing infrastructure
that may be relevant for security monitoring or logging:
- Azure WAF is in detection mode only — not yet in prevention mode
- MyChart is managed entirely by Epic — no Sentinel visibility into portal activity

---

## Notes

### What We Are Looking For

Anything about your environment that is important for our security team to know
that was not captured in the sections above. Think specifically about the prompts
below. Short answers are fine — we just want to make sure nothing important gets
missed.

### Why This Matters for Security Monitoring

Every environment has unique context that doesn't fit neatly into a template.
Our engineers read this section carefully when first working in your environment
and it helps us avoid making wrong assumptions.

### Considerations

- Crown Jewels — what are the most critical systems, applications, or data in
  your environment? If something were compromised what would cause the most damage
  or disruption to your business? Knowing this helps us prioritize monitoring and
  ensure the right detections are in place
- Third Party Managed Services — do any outside companies manage parts of your
  technology environment on your behalf? For example a managed network provider,
  a company that manages your endpoints or servers, or another IT services company
  that owns or operates any part of your infrastructure. If yes please describe what
  they manage and who they are — this helps us know who to contact if we have
  questions about specific parts of your environment
- Anything That Would Surprise Us — is there anything unusual or unexpected about
  your environment that even your own IT team had to learn the hard way? Legacy
  systems that can't be touched, unusual configurations, known quirks, or anything
  else a security engineer should know before working in your environment

### Example

Crown Jewels:
- Epic EHR is the most critical application — patient data and clinic operations
  depend entirely on it. Any anomalous access or authentication failure related
  to Epic should be treated as high priority
- Azure Key Vault stores all Epic integration credentials — compromise of Key Vault
  would affect all clinic integrations

Third Party Managed Services:
- Network infrastructure at all 12 clinic locations is managed by Contoso IT
  Partners — they own the firewall configurations and syslog forwarding setup.
  Contact: network@contosoitpartners.com
- Azure environment is co-managed between internal IT and CloudOps Inc who handles
  DCR configuration and AMA agent deployment

Anything That Would Surprise Us:
- Legacy Windows Server 2008 machines still running at two clinic locations — cannot
  be updated due to Epic compatibility requirements, no agents can be installed
- Clinic locations 4 and 7 have intermittent internet connectivity — expect periodic
  gaps in log data from those sites that are not security related