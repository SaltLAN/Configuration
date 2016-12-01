# SaltLAN Installation


How we built it.



# 0. Side note

The goals of this project were more than one, and can be broken down into a few sections.

1. Be a router

  * It needs to be able to sit inbetween the party atendees and the outside network.

  * This also means it needs to have DHCP, DNS, Firewall, and NAT.

2. Cache content

  * To reduce network load and prevent the possibility of saturating the venue's bandwidth.

  * This means caching Steam, Origin, Uplay, Blizzard, Windows updates, Hirez, RSI, Frontier, Twitch, and Youtube.

3.  Be a (single) server

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
   network 10.0.0.0 # vase address for the network
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

`sudo cp /etc/dndmasq.conf /etc/dnsmasq.conf.backup`

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
