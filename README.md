# SaltLAN

Single-server LAN party

## What

The goal of SaltLAN's network and server infrastructure was designed to be small, single-system, and cheap, while meeting all of our requirements: Caching, Game servers, monitoring, and 100Mb/s to the client.

I wanted to build SaltLAN after attending a LAN party in May, 2016. While the LAN was a lot of fun and it featured a great giveaway, awesome unlimited drinks and cool people, however, there were some massive networking problems.

 1. No dedicated LAN servers for games like TF2. People had to try and host local games, but nobody could see/connect to the game.
 2. Bad switches/using home routers instead of an unmanaged switch with distribution switches. Nobody could talk to each other.
 3. Slow internet (20Mbps max). Not everybody had certain games, and the day of the LAN, there was a massive TF2 update. It killed the network several times.
 
 
## Configuration

###Server

//side note:
For anyone who's wondering why I'm releasing **so** much information on the setup and the possibility that this information might lead to a breach of network security:
 1. I am moderately confident in our network security and IDS.
 2. ALL of the network information is readily available to anyone who runs a basic nmap scan at the event.
 3. I want this information to be public. I spent a lot of time piecing pieces of information together from tons of sites and I want it to be available to anybody who looks.

Component | Hardware
--- | ---
**CPU** |Xeon E3 1220v5 
**RAM** |32GB DDR4      
**Storage** | 512GB Samsung EVO SSD
  | 3TB WD BLACK HDD
**Networking**| 3x 1Gbps NICS
  | 1x 1Gbps NIC for WAN
  | 2x 1Gbps NICs bonded for 2Gbps into LAN
  
###Network

Component | Hardware
--- | ---
**Core switch** | 1x Cisco SLM 2048 48 port Gigabit switch
**Disribution switches** | 6+x Netgear Prosafe 24 port 10/100 switch with 2x 1Gbps uplink

###Software

Component | Software
--- | ---
OS | Ubuntu server 16.04
DHCP | Dnsmasq
DNS | Dnsmasq
Caching | Docker
Game servers | Linux Game Server Manager
Usage monitoring | Grafana
Usage storage | InfluxDB
Usage collection | Telegraf

##Address definition

IP Address | Hostname | Port | Interface | Service | Comments
:--- | --- | --- | --- | :--- |--- | 
`10.0.0.1` | `router.saltlan.org` | N/A | `eno2` | Router | Basic NAT and IPv4 forwarding
`10.0.0.1` | `dns.saltlan.org` | `53` | `eno2` | Dnsmasq | DNS
`10.0.0.1` | `dhcp.saltlan.org` | N/A | `eno2` | Dnsmasq | DHCP
`10.0.0.2` | `steam.cache.saltlan.org` | `80` | `eno2:2` |  Docker | Steam cache
`10.0.0.3` | `blizzard.cache.saltlan.org` | `80` |  `eno2:3` |  Docker | Battle.net cache
`10.0.0.4` | `origin.cache.saltlan.org` | `80` | `eno2:4` |  Docker | Origin cache
`10.0.0.5` | `uplay.cache.saltlan.org` | `80` |  `eno2:5` | Docker | Uplay cache
`10.0.0.6 - 9` | Reserved | N/A | N/A | N/A
`10.0.0.10` | `csgo.server.saltlan.org` | `27015` | `eno2:10` |  LGSM | CS:GO server
`10.0.0.11` | `tf2.server.saltlan.org` | `27015` | `eno2:11` |  LGSM | TF2 server
`10.0.0.12` | `q3.server.saltlan.org` | `27015` | `eno2:12` |  LGSM | Quake 3 server
`10.0.0.13` | `insurgency.server.saltlan.org` | `27015` | `eno2:13` |  LGSM | Insurgency server
`10.0.0.14` | `ss3.server.saltlan.org` | `27015` | `eno2:14` |  LGSM | Serious Sam 3 server
`10.0.0.15` | `chiv.server.saltlan.org` | `27015` | `eno2:15` |  LGSM | Chivalry server
`10.0.0.16` | `starbound.server.saltlan.org` | `27015` | `eno2:16` |  LGSM | Starbound server
`10.0.0.17` | `factorio.server.saltlan.org` | `27015` | `eno2:17` |  LGSM | Factorio server
`10.0.0.18 - 19` | Reserved | N/A | N/A | N/A | N/A 
`10.0.0.20 - 250` | DHCP Range | N/A | N/A | N/A | N/A 
`10.0.0.251 - 253` | Reserved | N/A | N/A | N/A | N/A 
`10.0.0.254` | `core.network.saltlan.org` | `80/443` | N/A | Cisco | SLM2048 core switch
`10.0.0.255` | Broadcast | N/A | N/A | N/A | N/A 
##Netmap

![Network Map](https://i.imgur.com/8V0OGwn.png)
