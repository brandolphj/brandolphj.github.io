---
report_number: "002"
title: "Honeypot"
description: "A brief description of the project."
tools:
  - Azure
  - Docker
  - Cowrie
---
## Background

Your explanation here.

## TLDR

https://vasttoast2261.grafana.net/public-dashboards/9bc39cf725af40488b8775c93f51d4b4

### Setup

1. Create Exposed Network and Machine

We don't want to expose the rest of my TCD corporate network to nation state bots so lets create a segmented network for our honeypot VM to live

<img width="1248" height="245" alt="{955337AE-B8D1-44AB-AD34-49F4689D1767}" src="https://github.com/user-attachments/assets/afa744fb-4b60-4fe3-b8f8-85a0ca8336d8" />

That's a good start. We still need to add access controls with a NSG.

<img width="1249" height="159" alt="{E5CC3CD5-33C2-4259-BB82-4380204E46C9}" src="https://github.com/user-attachments/assets/33491f39-b328-471d-a108-0ffacbac2b24" />

Here we attach a a new blank NSG to our previously created honey pot subnet. 

<img width="1036" height="276" alt="{FAEB5FC8-CC10-4DDC-AAA9-9EDB2705FC6E}" src="https://github.com/user-attachments/assets/56d6f778-47d1-4268-8f13-73654646b9c6" />

Here are the rules for a default NSG. The biggest issue is that our deny traffic rules are too low in priority. We dont want any outbound traffic to occur for this honeypot, since we only want to receive and monitor inbound traffic.

<img width="1041" height="56" alt="{EDC58616-D6FD-4CF8-8CEF-8D91895DDD0E}" src="https://github.com/user-attachments/assets/235c09f5-4d39-404d-951f-b214b8d3b042" />

Just in case, we also will deny all inbound traffic from any part of our internal network, as there is no reason our systems need to communicate with it.

<img width="1043" height="65" alt="{0C776009-400C-4B80-99B9-32D0B478ED19}" src="https://github.com/user-attachments/assets/fe0a6959-589b-4aa0-89da-46756a84599f" />


Turns out azure is constrained at the moment and there are no available VMs in the West Region so I will have to create all this again on the East Region, BRB.

So our VM is now running and now we need to provide it an internet facing address so it is able to connect with and download Cowrie :

<img width="432" height="572" alt="{09A72A77-19D0-4FA2-99FD-702ADF97CD0F}" src="https://github.com/user-attachments/assets/646dc675-4a63-468d-9b48-a3cc08599c23" />

Right now all traffic is denied, so we need to open up a temporary HTTPS channel via NSG to do so as well:

<img width="431" height="558" alt="{0B696C23-14D7-4905-BA16-FCAABA024843}" src="https://github.com/user-attachments/assets/c5e83eeb-fbde-4d26-8d38-c740065e4ec5" />

Time to test!

curl -I --connect-timeout 5 --max-time 10 https://bethesda.net

Enable succeeded: 
[stdout]
HTTP/2 302 
content-type: text/plain; charset=utf-8
content-length: 0
location: https://bethesda.net/en-US
server: awselb/2.0
date: Thu, 22 Oct 2026 18:53:21 GMT
x-powered-by: Express
vary: Accept

We are in business. Lets get to downloading Docker

First get the latest and greatest:

apt-get update

set -e

DEBIAN_FRONTEND=noninteractive apt-get install -y docker.io

systemctl enable --now docker

Enable succeeded: 
[stdout]
Docker version 29.1.3, build 29.1.3-0ubuntu3~24.04.2
active

[stderr]

Container is good. Now for Cowrie.

docker pull cowrie/cowrie:latest

Digest: sha256:69e3317c6ecf8811d18faecb8a2fcedee0af1f0f864b50de07b688c7409f515c
Status: Downloaded newer image for cowrie/cowrie:latest
docker.io/cowrie/cowrie:latest

Cowrie's Docker image downloaded successfully.


I found this command online to check the current configuration that we have deployed:

docker image inspect cowrie/cowrie:latest \
  --format 'User: {{.Config.User}}
Exposed ports: {{json .Config.ExposedPorts}}
Working directory: {{.Config.WorkingDir}}
Volumes: {{json .Config.Volumes}}'


Returns:
User: 999:999
Exposed ports: {"2222/tcp":{},"2223/tcp":{}}
Working directory: /cowrie/cowrie-git
Volumes: {"/cowrie/cowrie-git/etc":{},"/cowrie/cowrie-git/var":{}}

999 means that it is a non-root user and we have 2 exposed ports. 
-2222 is Cowries SSH port
-2223 is an additional custom port

Containers can be deleted and recreated. Without persistent storage, important data may be lost.
For our honeypot, we want to preserve attack logs even if we replace the Cowrie container.

We need to create two Docker volumes:
-cowrie-etc for Configuration
-cowrie-var for Logs and runtime data

docker volume create cowrie-etc
docker volume create cowrie-var

docker volume ls

Enable succeeded: 
[stdout]
cowrie-etc
cowrie-var
DRIVER    VOLUME NAME
local     cowrie-etc
local     cowrie-var


Docker setup is done.

### Cowrie Config

We need to free up SSH access on port 22 so Cowrie can do its thang.

systemctl disable --now ssh.socket - disables auto-start
systemctl disable --now ssh.service - disables SSH

Now for the full config file:

mkdir -p /opt/cowrie (Create directory)

cat > /opt/cowrie/compose.yaml (Put into file for Docker Compose)

services:
  cowrie:
    image: cowrie/cowrie@sha256:69e3317c6ecf8811d18faecb8a2fcedee0af1f0f864b50de07b688c7409f515c  (This is from our Docker Pull)
    container_name: cowrie
    restart: unless-stopped
    
   "127.0.0.1:2222:2222" (The SSH Port)

  volumes:
  - cowrie-etc:/cowrie/cowrie-git/etc (Working Dir for our volumes)
  - cowrie-var:/cowrie/cowrie-git/var
    
    mem_limit: 512m (Small starting value so I don't get charged out the wazoo)
    cpus: 0.5

    cap_drop:
  - ALL    (Removes Linux capabilities Docker would otherwise grant)

  security_opt:
  - no-new-privileges:true (Prevents processes from gaining privileges through mechanisms such as setuid executables)

volumes: (Because we manually created these)
  cowrie-etc:
    external: true
  cowrie-var:
    external: true

Lets start it now that its settings are configured:

cd /opt/cowrie

docker compose up -d

docker compose ps

Enable succeeded: 
[stdout]
NAME      IMAGE                                                                                   COMMAND                  SERVICE   CREATED        STATUS                  PORTS
cowrie    cowrie/cowrie@sha256:69e3317c6ecf8811d18faecb8a2fcedee0af1f0f864b50de07b688c7409f515c   "/cowrie/cowrie-env/…"   cowrie    1 second ago   Up Less than a second   127.0.0.1:2222->2222/tcp, 2223/tcp

[stderr]
 Network cowrie_default  Creating
 Network cowrie_default  Created
 Container cowrie  Creating
 Container cowrie  Created
 Container cowrie  Starting
 Container cowrie  Started

 So it started but is it working?

 docker logs --tail 50 cowrie

 This pulls the first 50 logs 

2026-10-22T19:38:49.460072Z [-] CowrieSSHFactory starting on 2222
2026-10-22T19:38:49.460156Z [cowrie.ssh.factory.CowrieSSHFactory#info] Starting factory <cowrie.ssh.factory.CowrieSSHFactory object at 0x771cd7d24ec0>
2026-10-22T19:38:49.551848Z [cowrie.ssh.factory.CowrieSSHFactory#info] Ready to accept SSH connections

 Indeed it is working. Lets verify its collecting Logs as well:

nc -vz 127.0.0.1 2222

this is just a simple ssh command- lets see the logs.

 docker logs --tail 10 cowrie

 2026-10-22T19:51:14.115707Z [ssh,8663e701dcf0,172.18.0.1] New connection: 172.18.0.1:42396 (172.18.0.2:2222) [session: 8663e701dcf0]
2026-10-22T19:51:14.116563Z [HoneyPotSSHTransport,5,172.18.0.1] connection lost
2026-10-22T19:51:14.117004Z [ssh,8663e701dcf0,172.18.0.1] Connection lost after 1 milliseconds

It logged that netcat command, nice!

### Open the Network

We no longer need our temporary HTTPS rule now that cowrie is up and running so lets delete that rule.

Lets also just double check our SSH ports are disabled

systemctl is-enabled ssh.socket ssh.service

disabled
disabled

Check,check. 

We want Cowrie to accept internet connections on standard SSH port 22, while still running internally on port 2222. We have to change that via the compose file we made earlier, we have no interactive terminal so we have to edit this way. sed edits text in a file and the left is the old mapping, right is the new allowing connections on all VM network interfaces.

sed -i 's/127.0.0.1:2222:2222/22:2222/' /opt/cowrie/compose.yaml

grep -A 2 'ports:' /opt/cowrie/compose.yaml (checking if it worked)

ports:
      - "22:2222"

Now lets restart it:

docker compose -f /opt/cowrie/compose.yaml up -d

 Container cowrie  Recreate
 Container cowrie  Recreated
 Container cowrie  Starting
 Container cowrie  Started

### Open The Gate

 Its go time 

 <img width="429" height="564" alt="{2E73FCF7-1C6B-43B5-80C4-6B89D59B9BAD}" src="https://github.com/user-attachments/assets/7c3657d2-d8b3-4d6c-ba64-73f67e5b50f9" />

 Lets test usinng my own PC

PS C:\WINDOWS\system32> Test-NetConnection 20.85.212.43 -Port 22                                                                                                                                                                                                                                                                                                        ComputerName     : 20.85.212.43
RemoteAddress    : 20.85.212.43
RemotePort       : 22
InterfaceAlias   : Wi-Fi
SourceAddress    : 192.168.1.168
TcpTestSucceeded : True

The honeypot is reachable from the internet.

### Visualize

We will need a way to gather the Cowrie logs into our Log Analytic Workspace in Azure. For that we will use the Azure Monitor Service.

First we create a Data Collection Endpoint for the data collection

<img width="489" height="272" alt="{1D6D42B5-A1B9-44C8-81C5-1D9F250E41E2}" src="https://github.com/user-attachments/assets/b855ae5e-a943-4a90-8cae-8eefd88704fb" />

Next we need a custom Cowrie_CL to Query 


First we create a Data Collection Rule to configure how Cowries JSON logs are collected

<img width="1226" height="167" alt="{B0B8D6E7-2283-433E-B180-D2B933478973}" src="https://github.com/user-attachments/assets/3c714d72-e215-48a1-a07e-8a6184a50611" />


We also need to add a rule to allow our Azure Monitor to connect to our VM.

<img width="429" height="559" alt="{837B171C-8BB4-4E4E-A1BF-37D4F963CCD8}" src="https://github.com/user-attachments/assets/ee968098-1afb-4b37-9105-c772a6906c7f" />

Lets go see if our table starts populating:

<img width="1219" height="541" alt="{2EC68436-55D2-4ED5-B378-D0BBDC144DB5}" src="https://github.com/user-attachments/assets/b3a74e04-a647-4452-9821-7971946c7f4f" />

### Dashboard

We will create our dashboard in Azure Workbooks where we can create visuals based off querys to our Cowrie_CL Table


###  Dashboard Upgrade

Grafana App Registration

<img width="1234" height="540" alt="{81FC7EE1-DEDB-45AE-9978-625DBB2C9F81}" src="https://github.com/user-attachments/assets/30ca04ea-72a7-4a35-a4a0-5ba17bf8e352" />

Grant Grafana read-only access

<img width="1057" height="356" alt="{10E11DB2-DA08-4CFE-9825-F49CBB949A7C}" src="https://github.com/user-attachments/assets/37609828-ea03-4261-bc5b-b0ee69623915" />




## Results

Your findings here.
