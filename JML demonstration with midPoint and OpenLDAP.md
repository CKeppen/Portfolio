# JML demonstration with midPoint and OpenLDAP

Author: Cody Keppen [LinkedIn Profile](https://www.linkedin.com/in/cody-keppen-a09068355/)

Date: 09/03/2026

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
This is a simple demonstration of foundational Joiner Mover Leaver (JML) functions related to Identity and Access Management (IAM). This lab and virtual machines provided by [SimplifyIAM](https://www.skool.com/simplify-iam-6792/classroom/d0e4038f?md=fe9b0d7da2ff4760a7814280ced2a596). 

The lab uses [midPoint](https://evolveum.com/midpoint/) as the Identity Governance and Administration (IGA) platform to handle HR tasks from a custom HR app to simulate what you would see from something like Workday.

MidPoint then will automate tasks into [OpenLDAP](https://www.openldap.org/), acting as the Directory for the company. Storing the user objects. 

The SimplifyHR app needs to be thought of as the "source of truth". HR will perform their tasks and update their portal to show the most current standing of the employees at the company.

The intended process flow is as follows:

HR representatives uses SimplifyHR to add, move and terminate employees. The basic Joiner, Mover, Leaver flow.

The Identity and Access Management member will use SimplifyIAM to handle the provisioning of the HR actions to sync with the OpenLDAP Directory.

#  Concepts Demonstrated

1. Joiner
2. Mover
3. Leaver 

---
# Instructional Steps

These are the steps taken during this demonstration. There will be links throughout this section to the [Lessons Learned](#lessons-learned) section to notate obstacles and solutions during the demonstration.

This lab comprises of three components. A custom HR app called SimplifyHR, an instance of the midPoint IGA using the name SimplifyIAM and OpenLDAP as the SimplifyIAM domain controller (DC).

Here is the SimplifyHR app that currently has six employees listed. It also has an "+ Add Employee" button at the top right. You will see five columns for the Employee Directory. Employee ID, Name, Department, Status and Actions.

![](images/Pasted%20image%2020260903185326.png)

<br> Here is the SimplifyIAM user list. Which includes the same six employees and the admin account. There is a lot of functionality for this service.

![](images/Pasted%20image%2020260903185436.png)

<br> Now, here is the Directory for SimplifyIAM with the same six employees.

![](images/Pasted%20image%2020260903185648.png)

<br> Now that all three services have been shown for a visual, I'll start the process of onboarding a new employee.

## Joiner - Adding a new employee

For us to add a new employee, we'll start with the button at the top right, "+ Add Employee".

In this instance, we are pretending to be an HR representative entering a new employee to the database. The task for the new employee will have the following fields to enter.

First Name: Bob <br>
Last Name: Boberton <br>
Department: Legal <br>

When entered into the "Add New Employee" popup, you can see a "CSV Preview" at the bottom that will go into the HR file.

You'll notice it automatically adds the Employee ID to an available ID as well as the other information we entered. What was not asked of us is the initial Status of the employee. Which is defaulted to "Active".

![](images/Pasted%20image%2020260903190421.png)

<br> Now that the new employee has been added, you can see him as "Active" at the bottom of the "Employee Directory".

![](images/Pasted%20image%2020260903190652.png)

<br> Let's check the other services to see if he is there. 

First to check is SimplifyIAM and he is not there.

![](images/Pasted%20image%2020260903190758.png)

<br> Nor in the OpenLDAP Directory.

![](images/Pasted%20image%2020260903190830.png)

<br> Now, we could enter the new employee into the Directory manually, but that is a bit tedious. You are creating a double entry situation in which you are typing something someone else has already entered into a system.

![](images/Pasted%20image%2020260903191048.png)

<br> Plus you are increasing the chances of typos and syncing issues. Introducing scenarios where entries may no longer be uniform anymore. 

Here I manually entered Bob Boberton into the Directory, but he doesn't exactly match the other employees. The below can easily be done with new IT personnel who have to follow procedures. And that is if procedures are updated.

![](images/Pasted%20image%2020260903191714.png)

<br> Instead we'll use SimplifyIAM's Server Task function to create some automation. From the "All Tasks" module, select "New Task" at the bottom, represented as a "+".

![](images/Pasted%20image%2020260903191233.png)

<br> Then I'll select the "Reconciliation Task" to create a new one with the help of a wizard.

![](images/Pasted%20image%2020260903191951.png)

<br> This will be a simple task creation, starting with selecting the "Resource", which in this case is going to be the SimplifyHR app.

Then going through to choose the "Kind" to "Account", the intent kept to the "default" then "Object Class" set to the "AccountObjectClass".

This is to define the resources as an Account when it provisions the data. Everything else stays the same for this demonstration.

![](images/Pasted%20image%2020260903192233.png)

<br> Once I click save, I'm back to the All Tasks list, with the newly create Reconciliation Task. It is currently Suspended, but I can click "Resume" from here to start the task. In the previous window, we also had the option to "Save & Run" to have the task run right after it is saved.

![](images/Pasted%20image%2020260903192643.png)

<br> The icon and Execution column will change providing tooltip updates, or you can click the refresh button at the bottom. After hitting refresh, I can see that the task ran successfully.

![](images/Pasted%20image%2020260903192810.png)

<br> Now to check what happened. On SimplifyIAM, I can see "Employee.Bob" now listed in the users, following the typically format the company uses.

![](images/Pasted%20image%2020260903193158.png)

<br> Going into the account, I can see everything from the SimplifyHR app was pushed over into the SimplifyIAM table. Including the Organizational Unit.

![](images/Pasted%20image%2020260903193252.png)

<br> When looking at the OpenLDAP Directory, we can see both entrees for Bob now.

![](images/Pasted%20image%2020260903193120.png)

<br> Let's take a comparison look between the two.

You can see some difference. But specifically the User Name difference, which ties to the UID object for the user. Notice how there was no conflict before the manually entered account was `Bob.Boberton` and the automated one followed the company format of `Employee.Boberton`.

![](images/Pasted%20image%2020260903193409.png)
![](images/Pasted%20image%2020260903193445.png)

<br> I'll do a little cleanup here and get rid of the manually entered account for now. And to note, let's say the incorrect Bob account was removed. Running the same SimplifyIAM task would bring the account back, as it needs to reconcile with the HR "source of truth" table.

![](images/Pasted%20image%2020260903193628.png)

## Mover - Employee promotion or demotion

Now for the second part of the JML workflow, Moving an employee around.

I had to do same modifications to allow for this to work, as the SimplifyHR app doesn't include department moves. I go over that here in the [Lessons Learned](#missing-mover-function) section.

Here'll we'll have the Engineer Sophie Muller move to the Sales department. Currently she is listed as an Engineer on both the HR  and SimplifyIAM portal. The Directory doesn't setup OU's by department, so that isn't relevant on this portion.

SimplifyHR:
![](images/Pasted%20image%2020260903194141.png)

SimplifyIAM:
![](images/Pasted%20image%2020260903194343.png)

<br> After modifying the HR file, I now have Sophie Muller in the Sales department on the HR app.

![](images/Pasted%20image%2020260903195105.png)

<br> I then go back to the SimplifyIAM Task portal to run and double check that the department has updated. Success!

![](images/Pasted%20image%2020260903195236.png)

<br> Obviously when it comes to Mover function there will follow other tasks like permission changes and verifications, etc. I do not have that setup and this is a simple demonstration for now.

## Leaver

The last step is Leaver. There are going to be times when an employee is no longer with an organization. HR will place the ticket it, then the Employee account is deactivated by IT for security purposes. The account is not immediately remove, as there is usually a grace period to have access to the users data for business operations or HR for continuity reasons. Like excels sheets the employee owned or to conclude an investigation.

Here we will have a termination request for Laura Martinez.

![](images/Pasted%20image%2020260903195725.png)

<br> In the HR system, she now shows as "Terminated". But SimplifyIAM and the Directory still have her as active. We'll start the same automation task from SimplifyIAM to update the two services.

![](images/Pasted%20image%2020260903195910.png)

<br> After the task has ran, I go to the "All Users" table and can see a new icon next to Laura's name.

![](images/Pasted%20image%2020260903200002.png)

<br> When I click on her account, I can see that she is shown as "Disabled".

![](images/Pasted%20image%2020260903200059.png)

<br> On the OpenLDAP Directory, I hit refresh and I see her now in the "inactive" OU.

![](images/Pasted%20image%2020260903200155.png)

<br> You'll also notice back on the HR portal that a "Reactivate" button is available. Which will move the user back from the "inactive" OU to the "people" OU once the SimplifyIAM task is ran.

![](images/Pasted%20image%2020260903202138.png)

<br> Once I select this, Laura is now showing back as "Active" in both SimplifyIAM and the Directory. If you go to Laura's account, you can see the history of her being "Disabled" then "Enabled" again.

![](images/Pasted%20image%2020260903202917.png)

<br> Selecting the oldest entry, you can see Laura as "Disabled".

![](images/Pasted%20image%2020260903203003.png)

# Verification Steps

## Joiner Task

An new employee was hired and needed to be onboard. Here is the employees information:

```
First Name: Bob
Last Name: Boberton
Department: Legal
```

SimplifyHR confirmed.
![](images/Pasted%20image%2020260903203145.png)

SimplifyIAM confirmed:
![](images/Pasted%20image%2020260903203219.png)

Directory confirmed: <br> 
![](images/Pasted%20image%2020260903203420.png)

## Mover Task

An employee has been promoted from the Engineering department to the Sales department. Employee Sophie Muller needs to be update to the Sales department.

SimplifyHR confirmed:
![](images/Pasted%20image%2020260903203631.png)

SimplifyIAM confirmed:
![](images/Pasted%20image%2020260903203653.png)

## Leaver Task

An employee's last day is today and needs to go through the Termination process flow. The employee Laura Martinez needs to be deprovisioned.

SimplifyHR confirmed:
![](images/Pasted%20image%2020260903204400.png)

SimplifyIAM confirmed:
![](images/Pasted%20image%2020260903204658.png)

Directory confirmed:
![](images/Pasted%20image%2020260903204728.png)

---
# Lessons Learned

## Missing Mover Function

I wanted to do a JML demo, but I realized the HR app for the lab doesn't have a function to update the employee department. Which almost led me away from using the lab, but there really is a lot of good functionality and resources added here. I noticed the top of the HR portal shows the path to the `hr.csv`, so I decided to poke around.

![](images/Pasted%20image%2020260903194620.png)

Once I got to the file, I then opened it to modify the file and see if it would actually update on the portal.

![](images/Pasted%20image%2020260903194835.png)

I remembered the Department field was a drop down when adding the new employees, so I went back to double check spelling, capitalization, etc.

![](images/Pasted%20image%2020260903194944.png)

Then changed Sophie to Sales from Engineering.

![](images/Pasted%20image%2020260903195002.png)

When I returned, she was showing in the Sales department! Success!

![](images/Pasted%20image%2020260903195059.png)
# Summary

This is a nice little playground to learn different functions between midPoint and LDAP.

MidPoint itself has more features that I'll play around with. Like creating schedules for the reconciliation tasks. It is pretty deep and intimidating for my first time using it. I see a lot of Entra ID functions.

The directory and the HR app are missing some functionality to do some Mover functions. Like department, OU moves and permission changes.

In the end, it was a good way to demonstrate a JML process flow. Having automation available is a plus, as I'm always going to be a fan of automation. 

Plus having uniform actions take play over the different services is always a plus.

---

# Resources
- [SimplifyIAM](https://www.skool.com/simplify-iam-6792) for labs and other IAM related lessons
- [midPoint](https://evolveum.com/midpoint/) for the IGA platform
- [OpenLDAP](https://www.openldap.org/) for the LDAP platform

