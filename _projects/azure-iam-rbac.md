---
report_number: "001"
layout: project
title: Post-Breach Reconstruction
description: Moving through the cyber kill chain on a live Azure network
date: 2026-10-18
featured: true
tools:
  - Azure
  - KQL
  - Sentinel
---

## Project Overview
Capstone Project in MadHat.io
Ten stages. Each maps to one phase of the kill chain, then a final report.
Submit each value exactly as it appears in the portal or the query result. Only the final after-action report (stage 10) uses the MadHat{...} format. 


## Objectives

Operative, three days ago an attacker reached the customer PII blob container in our production tenant and exfiltrated it to an external endpoint. The SOC closed the kill chain on the live attack twelve hours ago. You are now the post-incident analyst. Your job is to reconstruct every stage of the breach using only the telemetry and configuration the platform left behind, then file the after-action report so the team can fix what they broke.


## Investigation Process

### Stage 1: Entry
The breach started in identity, the way most breaches do. Recently, a Mad Hat Labs employee got phished and their credentials were captured on a typosquat login page. The attacker waited several hours, then signed in using the stolen session token from a location the user has never been to. And unfortunately the sign-in succeeded because passkey (with a proximity check) was not enforced on the account.  The lab's sign-in telemetry is in SigninLogs_LAB_CL.

Ok, so we need to figure out who was able to successfully logon or with what identity they were able to take. To do that I will start by querying for anomalous sign on attempts via Microsoft Sentinel.

We are already given that we need to query SigninLogs_LAB_CL so that helps.

Lets build a query:

SigninLogs_LAB_CL
| where TimeGenerated > ago(180d)
| take 10

here we get the last 10 logs from sign ins. We see that there is a UserDisplayName we can sort by

<img width="865" height="201" alt="{C2A05C8A-401C-4341-9B08-C3204B5C72A4}" src="https://github.com/user-attachments/assets/72cdb914-5993-432a-ab47-dd34d9b4b082" />


SigninLogs_LAB_CL
| where TimeGenerated > ago(180d)
| summarize count() by UserDisplayName
| sort by count_

<img width="384" height="55" alt="{6FD8A218-E7BC-4895-BD21-B7DD205AEE82}" src="https://github.com/user-attachments/assets/d01a4cba-7b64-435f-84d2-0708dd7bc860" />

Dana "white" Wright loves logging in as much as watching UFC. Lets dive deeper into his activity


I started by sorting Dana Wright's sign-in activity in ascending order so I could see the events in the order they happened. I noticed that he had multiple successful sign-ins from Austin, Texas. After that, there were several failed sign-in attempts from Sao Paulo, Brazil, due to an invalid password. These attempts were using single-factor authentication.

Following those failed attempts, I noticed one with a result type of 0, which indicates a successful authentication. There were then several more unsuccessful attempts after and finsished with successful logins from the original Texas IP.

This all happened from Jul 26, 2026 12:08:13 PM - Jul 26, 2026 12:08:14 PM

Idk about you, but I cant log in 80 times in a minute.

 Submit the compromised account's UPN exactly as it appears in the UserPrincipalName column:

 dwright@madhatlabs.io

 <img width="750" height="106" alt="{A7C8FE43-E47D-49B6-9DC0-4FF73108DDE4}" src="https://github.com/user-attachments/assets/729e09de-1188-498c-83ab-63b8a1cdca8d" />


### Stage 2: Foothold

Stealing an employees credentials and re-using them is often noticed by built in protections like Microsoft's Entra ID Protection. The attacker knew it was only a matter of time before their access was cut off. So, the very next thing they did was create a non-human identity that could keep operating after the password was reset. They registered an app inside the tenant under a name that looks like a normal internal backup service. To the casual Azure portal viewer, it blends in. To the post-incident analyst, the recent creation date and unfamiliar owner are glaring red flags.
Now find it.

Ok, so we need to find what app was created. Azure stores this record via App Registrations, lets go.

<img width="1233" height="494" alt="{F8A5206F-A32D-49C0-BD8A-F6017A2B04A2}" src="https://github.com/user-attachments/assets/4ab4b24e-062f-4727-a59e-1de01a9fb72d" />

hmmm 7/26/2026....
Jul 26, 2026 12:08:13 PM

Submit that app's display name exactly.

MadHatLabs-BackUp-Sync-Service

<img width="729" height="113" alt="{FBAE79EB-8938-4F06-915C-F107D81A2B1F}" src="https://github.com/user-attachments/assets/2b36dceb-65b1-4188-a6b2-f4a004e65fc1" />

### Stage 3: Persist

 The attacker added a client secret to the rogue app with an expiry far enough out that no automated rotation policy would ever clear it. As long as that secret lives, the attacker has a credential the IT team will not see expire.On the rogue app, find the client secret engineered to outlive any rotation policy. Submit that secret's ID value exactly.

 <img width="758" height="32" alt="{AAE1B50A-CB25-477E-B443-871399A9BC19}" src="https://github.com/user-attachments/assets/c7ddcd91-e5b4-482e-b364-3fccbdaeec78" />

12/31/2099 seems excessive

29568ae5-645f-444d-99d6-fec131891ff0

<img width="725" height="108" alt="{C4909AF4-E042-4190-B59E-00B27D8EED73}" src="https://github.com/user-attachments/assets/c44e52ae-5c8a-4d57-b712-0051515e3580" />


### Stage 4: Escalate

A credential is useless without permissions attached to it to do anything and call something. The attacker needed enough access to read secrets out of Key Vault, so they granted the rogue service principal a Key Vault data-plane role directly on the vault. On kv-madhat-lab-chap10, find that role and submit its exact name. 

All we asked to do here is to find what the literal name of the role that can assigned to perform these actions. Lets go find the roles on this key vault. 
We will download it to a csv

<img width="1136" height="43" alt="{ED549D8B-F891-4BA0-89B5-03226B41B86D}" src="https://github.com/user-attachments/assets/394b7a36-3374-434f-ab29-e92927d39c1f" />

Here she is.
Key Vault Secrets User

<img width="729" height="110" alt="{CA3F418F-8D81-4F8B-A3E4-A28FBEEB8CB9}" src="https://github.com/user-attachments/assets/6a9c3bcd-d1ad-41a4-9b34-b2679b7ec956" />


### Stage 5: Evade

The attacker got lazy and did not bother covering their tracks for the actions they took next - the secret pull, the storage account access, and the blob download - because they knew the resource they touched had never had its diagnostic settings turned on.  Find the storage account that has no diagnostic settings and Submit that storage account's name exactly.

<img width="536" height="81" alt="{6358E62E-6A20-48E4-85C1-00537066722C}" src="https://github.com/user-attachments/assets/c4b4365b-7bec-4d05-a7b9-00fc75ca26d8" />

There are 2 storage accounts, lets start with stmadhatlegacy

<img width="644" height="154" alt="{6CED33E0-F59A-4E0C-B960-1F6793021B59}" src="https://github.com/user-attachments/assets/a2e26d32-2a49-4ae3-80e2-4fa223dfb264" />

disabled. 

and the other?

<img width="620" height="166" alt="{0F5DE334-1927-4308-95D9-B6F14E6F6A1A}" src="https://github.com/user-attachments/assets/78020f76-b352-42e8-a246-3bd6e0a9dda5" />

at the file level yes

stmadhatlegacy

<img width="770" height="117" alt="{9DC938AC-BC6A-4B95-9B73-5D39C86C185C}" src="https://github.com/user-attachments/assets/8a645821-029d-4be8-83da-5dadbc27496c" />


### Stage 6: Credential Access

The attacker is now in control of that vault role, authenticated as the rogue service principal and read a secret out of the vault. The storage audit was dark, but the vault's own audit stream was on the whole time and that is what gives you helpful insight to be able to piece what happened together.
Query the table where Key Vault audit events land, filter to a SecretGet by the rogue app, and read which secret was pulled. Submit that secret's name exactly.

Lets go to the Log Analytics Workspace and pull some logs shall we.

We will also need the rogue apps ID which we got earlier when we identified it - :58dbc87c-5b75-484a-9042-831feb094a16

Lets do an initial query to see the fields:

AzureDiagnostics
| where TimeGenerated > ago(180d)
| take 10

<img width="740" height="282" alt="{1A93EBA1-B9C3-4DE2-99E6-3E04B0C9C9AD}" src="https://github.com/user-attachments/assets/dd37eac1-762d-4bbc-b2cd-fc0d53540474" />

Here is how we will be able to sort by activity happening on this APP.

AzureDiagnostics
| where TimeGenerated > ago(180d)
| where identity_claim_appid_g contains "58dbc87c-5b75-484a-9042-831feb094a16"

<img width="760" height="274" alt="{7F5E62D7-A5A5-4C34-9B26-A069E1F6B922}" src="https://github.com/user-attachments/assets/1373b99a-29a5-4a47-98a0-fa36c46cd2e0" />

We see a secret successfully pulled using SecretGet operation. 

customer-database-connection-string

<img width="726" height="107" alt="{730A7FE3-0EA6-42DD-A53B-1114CF92D034}" src="https://github.com/user-attachments/assets/f2d53877-2747-4773-a638-13e17edc44e5" />


### Stage 7: Collect

The secret you just traced contained a Storage account connection string. The attacker’s first move was to try and download the data directly to their external attack device, but they hit a wall: the storage account's network settings were locked down to only allow traffic from the internal corporate VNet. The connection string was useless on the open internet. To bypass the network firewall, the attacker used the hijacked session to log into the employee's corporate Cloud PC. Now sitting inside the trusted VNet, they used the connection string to authenticate and download the PII container locally to that virtual desktop.
Submit that container's name exactly.

Lets check that stmadhatlegacy storage account again. 

<img width="1003" height="129" alt="{F018F029-B9CD-46D4-A39B-5F0F9705F335}" src="https://github.com/user-attachments/assets/74ee6161-1d58-4941-a5b8-d9c9e0632c76" />

customer-pii-export

<img width="749" height="113" alt="{E5DC2E24-71FB-49B5-8E31-6FFC2911FFED}" src="https://github.com/user-attachments/assets/5ccb37f5-745b-44c0-aeb9-cab1149a8ea4" />



### Stage 8: Exfil

The download happened, but now the data was sitting on the employee's internal Cloud PC. How did they get it out of the network? The attacker couldn't push the data straight out to the public internet because Mad Hat Lab's Network Security Group (NSG) and egress proxy block bulk HTTPS transfers from user subnets. So, the attacker got fancy and used the one protocol that is never blocked, because blocking it breaks everything: DNS. 
Submit the domain the data was tunneled to.

We are given this info:

<img width="769" height="235" alt="{6B7F0AA0-02FF-4E1D-B346-F82A9E31995D}" src="https://github.com/user-attachments/assets/7dea4aef-bbc7-4886-bf2d-c4a3838cd1f5" />

We are also given a custom log: DnsEvents_LAB_CL

Lets mosey back over to Advanced Hunting in Defender.

First lets narrow down the time

DnsEvents_LAB_CL
|where TimeGenerated between (datetime(2026-07-26 08:30:00) .. datetime(2026-07-28 08:40:00))

<img width="880" height="282" alt="{6F24C164-F7B1-4349-B518-022AE7BE4697}" src="https://github.com/user-attachments/assets/71a822a1-4ffa-430b-8824-4f5d071cc2cc" />

Way too many results still.

We are told the Q type is commonly (TXT) for this technique so lets try that.

DnsEvents_LAB_CL
|where TimeGenerated between (datetime(2026-07-26 08:30:00) .. datetime(2026-07-28 08:40:00))
|where QueryType contains "TXT"

<img width="895" height="289" alt="{69001BD0-A0D0-471B-B76D-20CAEFF4481D}" src="https://github.com/user-attachments/assets/fda50e6d-c5b8-4124-b6b1-7581eaa1ebbd" />

500+ lookup query events from the same IP and look at that name

<img width="871" height="90" alt="{4DA72BB6-0D71-47E3-A99F-A5FAF910BBF3}" src="https://github.com/user-attachments/assets/50f3cbf1-cb22-4579-91ea-d93228f9b3f8" />

5LF4RLKGNQW6MV7D23SIJ5AGCIEMCZFEB3N5ISWJRS5ACIB2DEBHLIM.exfil-c2.badactor.io

exfil-c2.badactor.io

<img width="750" height="121" alt="{042DAA30-89CB-467C-840C-CBDEA146386E}" src="https://github.com/user-attachments/assets/6a036754-80d2-45dc-aa07-4290fea97dd5" />


### Stage 2: Foothold

Stage 9: Contain

The SOC briefed you that they closed the kill chain twelve hours ago. Their containment action was to disable the rogue service principal so its credentials, even though they technically still exist, can no longer be used to obtain tokens. If you disable the SP and every dangling capability the attacker holds (the never-expiring secret, the vault role assignment, the planted owner) becomes useless. The credential strings still exist, but no token request against them will work.
Confirm the SOC actually did it. Submit the value for the status of the Enterprise App exactly as written in the portal.

We go the Enterprise Apps and deselect all filters to be able to search for our app.

<img width="926" height="242" alt="{03E44B87-F69C-4F69-8E57-E6837C50C137}" src="https://github.com/user-attachments/assets/4a92c775-7885-4608-8aa8-90282099770e" />

We then view its properties:

<img width="887" height="546" alt="{4733F992-8A2B-478E-BCD6-87547D3498CF}" src="https://github.com/user-attachments/assets/76bdf30b-135e-4b71-9e46-d1a736f7d49f" />

Indeed nobody can sign into this now.

Deactivated

<img width="744" height="117" alt="{EEF46D1A-56DE-482E-8B67-05B0F9453EF6}" src="https://github.com/user-attachments/assets/76711677-5697-4662-b240-2c0932f5cb4d" />


### Stage 10: After-Action Report

You successfully traced the entry point, identified the foothold app, exposed the persistence credential, named the over-privileged grant, found the audit blind spot the attacker exploited, traced the Key Vault secret pull, identified the data target, documented the exfil channel, and verified containment. You did it from a read-only account across multiple surfaces of Microsoft's cloud security stack.


The last flag lives where real after-action incident reports live: a file in a storage account. Open stmadhatproddata (the storage account whose diagnostic settings were actually turned on, from stage 5). Navigate to Containers, open the case-closed container, and read the contents of case-closed.txt. The report body ends with your final flag!🎉

Here it is:

<img width="716" height="190" alt="{20D0E083-9071-4485-9ED6-4C6832646671}" src="https://github.com/user-attachments/assets/4f37bc81-55a8-402f-98fa-f76247335c10" />

<img width="516" height="292" alt="{A6487A7C-23D1-471C-8103-B658D5434129}" src="https://github.com/user-attachments/assets/66ae18f6-ff6a-48a9-ae25-23459f75d185" />

MadHat{thE_breACh_audit_contiNueS_WiTH_each_dePlOy}

<img width="748" height="114" alt="{E4647EA3-A203-4842-A956-62A2291E2589}" src="https://github.com/user-attachments/assets/78f2f7a8-06df-4356-86b8-5f197a7a0079" />


## Final Notes

Hooray, I have a super awesome certificate 

[View Credential](https://madhat.io/pages/credential/mhl-azf-9341051830347)

<img width="854" height="400" alt="{1809CB90-4426-4E4C-8D78-C73EC69FB30A}" src="https://github.com/user-attachments/assets/5e339f27-3227-45d7-a14d-07924b0ab464" />

I'll take top 5 all day on this extremely competitive leaderboard... ha first mover advantage

