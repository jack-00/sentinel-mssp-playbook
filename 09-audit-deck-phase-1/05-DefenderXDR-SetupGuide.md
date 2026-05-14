
## What Is the Defender XDR Data Connector

The Microsoft Defender XDR Data Connector is the single most important connection
between your Microsoft Defender security products and Microsoft Sentinel. It streams
security events, incidents, alerts, and advanced hunting data from your Defender
products directly into Sentinel where our team monitors and detects threats on your
behalf.

Without this connector configured correctly we may have significant blind spots in
our monitoring capability — security events happening in your Defender products would
not be visible to us in Sentinel.

---

## What It Connects

The connector brings together data from four Microsoft Defender products:

- **Microsoft Defender for Endpoint** — endpoint security events from your workstations
  and servers
- **Microsoft Defender for Identity** — identity threat detection from your Active
  Directory and Entra ID environment
- **Microsoft Defender for Office 365** — email, Teams, and SharePoint security events
- **Microsoft Defender for Cloud Apps** — cloud application activity and anomalies

---

## What Data It Sends to Sentinel

The connector populates the following tables in Microsoft Sentinel:

**Incidents and Alerts**
- SecurityIncident
- SecurityAlert
- AlertEvidence

**Endpoint — Defender for Endpoint**
- DeviceInfo
- DeviceEvents
- DeviceFileEvents
- DeviceImageLoadEvents
- DeviceLogonEvents
- DeviceNetworkEvents
- DeviceNetworkInfo
- DeviceProcessEvents
- DeviceRegistryEvents
- DeviceFileCertificateInfo

**Email — Defender for Office 365**
- EmailEvents
- EmailUrlInfo
- EmailAttachmentInfo
- EmailPostDeliveryEvents
- UrlClickEvents

**Identity — Defender for Identity**
- IdentityLogonEvents
- IdentityQueryEvents
- IdentityDirectoryEvents

**Cloud Apps — Defender for Cloud Apps**
- CloudAppEvents

---

## Important Notes Before You Begin

**Replaces standalone connectors**
When you enable the Defender XDR connector any previously connected standalone
Microsoft Defender connectors are automatically disconnected in the background.
They may continue to appear as connected but no data will flow through them.
This is expected behavior — the XDR connector replaces them all.

**E5 licensing and ingestion cost**
If your organization has Microsoft 365 E5 or equivalent licensing the ingestion
of Defender XDR advanced hunting tables into the Sentinel Analytics tier is
covered at no additional cost under the Microsoft security data grant. Note that
ingesting directly to the Sentinel Data Lake tier does not qualify for this free
grant — contact your Optiv engineer if you have questions about your licensing
and cost implications.

---

## How to Check If the Connector Is Already Configured

**In the Azure portal:**
1. Navigate to Microsoft Sentinel
2. Under Configuration select Data Connectors
3. Search for Microsoft Defender XDR
4. Check the status — Connected or Not Connected
5. If Connected select Open Connector Page to see what is enabled

**In the Microsoft Defender portal:**
1. Navigate to Microsoft Sentinel > Configuration > Data Connectors
2. Search for Microsoft Defender XDR
3. Check status and open the connector page to review configuration

---

## How to Configure the Connector

If the connector is not yet configured follow these steps:

**Step 1 — Install the Microsoft Defender XDR Solution**
1. In Microsoft Sentinel navigate to Content Hub
2. Search for Microsoft Defender XDR
3. Select Install
4. Wait for installation to complete

**Step 2 — Open the Connector Page**
1. Navigate to Configuration > Data Connectors
2. Search for Microsoft Defender XDR
3. Select Open Connector Page

**Step 3 — Connect Incidents and Alerts**
1. Check the box labeled Turn off all Microsoft incident creation rules for these
   products — this is recommended to avoid duplicate incidents
2. Select Connect incidents and alerts
3. Verify the connection shows as successful

**Step 4 — Connect Events**
Under the Connect events section you will see checkboxes for each Defender product.
Enable the event categories relevant to your licensed products:

- Defender for Endpoint Events — check if you have Defender for Endpoint licensed
- Defender for Identity Events — check if you have Defender for Identity licensed
- Defender for Office 365 Events — check if you have Defender for Office 365 licensed
- Defender for Cloud Apps Events — check if you have Defender for Cloud Apps licensed

Only enable categories for products you are licensed for. Enabling unlicensed
categories will not produce data and is not harmful but is unnecessary.

---

## What We Need From You

Once you have checked or configured the connector please reply in your ServiceNow
ticket with the following information:

**Connector Status**
Is the Microsoft Defender XDR connector currently installed and connected?
Yes / No / Not Sure

**Incidents and Alerts**
Is the incidents and alerts connection enabled?
Yes / No / Not Sure

**Events Enabled**
Which of the following event categories are currently enabled:
- Defender for Endpoint Events: Yes / No
- Defender for Identity Events: Yes / No
- Defender for Office 365 Events: Yes / No
- Defender for Cloud Apps Events: Yes / No

If the connector is not configured or you are unsure please let us know in the
ticket and we will open a separate ticket to work through the configuration with
you step by step.

---

## Questions

If you have any questions about this connector or are unsure about any of the
above please reply in your ServiceNow ticket and we will be happy to help.

---

## Reference

Microsoft official documentation:
https://learn.microsoft.com/en-us/azure/sentinel/connect-microsoft-365-defender