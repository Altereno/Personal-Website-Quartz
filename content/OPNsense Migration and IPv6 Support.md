# Summary
I wanted to support IPv6 in my home network. I have documented some notes for myself here.

OPNsense version as of now (May 10th, 2026) is: `OPNsense 26.1.7_3-amd64`

**PfSense was not cooperating with me, so instead of troubleshooting it, I nuked the installation (after taking screenshots and backing up the XML) and installed OPNsense**
# Important Resource(s)
This [video](https://www.youtube.com/watch?v=Yb7JdIFriKI) by [apalrdsadventures](https://www.youtube.com/@apalrdsadventures) is great, it covers all the basic setup for OPNsense.
*There are also other videos from this channel that cover IPv6*
# OPNsense Setup
**This assumes that the general setup wizard has already been run**
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
*Note that I'm just using Cloudflare's DoT for this; other providers such as Quad 9 will work.*
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

The following configurations are based heavily on the examples found on the [official OPNsense documention](https://docs.opnsense.org/manual/vpnet.html#wireguard)
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
- Leave `Endpoint` blank as this will be dynamic (*Not a Site-to-Site* VPN)
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
- Set `IPv6 Configuration Type` to `Identity association`
- Set the `Parent interface` to `WAN`
- Set the `Assign prefix ID` to a valid prefix
Under `Firewall -> NAT -> NPTv6`:
- Create a new Rule
- Set the `Interface` to `WAN`
- Set the `Internal IPv6 Prefix` to the IPv6 network of the tunnel (*`fd00:a12b:3c4d:5e6f::/64` from earlier*)
- Leave the `External IPv6 Prefix` blank to auto-detect
- Set the `Identity association` to the "dummy" interface from earlier

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
#### Tunnel
Under `VPN -> WireGuard -> Instances`:
*This is filled in using information from the `Interface` section of the configuration file*
- Create a new instance:
	- Check the `Enable` box
	- Give it a `Name`
	- Copy the `Private key` over (*Filling in the `Public key` is optional but to generate the public key from the private key run this: `printf "<PRIVATEKEYHERE>" | wg pubkey`*)
	- Copy over the `MTU`
	- Leave the `DNS servers` empty
	- Copy over the `Tunnel Address` (*Note Below*)
	- Check the `Disable routes` box
	- Toggle the `advanced mode`
	- Set the `Gateway` to one below the assigned IPv4 address (*This seems to be arbitrary, just needs to not conflict with another address*)
*For the tunnel addresses, fill in the `/32` route for IPv4, but use a `/127` for IPv6. This is because there is no concept of a `Far Gateway` for IPv6, thus the gateway address will need to be within the same subnet configured in the tunnel. Using a subnet calculator will show the gateway address for later. As an example, from the configuration posted above, the gateways would be `10.11.12.12` and `fd00:1234:1234:1234:1234:1234:1234:1235`*
#### Peers
*This is filled in using information from the `Peer` section of the configuration file*
Under `VPN -> Wireguard -> Peers`:
- Give it a `Name`
- Copy over the `Public key`
- Copy over the `Pre-shared key`
- Copy over the `Allowed IPs`
- Copy over the `Endpoint address`
- Copy over the `Endpoint port`
- Set the `Instances` to the one we just created
- Copy over the `Keepalive interval`
#### Interface
Under `Interfaces -> Assignments`:
- Assign the new WireGuard interface
- Enable the newly assigned interface (*Nothing else needs to be changed here. There is no need to set the IPv4 and IPv6 `Configuration Type`, it has already been set when the tunnel was created.*)
- Set the `MTU` and `MSS` to match the WireGuard tunnel created earlier
#### Gateway
Under `System -> Gateways -> Configuration`:
- Create a new Gateway
- Give it a `Name`
- Select the `Interface` we just created
- Select the `Address Family` (*In the end there should be 2 gateways, one for each address family*)
- Set a `Priority` (*I just set it to max (255), no reason behind it*)
- Set the `IP Address` (*For IPv4 it will be the one configured in the WireGuard tunnel earlier, and for IPv6, it would be the gateway address from the `/127`*)
- Check the `Far Gateway` box (*Only for IPv4*)
#### Outbound NAT
This will create the NAT44 and the NAT66 for the VPN. Since it is NAT, it is stateful and will translate the addresses between the two networks.
Under `Firewall -> NAT -> Outbound`:
- Change the mode to `Hybrid outbound NAT rule generation`
- Add a manual rule
- Select the `Interface` we just created
- Set the `TCP/IP Version` (*In the end there should be 2 gateways, one for each address family*)
- Set the `Protocol` to `any`
- Leave `Source invert` unchecked
- Set the `Source address` to the clients that will be routed through the VPN (*I have an alias for this*)
- Set the `Source port` to `any`
- Leave `Destination invert` unchecked
- Set the `Destination address` to `any`
- Set the `Destination port` to `any`
- Set the `Translation / target` to `Interface address`
#### PBR (Policy Based Routing)
This will force specific clients to use the VPN gateway instead of the WAN gateway.
Under `Firewall -> Rules`:
- Create a rule
- Check the `Enabled` box
- Leave `Invert Interface` unchecked
- Select the `Interface` that the client lives on
- Check the `Quick` box
- Select `Pass` for `Action`
- Select `In` for `Direction`
- Set the `Version` (*This rule will be created twice, one for IPv4 and one for IPv6*)
- Set `Protocol` to `any`
- Leave `Invert Source` unchecked
- Set the `Source` to the clients that will be routed through the VPN (*I have an alias for this*)
- Set the `Source Port` to `any`
- Check the `Invert Destination` box
- Set the `Destination` to either RFC 1918 or RFC 4193 depending on the IP family
- Set the `Destination port` to `any`
- Set the `Gateway` to the corresponding one based on the IP family
- Toggle `advanced mode` switch
- Set `Set local tag` to a custom tag (*I just used 'NO_WAN_EGRESS', although this could be anything. This will be used later to do the kill switch.*)
#### Kill Switch
This will block all attempts of the client trying to reach the internet through WAN.
Under `Firewall -> Rules`:
- Create a rule
- Check the `Enabled` box
- Leave `Invert Interface` unchecked
- Select the `WAN` for the `Interface`
- Check the `Quick` box
- Select `Block` for `Action`
- Select `Out` for `Direction`
- Set the `Version` to `IPv4+IPv6`
- Set `Protocol` to `any`
- Leave `Invert Source` unchecked
- Set the `Source` to `any`
- Set the `Source Port` to `any`
- Leave `Invert Destination` unchecked
- Set the `Destination` to `any`
- Set the `Destination port` to `any`
- Toggle `advanced mode` switch
- Set `Match local tag` to a custom tag created earlier
#### Port Forwarding
This assumes the VPN provider supports port forwarding.
Under `Firewall -> Rules`:
- Create a rule
- Check the `Enabled` box
- Leave `Invert Interface` unchecked
- Select the `Interface` that matches the VPN provider tunnel
- Check the `Quick` box
- Select `Pass` for `Action`
- Select `In` for `Direction`
- Set the `Version` (*This rule will be created twice, one for IPv4 and one for IPv6*)
- Set `Protocol` to support the client application that is being port forwarded
- Leave `Invert Source` unchecked
- Set the `Source` to `any`
- Set the `Source Port` to `any`
- Leave `Invert Destination` unchecked
- Set the `Destination` to client IP 
- Set the `Destination port` to client port
- Toggle `advanced mode` switch
- Set `Reply-to` to the correct gateway depending on the IP family
#### Aliases
For IPv4, under `Firewall -> Aliases`:
- Create an alias
- Check the `Enabled` box
- Give it a `Name`
- Use the `Host(s)` `Type`
- Fill in the hosts that need to be routed though the VPN under `Content`
For IPv6, under `Firewall -> Aliases`:
- Create an alias
- Check the `Enabled` box
- Give it a `Name`
- Use the `MAC Address` `Type`
- Fill in the MAC addresses of the hosts that need to be routed though the VPN under `Content`
*Note: For IPv6, the `Host(s)` `Type` would not work because the GUA prefix can change, thus bypassing the firewall rules created earlier.*
#### DNS Leaks
To prevent DNS leaks, I just manually configured a public DNS server on the client.
## IPv6 (For real this time)
*Below reflects my current understanding of IPv6 and networking, it may or may not be correct!*
### Notes
From some research and talking with the big G(emini), some general notes I have of IPv6:
#### Address Types
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

When a host attempts to send traffic to a destination, it will use the address that is scoped closest to the destination address. So if it wanted to reach a GUA, it would use its GUA to communicate.
#### IPv4 Parallels (kind of)
ICMPv6 now takes care of these:
- ARP - through Neighbor Solicitation and Neighbor Advertisement
- DHCP - through Router Advertisements (RA)

DHCPv6 is a thing, however some operating systems don't support it because SLAAC (Stateless Address Autoconfiguration) exists.
- DHCPv6 will function basically the same as DHCPv4, allowing the server to assign the client addresses in a pool
- SLAAC is done client side, where the client takes the prefix information from the RA and generates their own address
*This is turbo annoying because I wanted to manage static addresses from OPNsense, just like I already do with IPv4. More on this later...*
#### IPv6 PD (Prefix Delegation)
When an ISP assigns an IPv6 block to a customer, it will usually be either a `/48`, `/56` block from what I've seen online. In my case, the ISP have provided a `/56` block, which means that the first 56 bits of the IPv6 address will be "static" (in the sense that I won't be able to modify the first 56 bits, but they can change if the ISP reallocates another block for me) and I will have $2^{128-56}$ addresses to work with. (*This is important for subnetting IPv6*)
#### Router Advertisements
There are different types of router advertisements:
- `Unmanaged` (*A flag*)
- `Managed` (*M flag*)
- `Assisted` (*M+O+A flags*)
- `Stateless` (*O+A flags*)
And to break down the flags:
- `M` - Managed
	- Tells client device that a DHCPv6 server is running on the network, and the client should contact the DHCPv6 server in order to get an IP address
- `O` - Other
	- Tells client device that a DHCPv6 server is running on the network, and the client can request additional information such as the DNS server from it.
- `A` - Autonomous
	- Tells the client that they should take in the advertised `/64` prefix, and generate their own address using SLAAC
#### OPNsense Related
Another thing to note: PfSense seems to only have the `Track Interface` for  `IPv6 Configuration Type`, which is known as `Track Interface (Legacy)` on OPNsense. However, OPNsense has the option of `Identity association` which seem to work the same (?)
### Enabling on Interface
Assuming that the ISP has already assigned a prefix, enabling IPv6 on an interface is pretty straightforward:
- Under `Interfaces -> [$INTERFACE]`:
	- Set `IPv6 Configuration Type` to `Identity association`
	- Under `IPv6 Identity Association`:
		- Set the `Parent interface` to `WAN`
		- Set the `Assign prefix ID` to a valid hex value (*Note below*)
*SLAAC requires at least a `/64` block to work. If the ISP assigns a `/56` prefix, we will have $2^{64-56}$ subnets to hand out internally, which `0x00` to `0xff` covers.*
- Under `Services -> Router Advertisements`:
	- Create an entry
	- Check the `Enabled` box
	- Select the `Interface`
	- Select a `Mode` (*I will be using `Stateless`(SLAAC)* for everything)
	- Under `DNS Settings`:
		- Set `Recursive DNS Servers (RDNSS)` to the ULA VIP address for the interface (*Refer to [[OPNsense Migration and IPv6 Support#ULAs]]*)
### ULAs
To advertise a ULA prefix on a subnet:
Under `Interfaces -> Virtual IPs -> Settings`:
- Create a VIP
- Select the `IP Alias` mode
- Select the `Interface`
- Enter a `Network / Address` (*Use `/64` and `::1`, example: `fd00:1125:5232:2312::1/64`*)
- Set a `Description`
### Interface Isolation
Previously with IPv4, I would have a rule that passes the current interface network to NOT the RFC 1918. This was done with the invert destination and a network alias.

With IPv6, I thought it would be the same thing, so I blocked all ULA and LL addresses. However, this does not cover the GUA of the hosts, thus breaking the isolation between interfaces.

To remedy this, I created a new network alias that contained all the local addresses along with all the other networks. So something like this:
```
__lan_network
__opt1_network
__opt2_network
__opt3_network
__wireguard_network
10.0.0.0/8
172.16.0.0/12
192.168.0.0/16
fc00::/7
fe80::/10
127.0.0.0/8
::1/128
```
*Note: the `__` prefixed networks are predefined in OPNsense, they should be at the bottom of the alias list, you don't have to create them.*

Finally, for the firewall rule, I just changed the existing allow rule to support both IPv4 and IPv6 and changed the target alias to the one that was just created. With this, I can delete the rule for IPv4 and IPv6 and coalesce them into one.
# IPv6 Hosts
## Proxmox
To configure Proxmox to use SLAAC, append the following to `/etc/network/interfaces`:
```
iface vmbr0 inet6 auto
        accept_ra 2
        dhcp 0
```
- `auto` for SLAAC
- `accept_ra` to accept router advertisements 
- `dhcp 0` to disable the DHCP6 client
*Note: I had to disable IPv6 on another interface since it seemed to create a gateway metric tie?*
```
iface vmbr1 inet6 manual
		accept_ra 0
```
## TrueNAS
Circling back to the SLAAC vs DHCPv6, I have discovered that TrueNAS does not support DHCPv6. My original idea was to have a DHCPv6 server running on OPNsense, then have it hand out reserved addresses just like how I am currently doing with IPv4.

Since DHCPv6 is not supported on TrueNAS, I could have a hybrid setup where I have both SLAAC and DHCPv6 running. If that were the case, I could the `Assisted` mode for RAs. However, I didn't want to wrangle with reservations on OPNsense, and also setting setting static IPs on the hosts that don't support DHCPv6.

In the end I ended up just using `Unmanaged` (SLAAC) for everything. Since SLAAC is based off the MAC address, and all the hosts don't have any SLAAC privacy extensions enabled by default.
*Note: The IPv6 SLAAC privacy extensions uses random bits instead of the MAC address*

To configure TrueNAS to use SLAAC, check the `Autoconfigure IPv6` box under `Network -> Interfaces -> [$INTERFACE]`
## Docker
After doing some research, I think I have these options to support IPv6 on my home services:
- Enable IPv6 on the Docker daemon, then give it a ULA block which resolves using a static route. I would be able to access each application via their IPv4 port mapping or with their IPv6 ULA.
- Use host networking mode. This would remove all the port mappings and have everything run directly on the host addresses.
- Use Macvlan or IPvlan. This would become its own device on the network, having its own address.
- Use port mapping and bind it to the ULA (*`[fd00::1]:8080:80`*). This would be the most like the current IPv4 setup.
- Keep the IPv4 backend and only enable IPv6 on the reverse proxy. This would require the least configuration.
There are probably more ways to enable IPv6 connectivity, some being "better" than the others.

In my case, I'll choose to keep my IPv4 backend and enable IPv6 on my reverse proxy.
To keep it brief, here is what I had to do:
- Enable IPv6 in Docker
- Update the alias for the reverse proxy to include both IPs
- Update the rules to support dual stack
- Update the allow list on the reverse proxy
# Closing Notes
- IPv4 is ingrained into me, I love NAT (YAY!)
- IPv6 Is probably nicer to set up if I were to start my infrastructure from scratch (Future project?)
- I think Dual Stack has a place, however internal communication should just choose to use either IPv4 or IPv6, and only the edge should have support for both protocols.
