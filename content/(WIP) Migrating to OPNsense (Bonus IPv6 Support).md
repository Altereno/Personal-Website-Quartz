# Summary
I wanted to support IPv6 in my home network. I have documented some notes for myself here.

OPNsense version as of now (May 10th, 2026) is: `OPNsense 26.1.7_3-amd64`

**PfSense was not cooperating with me, so instead of troubleshooting it, I nuked the installation (after taking screenshots and backing up the XML) and installed OPNsense**
# Important Resource(s)
This [video](https://www.youtube.com/watch?v=Yb7JdIFriKI) by [apalrdsadventures](https://www.youtube.com/@apalrdsadventures) is great, it covers all the basic setup for OPNsense.
*There are also other videos from this channel that cover IPv6*
# OPNsense Setup
**This assumes that the general setup wizard has already been ran**
## DHCPv4
Dnsmasq DNS & DHCP is the default, but coming from PfSense, I was using Kea. 

Thus:
- Under `Services -> Dnsmasq DNS & DHCP -> General`, I unchecked the `enable` box.
- Under `Services -> Kea DHCP -> Kea DHCPv4 -> Settings`, I checked the `enable` box and also selected the interfaces I wanted under `Interfaces`
- Under `Services -> Kea DHCP -> Kea DHCPv4 -> Subnets`, I configured the subnet and address pools for each interface. (*Reservations were under `Services -> Kea DHCP -> Kea DHCPv4 -> Reservations`*)
## Unbound DNS
Instead of setting the DNS servers under `System -> Settings -> General`, I let the system use itself (the local Unbound server) to resolve hostnames.

Unbound is enabled by default but I needed to add DNS servers.
## DNS over TLS
- Under `Services -> Unbound DNS -> DNS over TLS`:
	- `Server IP`: `1.1.1.1`
	- `Verify CN`: `one.one.one.one`
*Note that I'm just using Cloudflare's DoT for this, other providers such as Quad 9 will work.*
## Overrides
Under `Services -> Unbound DNS -> Overrides`, I just copied all the previous host overrides that I had on PfSense.
## NUT
My PfSense box was acting as the NUT server for all my other machines, so this was pretty important to also have.

To install the NUT package:
- Under `System -> Firmware -> Plugins`:
	- Check  `Show community plugins`
	- Install `os-nut`
To configure NUT:
- Under `Services -> NUT -> Configuration`
	- Under `General Settings`:
		- Under `General Nut Settings`
			- Check the `Enable Nut` box
			- Set the `Service Mode` to `standalone`
			- Give the UPS a name
			- Add the interface address that you want NUT to be reachable over (*Keep the loopback addresses as removing them will cause the status page to break*)
		- Under `Nut Account Settings`: 
			- Set the `Admin Password`
			- Set the `Monitor Password` (*This will be the one I use on my other machines, the user for this would be `monuser`*)
	- Under `UPS Type`(*This is a drop down menu, this also took me a long time to figure out...*):
		- Under `SNMP-Driver`:
			- Check the `Enable` box
			- Add all the configurations from PfSense inside `Extra Arguments` (*see the end of this section for an example*)
### Extra Arguments Example
```
port=10.0.15.2
mibs=ietf
snmp_version=v3
secLevel=authPriv
secName=monitor
authPassword=password
privPassword=password
authProtocol=SHA
privProtocol=AES
```
## ACME Client
To install the ACME package:
- Under `System -> Firmware -> Plugins`:
	- Check  `Show community plugins`
	- Install `os-acme-client`
### Creating an Account
To create an account:
- Under `Services -> ACME Client -> Accounts`:
	- Create a new account:
		- Enter a valid email in `Email`
		- Use the default Let's Encrypt for the `ACME CA`
### DNS Challenge
I don't have anything to expose publicly currently, so I will just use a DNS challenge.
- Under `Services -> ACME Client -> Challenge Types:
	- Add a new challenge type:
		- Use `DNS-01` for the `Challenge Type` 
		- Select the `DNS Service` (*I use Cloudflare*)
		- Under `Restricted API Token` (*Use this over the `Global API Key`*):
			- Populate the ` CF Account ID` (*Can be found in the URI after logging into the Cloudflare dashboard*)
			- Populate `CF API Token` (*The token needs "Read" access to Zone.Zone and "Edit" access to Zone.DNS across all zones from an account*)
### Creating a Certificate
This will be a certificate only for OPNsense, not a wildcard.
- Under `Services -> ACME Client -> Certificates:
	- Enter a valid `Common Name` (*Something like opnsense.example.com*)
	- Everything else should be left default (*Make it is using the correct `ACME account` and `Challenge Type`, `Auto Renewal` should also be enabled*)
### Automations
When the certificate renews, OPNsense should also restart its web UI to use the new certificate.
- Under `Services -> ACME Client -> Automations:
	- Select `Restart OPNsense Web UI` for `Run Command`
	- Add this automation to the certification created in the previous step
### Enabling
Under `Services -> ACME Client -> Settings`:
- Check the `Enable Plugin` box
- Check the `Auto Renewal` box
- Uncheck the `Show introduction pages` box
### Using the Certificate
Under `System -> Settings -> Administration`:
- Change the `Web GUI -> SSL Certificate` to use the new certificate from Let's Encrypt
## Dynamic DNS
To install the DDNS package:
- Under `System -> Firmware -> Plugins`:
	- Check  `Show community plugins`
	- Install `os-ddclient`
### DDNS Entry
Under `System -> Services -> Dynamic DNS -> Accounts`:
- Create an entry for IPv4 and IPv6:
	- Select the correct DNS provider under `Service`
	- Fill in the `Username` (*`token` for Cloudflare*)
	- Fill in the `Password` (*The token needs "Read" access to Zone.Zone and "Edit" access to Zone.DNS across all zones from an account*)
	- Enter the correct `Zone`
	- Enter the a valid `Hostname` (*Make sure this is already created on Cloudflare*)
	- Select a `Check ip method` (*I used cloudflare-ipv4 and cloudflare-ipv6*)
### Enabling
Under `System -> Services -> Dynamic DNS -> General Settings`:
- Check the `Enable` box
## WireGuard
WireGuard in OPNsense actually has a nicer QoL compared to PfSense. It has a QR code generator that allows me to quickly add mobile peers.
### "Road Warrior" Setup
#### Tunnel
Under `VPN -> WireGuard -> Instances`:
- Create a new instance:
	- Check the `Enable` box
	- Give it a `Name`
	- Generate a new keypair
	- Configure the `Listen port`
	- Set the `Tunnel Address` (*A separate network for VPN hosts. something like: `10.20.30.1/24` and `fd00:a12b:3c4d:5e6f::1/64`*)
#### Interface
Under `Interfaces -> Assignments`:
- Assign the new WireGuard interface
- Enable the newly assigned interface (*Nothing else needs to be changed here. There is no need to set the IPv4 and IPv6 `Configuration Type`, it has already been set when the tunnel was created.*)
#### Peers
Under `VPN -> Wireguard -> Peer Generator`:
- Set the `Instance` to the one that was just created
- Leave ` Endpoint` blank as this will be dynamic (*Not a Site-to-Site* VPN)
- Give a friendly `Name` for the peer
- Set the static `Address` for the peer (*This will be the IP that the peer will use inside the network that was created for the Tunnel*)
- Generate a `Pre-shared key`
- Set the `DNS Servers` to the OPNsense tunnel address (*`10.20.30.1/24` and `fd00:a12b:3c4d:5e6f::1/64` from earlier*)
- Copy the configuration to the other peer, or scan the QR code
- Click `Store and generate next` (*Make sure to do this, I've forgotten to do this more times than I'd like to share*)
#### IPv6 GUA
WireGuard uses NAT44 for IPv4, which allows the peer to access the internet, for IPv6, I researched a couple options:
- Use NAT66 (Network Address Translation)
	- This basically mimics the functionality of NAT44, which makes OPNsense keep track of connections (stateful)
- Use NPTv6 (Network Prefix Translation)
	- This swaps the network prefix (leftmost side) of an address with another. This will allow a 1:1 mapping between the GUA network that the ISP assigns and the ULA network of the VPN. As a result, OPNsense doesn't need to keep track of the connections (stateless)
*Since the WireGuard tunnel interface doesn't have the `Configuration Type` set, IPv6-PD (Prefix Delegation) did not work*

To setup NPTv6:
Under `Interfaces -> Devices -> Loopback`:
- Create a "dummy" interface, which will be used to allow tracking on WAN
Under `Interfaces -> Assignments`:
- Assign the new "dummy" interface
- Enable the interface
- Set `IPv6 Configuration Type` to `Track Interface (legacy)`
- Set the `Parent interface` to `WAN`
- Set the `Assign prefix ID` to a valid prefix
Under `Firewall -> NAT -> NPTv6`:
- Create a new Rule
- Set the `Interface` to `WAN`
- Set the `Internal IPv6 Prefix` to the IPv6 network of the tunnel (*`fd00:a12b:3c4d:5e6f::/64` from earlier*)
- Leave the `External IPv6 Prefix` blank to auto-detect
- Set the `Track interface` to the "dummy" interface from earlier

**If this was set up correctly, IPv6 connectivity should be working when using the VPN**
### External VPN Provider
*Note: This assumes that the VPN provider has already provided the WireGuard configuration file. I will be referencing the example configuration file below:*
```
[Interface]
Address = 10.11.12.13/32,fd00:1234:1234:1234:1234:1234:1234:1234/128
PrivateKey = OH3U1HuSuBqpuBJ4kU041BDgFIv8ZkzUCWRatePbkFI=
MTU = 1320
DNS = 10.11.12.1, fd00:1234:1234:1234::1

[Peer]
PublicKey = 0SixTh/9bxYhtWrNkFApEpHjy46QnucU7l1rN88hsTI=
PresharedKey = L5soHEDdd1IRmXZWfH+OH/0ebBIq+O+qcUa/QEPv3Ng=
Endpoint = vpn.bello.internal
AllowedIPs = 0.0.0.0/0,::/0
PersistentKeepalive = 25
```

## IPv6 (For real this time)
*Below reflects my current understanding of IPv6 and networking, it may or may not be correct!*

From some research and talking with the big G(emini), some general notes I have of IPv6:
For unicast, there are 3 types of addresses:
- Global Unicast Address (GUA)
	- Prefix `2000::/3`
	- Routable on the internet
- Unique Local Address (ULA)
	- Prefix `FD00::/8`
	- Private networks, locally routable
- Link-Local Address (LLA)
	- Prefix `FE80::/10`
	- Local subnet only, routers will not route these
	- Assigned to each network interface

ICMPv6 now takes care of these:
- ARP - through Neighbor Solicitation and Neighbor Advertisement
- DHCP - through Router Advertisements (RA)

DHCPv6 is a thing, however some operating systems don't support it because SLAAC (Stateless Address Autoconfiguration) exists.
- DHCPv6 will function basically the same as DHCPv4, allowing the server to assign the client addresses in a pool
- SLAAC is done client side, where the client takes the prefix information from the RA and generates their own address
*This is turbo annoying because I wanted to manage static addresses from OPNsense, just like I already do with IPv4. More on this later...*
### Enabling on Interface
Assuming that the ISP has already assigned a prefix, enabling IPv6 on an interface is pretty straightforward:
- 

# WIP NOTES
TrueNAS Setup
Does not support dhcpv6
could use assisted but i dont like having it split so use slaac/unmanaged for everything
Proxmox Setup