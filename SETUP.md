# SaltLAN Installation


How we built it.



# 0. Side note

The goals of this project were more than one, and can be broken down into a few sections.

1. Be a router

  * It needs to be able to sit inbetween the party atendees and the outside network.

  * This also means it needs to have DHCP, DNS, Firewall, and NAT.

2. Cache content

  * To reduce network load and prevent the possibility of saturating the venue's bandwidth.

  * This means caching Steam, Origin, Uplay, Blizzard, Windows updates, Hirez, RSI, Frontier, Twitch.

3.  Be a server

  *  We will be serving like it's hot; Game servers, file servers, internal websites, control panels, etc.

  *  Self-contained and large. Scaling verticaly instead of horizontaly isn't always good in the way of servers, but with what we have at the moment, it's what we're going to do.

4. Monitor everything

  * Needs to be able to provide us with powerful analytics into as much as possible. DHCP,  DNS, Caching, Bandwidth, clients, Steam users, etc.



# 1. Routing

Our current setup is Ubuntu server 16.04. Once ubuntu is installed, we need to configure it to be a router. 

Routers need two ethernet ports at minumum; LAN and WAN. For those who don't know, LAN is your Local Area Network (where all of your clients connect) and WAN is your Wide Area Network (basically the internet), so the server needs to have at least two ports. To see which NICs (Network Interface Card) you have, `ifconfig -a` will tell you.

Pick a NIC for each WAN and LAN, and stick with it. As of Ubuntu 16 with the intruduction of systemd, your interfaces will be labled dfferently, such as `eno, eno1, eno2, enp, eno2s0, enp2s0f, etc.` This guide will call WAN `eth0` and LAN `eth1`, in the pre-systemd format. Make sure you read over everything and replace `eth0/eth1` with the correct interface before copying and pasting.

You also need to decide an IP scheme. Ours is `10.0.0.x`, because it's easy to remember and easy to type, and does not interfere with internal or external IP ranges. This guide will use that scheme, but you can use whatever you want.

## Interface configuration

First thing we do is edit the interfaces on the server. 

`sudo nano /etc/network/interfaces`

Make a note of which interface is which. our WAN is `eth0` and our LAN is `eth1`

```
#loopback, not important
auto lo
iface lo inet loopback

#WAN - we make sure that it had DHCP enabled on the interface so that it can get it's info from the upstream router.
auto eth0
iface eth0 inet dhcp

#LAN
auto eth1
iface eth1 inet static
 address 10.0.0.1 # router address
 netmask 255.255.255.0 # the netmask of the network
 network 10.0.0.0 # base address for the network
 broadcast 10.0.0.255 # broadcast address
```

Now the interfaces are configured for the network we will create.

## IP forwarding

Next, we need to enable IP forwarding. 

`sudo nano /etc/sysctl.conf`

look for the line `# net,ipv4.ip_forward=1` and uncomment it so that it is enabled. Save and exit.

We also want to turn it in without having to reboot, so run the next command.

`sudo sysctl -w net.ipv4.ip_forward=1`

## iptables (firewall)

Now we need to make sure the firewall had IP masquerading enabled.

`sudo nano /etc/rc.local`

and add these lines **before** `exit 0`:

```
/sbin/iptables -P FORWARD ACCEPT
/sbin/iptables --table nat -A POSTROUTING -o eth0 -j MASQUERADE
```

To make it work without rebooting, run

`sudo iptables -P FORWARD ACCEPT`

and

`sudo iptables --table nat -A POSTROUTING -o eth0 -j MASQUERADE`

## DHCP server

Now that it is routing, it needs to be able to assign IP addresses. We use `dnsmasq` for both DHCP and DNS.

`sudo apt install dnsmasq`

Now we need to configure dnsmasq to act like a DHCP server. First, lets backup it's default configration file.

`sudo cp /etc/dnsmasq.conf /etc/dnsmasq.conf.backup`

Next, we'll make a config file for it.

`sudo nano /etc/dnsmasq.conf`

Make the file look like this:

```
interface=eth1 #interface to serve addresses
dhcp-range=eth1,10.0.0.10,10.0.0.240,4h # interface, first address, last address, lease time.
dhcp-option=6,10.0.0.1 # we tell dnsmasq to give out 10.0.0.1 as the dns server address as well.

```

and save it. Then, restart dnsmasq

`sudo service dnsmasq restart`

Now, dnsmasq should be serving IP addresses and your server should be able to route them. Plug a computer into your LAN port on your server and see if you get an IP address and can get online.

# 2. Caching

We will be using custom Docker images running nginx caching proxies to handle all game caching. This makes it easier to turn on and off certan caches, as well as the ability to track the usage, backup, restore, and monitor each individual cache.

## Virtual Interfaces

Before we setup docker or any caches, we need to give each cache a dedicated IP address. Plan out your address scheme. For DHCP, we start the address assignment at `10.0.0.10`, which provides us with 9 dedicated internal IP addresses for whatever we need.

To give each cache a dedicated IP address, we will first need to make virtual interfaces in `/etc/network/interfaces`. 

`sudo nano /etc/network/interfaces`

and add this below your current configuration. Please make sure to change `eth0/eth1` and each `10.0.0.x` IP address with whatever you've decided.

```
...

# Steam
auto eth1:2
iface eth1:2 inet static
 address 10.0.0.2

# Battle.net
auto eth1:3
iface eth1:3 inet static
 address 10.0.0.3

# Origin
auto eth1:4
iface eth1:4 inet static
 address 10.0.0.4

# Uplay
auto eth1:5
iface eth1:5 inet static
 address 10.0.0.5

# Riot
auto eth1:6
iface eth1:6 inet static
 address 10.0.0.6

# Frontier
auto eth1:7
iface eth1:7 inet static
 address 10.0.0.7

# Windows
auto eth1:8
iface eth1:8 inet static
 address 10.0.0.8

# Twitch
auto eth1:9
iface eth1:9 inet static
 address 10.0.0.9
 ```

 Then save that file. To see if the Virtual Interfaces are appearing to the system, you can run `ifconfig | grep eth1:` to see.

## Docker

Now that we have dedicated internal IPs for each cache, we now need to setup the cache. To do this, we will be using Docker containers written by the [steamcache.net](https://steamcache.github.io/) team. You need docker though.

`sudo apt-get install docker docker.io docker-engine`

Once that is installed, we will provision the containers. SteamCache team has made two types of cache. The first type is specific to Steam and is optimized for Steam downloads. The second is a generic cache, which will work with any download, Origin, Uplay, Battle.net, etc. This generic cache will also work with steam, however it is not as efficient as the Steam-specific container. Meaning; We'll be running the Steam cache for Steam downloads, and the generic cache for all else.

The `docker run` command will need three arguments from us.

`sudo docker run --name steam-cache -p 10.0.0.2:80:80 steamcache/steamcache`

1. The first argument, `--name`, specifies the Docker container name. We will use this name to start, stop, and generally reference this container.
2. Second, `-p 10.0.0.2:80:80` This specifies the IP address to bind the container to, as well as the port. Make sure to double check your `/etc/network/interfaces` for the IP address for each container. The `:80:80` section tells Docker what port to forward inside the docker container.
3. Third, `steamcache/steamcache` refers to the hub.docker.io user (steamcache) and the container (also named steamcache), much the same that `SaltLAN/Configuration` is the user and repository on Github.
 * There are three containers, if we're using Twitch Cache
 * `steamcache/steam` (Steam)
 * `steamcache/generic` (Origin, Uplay, Battle.net, anything that supports HLS caching)
 * `steamcache/twitch` (Twitch)

To provision containers for the cache, we just need to run the same command and change the arguments for each container.

Service | Command 
 --- | --- 
Steam | `sudo docker run --name steam-cache -p 10.0.0.2:80:80 -d steamcache/steamcache`
Battle.net | `sudo docker run --name blizzard-cache -p 10.0.0.3:80:80 -d steamcache/generic`
Origin | `sudo docker run --name origin-cache -p 10.0.0.4:80:80 -d steamcache/generic`
Uplay | `sudo docker run --name uplay-cache -p 10.0.0.5:80:80 -d steamcache/generic`
Riot | `sudo docker run --name riot-cache -p 10.0.0.6:80:80 -d steamcache/generic`
Frontier | `sudo docker run --name frontier-cache -p 10.0.0.7:80:80 -d steamcache/generic`
Windows | `sudo docker run --name windows-cache -p 10.0.0.8:80:80 -d steamcache/generic`
Twitch | `sudo docker run --name twitch-cache -p 10.0.0.9:80:80 -d steamcache/twitch`
