---
tags:
  - Adguard
  - Docker
---

![adguard.png](../assets/images/adguard.png)

# AdGuard 

AdGuard is a comprehensive, customizable network software that blocks ads, trackers, and malicious websites. It enhances user privacy and improves browsing speed and security.

Docker Command Configuration for AdGuard
To install and run AdGuard using Docker, you can use the following command. This command sets up AdGuard Home, a network-wide software for blocking ads & tracking.

# Docker Run Command

```yaml
docker run --name adguardhome \
    --restart unless-stopped \
    -v /home/beardnetwork/docker/adguard/workdir:/opt/adguardhome/work \
    -v /home/beardnetwork/docker/adguard/confdir:/opt/adguardhome/conf \
    -p 192.168.68.120:53:53/tcp -p 192.168.68.120:53:53/udp \
    -p 80:80/tcp -p 443:443/tcp -p 443:443/udp -p 3000:3000/tcp \
    -p 853:853/tcp \
    -p 784:784/udp -p 853:853/udp -p 8853:8853/udp \
    -p 5443:5443/tcp -p 5443:5443/udp \
    -d adguard/adguardhome
```

!!! note

On line 5, a hardcoded IP address of the server is used due to a conflict with a broadcast address. Change the IP address and the volume directory to match your local server setup.

# Docker Hub Link

AdGuard Home on Docker Hub

https://hub.docker.com/r/adguard/adguardhome