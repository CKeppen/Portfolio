# Elastic SIEM

Author: Cody Keppen [LinkedIn Profile](https://www.linkedin.com/in/cody-keppen-a09068355/) <br>
Date: 06/02/2026 <br>
Source: [Investigating the ELK 101 - THM](https://tryhackme.com/room/investigatingwithelk101) </br>

---
# Objective:

 A quick demonstration of using the Elastic SIEM, which uses the Elastict stack ELK. ELK is made up of three open-source tools that are commonly used together to collect, store, analyze, and visualize data. Elasticsearch, Logstash, and Kibana (ELK). 

**Beats**:
Agents that handle the transfer of data.

**Logstash**:
Data processing engine made up of an Input that the user chooses, filters the users wants and where the user wants the data to be sent.

**Elasticsearch**:
Full-text search and analytics engine for JSON-formatted documents.

**Kibana**:
Web-based data visualization tool using the data for a dashboard.

![](images/Pasted%20image%2020251227160152.png)

## Concepts

- [Filtering](#filtering)
- [Quick filter](#quick-filter)
- [Investigating Spikes](#investigating-spikes)
- [Excluding Data](#excluding-data)
- [Custom Tables](#custom-tables)
- [KQL](#kql)
- [Visualization](#visualization)

---
# Demonstration:

## Filtering

The Discover tab is the main tab to see ingested logs for searching and filtering.

In this example I am looking at logs from Dec 31, 2021 to Feb 2, 2022.

![](images/Pasted%20image%2020251227162712.png)

## Quick filter

I want to see the IP source that has the highest number of connections. I go to the Index column on the left, select `source_ip`, and see the top source IP connections.

![](images/Pasted%20image%2020251227162906.png)

## Investigating spikes

In the previous timeline screen, I can see a spike in connections on Jan 11. I click on that bar to get a breakdown of that date.

With that date now filtered on, I go back to the index column to find the `source_ip` to find which has the most connections. The quick filter shows `172.201.60.191` as the highest. Which is worth investigating.

![](images/Pasted%20image%2020251227163158.png)

## Excluding Data

Here I need to look at connection `238.163.231.224` but not in the state of New York.

I set my date timeline from Dec to Feb. I go to the search bar and use `source_ip : 238.163.231.224` then go to the index column, select `source_state` and click the minus sign for New York. Which creates a `NOT` filter for New York, only leaving Michigan as the only state.

![](images/Pasted%20image%2020251227163734.png)

I could also use the `+ Add Filter` option at the top in the following manner for the same result.

![](images/Pasted%20image%2020251227163846.png)

## Custom tables

I can also make a quick custom table if there is specific data I want to see in the logs when scrolling through. In this example, I open up an event log, then clicked the `Toggle column in table` button that has a blue plus sign.

This creates a `Selected fields` section in the index column on the left showing what fields I have chosen for the custom table.

Now on the right side, I can quickly see these data points when scrolling through the logs.

![](images/Pasted%20image%2020251227164212.png)

## KQL

Let's say I want to see all the logs in the United States with a user name of either James or Albert. I'd use the following KQL (Kibana Query Language) query,

`Source_Country : "United States" AND (UserName : James OR Albert)`

In another example, there was a terminated employee, Johny Brown, on January 1st. It was discovered some of his access were not fully removed. 

I'd do a search with the following to see if there were any connections for further review, which there does appear to be one connection worth investigating.

`UserName: "Johny Brown"`

![](images/Pasted%20image%2020251227170128.png)

## Visualization

With the above examples, there does appear to be an issue with failed logins. With Elastic, I have the ability to use data to create visualizations and dashboards for a better presentation.

In the below, I'll go to `source_ip` then click `Visualize` at the bottom of the Top 5 values.

![](images/Pasted%20image%2020251227171041.png)

I'll now be in the Visualize Library dashboard, where I'll create this filter to find the failed logins to report on.

![](images/Pasted%20image%2020251227171336.png)

I'm going to change the visualization chart to a table. Then click and drag `UserName` and `source_ip` to the Rows section of the table.

![](images/Pasted%20image%2020251227171727.png)

I will do a little cleanup on the title of the data sets for better presentation.

![](images/Pasted%20image%2020251227171809.png)

There is also the ability to create multiple windows and save the visualizations into a live dashboard.

![](images/Pasted%20image%2020251227172051.png)

---
## Summary:

Elastic is a user friendly SIEM that makes investigating and reporting easy. Filtering is intuitive, and it features a nice quick glance of the top 5 values for the different categories you select.

The dashboard visualizations makes looking for high level anomaly detection easier, as well as a nice visual for executive summary reviews for non-technical people.

And of course, as a SIEM, making a central location to aggregate multiple log sources is impactful when needing to cross-reference events that occur across multiple logs.




