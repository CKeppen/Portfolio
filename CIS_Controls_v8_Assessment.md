# CIS Controls v8 Assessment

Author: Cody Keppen | [LinkedIn](https://www.linkedin.com/in/cody-keppen-a09068355/) <br>
Date: 06/07/2026

# Preview and Summary of Exercise

This is a simulated CIS control assessment from my project, [Creating a Microsoft Work Environment with AD DS](https://github.com/CKeppen/Portfolio/blob/main/Creating%20a%20Microsoft%20Work%20Environment%20with%20AD%20DS.md), using the [CIS Controls v8](https://www.cisecurity.org/controls/v8) Framework. I only used the safeguards listed in the framework that I actually touched on in the project. So every item is not touched on for a full CIS Benchmark review.

When I did the original project, I caught some errors in the verification process that I noted here, as well as the fixes. However, I found an additional item that I did correctly in the project, but didn't quite meet the framework standards, this being password length.

It's not entirely detrimental, and frameworks are designed to be guidance. Not be all, end all. Though it is something for me to carry with me that if no MFA is used, the recommended password length is at least 14 characters long. 

The simulated Control Assessment documentation is below.
<br>
<br>

---
# CIS Controls v8 \- Control Assessment

**Assessor:** Cody Keppen
**Date:** 06/07/2026   
**System:** ClickIT Inc. \- Windows Server 2022 (AD DS, DNS) \+ Windows 10 Client  
**Reference:** [CIS Controls v8](https://www.cisecurity.org/controls/v8) | [Creating a Microsoft Work Environment with AD DS](https://github.com/CKeppen/Portfolio/blob/main/Creating%20a%20Microsoft%20Work%20Environment%20with%20AD%20DS.md)

This assessment maps the controls implemented during the referenced lab exercise above to the CIS Controls v8 framework. Only the safeguards directly addressed by the project are included. This is not a full organizational assessment of all 18 applicable safeguard controls. The findings column reflects the assessor's own determination based on a simulated internal audit.

---

## CIS Control 4 \- Secure Configuration of Enterprise Assets and Software

Establish and maintain the secure configuration of enterprise assets and software.

| Safeguard | Description | What Was Tested | Finding |
| :---- | :---- | :---- | :---- |
| **4.3 \- Configure Automatic Session Locking on Enterprise Assets** | Configure enterprise assets containing sensitive data to automatically lock the screen after a period of inactivity. | `ClickIT - Lockout Timer` configured screensaver timeout to 900 seconds with password protection. Client left idle at 9:38 AM; screen lock confirmed engaged at 9:53 AM. Credential entry required to regain access. | **Pass** |
| **4.7 \- Manage Default Accounts on Enterprise Assets and Software** | Manage default accounts on enterprise assets and software, such as root, administrator, and other pre-configured vendor accounts. | Default domain Administrator renamed to `DoAcct` (SID-500 confirmed). Local Administrator renamed to `LoAcct` via `ClickIT - Rename Accounts`. Guest account confirmed disabled. Honeypot Administrator account created with no SID-500, set to disabled. | **Pass** |
| **4.8 \- Uninstall or Disable Unnecessary Services on Enterprise Assets and Software** | Uninstall or disable unnecessary services on enterprise assets and software, such as an unused file sharing service, web application module, or service function. | Standard users blocked from CMD via `ClickIT - Restrict CMD Usage`. Standard users blocked from PowerShell via `ClickIT - Restrict PowerShell` (AppLocker). Standard users blocked from Control Panel via `ClickIT - Control Panel Access`. IT\_Admins confirmed access retained for all three. During CMD verification, PowerShell was found accessible to standard users. PowerShell restriction applied via AppLocker and re-verified. | **Fail \- PowerShell was accessible to standard users after CMD restriction was applied. AppLocker rules corrected; Standard user `KLycker` confirmed blocked, `cody_admin` confirmed access retained.** |

---

## CIS Control 5 \- Account Management

Use processes and tools to assign and manage authorization to credentials for user accounts, including administrator accounts, as well as service accounts, to enterprise assets and software.

| Safeguard | Description | What Was Tested | Finding |
| :---- | :---- | :---- | :---- |
| **5.1 \- Establish and Maintain an Inventory of Accounts** | Establish and maintain an inventory of all accounts managed in the enterprise. The inventory must include user, administrator, and service accounts. Validate that all active accounts are authorized on a recurring schedule. | Department OUs created: Executive, Finance, HR\_Legal, Operations, IT. User accounts provisioned with \[First Initial\]\[Last Name\] naming convention. Security groups verified: Management\_Access, Finance\_Access, HR\_Access, Legal\_Access, Operations\_Access, IT\_Admins. All accounts confirmed active and authorized. Security group All\_Employees was found to be missing user `Rkitt`. | **Fail \- `RKitt` was found missing from All\_Employees during onboarding verification. User added to group and membership re-verified.** |
| **5.2 \- Use Unique Passwords** | Use unique passwords for all enterprise assets. Best practice implementation includes, at a minimum, an 8-character password for accounts using MFA and a 14-character password for accounts not using MFA. | Default Domain Policy configured: minimum 12-character length, complexity enabled, history of last 24 passwords enforced. Tested against user `RKitt`: password reuse rejected, short password rejected, non-complex password rejected. No MFA in use. | **Fail \- Password minimum set to 12 characters. CIS guidance requires 14 characters for accounts not using MFA. Needs remediation.** |
| **5.4 \- Restrict Administrator Privileges to Dedicated Administrator Accounts** | Restrict administrator privileges to dedicated administrator accounts on enterprise assets. Conduct general computing activities, such as internet browsing, email, and productivity suite use, from the user's primary, non-privileged account. | Use of the shared domain Administrator account was stopped. `cody_admin` was created as a dedicated admin account added to Domain Admins. All privileged activity conducted as `cody_admin`. IT\_Admins security group used to scope privileged access; RDP, PowerShell, CMD, and Control Panel access restricted to this group only. | **Pass** |
| **5.6 \- Centralize Account Management** | Centralize account management through a directory or identity service. | All accounts managed through Active Directory on Windows Server 2022 acting as domain controller. Group Policy Objects applied centrally through Group Policy Management. | **Pass** |

---

## CIS Control 6 \- Access Control Management

Use processes and tools to create, assign, manage, and revoke access credentials and privileges for user, administrator, and service accounts for enterprise assets and software.

| Safeguard | Description | What Was Tested | Finding |
| :---- | :---- | :---- | :---- |
| **6.1 \- Establish an Access Granting Process** | Establish and follow a documented process, preferably automated, for granting access to enterprise assets upon new hire or role change of a user. | New employee onboarding tested with `RKitt`: added to Operations OU and Operations\_Access group. Initial password set with forced change at first logon. Access to Operations and Company Resources shared folders confirmed; Finance folder access correctly denied. | **Pass** |
| **6.2 \- Establish an Access Revoking Process** | Establish and follow a process, preferably automated, for revoking access to enterprise assets, through disabling accounts immediately upon termination, rights revocation, or role change of a user. | Department transfer tested with `KLycker`: removed from Operations\_Access, added to Finance\_Access, moved to Finance OU. Access to Operations folder correctly denied post-transfer; Finance folder access confirmed. | **Pass** |
| **6.7 \- Centralize Access Control** | Centralize access control for all enterprise assets through a directory service or SSO provider, where supported. | All access controlled through Active Directory security groups and GPOs applied at the domain level. Share and NTFS permissions enforced via group membership. No local account exceptions. | **Pass** |
| **6.8 \- Define and Maintain Role-Based Access Control** | Define and maintain role-based access control, through determining and documenting the access rights necessary for each role within the enterprise to successfully carry out its assigned duties. | Security groups mapped to roles: IT\_Admins, Finance\_Access, HR\_Access, Legal\_Access, Operations\_Access, Management\_Access, All\_Employees. Shared folder NTFS permissions enforced per group. RDP scoped to IT\_Admins via Restricted Groups GPO. | **Pass** |

---

## CIS Control 8 \- Audit Log Management

Collect, alert, review, and retain audit logs of events that could help detect, understand, or recover from an attack.

| Safeguard | Description | What Was Tested | Finding |
| :---- | :---- | :---- | :---- |
| **8.2 \- Collect Audit Logs** | Collect audit logs. Ensure that logging, per the enterprise's audit log management process, has been enabled across enterprise assets. | Audit Policy configured in `Default Domain Policy`: Audit Account Logon Events, Audit Account Management, Audit Logon Events, and Audit Policy Change all set to Success and Failure. Policy applied centrally via GPO. Logging confirmed enabled on both Server and Client. Event IDs captured: `4625 (Failed Logon)` on Client, `4740 (Account Lockout)` on Server, `4724 (Password Reset by Admin)` on Server, `4767 (Account Unlock)` on Server. | **Pass** |
| **8.5 \- Collect Detailed Audit Logs** | Configure detailed audit logging for enterprise assets containing sensitive data. Include event source, date, username, timestamp, source addresses, destination addresses, and other useful elements that could assist in a forensic investigation. | Pre- and post-policy audit state compared using `auditpol /get /category:*` via `Invoke-Command`. Expanded event capture confirmed after GPO push. Events retrieved via PowerShell using `Get-WinEvent` filtered by Event ID with timestamp and message detail. | **Pass** |
| **8.11 \- Conduct Audit Log Reviews** | Conduct reviews of audit logs to detect anomalies or abnormal events that could indicate a potential threat. Conduct reviews on a weekly, or more frequent, basis. | Event Viewer reviewed on Server and Client. Events filtered by ID and reviewed for account lockout, password reset, and failed logon activity. PowerShell queries used to cross-reference Server and Client logs across the same timeframe. Review conducted as part of the exercise build; process established for recurring weekly reviews going forward. | **Pass** |

---

*Safeguard descriptions sourced from the [CIS Controls Assessment Specification v8.1](https://cas.docs.cisecurity.org/en/latest/).*  
