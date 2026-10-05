# Microsoft Entra ID, Azure RBAC & Governance

---

## 1. Microsoft Entra ID (formerly Azure Active Directory) vs Windows Server AD

| Feature | Microsoft Entra ID | Windows Server Active Directory (AD DS) |
| :--- | :--- | :--- |
| **Protocol** | Web protocols: OAuth 2.0, OpenID Connect, SAML, WS-Fed. | Kerberos, NTLM, LDAP, DNS. |
| **Structure** | Flat structure (Tenants, Users, Groups, App Registrations). | Hierarchical (Forest, Trees, Domains, Organizational Units). |
| **Device Management**| Intune, Entra ID Registered, Entra ID Joined. | Group Policy Objects (GPOs), Domain Join. |

---

## 2. Azure Role-Based Access Control (RBAC) Scopes & Roles

```text
Management Groups -> Subscriptions -> Resource Groups -> Individual Resources
```

### Built-in Core Roles:
* **Owner:** Full access to all resources, including permission to delegate access to other principals.
* **Contributor:** Full access to create and manage all resources, but **cannot** grant permissions to others.
* **Reader:** View-only access to existing resources.
* **User Access Administrator:** Manage user access to Azure resources without granting resource control.

---

## 3. Privileged Identity Management (PIM) & Conditional Access

* **PIM:** Just-In-Time (JIT) role activation with time-bound eligibility (e.g. 8 hours max), mandatory MFA, and approval workflows.
* **Conditional Access:** "If-Then" engine evaluating Signals (User risk, IP location, Device compliance) &rarr; Decisions (Require MFA, Block, Require compliant device).
