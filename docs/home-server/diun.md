---
tags:
  - Diun
  - Docker
---

![diun.png](../assets/images/diun.png)


## DIUN: Docker Image Update Notifier

I believe we all agree that automate updates (in general) might not be a good idea. If we are talking about Docker images, we need to monitor the registries to see if a new release is out, than read the release notes and decide whether to update to the new version or not.
Well, what if I tell you that we can at least skip the monitoring part and get a notification to our phone when a new version of an image is released? Maybe it doesn’t seem much, but if you like to manage your time as efficiently as possible, like I do, you can see the benefit of it.
Let me introduce you Diun! Long story short, Diun receives a notification when a Docker image is updated on a Docker registry.
As you can see in the documentation, you can run Diun in different ways, but I chose docker-compose.
What a surprise!

```bash
mkdir diun
cd diun
touch docker-compose.yml
nano docker-compose.yml
```
`docker-compose.yml`

```yaml
---
version: "3.5"

services:
  diun:
    image: crazymax/diun:latest
    container_name: diun
    command: serve
    volumes:
      - "./data:/data"
      - "/var/run/docker.sock:/var/run/docker.sock"
    environment:
      - "TZ=Europe/London"
      - "LOG_LEVEL=info"
      - "LOG_JSON=false"
      - "DIUN_WATCH_WORKERS=20"
      - "DIUN_WATCH_SCHEDULE=0 0 09 * * *"
      - "DIUN_PROVIDERS_DOCKER=true"
    labels:
      - "diun.enable=true"
    restart: unless-stopped
```
I changed the schedule to make it run every day at 09:00 (take a look at CRON Expressions Format if you need).
But how do we get notified?
Diun supports many services, but for this kind of projects I usually choose Discord.
Set up a Discord webhook which will send the notifications and, once done, add some environment variables in the docker-compose file:

```yaml
- "DIUN_NOTIF_DISCORD_WEBHOOKURL=https://discord.com/api/webhooks/1230110122752217159/OWcRAUUbT3QFUSs3z35TCD9dUkM26PH0iNY1RNdgqlzoAMC81SZM_iwQ5wuyY8cyFoqL" # change to your webhook
# - "DIUN_NOTIF_DISCORD_MENTIONS" # (comma separated)
- "DIUN_NOTIF_DISCORD_RENDERFIELDS=true"
- "DIUN_NOTIF_DISCORD_TIMEOUT=10s"
 # - "DIUN_NOTIF_DISCORD_TEMPLATEBODY"
```
You can reference the documentation to find more details, including the default templateBody which you can modify according to your needs. Highly recommended if you run Diun on multiple machines.
Now we can finally spin it up:

```bash
sudo docker-compose up -d
```

A quick note: to make sure everything is working fine, run a test using:

```bash
sudo docker-compose exec diun diun notif test
```

You should receive a notification on your discord server from your new Diun set up.

