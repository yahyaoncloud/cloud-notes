# Azure Cloud Security Operations
### Azure Monitor • Microsoft Sentinel (SIEM/SOAR) • Microsoft Defender for Cloud

---

## 1. Microsoft Sentinel Architecture & KQL Analysis

Microsoft Sentinel is a cloud-native SIEM and Security Orchestration, Automation, and Response (SOAR) solution built on Azure Monitor Log Analytics.

```text
[Data Connectors] -> [Log Analytics Workspace (Tables)] -> [Analytics Rules (KQL)] -> [Incidents & Playbooks (Logic Apps)]
```

### Essential Kusto Query Language (KQL) Queries:

#### Failed Sign-in Spikes (Brute Force Detection):
```kusto
SigninLogs
| where TimeGenerated >= ago(24h)
| where ResultType != 0 // Non-zero indicates failure
| summarize FailedCount = count() by IPAddress, UserPrincipalName, bin(TimeGenerated, 15m)
| where FailedCount > 10
| order by FailedCount desc
```

#### Privileged Role Assignment Audit:
```kusto
AuditLogs
| where TimeGenerated >= ago(7d)
| where OperationName has "Add member to role"
| extend RoleName = tostring(TargetResources[0].modifiedProperties[1].newValue)
| extend TargetUser = tostring(TargetResources[0].userPrincipalName)
| extend Initiator = tostring(InitiatedBy.user.userPrincipalName)
| project TimeGenerated, Initiator, TargetUser, RoleName
```

---

## 2. Microsoft Defender for Cloud & CSPM

* **Cloud Security Posture Management (CSPM):** Secure Score assessment, compliance benchmark auditing (NIST, CIS, PCI-DSS, ISO 27001), automated regulatory remediation.
* **Cloud Workload Protection (CWPP):** Real-time endpoint, container (AKS), SQL database, and storage malware protection.
