This has been successfully tested. It is based on [Genghis1227/guide_eap_proxy](https://github.com/Genghis1227/guide_eap_proxy)

- [Configuring Edgerouter 10X with EAP Proxy for AT&T Gigabit](#configuring-edgerouter-10x-with-eap-proxy-for-att-gigabit)
	- [Basic Reading](#basic-reading)
- [0. Prework](#0-prework)
	- [On your AT&T Router](#on-your-att-router)
	- [On your EdgeRouter](#on-your-edgerouter)
		- [Backup and update](#backup-and-update)
	- [On your computer/laptop](#on-your-computerlaptop)
- [1. Basic Setup](#1-basic-setup)
	- [Setup DHCP](#setup-dhcp)
	- [Setup DNS](#setup-dns)
- [3. Install EAP_Proxy](#3-install-eap_proxy)
	- [Make the scripts executable](#make-the-scripts-executable)
- [4. Firewall and IPv6 Setup](#4-firewall-and-ipv6-setup)
	- [Setup firewall rules](#setup-firewall-rules)
- [5. Finish Bypass Setup](#5-finish-bypass-setup)
	- [Setup NAT](#setup-nat)
- [6. Additional Configuration](#6-additional-configuration)
	- [Configure service gui ports and disable older-ciphers](#configure-service-gui-ports-and-disable-older-ciphers)
	- [Configure SSH](#configure-ssh)
	- [Setup UPNP Service](#setup-upnp-service)
- [7. Start the proxy service](#7-start-the-proxy-service)
- [Credits](#credits)

# Configuring Edgerouter 10X with EAP Proxy for AT&T Gigabit
This guide was forked from [Genghis1227/guide_eap_proxy](https://github.com/Genghis1227/guide_eap_proxy). I am reworking it as I learn to configure my new EdgeRouter 10X (ER-10X) to replace my EdgeRouter X (ER-X). 

This guide configures the EdgeRouter ports with the following:

	eth0 = WAN (the connection from the ONT)
	eth1 = AT&T Router
	eth2 = _Reserving for failover internet_
	eth3 = LAN
	eth4 = LAN
	eth5 = LAN
	eth6 = LAN
	eth7 = LAN
	eth8 = LAN
	eth9 = LAN

## Basic Reading
* [EdgeRouter - Configuration and Operational Mode](https://help.ui.com/hc/en-us/articles/204960094-EdgeRouter-Configuration-and-Operational-Mode)
* [Intro to Networking - How to Establish a Connection Using SSH](https://help.ui.com/hc/en-us/articles/218850057)
* [CLI Command Primer](https://community.ui.com/questions/EdgeOS-CLI-Primer-part-1/dc0a7754-1bcf-4ca0-9d02-239100dbc926)
* (Setup SSL with LetsEncrypt)[https://github.com/j-c-m/ubnt-letsencrypt]
* (Setup VPN)[https://help.ui.com/hc/en-us/articles/204950294-EdgeRouter-L2TP-IPsec-VPN-Server]


# 0. Prework

This process is best completed with a computer that can be connected in the same room as the router. If your computer is in a different room than your ATT router and your EdgeRouter I recommend using a laptop with a network port. 

## On your AT&T Router
Factory reset your AT&T router (optional)

Record the MAC address of your AT&T router, this is needed in section 4.  If using IPv6, record the duid value required.

From [jimbair](https://github.com/jimbair):

	For IPv6, be sure to change the duid value to the duid of your AT&T router, or wait ~2 weeks for the lease to expire to get a fresh lease. 
	You can sniff the traffic from your AT&T router to find the duid, or generate one with a script like gen-duid.sh from pfatt on github.

	For firewall rules, note that the setup wizard creates rules named WANv6_* if you check the box to enable IPv6, whereas the above rules are WAN6_*.

Disable the WiFi on the AT&T router.

## On your EdgeRouter
Perform a factory reset.

### Backup and update
* Backup config of the EdgeRouter (System > Back Up Config) - Old firmware
* Check www.ubnt.com/download to see if there is an updated firmware, download (if desired)
* Upgrade firmware of device (if desired) (System > Upgrade System Image)
* Backup config of the EdgeRouter (System > Back Up Config) - New firmware

## On your computer/laptop

Install [WinSCP](https://winscp.net/eng/index.php) - To transfer files

Install [Putty](https://www.putty.org/) - To configure the router

Plug in a computer to eth0

Set IP of computer to 192.168.1.100, 255.255.255.0, 192.168.1.1 - [How to: Change IP address in Windows](https://www.thewindowsclub.com/change-ip-address-windows-10)

Login to the EdgeRouter EdgeOS by going to https://192.168.1.1 in your browser

# 1. Basic Setup

Now that you are at a clean slate. Go ahead and log back into your router https://192.168.1.1

Open Putty and connect to 192.168.1.1.

In Putty, type `configure` then Enter to get into the configure prompt

```
set interfaces switch switch0 address 192.168.2.254/24
set interfaces switch switch0 description LAN
set interfaces switch switch0 switch-port interface eth3
set interfaces switch switch0 switch-port interface eth4
set interfaces switch switch0 switch-port interface eth4
set interfaces switch switch0 switch-port interface eth5
set interfaces switch switch0 switch-port interface eth6
set interfaces switch switch0 switch-port interface eth7
set interfaces switch switch0 switch-port interface eth8
set interfaces switch switch0 switch-port interface eth9
save;commit
```

## Setup DHCP

```
set service dhcp-server disabled false
set service dhcp-server hostfile-update enable
set service dhcp-server shared-network-name LAN2
set service dhcp-server shared-network-name LAN2 authoritative enable
set service dhcp-server shared-network-name LAN2 subnet 192.168.2.0/24 default-router 192.168.2.254
set service dhcp-server shared-network-name LAN2 subnet 192.168.2.0/24 dns-server 192.168.2.254
set service dhcp-server shared-network-name LAN2 subnet 192.168.2.0/24 lease 86400
set service dhcp-server shared-network-name LAN2 subnet 192.168.2.0/24 start 192.168.2.5 stop 192.168.2.200
set service dhcp-server static-arp disable
set service dhcp-server use-dnsmasq disable
save;commit
```

## Setup DNS
Change the nameservers to whatever you want.

* Cloudflare: 
	* 1.1.1.1 
	* 1.0.0.1
* Google: 
	* 8.8.8.8 
	* 8.8.4.4

```
set service dns forwarding cache-size 150
set service dns forwarding listen-on switch0
set service dns forwarding name-server 1.1.1.1
set service dns forwarding name-server 1.0.0.1
set service dns forwarding name-server 8.8.8.8
set service dns forwarding name-server 8.8.4.4
save;commit
```

# 3. Install EAP_Proxy

Downloaded eap-proxy files from: https://github.com/jaysoffian/eap_proxy

Edit the eap_proxy.sh file.

Change `IF_WAN` and `IF_ROUTER` to be the following.

```
IF_WAN=eth0
IF_ROUTER=eth1
```

Connect to EdgeRouter using WinSCP (IP address for the EdgeRouter is 192.168.1.1)

Copy using binary mode eap_proxy.sh to /config/scripts/post-config.d/

Copy using binary mode eap_proxy.py to /config/scripts/

## Make the scripts executable
Connect to EdgeRouter with Putty

Change permissions of files for execution using the following commands

	sudo chmod +x /config/scripts/post-config.d/eap_proxy.sh
	sudo chmod +x /config/scripts/eap_proxy.py
	
# 4. Firewall and IPv6 Setup

Configure router using the below, this will setup the firewall, all interfaces and give you working IPv6

Copy and paste each section below into Putty and run through save and commit. You should be able to see if each section ran properly before continuing. 

## Setup firewall rules
```
set firewall all-ping enable
set firewall broadcast-ping disable
set firewall ipv6-name IPv6_WAN_IN default-action drop
set firewall ipv6-name IPv6_WAN_IN description 'WAN to internal'
set firewall ipv6-name IPv6_WAN_IN enable-default-log
set firewall ipv6-name IPv6_WAN_IN rule 10 action accept
set firewall ipv6-name IPv6_WAN_IN rule 10 description 'Allow established/related'
set firewall ipv6-name IPv6_WAN_IN rule 10 state established enable
set firewall ipv6-name IPv6_WAN_IN rule 10 state related enable
set firewall ipv6-name IPv6_WAN_IN rule 20 action drop
set firewall ipv6-name IPv6_WAN_IN rule 20 description 'Drop invalid state'
set firewall ipv6-name IPv6_WAN_IN rule 20 log enable
set firewall ipv6-name IPv6_WAN_IN rule 20 state invalid enable
set firewall ipv6-name IPv6_WAN_IN rule 30 action accept
set firewall ipv6-name IPv6_WAN_IN rule 30 description 'Allow ICMPv6 destination-unreachable'
set firewall ipv6-name IPv6_WAN_IN rule 30 icmpv6 type destination-unreachable
set firewall ipv6-name IPv6_WAN_IN rule 30 protocol icmpv6
save;commit
```

```
set firewall ipv6-name IPv6_WAN_IN rule 31 action accept
set firewall ipv6-name IPv6_WAN_IN rule 31 description 'Allow ICMPv6 packet-too-big'
set firewall ipv6-name IPv6_WAN_IN rule 31 icmpv6 type packet-too-big
set firewall ipv6-name IPv6_WAN_IN rule 31 protocol icmpv6
set firewall ipv6-name IPv6_WAN_IN rule 32 action accept
set firewall ipv6-name IPv6_WAN_IN rule 32 description 'Allow ICMPv6 time-exceeded'
set firewall ipv6-name IPv6_WAN_IN rule 32 icmpv6 type time-exceeded
set firewall ipv6-name IPv6_WAN_IN rule 32 protocol icmpv6
set firewall ipv6-name IPv6_WAN_IN rule 33 action accept
set firewall ipv6-name IPv6_WAN_IN rule 33 description 'Allow ICMPv6 parameter-problem'
set firewall ipv6-name IPv6_WAN_IN rule 33 icmpv6 type parameter-problem
set firewall ipv6-name IPv6_WAN_IN rule 33 protocol icmpv6
set firewall ipv6-name IPv6_WAN_IN rule 34 action accept
set firewall ipv6-name IPv6_WAN_IN rule 34 description 'Allow ICMPv6 echo-request'
set firewall ipv6-name IPv6_WAN_IN rule 34 icmpv6 type echo-request
set firewall ipv6-name IPv6_WAN_IN rule 34 limit burst 1
set firewall ipv6-name IPv6_WAN_IN rule 34 limit rate 600/minute
set firewall ipv6-name IPv6_WAN_IN rule 34 protocol icmpv6
set firewall ipv6-name IPv6_WAN_IN rule 35 action accept
set firewall ipv6-name IPv6_WAN_IN rule 35 description 'Allow ICMPv6 echo-reply'
set firewall ipv6-name IPv6_WAN_IN rule 35 icmpv6 type echo-reply
set firewall ipv6-name IPv6_WAN_IN rule 35 limit burst 1
set firewall ipv6-name IPv6_WAN_IN rule 35 limit rate 600/minute
set firewall ipv6-name IPv6_WAN_IN rule 35 protocol icmpv6
save;commit
```

```
set firewall ipv6-name IPv6_WAN_LOCAL default-action drop
set firewall ipv6-name IPv6_WAN_LOCAL description 'WAN to router'
set firewall ipv6-name IPv6_WAN_LOCAL enable-default-log
set firewall ipv6-name IPv6_WAN_LOCAL rule 10 action accept
set firewall ipv6-name IPv6_WAN_LOCAL rule 10 description 'Allow established/related'
set firewall ipv6-name IPv6_WAN_LOCAL rule 10 state established enable
set firewall ipv6-name IPv6_WAN_LOCAL rule 10 state related enable
set firewall ipv6-name IPv6_WAN_LOCAL rule 20 action drop
set firewall ipv6-name IPv6_WAN_LOCAL rule 20 description 'Drop invalid state'
set firewall ipv6-name IPv6_WAN_LOCAL rule 20 state invalid enable
set firewall ipv6-name IPv6_WAN_LOCAL rule 30 action accept
set firewall ipv6-name IPv6_WAN_LOCAL rule 30 description 'Allow ICMPv6 destination-unreachable'
set firewall ipv6-name IPv6_WAN_LOCAL rule 30 icmpv6 type destination-unreachable
set firewall ipv6-name IPv6_WAN_LOCAL rule 30 protocol icmpv6
set firewall ipv6-name IPv6_WAN_LOCAL rule 31 action accept
set firewall ipv6-name IPv6_WAN_LOCAL rule 31 description 'Allow ICMPv6 packet-too-big'
set firewall ipv6-name IPv6_WAN_LOCAL rule 31 icmpv6 type packet-too-big
set firewall ipv6-name IPv6_WAN_LOCAL rule 31 protocol icmpv6
save;commit
```

```
set firewall ipv6-name IPv6_WAN_LOCAL rule 32 action accept
set firewall ipv6-name IPv6_WAN_LOCAL rule 32 description 'Allow ICMPv6 time-exceeded'
set firewall ipv6-name IPv6_WAN_LOCAL rule 32 icmpv6 type time-exceeded
set firewall ipv6-name IPv6_WAN_LOCAL rule 32 protocol icmpv6
set firewall ipv6-name IPv6_WAN_LOCAL rule 33 action accept
set firewall ipv6-name IPv6_WAN_LOCAL rule 33 description 'Allow ICMPv6 parameter-problem'
set firewall ipv6-name IPv6_WAN_LOCAL rule 33 icmpv6 type parameter-problem
set firewall ipv6-name IPv6_WAN_LOCAL rule 33 protocol icmpv6
set firewall ipv6-name IPv6_WAN_LOCAL rule 34 action accept
set firewall ipv6-name IPv6_WAN_LOCAL rule 34 description 'Allow ICMPv6 echo-request'
set firewall ipv6-name IPv6_WAN_LOCAL rule 34 icmpv6 type echo-request
set firewall ipv6-name IPv6_WAN_LOCAL rule 34 limit burst 5
set firewall ipv6-name IPv6_WAN_LOCAL rule 34 limit rate 5/second
set firewall ipv6-name IPv6_WAN_LOCAL rule 34 protocol icmpv6
save;commit
```

```
set firewall ipv6-name IPv6_WAN_LOCAL rule 35 action accept
set firewall ipv6-name IPv6_WAN_LOCAL rule 35 description 'Allow ICMPv6 echo-reply'
set firewall ipv6-name IPv6_WAN_LOCAL rule 35 icmpv6 type echo-reply
set firewall ipv6-name IPv6_WAN_LOCAL rule 35 limit burst 5
set firewall ipv6-name IPv6_WAN_LOCAL rule 35 limit rate 5/second
set firewall ipv6-name IPv6_WAN_LOCAL rule 35 protocol icmpv6
set firewall ipv6-name IPv6_WAN_LOCAL rule 36 action accept
set firewall ipv6-name IPv6_WAN_LOCAL rule 36 description 'Allow ICMPv6 Router Advertisement'
set firewall ipv6-name IPv6_WAN_LOCAL rule 36 icmpv6 type router-advertisement
set firewall ipv6-name IPv6_WAN_LOCAL rule 36 protocol icmpv6
set firewall ipv6-name IPv6_WAN_LOCAL rule 37 action accept
set firewall ipv6-name IPv6_WAN_LOCAL rule 37 description 'Allow ICMPv6 Neighbor Solicitation'
set firewall ipv6-name IPv6_WAN_LOCAL rule 37 icmpv6 type neighbor-solicitation
set firewall ipv6-name IPv6_WAN_LOCAL rule 37 protocol icmpv6
set firewall ipv6-name IPv6_WAN_LOCAL rule 38 action accept
set firewall ipv6-name IPv6_WAN_LOCAL rule 38 description 'Allow ICMPv6 Neighbor Advertisement'
set firewall ipv6-name IPv6_WAN_LOCAL rule 38 icmpv6 type neighbor-advertisement
set firewall ipv6-name IPv6_WAN_LOCAL rule 38 protocol icmpv6
save;commit
```

```
set firewall ipv6-name IPv6_WAN_LOCAL rule 50 action accept
set firewall ipv6-name IPv6_WAN_LOCAL rule 50 description 'Allow DHCPv6'
set firewall ipv6-name IPv6_WAN_LOCAL rule 50 destination port 546
set firewall ipv6-name IPv6_WAN_LOCAL rule 50 protocol udp
set firewall ipv6-name IPv6_WAN_LOCAL rule 50 source port 547
save;commit
```

```
set firewall ipv6-receive-redirects disable
set firewall ipv6-src-route disable
set firewall ip-src-route disable
set firewall log-martians enable
save;commit
```

```
set firewall name WAN_IN default-action drop
set firewall name WAN_IN description 'WAN to internal'
set firewall name WAN_IN rule 10 action accept
set firewall name WAN_IN rule 10 description 'Allow established/related'
set firewall name WAN_IN rule 10 state established enable
set firewall name WAN_IN rule 10 state related enable
set firewall name WAN_IN rule 20 action drop
set firewall name WAN_IN rule 20 description 'Drop invalid state'
set firewall name WAN_IN rule 20 state invalid enable
save;commit
```

```
set firewall name WAN_LOCAL default-action drop
set firewall name WAN_LOCAL description 'WAN to router'
set firewall name WAN_LOCAL rule 10 action accept
set firewall name WAN_LOCAL rule 10 description 'Allow established/related'
set firewall name WAN_LOCAL rule 10 state established enable
set firewall name WAN_LOCAL rule 10 state related enable
set firewall name WAN_LOCAL rule 50 action drop
set firewall name WAN_LOCAL rule 50 description 'Drop invalid state'
set firewall name WAN_LOCAL rule 50 state invalid enable
save;commit
```

```
set firewall receive-redirects disable
set firewall send-redirects enable
set firewall source-validation disable
set firewall syn-cookies enable
save;commit
```

# 5. Finish Bypass Setup
```
set interfaces ethernet eth0 description WAN
set interfaces ethernet eth0 duplex auto
set interfaces ethernet eth0 firewall in ipv6-name IPv6_WAN_IN
set interfaces ethernet eth0 firewall in name WAN_IN
set interfaces ethernet eth0 firewall local ipv6-name IPv6_WAN_LOCAL
set interfaces ethernet eth0 firewall local name WAN_LOCAL
set interfaces ethernet eth0 speed auto
save;commit
```
> ⚠ **Before you continue:** Edits are required for the next step
> 
> Replace **aa:bb:cc:dd:ee:ff** in this section of the configuration to reflect the mac address of your AT&T Router

```
set interfaces ethernet eth0 vif 0 address dhcp
set interfaces ethernet eth0 vif 0 description 'WAN VLAN 0'
set interfaces ethernet eth0 vif 0 dhcp-options default-route update
set interfaces ethernet eth0 vif 0 dhcp-options default-route-distance 210
set interfaces ethernet eth0 vif 0 dhcp-options name-server update
set interfaces ethernet eth0 vif 0 firewall in ipv6-name IPv6_WAN_IN
set interfaces ethernet eth0 vif 0 firewall in name WAN_IN
set interfaces ethernet eth0 vif 0 firewall local ipv6-name IPv6_WAN_LOCAL
set interfaces ethernet eth0 vif 0 firewall local name WAN_LOCAL
set interfaces ethernet eth0 vif 0 mac 'aa:bb:cc:dd:ee:ff'
save;commit
```

```
set interfaces ethernet eth1 description 'AT&T Router'
set interfaces ethernet eth1 duplex auto
set interfaces ethernet eth1 speed auto
save;commit
```

```
set port-forward auto-firewall enable
set port-forward hairpin-nat enable
set port-forward lan-interface switch0
set port-forward wan-interface eth0.0
save;commit
```

```
set interfaces ethernet eth0 vif 0 dhcpv6-pd pd 1 interface switch0 ost-address '::1'
set interfaces ethernet eth0 vif 0 dhcpv6-pd pd 1 interface switch0 no-dns
set interfaces ethernet eth0 vif 0 dhcpv6-pd pd 1 interface switch0 refix-id ':0'
set interfaces ethernet eth0 vif 0 dhcpv6-pd pd 1 interface switch0 ervice slaac
set interfaces ethernet eth0 vif 0 dhcpv6-pd pd 1 prefix-length 60
set interfaces ethernet eth0 vif 0 dhcpv6-pd prefix-only
set interfaces ethernet eth0 vif 0 dhcpv6-pd rapid-commit disable
set interfaces ethernet eth0 vif 0 firewall in ipv6-name IPv6_WAN_IN
set interfaces ethernet eth0 vif 0 firewall local ipv6-name IPv6_WAN_LOCAL
set interfaces ethernet eth0 vif 0 ipv6 dup-addr-detect-transmits 1
set system offload ipv6 forwarding enable
set system offload ipv6 vlan enable
save;commit
```

## Setup NAT
```
set service nat rule 5010 description 'masquerade for WAN'
set service nat rule 5010 outbound-interface eth0.0
set service nat rule 5010 type masquerade
set service nat rule 6000 description 'MASQ all networks NTP to WAN'
set service nat rule 6000 log disable
set service nat rule 6000 outbound-interface switch0
set service nat rule 6000 outside-address port 1024-65535
set service nat rule 6000 protocol udp
set service nat rule 6000 source port 123
set service nat rule 6000 type masquerade
save;commit
```

If your router supports ipv4 vlan offload use this command:

```
set system offload ipv4 vlan enable`
save;commit
```
If not, use this command (ER-X, ER-10X):
```
set system offload hwnat enable
save;commit
```

# 6. Additional Configuration
## Configure service gui ports and disable older-ciphers
```
set service gui http-port 80
set service gui https-port 443
set service gui older-ciphers disable
save;commit
```

## Configure SSH
```
set service ssh port 22
set service ssh protocol-version v2
set service unms disable
save;commit
```

## Setup UPNP Service
```
set service upnp
set service upnp2 acl rule 9000 action deny
set service upnp2 acl rule 9000 description 'Deny All'
set service upnp2 acl rule 9000 external-port 0-65535
set service upnp2 acl rule 9000 local-port 0-65535
set service upnp2 acl rule 9000 subnet 0.0.0.0/0
set service upnp2 listen-on switch0
set service upnp2 nat-pmp enable
set service upnp2 secure-mode enable
set service upnp2 wan eth0.0
save;commit  
```
	
Once everything returns and committed then `exit` from the configure mode.

# 7. Start the proxy service

Enter the following in putty to monitor the data:

	sudo /usr/bin/python /config/scripts/eap_proxy.py eth0 eth1 --restart-dhcp --ignore-when-wan-up --ignore-logoff --ping-gateway

Plug in cable in from ONT to eth0

Plug in cable in from AT&T router to eth1 

Plug in cable out to LAN from eth3 - eth9

In you should see EAPOL start with a whole bunch of communication

Log into the EdgeOS (https://192.168.2.254/) to see everything setup with:

	WAN (eth0) as 192.168.1.1
	WAN VLAN 0 (eth0.0) with WAN IP
	AT&T Router (eth1) has no IP
	LAN (switch0) as 192.168.2.254

Example of what it should look like:

![Configured EdgeRouter with Bypass](https://i.imgur.com/LrEg9WW.png)

Good idea to backup your EdgeRouter configuration if everything is in working order.

# Credits
[EAP Proxy source](https://github.com/jaysoffian/eap_proxy) as a required dependency for this all to work.

[Genghis1227/guide_eap_proxy](https://github.com/Genghis1227/guide_eap_proxy) for pulling all of the knowledge together.

[SScorpio](https://www.reddit.com/user/SScorpio/) provided much of hte background for the script [eli5 how to use eaproxy and att uverse](https://www.reddit.com/r/uverse/comments/9ii4t5/eli5_how_to_use_eaproxy_and_att_uverse/e9fbtoa/)

Thanks to user [h-parks](https://github.com/h-parks) for figuring out and merging the IPv6 portion from [jimbair](https://github.com/jimbair) posted [here](https://github.com/jaysoffian/eap_proxy/pull/36/commits/0abe21cfcfd181372cf1dbee91bdeea5926ff1e6) from the original eap_proxy.
