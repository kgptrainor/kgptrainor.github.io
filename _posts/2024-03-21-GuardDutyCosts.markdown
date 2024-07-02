---
layout: post
title:  "K8S Commands"
date:   2024-03-21 12:13:26 +0000
tags: [AWS,Cost]
---

### Usefull commands

We are seeing very high Guard Duty Costs 

How to investigate: (7 days cost)

| Entry | Total | Cost |
|----------|----------|----------|
| Total Entries | 	2549924 | $11.22 |


| Entry | Total | Cost |
|----------|----------|----------|
| StealthWatch | 125266 | $0.55 |
| DivvyCloud | 418954 | $1.84 |
| ECS_STS_Session | 317884| $1.40|
| trustedadvisor | 1288013 | $5.68 |



Cloudtrail Logs 

Total : 

Athena qureies :

**Total Entries in cloudwatch***

SELECT COUNT(*) AS TotalEvents
FROM "cloudtrail_logs_427219654253"
WHERE eventtime >= '2024-04-01T00:00:00Z'
      AND eventtime < '2024-04-08T00:00:00Z';

**Number of times a user or principal appears**

SELECT useridentity.invokedby,
       useridentity.arn,
       useridentity.principalid,
       COUNT(*) AS EventCount
FROM "cloudtrail_logs_427219654253"
WHERE eventtime >= '2024-04-01T00:00:00Z'
      AND eventtime < '2024-04-08T00:00:00Z'
GROUP BY useridentity.invokedby, useridentity.arn, useridentity.principalid
ORDER BY EventCount DESC;


**Total number of times AWSServiceRoleForTrustedAdvisor is seen in Cloudtrail**

SELECT COUNT(*) AS TotalEventCount
FROM "cloudtrail_logs_427219654253"
WHERE eventtime >= '2024-04-01T00:00:00Z'
      AND eventtime < '2024-04-08T00:00:00Z'
      AND useridentity.arn LIKE '%AWSServiceRoleForTrustedAdvisor%';


