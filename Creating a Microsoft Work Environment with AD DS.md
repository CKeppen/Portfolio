By: Cody Keppen [LinkedIn Profile](https://www.linkedin.com/in/cody-keppen-a09068355/)
Last update: 05/20/2026

---
# Table of Contents
- [Preview of Exercise](#preview-of-exercise)
- [Concepts Demonstrated](#concepts-demonstrated) 
- [Instructional Steps](#instructional-steps)
- [Verification Steps](#verification-steps)
- [Lessons Learned](#lessons-learned)
- [Summary](#summary)
- [Resources](#resources)
---

# Preview of Exercise

Using the [MS Home Lab VM Setup VM](https://github.com/CKeppen/Portfolio/blob/main/MS%20Home%20Lab%20VM%20Setup.md) enterprise work environment, this demonstration is the creation of a fictional Windows-centric, on-prem organization.

The VM's used will be a Windows 2022 Server ("Server") acting as the DC with AD DS and DNS, a Windows 10 Client ("Client") acting as the workstation for users.

On the Server, AD DS is used to create Organization Units (OU) and employee user accounts. GPOs and system hardening are configured from an IT admin perspective.

The Client will be used by the "Users" acting like regular employees. Demonstrating the success of the environment setup.

#  Concepts Demonstrated

1. [Creation of Multiple Organization Units and Users](#creating-the-organization-units-and-users)
2. [Security Group Creation](#security-group-creation)
	1. [IT Admin Group Permissions](#rdp-access-and-admin-hardening)
		1. [RDP Access](#rdp-access)
		2. [Domain Admin User Management](#domain-admin-user-management)
		3. [WinRM Management](#winrm-management)
	2. [Shared Folder Permissions](#share-folder-permissions)
3. [Group Policy Management](#group-policy-management)
	1. [Login Banner](#login-banner)
	2. [Disable Auto Updates](#disable-auto-updates)
	3. [Desktop Wallpaper Enforcement](#desktop-wallpaper-enforcement)
		1. [Shared Folder Creation](#shared-folder-creation)
	4. [CMD Restriction](#cmd-restriction)
	5. [PowerShell Restriction](#powershell-restriction)
	6. [Control Panel Access](#control-panel-access)
	7. [Idle Timeout](#idle-timeout)
4. User Account Management
	1. [Initial Login Settings](#initial-login-settings)
	2. [Account Unlock](#account-unlock)
	3. [Password Reset](#password-reset)
	4. [Department Moves](#department-moves)
	5. [New Employee Onboarding](#new-employee-onboarding)
5. Hardening Settings
	1. [NLA Authentication for RDP](#rdp-access)
	2. [Password Hardening](#password-hardening)
	3. [Disabling Accounts](#disable-accounts)
	4. [Domain Admin Accounts](#domain-admin-name-change)
	5. [Rename Local Admin GPO](#rename-local-admin-gpo)
	6. [Create Honeypot Administrator](#domain-administrator-honeypot)
	7. [Login Attempt Limits](#login-attempt-limits)
	8. [Event Viewer Use](#event-viewer-use)
	9. [Audit Policy Settings](#audit-policy-settings)

---

# Instructional Steps

These are the technical steps taken during this demonstration. There will be links throughout this section to the [Lessons Learned](#lessons-learned) section to notate obstacles and solutions during the demonstration. 

Also, there is a [Verification](#verification) section that focuses solely on the verification of the concepts demonstrated, without the technical details. This can be viewed if only the final product of this demonstration wants to be reviewed.

## Setting up the Corporate Environment

In the Server, I open the Server Manager and go to Tools. Then Active Directory Users and Computers.

![](Attachments/Pasted%20image%2020260513171735.png)

The company name I decided on was "ClickIT Inc.". I don't know exactly what they do, but someone is gonna click something they shouldn't. 

On the `cyberlab.local` domain, I right click and go to  "New", then click "Organizational Unit". Giving it the name, `ClickIT`. This Organizational Unit (OU) will be where the fake corporate environment sits.

![](Attachments/Pasted%20image%2020260505171359.png)

## Creating the Organization Units and Users

 I've decided on the following company structure. 

- Executive
- Finance
- HR_Legal
- Operations
- IT
- Workstations

For each department, I will make sub organizational units (sub-OU) under the  "ClickIT" OU. I do this by selecting the newly created OU, and once again finding "New" and clicking "Organizational Unit". I also make a sub-OU for Workstations that the Windows Client will join. (After right clicking "ClickIT", a fast way to create these sub-OUs is pressing, "N", then "O" to create the new OU.)

![](Attachments/Pasted%20image%2020260505171445.png)

Each department will now have employees populated into it. I used fake First and Last Names. Their Middle Initial will hint at their role to help me remember who does what. The User Logon Name has the format of `[First Initial][Last Name]`.

For the department the employee goes under, I select the department, then "New" then "User".

![](Attachments/Pasted%20image%2020260505171923.png)

![](Attachments/Pasted%20image%2020260505172047.png)

### Initial Login Settings

Their initial password is set to `Welcome123!` with the setting "User must change password at next logon" checked. This is to allow a user to first login with a default password, then change their password to a unique one they can remember. Hopefully a strong one too. 

![](Attachments/Pasted%20image%2020260505174537.png)

Once the password is set, a review window is shown. After confirming details are correct, click "Finish".

Below is the company structure of the newly created company, ClickIT Inc.

| **Department (OU)** | **First Name** | **Middle (Role)** | **Last Name** | **User Logon Name** |
| ------------------- | -------------- | ----------------- | ------------- | ------------------- |
| **Executive**       | Donny          | CEO               | Phlips        | `DPhlips`           |
| **Finance**         | Penny          | Acct              | Banks         | `PBanks`            |
| **HR_Legal**        | Jan            | HR                | Jolly         | `JJolly`            |
| **HR_Legal**        | Larry          | Legal             | Lawless       | `LLawless`          |
| **Operations**      | Carl           | Mgr               | Wright        | `CWright`           |
| **Operations**      | Alice          | Staff             | Cypher        | `ACypher`           |
| **Operations**      | Bob            | Staff             | Token         | `BToken`            |
| **Operations**      | Karen          | Staff             | Lycker        | `KLycker`           |
| **IT**              | Cody           | IT                | Admin         | `cody_admin`        |

Here is an example of the "Operations" department.

![](Attachments/Pasted%20image%2020260505173749.png)

## Security Group Creation

Now I'll create permission Groups for the organization. While an employee is allocated to a sub-OU, they can have more than one group associated to them. Giving them different permissions than others. For example, both HR and Legal are in the same OU, but Legal shouldn't see employee records like HR. So different groups will be created and users assigned to them, granting different permissions.

I'll start by creating a Group sub-OU to store the groups. I've decided on the following group structure.

- All_Employees - everyone
- Management_Access
	- Donny Phlips (CEO)
	- Jan Jolly (HR)
	- Carl Wright (Mgr)
- Finance_Access
	- Penny Banks (Accountant)
- HR_Access
	- Jan Jolly (HR)
- Legal_Access
	- Larry Lawless (Legal)
- Operations_Access
	- Carl Wright (Mgr)
	- Alice Cypher (Staff)
	- Bob Token (Staff)
	- Karen Lycker (Staff)
- IT_Admins
	- Cody Admin (IT)

Here I am making the All_Employees group.

For Group Scope, I leave it on the default of "Global", as I only have one domain. "Security" is checked to allow permissions to be set for this group.

![](Attachments/Pasted%20image%2020260505205341.png)

Here are all the groups.

![](Attachments/Pasted%20image%2020260505210012.png)

Each group I'll double click to bring up the properties window, then clicking "Add" to add the employee that belongs to that group. I use their username, like `DPhlips`. Then after adding them, click "Check Names" to allow the system to find the actual user. 

Though there are multiple ways to use this, as it's just a search function.

![](Attachments/Pasted%20image%2020260505210756.png)

Once I click "OK", I see all the names in the Members section for the "All_Employees" group.

![](Attachments/Pasted%20image%2020260505210936.png)

## Group Policy Management

Now that all the Groups are created, I'll use the Group Policy Management (GPM), creating Group Policy Objects (GPO). This will enforce  different policy settings on the users and workstations.

These are the GPOs settings I'll create.

- Login Banner
- Disable Auto Updates
- Desktop Wallpaper Enforcement 
- CMD Restriction (except IT)
- PowerShell Restriction (except IT)
- Restrict Control Panel Access
- Idle Timeout
- Enable RDP
- Rename Accounts

To start, I'll open the GPM from the Server Manager.

![](Attachments/Pasted%20image%2020260520212943.png)

Then I'll go to the "cyberlab" domain, find the "ClickIT" sub-domain. Right clicking and choosing, "Create a GPO in this domain, and Link it here...".

![](Attachments/Pasted%20image%2020260505213132.png)

After that, I will create a name then edit the GPO for its specific purpose. This will be repeated for all the GPOs created.

### Login Banner

A Login Banner is a legal warning message that pops up at the login screen before a user attempts to login. Usually similar to, "If you are not authorized to access this machine, do not attempt to login."

Here I am creating the new GPO for the Login Banner, calling it `ClickIT - Login Banner`.

![](Attachments/Pasted%20image%2020260505213232.png)

Once created, I right click the GPO then select "Edit".

![](Attachments/Pasted%20image%2020260505213431.png)

Here I'll navigate to "Security Options", finding the two "Interactive Logon" options needed for the login banner, "Message text" and "Message title".

The path to these policy settings are:
**Computer configuration > Policies > Windows Settings > Security Settings > Local Policies > Security Options**

"Interactive logon: Message title..." is the title of the popup. 
"Interactive logon: Message text..." is the body of the message.

![](Attachments/Pasted%20image%2020260505214116.png)

Here is the banner message settings.

![](Attachments/Pasted%20image%2020260505214243.png)
![](Attachments/Pasted%20image%2020260505214324.png)

Now to test! I'll need to log on to the Windows Client to test this...
Welp. An unexpected update slowed me down.

![](Attachments/Pasted%20image%2020260505215139.png)

I'm now adding to my list of GPOs to disabling automatic updates to prevent me sitting around in the future. Back to the Login Banner after the update finishes.

I log in as `cody_admin`, then open up Command Prompt "CMD". There I force the new GPO update with the following command, to push the new GPO settings to the effected users and workstations. Which will be repeated throughout the demonstration.

```shell
gpupdate /force
```

The CMD terminal provides an output of a successful policy update.

![](Attachments/Pasted%20image%2020260505215915.png)

Here is the moment of truth. 

When I attempt to log back in, I am met with the following Login Banner I created. Success!

![](Attachments/Pasted%20image%2020260505215657.png)

Now I want to disable auto updates for the workstations. Coming from a production management background, having auto updates during production hours can do more harm than go. And I felt it earlier with the unexpected update when I was in a groove. 

### Disable Auto Updates

Here I'll be making a GPO to disable auto updates, allowing me to control when the update takes place. Possibly even testing updates in a sandbox first to ensure an update will not break anything first. Or test exploits recently patched, but having an outdated version. As these are common reasons to implement this.

This time I'll make the GPO in the "Workstations" OU. Called `ClickIT - Disable Auto Update`. That way the policy settings take place on the workstations in that OU.

![](Attachments/Pasted%20image%2020260506152322.png)

Once created, the path I take to get to the auto update is as follows:
**Computer Configuration > Policies > Administrative Templates: Policy.. > Windows Components > Windows Update**

![](Attachments/Pasted%20image%2020260506153810.png)

Now I'll open the settings and apply the following. Choosing "Disabled" to allow full control on of updates by the admin team. As well as a comment for future information.

![](Attachments/Pasted%20image%2020260506154319.png)

Again, to test I log onto the Windows Client as `cody_admin`, then do a force a `gpupdate`. 

Once the update is pushed, I'll confirm by going to the Windows Update settings. Where I find a notification that the Windows updates are managed by the organization. Success!

![](Attachments/Pasted%20image%2020260506155221.png)

### Resource Issues

Here I started running into the limitations of my home lab. I do plan to switch from my Home Server to my ThinkPad T14 as the VM's were running too slow, and I was getting kicked out of Chrome Remote. For now, I drop the RAM by one for each VM. I go over this more in the [Host Limitations](#host-limitations) section of Lessons Learned. 

After I drop the RAM and restart the VM's, I continue.

### Desktop Wallpaper Enforcement and Shared Folder Creation

Here, I'll make the first Shared Folder for the Wallpaper Enforcement policy. This is a common practice at companies to display company logos, policies, key values, etc.

#### Shared Folder Creation

For me to set the Desktop Wallpaper Enforcement, I need to also create a shared folder on the Server so that all devices on the domain can access the wallpaper to be used.

On the Server, I make a new folder, `C:\Shared\Wallpapers`. Then go into the properties of that folder.

![](Attachments/Pasted%20image%2020260506171206.png)

Then I'll go to the "Sharing" tab to select "Advanced Sharing..."

![](Attachments/Pasted%20image%2020260506171402.png)

In the "Advanced Sharing" window, I check the "Share this folder" box. Which populates `Wallpapers` in the share name box. Then I click "Permissions" to verify user permissions for this folder.

![](Attachments/Pasted%20image%2020260506171540.png)

I see from the below that all users will have "Read" access, allowing them access to the image I choose as their background.

![](Attachments/Pasted%20image%2020260506171624.png)

In an effort to really push security into the culture of the organization, a NSA "Do Your Part" wallpaper is used.

![](Attachments/Pasted%20image%2020260506171741.png)

Now to test if the file share is working. Logging back into the Windows Client, I look into the "Network" folder, but find nothing. 

Going back to the Server "Network" folder, I see this error.

![](Attachments/Pasted%20image%2020260506172613.png)

Network Discovery is off. So I'll turn this on to see if this solves the issue.

![](Attachments/Pasted%20image%2020260506172735.png)

I can now see the Windows Client. Time to check if the Client can see the Server.

![](Attachments/Pasted%20image%2020260506172823.png)

On the Client, I see the same issue now with the Network discovery being off. When I choose to turn it on, a User Account Control pops up asking for admin credentials. Which I use the domain admin to enter.

![](Attachments/Pasted%20image%2020260506173008.png)

Which, after I do, Success! I can see the "Wallpapers" shared folder from the Server on the Client.

![](Attachments/Pasted%20image%2020260506173936.png)

Here is the wallpaper in the folder.

![](Attachments/Pasted%20image%2020260506173950.png)

#### Desktop Wallpaper Enforcement

With the Shared wallpaper folder now created, I'll go back into the GPM for the User Configuration on the Server.

The path for the settings are:
**User Configuration > Policies > Administrative Templates: ... > Desktop > Desktop**

Then I'll configure the "Desktop Wallpaper" setting.

![](Attachments/Pasted%20image%2020260506174549.png)

From there, I'll enter the UNC path of the shared folder and the image. Then set to Fill.

![](Attachments/Pasted%20image%2020260506175109.png)

Now to enforce the new GPO and test on the Windows Client.

So far, it is a black screen, not the normal Windows background. And I don't see the shared Wallpaper folder anymore.

![](Attachments/Pasted%20image%2020260506175541.png)

I'll go to search for the folder with the UNC and find it in the drop down already.

![](Attachments/Pasted%20image%2020260506175649.png)

Selecting it connects the folder once again, but I don't see the background. So I might have done something wrong with the path name.

![](Attachments/Pasted%20image%2020260506175718.png)

Back on the server, I do see I have `.jpeg` not `.jpg`. Even though the file type is "JPEG". I'll try this.

![](Attachments/Pasted%20image%2020260506180050.png)

Same situation. Though, I confirmed here that it is a `.jpg` file.

![](Attachments/Pasted%20image%2020260506180714.png)

![](Attachments/Pasted%20image%2020260506180839.png)

This time I'll try without the underscore.

![](Attachments/Pasted%20image%2020260506180928.png)

Back on the Client, I tested opening the file with the exact address.

First with the underscore, which gave me an error.

![](Attachments/Pasted%20image%2020260506181057.png)

Then with a space, and opened the image! Which is promising.

![](Attachments/Pasted%20image%2020260506181134.png)

Success!

I see the wallpaper now on the desktop. The scaling and position isn't exact, but I don't want to spend too much time adjusting for this demonstration. As it doesn't look too bad in the end. It's right in the users face and a reminder that the End User is the most vulnerable in the organization.

![](Attachments/Pasted%20image%2020260506181304.png)

### CMD Restriction

Now I'll set a GPO to restrict users from using the command prompt (CMD). Only those in the IT Group will have access to this. CMD is a powerful app that you'll want to limit access to.

I'll go to the GPM console and go to the "Prevent access to the command prompt" settings.

The path is as follows:
**User Configuration > Policies > Administrative Template:.. > System**

![](Attachments/Pasted%20image%2020260520213209.png)

From here, I'll "Enable" this setting.

![](Attachments/Pasted%20image%2020260506182844.png)

Now I need to filter this GPO so it applies restrictions to all users, except for IT members. Allowing them to continue to use CMD.

In the GPM, I select "ClickIT - Restrict CMD Usage" on the left panel. Towards the bottom of this menu is the "Security Filtering" menu. Currently, it only has "Authenticated Users".

![](Attachments/Pasted%20image%2020260506183038.png)

I select the "Delegation" tab at the top, then "Add". Inserting the `IT_Admins` Group.

![](Attachments/Pasted%20image%2020260506183242.png)

The default will be "Read" for the group, but I'll change it to "Deny". It might be a little backward at first, but this is a "Restriction" policy. By using a "Deny" on a "Restriction" policy, it becomes an exemption to the restriction for the  `IT_Admin` group. Allowing them to continue to use CMD.

Now I select, "Advanced" on the bottom right. In the new "Security Settings" window, I select `IT_Admins`. Below is permissions for this group.

I leave "Read" on "Allow", but go to "Apply group policy" and select "Deny".

![](Attachments/Pasted%20image%2020260506184344.png)

After the policy update is pushed, it is the time to test. I'll be logging in as one of the staffers for the first time to test restriction.

While logging into Karen Lycker (`KLycker`) for the first time, I am prompted to change the password. Success for that setting!

![](Attachments/Pasted%20image%2020260506184840.png)

`KLycker` sets what we all hope is a strong password. And once again, the "Do Your Part" wallpaper is displayed.

Now to test if Karen can open the command prompt.

The command prompt does open, but it displays the message "The command prompt was disabled by your administrator." Success!

![](Attachments/Pasted%20image%2020260506203103.png)

Now to log back into `cody_admin` to test if CMD is still accessible. Success!

![](Attachments/Pasted%20image%2020260506202932.png)

### Restrict PowerShell

Here I'll be creating two GPOs to use AppLocker to restrict the use of PowerShell like I just did above for CMD. IT admins will still have access to using PowerShell. Like CMD, this is not something you'll want normal employees to have access to.

One GPO is created for the Application Identity service to run automatically at computer start.

The other GPO is created for the AppLocker policies to restrict PowerShell.

#### Enable AppIDSvc

I create a GPO policy named `ClickIT - EnableAppIDSvc`. This is to enable the Application Identity service to launch when the computer starts.

![](Attachments/Pasted%20image%2020260514134747.png)

Then I'll get to the "System Service" folder in the GPM.

The path is:
**Computer Configuration > Policies > Windows Settings > Security Settings > System Services**

Selecting the "Application Identity" service and going to settings, I turn "Automatic" startup mode on.

![](Attachments/Pasted%20image%2020260515194233.png)

#### AppLocker PowerShell Restriction

Now I'll make a new GPO, `ClickIT - Restrict PowerShell`.

![](Attachments/Pasted%20image%2020260515194939.png)

Back in the GPM, I go to to "AppLocker" to define the rules needed.

Path:
**Computer Configuration > Policies > Window Settings > Security Settings > Application Control Policies > AppLocker**

Selecting "Executable Rules", I see a blank window with no current rules. Right-clicking "Executable Rules" brings up the "Create Default Rules" option, which will create the default rules that will not block everything. Which I'll do.

![](Attachments/Pasted%20image%2020260515195624.png)

Now you can see the three new rules created. These allow everyone to run anything in Windows, and Program Files. Along with Administrators allowed to run anything everywhere.

![](Attachments/Pasted%20image%2020260520193253.png)

After that, I'll right click the "All files located in the Windows folder" rule that was just created and select "Properties".

Going to the "Exceptions" tab at the top, I change the drop down from "Publisher" to "Path". Then select "Add" to start adding the exception rules.

![](Attachments/Pasted%20image%2020260519114337.png)

In the "Path Exception" window, I select "Browse Files..." then find the PowerShell applications to be exempt from this "Allow" rule. 

![](Attachments/Pasted%20image%2020260519114549.png)

Meaning, these are not allowed for `Everyone`, as that is what this default rule was originally for. (It loses the "Default Rules" in its name after this setting.) PowerShell is now exempted from the "Allow" rule.

![](Attachments/Pasted%20image%2020260519114805.png)

Now two more rules are needed to allow the `IT_admins` to use PS.

Right clicking on the rules panel, I select "Create New Rule..." to get started.

![](Attachments/Pasted%20image%2020260519115005.png)

This will be an "Allow rule" for the `IT_Admins`.

![](Attachments/Pasted%20image%2020260519115130.png)

In the next page, I'll select "Path", then "Next".

![](Attachments/Pasted%20image%2020260519115159.png)

Then choosing, "Browse Files...", I go to the PowerShell application to add to this rule. Skipping the "Exceptions" portion.

![](Attachments/Pasted%20image%2020260519115256.png)

I then give it the name `Allow PS for IT_Admins` with the file path as the description, `%SYSTEM32%\WindowsPowerShell\v1.0\powershell.exe`

![](Attachments/Pasted%20image%2020260519115452.png)

I did this for both PowerShell and PowerShell ISE (Integrated Scripting Environment).

Next is creating the "Package app Rules" default to allow the apps that come with Windows to be accessible still. Like the Start menu, Search and Store, as an example. 

![](Attachments/Pasted%20image%2020260517191203.png)

Here is the default rule.

![](Attachments/Pasted%20image%2020260519115754.png)

Now I need to right-click "AppLocker", then select " Properties" to enforce the "Executable Rules" and "Packaged app Rules" I just set.

![](Attachments/Pasted%20image%2020260515203507.png)

From the properties window, I select "Executable Rules", check the "Configured" box, then "Enforce rules", in the drop down. The same for "Package app Rules". This is to enforce the new rules I just created in the "Executable rules" and "Packaged app Rules" categories.

![](Attachments/Pasted%20image%2020260517191511.png)

I push the GPO update to the client and verify "AppIDSvc" is running with the following command. (This was done after I setup RDP in the later sections. Which is why I am able to use `Invoke-Command`.)

```PowerShell
Invoke-Command -ComputerName DESKTOP-MBV8D69 -ScriptBlock{Get-Service AppIDSvc}
```

![](Attachments/Pasted%20image%2020260515204023.png)

When I sign in as the user `KLycker` and attempt to open PowerShell, I get a "app has been blocked by your administrator" message. Success!

![](Attachments/Pasted%20image%2020260515204458.png)

When I sign in as `cody_admin`, I am able to use PowerShell. Success!

![](Attachments/Pasted%20image%2020260519121411.png)

(This was added later as a security gap discovered during the [CMD Restriction](#cmd-restriction) verification process. I talk about the discovery here, [PowerShell Security Gap](#powershell-security-gap). I ran into a couple different problems that required some research and learning on my end.)

### Control Panel Access

In an effort to not allow employees to modify too much on their machines, we'll also implement a GPO to restrict access much like the CMD restriction we just created.

In the GPM, I go along the following path:
**User Configuration > Policies > Administrative Template:.. > Control Panel**

Then select the "Prohibit access to Control Panel and PC Settings" setting.

![](Attachments/Pasted%20image%2020260506203716.png)

Select "Enabled" and hit "OK".

![](Attachments/Pasted%20image%2020260506203555.png)

With the Control Panel Access GPO selected in the left panel, I'll go to the "Delegation" tab, then click "Advanced".

![](Attachments/Pasted%20image%2020260506204105.png)

With the `IT_Admins` added to the "Delegation" list, I'll go to the advanced settings to select "Deny" for "Apply group policy".

![](Attachments/Pasted%20image%2020260506204320.png)

Now I'll log onto `cody_admin` on the Client to do a GPO update, then test between `cody_admin` and `KLycker`.

While on `KLycker`, I was able to open the Control Panel. So not a success.

![](Attachments/Pasted%20image%2020260506204934.png)

In an effort to be faster, I was choosing, "Switch User" when switching between `cody_admin` and `KLycker`. I'm going to try logging completely off this time as I don't think the settings were fully enforced.

After logging out, then back in, Success!

![](Attachments/Pasted%20image%2020260506205103.png)

Here is the difference between "Switch user" and "Sign out" in [Logout Difference](#logout-difference) in Lessons Learned.

After logging out and then into `cody_admin`, I was able to successfully open the Control Panel with no errors. Success!

![](Attachments/Pasted%20image%2020260506205745.png)

### Idle Timeout

Now I'll set the screensaver timeout settings. This will automatically lock a computer screen after a predetermined amount of inactivity time on the device is reached.

The idea is to enhance security on a device by not allowing it to be endlessly available in case a user is away from the machine for a long period of time. Forcing the user to need to sign back in after it locks on its own.

I'll name the GPO `ClickIT - Lockout Timer`. For this, I'll need multiple settings enabled. The path is the following in the GPM,

The path:
**User Configuration > Policies > Administrative Templates:.. > Control Panel > Personalization**

The settings,
- Enable Screensaver - Enabled
- Password protect the screensaver - Enabled
- Screen saver timeout - Enable, 900 seconds (15 minutes)

![](Attachments/Pasted%20image%2020260506210744.png)

Now to test. (I did drop the timer to 15 seconds for testing so I didn't sit for 15 minutes.)

Failure.

Something is occurring that is disconnecting my mouse and keyboard inputs to Chrome Remote Desktop, which I've been using to access my home server with the VM's from my laptop.

I go into more details here, [Switch to RDP](#switch-to-rdp), but in the end, I decided to go with TailScale on the Windows 2022 server, to RDP into it from the laptop. With the intent to RDP from the Server, into the Client. Which means, I needed to do a RDP setup.

## RDP Access and Admin Hardening

Now I'll get the `IT_Admin` group setup with RDP to allow `cody_admin` to RDP into the Client. This should save some time from me switching between VM's, plus I already have the Client running headless in the background.

This really all comes from the CRD issues I've been having and wanting to prove the idle screen lockout works. But I need to RDP into the workstation first.

### Domain Admin User Management

First I need to start doing some better security practice. I've been logging into the Server as the domain `Administrator`. This gives me all the accesses needed, but really, for better record keeping I should be logging in as `cody_admin`.

Imagine trying to see who did what when three different IT admins use the same `Administrator` login. Or just allowing users with lots of privileges on at all times.

With me using `cody_admin` exclusively now, I'll need to add the account to the "Domain Admins".

I'll go to the "AD Users and Computers Window ". Find the "IT" OU, then select `Cody IT. Admin` to open the properties.

Once this is opened, I'll go to "Member Of", then click "Add".

![](Attachments/Pasted%20image%2020260507195019.png)

I then sign out as the `Administrator`, to log back in as `cody_admin`.

Which led to the other discovery with Tailscale. You need to setup Tailscale to stay up and running when all users log out in the settings. If not, it'll shut down and you lose connection. Which I went over in the Lessons Learned section.

### RDP Access

Now that I'm logged back in as `cody_admin` on the Server, I'm going to create a GPO for `IT_Admins` to RDP.

I'll call it `Enable RDP`.

The GPM path is lengthy:
**Computer Configuration > Policies > Administrative Templates:.. > Windows Components > Remote Desktop Services > Remote Desktop Session Host > Connections.**

Finding "Allow users to connect remotely..." and Enabling it.

![](Attachments/Pasted%20image%2020260507202914.png)

I'm also going to go to the **Security** folder in the same **Remote Desktop Session Host** folder to Enable the NLA Authentication.

NLA is "Network Level Authentication". The concept is that credential authentication is needed upfront to even see the login screen. Not allowing a threat actor to make a connection, see the login screen, and try to manipulate the machine or resources in any way.

![](Attachments/Pasted%20image%2020260507203649.png)

Now to force the `IT_Admins` group to always be in a computer's "Remote Desktop User" group, using "Restrictive Groups".

It sounds a little counter-intuitive, but a "Restrictive Group" is a feature to control the groups on a local machine. By adding `IT_Admins` to the "Remote Desktop Users" group of every workstation, you are ensuring an updated member list of IT admins is pushed to the local machine to allow those members to use RDP.

![](Attachments/Pasted%20image%2020260507204614.png)

With this new GPO setup, I'm going to try using PowerShell on the Server as `cody_admin` to do the same `gpupdate` I've been doing. Instead of logging into the workstation itself to do the same command, I can do the GPO update for the Client while logged into the Server.

I'll use the following,

```PowerShell
Invoke-Command -ComputerName DESKTOP-MBV8D69 -ScriptBlock {gpupdate /force}
```

Unfortunately, I got an error for `WINRM` not being on for the Client. So the GPO update didn't go through.

![](Attachments/Pasted%20image%2020260507210107.png)

I'm going to try to log in with RDP anyways to finish the troubleshooting for the Idle Lock Screen, by using RDP.

Success! I was able to connect remotely. Though I need to kick the Administrator out that was still logged in. 

![](Attachments/Pasted%20image%2020260507210137.png)

I'm going to do the GPO update just to be safe once logged in.

![](Attachments/Pasted%20image%2020260507210407.png)

### Idle Lock Timeout Fix

With the remote connection issue fixed, I'm going to set the idle lockout timer back to 15 seconds, do another GPO update, then log back in and wait for the lockout screen.

I've logged in at around this time shown below at `21:15:11`

![](Attachments/Pasted%20image%2020260507211543.png)

Then, while typing this up, the screen locks a little before `21:16:06`.

![](Attachments/Pasted%20image%2020260507211624.png)

Success for the idle time! (I turn this back to 900 seconds for 15 minutes for idle time afterwards.)

### NLA Authentication for RDP

I also verify that NLA settings is set to required by going to settings, Remote Desktop, then selecting, "Advanced Settings".

![](Attachments/Pasted%20image%2020260519142202.png)

Another success!

![](Attachments/Pasted%20image%2020260507212012.png)

### WinRM Management

I went back to fix the WinRM afterwards by adding "Allow remote server management.." to the `ClickIT - Enable RDP` GPO.

The path was:
**Computer Configuration > Policies > Administrative Templates:.. > Windows Components > Windows Remote Management (WinRM) > WinRM Service**

![](Attachments/Pasted%20image%2020260507212610.png)

And enabled the "Windows Remote Management", "Inbound Rules".

The path here:
**Computer Configuration > Policies > Windows Settings > Security Settings > Windows Defender Firewall.. > Windows Defender Firewall.. > Inbound Rules**

Right Clicking on "Inbound Rule" will bring up a Rule Wizard window. Selecting "Predefined:", then using the drop down to find "Windows Defender Firewall Remote Management".

![](Attachments/Pasted%20image%2020260511115623.png)

After selecting "Next", I'll choose the two pre-built rules,

![](Attachments/Pasted%20image%2020260511115758.png)

Then "Allow the connection" and selecting the "Finish" button.

![](Attachments/Pasted%20image%2020260511115852.png)

The rules will now show up in the "Inbound Rules" section of "Windows Defender.."

![](Attachments/Pasted%20image%2020260507212854.png)

I try to push a GPO update through the server again, but I got the same error.

I opened the command prompt as Administrator and entered `winrm quickconfig` to double check if it was setup. Which it was not. I assumed it would have already been enabled. Next time I'll have to double check this upfront.

Below I used `winrm quickconfig` to see the status of WinRM. Then accepting the prompt to start WinRM by pressing `y`. Then accepting the prompt to enable the firewall rules with `y`.

![](Attachments/Pasted%20image%2020260511121452.png)

Success! The GPO update from the Server as `cody_admin` went through to the Client, using `Invoke-Command`. Basically, using `cody_admin` on the Server, to use PowerShell on the Client. No need to log onto the Client anymore as `cody_admin` to do a GPO update on it.

![](Attachments/Pasted%20image%2020260511122400.png)

## Rename Default Admin Accounts and Make Honeypots

Here I'll do some more hardening steps for better security. Admin accounts are high value targets for threat actors. By changing the default names, it takes away from automation or clear targets.

In this exercise I will do the following name changes,
Domain Administrator = `DoAcct` (Domain Account)
Local Administrator = `LoAcct` (Local Account)

Though I will be using honeypots for Active Defense and Cyber Deception techniques. A honeypot is a defensive technique that creates fake, high value targets that will trigger alerts or actions when a threat actor stumbles in the trap try to access it.

For me to do that, I'll be making new accounts, using the `Administrator` names I am renaming, then disabling them. That way, if someone tries to access a disabled account, Event ID `4625` will generate as a failed logon event. Giving an early indicator someone is trying to access an account, that the Admin team should be well aware of is disabled.

The reason for renaming the original accounts and creating new accounts with the previous name, is to keep the SID and settings in the environment, without breaking anything.

### Domain Admin name change

To change the Domain Admin name, I'll need to be on the Server Manager, then go into the AD Users and Computers window.

Go to the Domain "Users" folder, then find "Administrator". Right click "Rename"

![](Attachments/Pasted%20image%2020260511144543.png)

In the "Rename User" window, I make the following changes.

![](Attachments/Pasted%20image%2020260511145001.png)

### Domain Administrator Honeypot

Now I'll create a new user in the same "User" folder with the normal default name `Administrator`. 

![](Attachments/Pasted%20image%2020260511150429.png)

Now I create the user to be as close to the Administrator as I can.

![](Attachments/Pasted%20image%2020260511150613.png)

Now I set a long password, unchecked "User must change password at next logon", and checked "Password never expires" and "Account is disabled". The intent is to set this account as disabled, and let it sit. After review, I select "Finish".

![](Attachments/Pasted%20image%2020260511150947.png)

An extra step is to copy the description from the original Admin account to the honeypot, then modifying the real admin account description.

![](Attachments/Pasted%20image%2020260519154831.png)

### Disable Accounts

Luckily the Guest account is already disabled, so this will be more of a security verification step. You can tell it is disabled as the icon next to the name has a down arrow. Both are shown here to confirm the honeypot Admin user has been created and the Guest account is as well.

Though the same concept exists. If someone does a password spray and tries to log into "Guest", the same `4625` Event ID will trigger.

![](Attachments/Pasted%20image%2020260511151344.png)

### Rename Local Admin GPO

The Local Administrator name change is handled through GPO.

I create another GPO at the domain level called, `ClickIT - Rename Accounts`.

The path is, 

**Computer Configuration > Policies > Windows Settings > Security Settings > Local Policies > Security Options**

Then change the "Accounts: Rename administrator account" setting to `LoAcct`.

![](Attachments/Pasted%20image%2020260511204433.png)

Now I'm going to do another GPO push from the Server to the Client. 

```Powershell
Invoke-Command -ComputerName DESKTOP-MBV8D69 -ScriptBlock {gpupdate /force}
```

Then to verify, I'll use `GET-LocalUser` to see the account names on the workstation.

```Powershell
Invoke-Command -ComputerName DESKTOP-MBV8D69 -ScriptBlock {GET-LocalUser}
```

![](Attachments/Pasted%20image%2020260511203707.png)

Within the information from that command, I find `LoAcct` with a SID of `500`. Success!

![](Attachments/Pasted%20image%2020260511204058.png)

I do the same for the Domain Admin on the Server. 

```Powershell
GET-LocalUser
```

Success! `DoAcct` shows up as the admin account.

![](Attachments/Pasted%20image%2020260511204224.png)

## Password Hardening

Now is to set the password difficulty and put a limit to the number of attempts someone can make before they get locked out.

### Password Difficulty Enforcement

When it comes to password difficulty, length is going to be the greatest strength to a strong password. Using numbers, upper and lowercase, and symbols increase difficulty as well but not as much as length. Below is a great chart displaying this.

![](Attachments/Pasted%20image%2020260511205643.png)

For this change I need to keep it at the domain level, changing the "Default Domain Policy" settings under `cyberlab.local`. That's because I am enforcing a policy on a domain user, thus a domain level policy is needed.

By choosing the "Default Domain Policy", then "Settings" in the window on the left, I can see what the current policy settings are.

A summary of the below,
- Enforce password history: Remembers the last 24 passwords used to prevent reuse.
- Max password age: 42 days till next new password is required
- Min password age: User must wait 1 day till they can change the password again.
- Min password length: At least 7 characters must be used for the password
- Password must meet complexity requirements: Special characters, upper and lower case letters and numbers must be used.
- Store password using reversible encryption: Disabled means, non-reversible hashes are used
- Account lockout threshold: User will be locked out after the number of invalid logins are met. This is a zero, meaning infinite.

![](Attachments/Pasted%20image%2020260511211049.png)

Based on this, there are two changes I need to make. Increase the length to 12 for stronger passwords and putting a password attempt limit of 5. As well as adjusting the lockout time.

I'll go to the below with,
**Computer Configuration > Policies > Windows Settings > Security Settings > Account Policies > Password Policy**

Then go to "Minimum password length" and change it to 12 characters.

![](Attachments/Pasted%20image%2020260511212041.png)

### Login Attempt Limits

As for the lockout settings, I go back one folder from the previous path, then choose, **Account Lockout Policy**

Selecting "Account lockout duration" I change to 15 minutes. Which prompts the below suggestions for the other two policies. Which was a quick way to set three settings at once.

![](Attachments/Pasted%20image%2020260511212352.png)

After that, I now have the following settings for the lockout.

- Account lockout duration: 15 minutes till user can attempt to login after hitting the invalid login number
- Account lockout threshold: A user will be locked out for 15 minutes after 5 invalid login attempts
- Reset account lockout counter after: 15 minutes of no login activity will reset the failed login attempts counter

![](Attachments/Pasted%20image%2020260511212455.png)

## Account Lockout and Password Reset

At this point, I need to do a test of the password enforcement after doing a GPO update push.

This is also a great time to demonstrate unlocking a locked out account, as well as doing a password reset.

Unfortunately, since I'm now using headless VMs, I can't just RDP into a user that hasn't already logged in. Which is what prompted the below error when trying to log in a Bob Token.

![](Attachments/Pasted%20image%2020260512192033.png)

I've only logged in as Karen and have seven other employees who have not logged in. Which would take some time to handle manually. I've already shown it works earlier.  So, I'm going to clear the first-login flag for all current users.

```Powershell
Get-ADUser -Filter {PasswordLastSet -eq 0} | Set-ADUser -ChangePasswordAtLogon $false
```

(Later I would need to use this command to test the password reset again.)

```PowerShell
Set-ADUser -Identity DPhlips -ChangePasswordAtLogon $true
```

Now I no longer get the error message and log in with the original default password I set.

![](Attachments/Pasted%20image%2020260512192557.png)

But now I get this error. I'm now realizing a limitation of using RDP like this. I can't log in with my regular users using RDP, as they don't have access. They aren't in the `IT_admin` group. 

![](Attachments/Pasted%20image%2020260512193740.png)

In the [VRDE Setup](#vrde-setup) section in Lessons Learned, I continue to go away from resource heavy CRD and use VRDE instead.

![](Attachments/Pasted%20image%2020260512213130.png)

Now that `BToken` has logged in for the first time, now I'll purposefully enter an incorrect password 5 times.

I now have an error message about attempting to log in and getting locked out. Success!

![](Attachments/Pasted%20image%2020260512213924.png)

I attempt to log in and get the same message. After a GPO update to shorten the lockout timer to 1 minute, I attempt again, and am able to attempt to log in again.

![](Attachments/Pasted%20image%2020260512214712.png)

Now I'm going to lock Bob out again to use admin controls to unlock him. I already changed the lockout timer back to 15 minutes.

![](Attachments/Pasted%20image%2020260512215139.png)

### Account Unlock

Going into AD Users and Computers, I need to go to `BToken` in the "Operations" OU. Once I find him, I'll right click his account, select "Properties" then find the "Account" tab.

There is a checkbox indicating the account is currently locked. Checking the box and clicking apply will unlock the account.

![](Attachments/Pasted%20image%2020260512215414.png)

As Bob, I'm able to attempt to log in again, but we are assuming the password is forgotten. 

### Password Reset

Now as `cody_admin` I'll do a password reset for Bob to create a new one they'll hopefully remember.

In the same place, I'll right click Bob again and select "Reset Password".

![](Attachments/Pasted%20image%2020260512215801.png)

From here, I'll make a generic password for Bob to use for initial login.

With "User must change password at next logon" checked, Bob will be able to make their own unique password. That way even the admin doesn't have the password to log in as Bob later.

You'll also notice another lock status and unlock checkbox in this window. That way the previous step to unlock Bob and this step to reset their password can be done all at once.

![](Attachments/Pasted%20image%2020260512215855.png)

After a confirmation window, I'll go back to the Client to attempt to log in as Bob.

In the Client as Bob, I get a prompt informing me that Bob must change their password. Success! 

![](Attachments/Pasted%20image%2020260512220132.png)

And Bob now has a new password!

![](Attachments/Pasted%20image%2020260512220353.png)

## Event Viewer Use

We just had some account events occur with failed login attempts, locked accounts being unlocked, password changes, etc.

So let's do some Event Viewer discovery to find these.

Going to the Event Viewer, then Windows Logs and right clicking Security, I select "Filter Current Log..."

![](Attachments/Pasted%20image%2020260512221204.png)

The Event IDs I'll be looking for are `4625, 4740, 4724, 4767`

4625 - Failed Logon Attempts
4740 - Account Lockout
4724 - Password Reset by Admin
4767 - Account Unlock

![](Attachments/Pasted%20image%2020260512221344.png)

I do see a failed logon (`4625`) earlier today, but that was when I tried to RDP into the Client with `KLycker`. But nothing about Bob failing to log in.

![](Attachments/Pasted%20image%2020260512221922.png)

That's because events on different machines may not all get logged in a centralized place. This is why log aggregators are important. Pulling logs from multiple places.

I'll check the remaining Event ID's for now and use PowerShell to retrieve the other Events from the Client.

Next event is the account for Bob getting locked (`4740`) from the incorrect password attempts.

![](Attachments/Pasted%20image%2020260512222602.png)

After that, you can see the two times I unlocked Bob's account with ID `4767`.

![](Attachments/Pasted%20image%2020260512222736.png)

Then when I reset Bob's password (`4724`).

![](Attachments/Pasted%20image%2020260512222827.png)

Now to look at the events on the Client machine with PowerShell.

```PowerShell
Invoke-Command DESKTOP-MBV8D69 -ScriptBlock 
{
Get-WinEVENT -LogName Security | 
Where-Object {($_.Id -eq 4625)} |
 Select-Object Id, TimeCreated
 }
```

From the return, you can see the quick succession of incorrect password attempts by me for the demonstration.

![](Attachments/Pasted%20image%2020260512224322.png)

If I wanted the same detail as the Event Viewer, I just add `Message` to `Select-Object`

```PowerShell
Invoke-Command DESKTOP-MBV8D69 -ScriptBlock 
{
Get-WinEVENT -LogName Security | 
Where-Object {($_.Id -eq 4625)} | 
Select-Object Id, TimeCreated, Message
}
```

![](Attachments/Pasted%20image%2020260512224725.png)

## Audit Policy Settings

I'll be doing some user account management next. So I want to verify that certain Audit Policy settings are in place so I can do similar reviews as above, but with more auditing enhancements.

This will enable failure event creations, when only success is set as the default. And centralize the policy by controlling through GPO, and not the individual OS defaults on the different machines.

These will be the policies I'll be modifying to allow these events to generate in the logs. The others can create more noise than necessary.

![](Attachments/Pasted%20image%2020260513120353.png)

I get here with by editing the "Default Domain Policy", then going the following path.

**Computer Configuration > Policies > Windows Settings > Security Settings > Local Policies > Audit Policy

Then, going into each, I right click, select "Properties", then mark "Success" and "Failure".

![](Attachments/Pasted%20image%2020260519184242.png)

**Audit Account Logon Events**

This setting will enable the audit event for account credentials every time the computer validates them.

**Audit Account Management**

This setting will audit each account management event. Like user account or groups being created or deleted, or modified. Or password modifications.

**Audit Logon Events**

This setting will audit each log on and log off attempt. Including failures, which is not a default.

**Audit Policy Change**

This setting will audit whether different policies have been modified. Like trust policy, account policy, etc.

Before I do a GPO update on the Client, I use the following command to look at the detailed settings for a before and after comparison.

```PowerShell
Invoke-Command -ComputerName DESKTOP-MBV8D69 -ScriptBlock
{
cmd /c "auditpol /get /category:*"
}
```

![](Attachments/Pasted%20image%2020260513122049.png)

Then I push the GPO updates and re-run that PowerShell command in a new terminal.

Here is the difference for "Logon/Logoff", with before on the left.

![](Attachments/Pasted%20image%2020260513122458.png)

Then the difference between "Policy Change", "Account Management" and "Account Logon".

![](Attachments/Pasted%20image%2020260513122723.png)

Success!

## Share Folder Permissions

Here will be the creation of department folders and setting the permissions. That way only those in Finance can view the files and folders in the Finance department Shared Folder.

These are the Shared Folders to be created.

- Finance
- HR
- Legal
- Operations
- Company Resources

When it comes to the permissions levels, I'm going to use two layers to manage this. The Share Permissions level, then the NTFS Permissions level.

The most restrictive permissions takes priority between the two levels. So even if "Full Control" is given at Share Permissions, but "Read" is only given at the NTFS level, only "Read" is allowed. As NTFS didn't grant the other permissions that the Share Permissions allowed.

At the Share Permissions on the folder, I'll have `IT_Admins` at Full Control. Then the `Domain Users` will be set to Change.

The idea is to allow the IT admins to have full control, including permissions. While the regular users can have typical controls, like read, write, delete, etc. They just can't change the permissions.

At the NTFS Permissions level, I'll add the Security groups to the Security tab for the folders at "Modify". This will determine who can access the folder.

This is an approach to keep from complex collisions in permissions causing conflict issues.

Now I'm going to go set the folders to share and set the permissions. (If I didn't want to use the visuals, I'd use AI to create a quick script for me to review, use, then verify).

`IT_Admins` will be set to "Full Control"

![](Attachments/Pasted%20image%2020260513134556.png)

`Domain Users` will be set to "Change".

![](Attachments/Pasted%20image%2020260513134832.png)

As I set the NTFS settings in the "Security" tab.

Example for `Computer Resources`, I have `Management_Access` as "Modify" and `Users` as "Read". As management would need to modify the company resources and the employees should just be reading the documents.

![](Attachments/Pasted%20image%2020260513135850.png)

When modifying the "Security" tab, I got an error informing me of "inheriting permissions from parent". 

For me to modify these folders more uniquely, I need to disable this so the current settings are cloned individually to the folders for me to change.

For the `Finance` folder, I now have `Finance_Access` at "Modify" for the folder.

![](Attachments/Pasted%20image%2020260513140411.png)

Now to test with `KLycker`, who is in the operations department. As signed in as Karen on the Client, I'll go to the Network folder and see all the shared folders I just created.

![](Attachments/Pasted%20image%2020260513141126.png)

When trying to access the `Finance` folder, I get a permission denied error. Success!

![](Attachments/Pasted%20image%2020260513141203.png)

When selecting the `Operations` folder, Karen is able to access the folder. Success!

![](Attachments/Pasted%20image%2020260513141240.png)

## Department Moves

Now for some real world work environment actions. Karen Lycker is going to be promoted to Finance to help with Penny Banks.

For me to do so, I'll need to go into the `Operations` OU in AD Users and Computers, and find Karen. Go to properties, then find the "Member of" tab.

Adding `Finance_Access` and removing `Operations_Access`.

![](Attachments/Pasted%20image%2020260513142233.png)

Then right click Karen and select the "Move.." option. Selecting the `Finance` OU.

![](Attachments/Pasted%20image%2020260513142438.png)

Now Karen is in the `Finance` OU and the `Finance_Access` Group.

Let's log out and back in for Karen and test her folder access again.

When attempting to go back into the `Operations` folder with Karen after her promotion, I get an access denied. Opposite of last time. Success!

![](Attachments/Pasted%20image%2020260513142723.png)

Now let's see if she'll be able to access her `Finance` folder for her new position. Success!

![](Attachments/Pasted%20image%2020260513142821.png)

## New Employee Onboarding

With Karen getting a promotion, the company has hired a new employee for the operations team. Ryan Kitt. They'll have the username `RKitt`.

I add Ryan to the `Operations` OU and make him a member of the `Operations_Access` group.

![](Attachments/Pasted%20image%2020260513144006.png)

(I later caught I forgot to add Ryan to the `All_Employees` group during [Verification](#verification). He was later added, shown below).

![](Attachments/Pasted%20image%2020260514121243.png)

# Verification Steps

That is the completion of the Microsoft Work Environment setup with AD DS. In this section, I verify each concept to confirm the final product is complete and functional. That no changes throughout the process altered previous accomplishments. As sometimes, a later change, can effect an earlier one.

**Creation of OU and Users**

The `ClickIT` OU was created with sub-OUs for each department. In each department, are the employees for that department. Verified each department exists with each employee accounted for.

![](Attachments/Pasted%20image%2020260514115214.png)

**Security Group Creation**

Creation of the `Groups` OU to manage the Security Groups for ClickIT Inc. Verification of all employees for each Group.

![](Attachments/Pasted%20image%2020260514115615.png)

![](Attachments/Pasted%20image%2020260514120647.png)

(Ryan was found to be missing in `All_Employees` during this step. It was missed in [New Employee Onboarding](#new-employee-onboarding) and fixed during this step. Now added.)
 
 **IT Admin Group Permissions**

As a member of the `IT_Admin` group, IT user `cody_admin` is able to RDP into the Windows Client VM.

![](Attachments/Pasted%20image%2020260514121629.png)

![](Attachments/Pasted%20image%2020260514121752.png)

Signed in as `KLycker` in the `Finance_Access` group, not `IT_admin` group, has an admin password credentials request when attempting to use RDP. Restricting access.

![](Attachments/Pasted%20image%2020260514122519.png)

These are the GPO settings below.

![](Attachments/Pasted%20image%2020260519142739.png)
![](Attachments/Pasted%20image%2020260519142843.png)
![](Attachments/Pasted%20image%2020260519142951.png)

**Shared Folder Permissions**

All shared folders are reviewed. Correct permissions are assigned. Specific department access is given to the specific department folders.

![](Attachments/Pasted%20image%2020260514124324.png)

Signed in as `KLycker`, a member of the Finance department in the `Finance_Access` group, has access to the shared Finance folder.

![](Attachments/Pasted%20image%2020260514124512.png)

Still signed in as `KLycker`, no longer a member of the operations team after a promotion, does not have permissions to the Operations folder.

![](Attachments/Pasted%20image%2020260514124438.png)

---

***Group Policy Management***

**Login Banner**

When attempting to log into the Client workstation, a Login Banner warning message appears before attempting to log in.

![](Attachments/Pasted%20image%2020260514124706.png)

GPO settings below for `ClickIT - Login Banner`.

![](Attachments/Pasted%20image%2020260519183059.png)

**Disable Auto Updates**

When in the Windows Update window, a "Some settings are managed by your organization" warning is displayed.

![](Attachments/Pasted%20image%2020260514125003.png)

Below is the GPO setting for `ClickIt - Disable Auto Update`.

![](Attachments/Pasted%20image%2020260514125942.png)

**Desktop Wallpaper Enforcement**

When signing into the Windows Client, the enforced NSA Cybersecurity wallpaper is displayed.

![](Attachments/Pasted%20image%2020260514130112.png)

GPO Settings below for `ClickIT - Wallpaper Enforcement`.

![](Attachments/Pasted%20image%2020260514130325.png)

**CMD Restriction**

When attempting to use the CMD logged in as normal user, a message is displayed indicating CMD has been disabled.

![](Attachments/Pasted%20image%2020260514130510.png)

As `cody_admin`, a member of the `IT_Admins` group, CMD is available for use.

![](Attachments/Pasted%20image%2020260514131224.png)

GPO Settings below `ClickIT - Restrict CMD Usage`.

![](Attachments/Pasted%20image%2020260514131326.png)

(I found a security gap here, as `KLycker` still had access to PowerShell. I speak on it slightly here, [PowerShell Security Gap](#powershell-security-gap), then fix it here, [Restrict PowerShell](#restrict-powershell) )

**PowerShell Restriction**

When attempting to use PowerShell as regular employee `KLycker`,  a message of "This app has been blocked by your system administrator" is displayed.

![](Attachments/Pasted%20image%2020260519121200.png)

When attempting to use PowerShell as `cody_admin` of the `IT_Admins` group, PowerShell successfully opens.

![](Attachments/Pasted%20image%2020260519121411.png)

GPO Settings below `ClickIT - Restrict PowerShell` and `ClickIT Enable AppIDSvc`.

![](Attachments/Pasted%20image%2020260519122050.png)

![](Attachments/Pasted%20image%2020260519183504.png)

**Control Panel Access**

When attempting to open the Control Panel while signed in as `KLycker` as a regular employee, the Control Panel displays a "This operation has been cancelled due to restrictions in effect on this computer. Please contact your system administrator."

![](Attachments/Pasted%20image%2020260519123326.png)

When attempting to open the Control Panel while signed in as `cody_admin` of the `IT_Admins` group, the Control Panel successfully opens.

![](Attachments/Pasted%20image%2020260519121719.png)

GPO Settings below for `ClickIT - Control Panel Access`.

![](Attachments/Pasted%20image%2020260519123451.png)

**Idle Timeout**

While logged in as `KLycker` the computer is left idle at 9:38 AM.

![](Attachments/Pasted%20image%2020260519123836.png)

Upon return to the computer at 9:53 AM, the computer is found locked.

![](Attachments/Pasted%20image%2020260519125400.png)

GPO settings below for `ClickIT - Lockout Timer`.

![](Attachments/Pasted%20image%2020260519125455.png)

---
***User Account Management***

**Initial Login Settings**

When signing in for the first time as the user `DPhlips`, using the initial password at account creation, `Welcome123!`, 

![](Attachments/Pasted%20image%2020260519125637.png)

user is prompted that their password must be changed before signing in.

![](Attachments/Pasted%20image%2020260519130721.png)

**Account Unlock**

Using user `DPhlips`, is locked out and needs to be unlocked.

![](Attachments/Pasted%20image%2020260519130945.png)

Going to the AD Users and Computers window. User account `DPhlips` is found, and unlocked.

![](Attachments/Pasted%20image%2020260519131257.png)

**Password Reset**

Continuing with the above scenario, `DPhlips` is locked out again, as they have forgotten their password.

![](Attachments/Pasted%20image%2020260519131518.png)

Finding, `DPhlips` again, the "Reset Password" option is chosen. Giving the user a new password, making the user change their password at login, and unlocking the account all at once.

![](Attachments/Pasted%20image%2020260519131918.png)

![](Attachments/Pasted%20image%2020260519132057.png)

User is confirmed to have the option to enter a new password, to log back into the system.

![](Attachments/Pasted%20image%2020260519132155.png)

**Department Moves**

Logged in as the user `KLycker`, who was recently promoted from the Operations department, to the Finance department, now has access to the Finance shared folder.

![](Attachments/Pasted%20image%2020260519135326.png)

User no longer has access to the Operations shared folder.

![](Attachments/Pasted%20image%2020260519135429.png)

**New Employee Onboarding**

A new employee has joined the company, Ryan Kitt, with the user name `RKitt`. They will be in the Operations department.

AD Users and Computers shows `RKitt` in the `Operations` OU, and a member of `All_Employees`, `Domain Users` and `Operations_Access` Security Groups.

![](Attachments/Pasted%20image%2020260519140112.png)

When doing the initial login, user is prompted to enter a new password because of the Initial Login settings.

![](Attachments/Pasted%20image%2020260519140936.png)

Once new user is logged in, they are confirmed to have access to the Company Resources and Operations shared folders, but does not have access to the Finance folder, all as expected.

![](Attachments/Pasted%20image%2020260519141538.png)

---

***Hardening Settings***

**NLA Authentication for RDP**

While logged in as `cody_admin`, checking the Advanced Settings of Remote Desktop shows Network Level Authentication is turned on.

![](Attachments/Pasted%20image%2020260519142325.png)

GPO setting below, from the previous `ClickIT - Enable RDP` GPO.

![](Attachments/Pasted%20image%2020260519143427.png)

**Password Hardening**

Signed in as the user `RKitt`, I am going to test some of the main password hardening settings.

Password history:

Here I attempt to use the original password given to the user by the admin, `Welcome123!`.

An error prompt shows three possible errors not allowing the password to update. One of which being a history requirement. Which would follow the "Enforce password history" setting, to not allow the last 24 used passwords.

![](Attachments/Pasted%20image%2020260519152606.png)

Password length:

This time I'll attempt to change the password to a very short password.

![](Attachments/Pasted%20image%2020260519152756.png)

The same error is displayed, which would be triggered by the "Minimum password length" requirement of 12 characters.

![](Attachments/Pasted%20image%2020260519153007.png)

Password complexity:

Now I'll attempt to set a password with only upper and lower case letters, but with more than 12 characters.

![](Attachments/Pasted%20image%2020260519153303.png)

The same message is displayed, which would be triggered by the "Password must meet complexity requirements" setting being enabled.

![](Attachments/Pasted%20image%2020260519153400.png)

Password lockout threshold:

Here I'll attempt to log into the user with six unsuccessful attempts using random passwords. After the sixth attempt, a message will display indicating the user is currently locked out. This will be triggered by the 5 invalid logon attempt settings, and the user will need to wait 15 minutes before they can attempt to log in on their own again.

![](Attachments/Pasted%20image%2020260519153539.png)

GPO settings changes for "Default Domain Policy" below.

![](Attachments/Pasted%20image%2020260519144327.png)

**Create Honeypot Administrator**

Going to the AD Users and Computers window, I find the account, `Administrator`. But when looking at the attribute editor, you'll see it is a normal user and does not have the admin SID of 500.

![](Attachments/Pasted%20image%2020260519155431.png)

**Disabling Accounts**

Continuing on from above. Both the honeypot `Administrator` and the `Guest` accounts are set as disabled. This is to prevent threat actors from logging in and using these accounts. Also being early warning signs when someone attempts to log into what the IT team would know are disabled.

![](Attachments/Pasted%20image%2020260519155701.png)

**Domain Admin Renamed**

Below is the real Administrator account renamed as `DoAcct`. This can be verified by checking the SID, which is 500. The description of the account was also slightly modified. This is an attempt to make the domain account harder to find.

![](Attachments/Pasted%20image%2020260519155346.png)

On the Server, Domain Account is confirmed, `DoAcct`.

![](Attachments/Pasted%20image%2020260519160908.png)

**Rename Local Admin GPO**

The same concept for the Local Admin. The idea is to make it harder to find a high value target. Though the local admin is disabled by default with the domain-join.

On the Client, the Local Admin is confirmed, `LoAcct`.

![](Attachments/Pasted%20image%2020260519162514.png)

GPO settings below for `ClickIT - Rename Accounts`.

![](Attachments/Pasted%20image%2020260519162712.png)

**Event Viewer Use**

Using Event Viewer, I can see a lot of different login activity that is useful. This was from my earlier account and password verification settings.

![](Attachments/Pasted%20image%2020260519164623.png)

I can filter for specific Events like the following.

4625 - Failed Logon Attempts
4740 - Account Lockout
4724 - Password Reset by Admin
4767 - Account Unlock

You'll notice in the same time frame from of 5/19, events are split between the Server and Client.

Server confirmed to record events `4740 - Account Lockout`, `4724 - Password Reset by Admin`, `4767 - Account Unlock`.

![](Attachments/Pasted%20image%2020260519170602.png)

Client confirmed to record 4625 - Failed Logon Attempts, showing difference between Account Lockout and Logon Attempt.

![](Attachments/Pasted%20image%2020260519170818.png)

I can also view these using PowerShell.

Confirmed on Client.

```Powershell
Get-WinEvent -LogName Security | Where-Object {($_.ID -eq 4625)} | Select-Object Id, TimeCreated, TaskDisplayName
```

![](Attachments/Pasted%20image%2020260519171238.png)

Confirmed on Server.

```Powershell
Get-WinEvent -LogName Security | Where-Object {($_.ID -eq 4740 -or $_.ID -eq 4724 -or $_.ID -eq 4767 -or $_.ID -eq 4625)} | Select-Object ID, TimeCreated, Message
```

![](Attachments/Pasted%20image%2020260519171920.png)


**Audit Policy Settings**

Audit Policy changes made earlier confirmed in the Event Log, as from 5/12 and on, there are more Events recorded, that weren't being recorded before.

![](Attachments/Pasted%20image%2020260519172954.png)

Specifically the difference between 5/11 and 5/19. On 5/11, there was a Password Reset, and Account Unlock when testing with the user `BToken` in the instructions section. Only the password reset was logged. After 5/12 you can see multiple new logs that give more details to what is going on. Like accounts be locked out, unlocked and the passwords being reset.

![](Attachments/Pasted%20image%2020260519173247.png)

Updated Audit Policy Confirmed.

GPO setting changes for "Default Domain Policy" below.

![](Attachments/Pasted%20image%2020260519184112.png)

---
# Lessons Learned

Here is a section that will go over different lessons I learned as I went through this exercise. While going through this, I ran into multiple obstacles. Researched, experimented, pivoted.

## Host Limitations

My Home Server limitations were starting to show at this point in the demonstration. I already had an idea this was going to come up. CPUs were spiking and memory swap was occurring. 

![](Attachments/Pasted%20image%2020260506161702.png)

My home server only has 4 cores and 16 GB of RAM. But my ThinkPad T14 has 8 cores (16 threads) and 32 GB of RAM. 

This will end up being a future project to swap my Samsung 990 Pro 2 TB SSD from the Home Server to the T14 with the T14's SK hynix 512 GB SSD.

In an effort to keep the labs running smooth, I decided to drop the Windows Server from 4 GB to 3 GB and the Windows Client from 3 GB to 2 GB, to not slow this exercise down. 

After that my RAM went from 71% to 52%.

![](Attachments/Pasted%20image%2020260506165257.png)

Back to [Resource Issues](#resource-issues).

## Logout Difference

I ran into a GPO update issue because I chose to "Switch user" instead of "Sign out".

When switching users, the session stays active. Because the session is still active, GPO updates might not always process. This is a key difference between signing out when making changes, as that would end and create a new session.

When enacting changes on a user, they should be signing out to end and create a new session. This could cause policy issues or troubleshooting rabbit holes if not done properly.

Back to [Control Panel Access](#control-panel-access)

## Switch to RDP

I had been connecting from my laptop to my home server using Chrome Remote Desktop (CRD) to access the VM's on it. 

During the [Idle Timeout](#idle-timeout) settings, I would wait for the screensaver to activate, but it seemed to also disconnect my ability to use my mouse and keyboard on the laptop. I had to use the manual virtual keyboard offered by CRD to push any actions onto the home server.

I was already seeing some performance issues with the lack of RAM, mentioned here [Host Limitations](#host-limitations). I restarted the VM's and restarted CRD. Considering I kept losing connection to my keyboard and mouse, I was getting slowed down.

I already have Tailscale on my laptop and home server, but after multiple attempts to troubleshoot, I decided on trying RDP (Remote Desktop Protocol) over CRD.

On the Windows Server, I downloaded Tailscale and set up the device to have its own IP. 

Once installed, I activate RDP in the System settings. Then I shutdown the VM to restart it as headless. Meaning no GUI is launched on the host to save on resources.

```bash
VBoxManage startvm "Windows-Server-DC" --type headless
```

![](Attachments/Pasted%20image%2020260507190933.png)

Once I connect with RDP, the GUI is output on the laptop with the remote desktop client. I downloaded Remmina, as the remote desktop software client, to RDP into the Server.

This was a success and the visual and responsiveness increased tremendously. With the host not needing to handle the GUI as well, that frees up more resources.

I have RDP setup for both the Windows Server and Client. Though I only use Tailscale for the Server, since it is internet facing. I'll RDP into the Client using the Server.

Another issue that came up, was when I logged out as the Administrator on the Server using Tailscale, Tailscale went down because no users were signed in.

I had to use two commands for this. The first to confirm Tailscale starts automatic with the Server, and the second to allow Tailscale to run unattended, without a user signed in.

```PowerShell
Get-Service *tailscale* | Select-Object Name, StartType, Status
```

```PowerShell
tailscale set --unattended
```

![](Attachments/Pasted%20image%2020260507201826.png)

Back to [Idle Timeout](#idle-timeout) to see if that helps resolve the Windows lockout policy.

## VRDE Setup

As discussed in the [Account Lockout and Password Reset](#account-lockout-and-password-reset) section, I realized going headless and relying on RDP would only really work for the IT_Admin group, not the regular users.

For me to log in as those regular users on the Client, I'll need to have some sort of GUI. Which makes sense considering normal employees are primarily going to use a GUI as they aren't as technical with the terminal or even have access.

Here I'll use VRDE, VirtualBox Remote Desktop Extension. It's a built in way to use a desktop GUI with headless VM.

To setup VRDE. I use the below command with the Client started headless.

```bash
VBoxManage modifyvm "Windows-10-Client" --vrde on --vrde-port 3390
```

Then on Remmina, I make a RDP connection using the Tailscale IP of the Host with port 3390.

Unfortunately, the resolution doesn't scale like with the VBox Guest Additions, but it still works.

Back to [Account Lockout and Password Reset](#account-lockout-and-password-reset)

## PowerShell Security Gap

While going through the [CMD Restriction](#cmd-restriction) verification steps, I realized I was missing PowerShell restrictions on my checklist. Confirmed below, Karen can't use CMD but can still use PowerShell. Yikes!

![](Attachments/Pasted%20image%2020260514132850.png)

When I first set out on this project, I brainstormed a list of objectives to showcase. Not that different than creating playbooks or on-boarding task lists.

Regardless, when running through a predetermined checklist given to you. It's still good to keep your mind open on what is going on in the full picture of things.

Here I was checking to see if the CMD was restricted. It was. Then my next thought was, wait, I've been using PowerShell. I didn't do anything about PowerShell. So Karen most definitely should also have PowerShell.

And sure enough, Karen has access.

I also ran into some issues when setting up the restrictions. As I got `cody_admin` locked out of using PowerShell as well.

![](Attachments/Pasted%20image%2020260517184758.png)

I then checked if AppIDSvc was up and running, and found it was in a "STOPPED" state. I also found that the AppLocker "Executable Rules" was blank.

![](Attachments/Pasted%20image%2020260517184935.png)

I then did some deep research with Claude and found that the "Package App Rules" defaults were needed. Think Start menu, Search, Store, etc. that are the original apps on your Windows.

![](Attachments/Pasted%20image%2020260517191308.png)

After fixing the package rules, I was able to use the things I wasn't before, like using the Windows button, the search box and the Windows Store. But PowerShell was still blocked for everyone.

Here was another discovery of how I incorrectly set up the AppLocker Rules.

When making the rules, I tried the approach of Denying for `Everyone`, then Allowing for the `IT_Admins`. There was some confusion after what I learned with the File Sharing section. In which for the NTFS vs Share permissions, restrictive permissions are going to take priority between the two granting permissions.

But in this case, I was dealing with specific "Allow" and "Deny" permissions. As in, the "Deny" would take priority over an "Allow".

![](Attachments/Pasted%20image%2020260515203347.png)

So even though I allowed the admins to use PowerShell, the previous rule to not allow everyone was restricting that access.

Another learning point was the use of "Publisher" or "Path" in the rules conditions.

![](Attachments/Pasted%20image%2020260519111312.png)

When using "Publisher" on the Server, it pulls the version of the file. 

![](Attachments/Pasted%20image%2020260519113117.png)

I found that the version mismatch was causing the Client PS to still run.

![](Attachments/Pasted%20image%2020260519112032.png)

I didn't get to this realization till I ran the Rule Collections in PowerShell, while as `KLycker`. Considering she had access, might as well take advantage of it.

```Powershell
Get-AppLockerPolicy -Effective | Select-Object -ExpandProperty RuleCollections
```

(sorry for the poor screenshot)
![](Attachments/Pasted%20image%2020260519113834.png)

Afterwards, I went to just using "Path" though later I realized I could have just pulled the dial on the left up one to set the version to any, with (`*`).

![](Attachments/Pasted%20image%2020260519111525.png)

After all of that and a much simpler setup you'll see in [Restrict PowerShell](#restrict-powershell), I was able to correct the accesses.

# Summary

This was a lot. But it really put me through different scenarios and troubleshooting for better knowledge of the environment.

Having to navigate an enterprise-like setup like this allowed me to explore a lot. Mistakes were made. Settings didn't work as intended. Funny moments occurred, like having the screen lockout every 15 seconds while trying to document, before I finally decided to change the GPO.

I tried to have a security mindset as well as I created this environment, as this will be used in offensive and defensive scenarios with my Kali VM.

The PowerShell Restriction setup was the most difficult. That was a lot of trial and error. Two different GPOs, multiple users, different application versions and types.

Luckily I just spent some time on the Share Folders prior, which helped me understand my mistake.

A huge wrench in the project came when I started having the resource issues and I had to switch to VRDE. I actually tried to do XRDP at first, but had to abandon that one too.

The PowerShell Restriction, host resource management and RDP issues took up the most of my time.

I did have Claude as an assistant throughout. Multiple instances of having to push back on suggestions. I have it set to teach me throughout the session, as to not just give answers. Allowing me to make the calls and navigate the setup process.

The Event Logs audit policy and Shared Folder permission setup I spent time on having to research deeper myself. As I wanted to have a better understanding of those.

Even with all the time spent on this, and the different obstacles I encountered throughout. It was a huge learning experience. 

Being in an office environment for almost a decade, then playing around with all the behind the scene foundations that allowed me to do the work was interesting.

VM Snapshots taken, as the full lab is now setup.

Next up, letting loose with Kali to start the cat and mouse game. The back and forth of system exploitation and hardening. Plus some IAM practice while I work on my SC-300, as I could start trying to use Azure.

---

# Resources

- https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/
- https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/group-policy/group-policy-overview
- https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/plan/security-best-practices/audit-policy-recommendations?tabs=winclient
- https://netwrix.com/en/resources/blog/ntfs-vs-share-permissions/
- https://www.varonis.com/blog/ntfs-permissions-vs-share
- https://www.tenfold-security.com/en/set-ntfs-permissions/
- https://www.tenfold-security.com/en/ntfs-permissions-and-share-permissions-whats-the-difference/
- https://www.lepide.com/blog/top-10-things-to-audit-in-active-directory/
- https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/plan/security-best-practices/advanced-audit-policy-configuration
- https://activedirectorypro.com/account-lockout-policy/
- https://learn.microsoft.com/en-us/services-hub/unified/health/remediation-steps-ad/set-the-account-lockout-threshold-to-the-recommended-value