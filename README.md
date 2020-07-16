# nintendo-pfsense walkthrough
### Documentation on configuring a pfsense firewall to allow online play for Nintendo Switch. 

These settings should work for most games. There are a lot of conflicting posts on this topic and a lot of confusion. Hopefully this readme will provide some clarity.

A high level solution that works for this setup can be found [HERE](https://www.reddit.com/r/NintendoSwitch/comments/agv6hw/fixing_nat_for_a_nintendo_switch_using_upnp_w/). I am offering this README to provide additional details and screenshots. 

I will attempt to explain the underlying problem with these connections, so this readme may help you if you need more details, or if you don't have a pfsense firewall.

---

## Problem Defined

Nintendo's [P2P](https://en.wikipedia.org/wiki/Peer-to-peer) connections need outbound ports to stay static. I verified (via packet capture) that ports seem to be used randomly between 1024-65355. This is in conflict to some online posts that suggest only needing 45000-65355.

[Nintendo Support](https://en-americas-support.nintendo.com/app/answers/detail/a_id/22455/~/troubleshooting-issues-related-to-nat) doesn't tell us what the port range is, and instead their first suggestion is to completely disable the firewall. Another suggestion is to forward _all_ 65355 ports from the web directly to your Nintendo, including the range of standard ports from 1-1023. 

I find these to be ridiculous suggestions. They are not at all helpful to resolve this issue.

Nintendo's only valid solution is to setup a [DMZ](https://en.wikipedia.org/wiki/DMZ_%28computing%29) for the Switch to use. However, depending on the DMZ configuration, a user may still run into the same connection problems.

The reason pfsense needs a custom configuration for these connections is because pfsense automatically changes the source port on outgoing connections. This is a security a security best practice. pfsense [recognizes](https://docs.netgate.com/pfsense/en/latest/nat/static-port.html) that this may cause some applications to break.

In addition, pfsense has [uPNP](https://en.wikipedia.org/wiki/Universal_Plug_and_Play) and [NAT PMP](https://en.wikipedia.org/wiki/NAT_Port_Mapping_Protocol) disabled by default to keep devices from making their own network configurations. This is another security best practice. 
- These protocols allow an applications to make their own NAT settings and port forwarding configurations without intervention. If malware were to use these protocols, it can make it's own connections at will.

Nintendo doesn't publish much about the Switch's security implementation, and a quick Google search suggests that it [hasn't](https://www.deseret.com/entertainment/2020/4/21/21230137/nintendo-switch-hack-two-step-verificaiton-secure-device-data-compromised-eshop-security) been adequately tested. 

This would fall in line with Nintendo's poor firewall configuration suggestions.

---

## Solution

A fairly easy solution is to allow _only_ the switch to use `NAT PMP` (or `uPNP`) connections and static outbound port mappings. 

I tested both `uPNP` and `NAT PMP` and they both work, but you only need _one_ of them turned on.

While this comes with the security risk mentioned above, we can at least limit this traffic to the IP address of the Switch. If your router only supports these settings to an entire network, you may be able to use a [CIDR](https://en.wikipedia.org/wiki/Classless_Inter-Domain_Routing#CIDR_notation) notation of `/32` to specify _only_ your Switch and not the entire network.

Still, if an attacker (or malware) were to use the IP address assigned to the Switch, they could make their own port mappings and NAT connections without hinderance. We have to accept this risk if we want to make online P2P connections with the switch.

---

## General Firewall Configuration

- The switch needs a static IP address so you can target it directly.
- `NAT PMP` connections are allowed only to the switch's IP address using [CIDR](https://en.wikipedia.org/wiki/Classless_Inter-Domain_Routing#CIDR_notation) notation `/32`.
- A NAT rule is set to use static outbound port mappings, only for the switch's IP address (again using [CIDR](https://en.wikipedia.org/wiki/Classless_Inter-Domain_Routing#CIDR_notation) notation).

**Port forwarding is not needed:**

- If you are using a separate firewall, make sure your ISP equipment is forwarding all traffic directly to your firewall and only your firewall is implementing `NAT`. 
- If we use `NAT PMP` or `uPNP`, the Switch is allowed to forward whatever port it needs for any specific connection.
- Some routers limit the amount of ports that you can forward, which make a port forwarding solution tedious at best. For instance, pfsense documentation states that only a range of 500 ports can be forwarded for any one rule, so to forward all ports from 1024-65535 would require some 129 NAT rules.

--- 

## pfsene Configuration

To get things working we need to make 3 changes:
- Static DHCP IP Address Mapping to the Switch.
- Turning on `NAT PMP`
- Configuring Static Outbound Port Mappings

### Setting up a static DHCP IP Address Mapping.

#TODO: Add Screenshots

**pfsense Documentation:**
- [Static IP Mappings](https://docs.netgate.com/pfsense/en/latest/dhcp/dhcp-server.html#static-ip-mappings)


Instead of changing the networking settings on the Switch and manually configuring the IP address to be static, we can configure the router (DHCP Server) to _always_ assign the same IP address to the switch.

To do this, we need:
- The Switch's MAC address
- The IP address we want the Switch to use
- The ability to setup DHCP static IP Mappings in our firewall of choice.

### Find the [MAC](https://en.wikipedia.org/wiki/MAC_address) address of the Switch:

- From the Switch home screen, go to Settings > Internet and look for `System MAC Address`. It should be in this format: `xx:xx:xx:xx:xx:xx`

- [Nintendo Docs](https://en-americas-support.nintendo.com/app/answers/detail/a_id/22397/~/how-to-find-a-nintendo-switch-consoles-mac-address)

### Static IP Mappings in pfsense:

#TODO: Add Screenshots

- Click on Services and choose `DHCP Server`.

- Choose the network interface where your Switch resides.

- For the `Range` settings, make sure to leave at least 1 address _out_ of the DHCP range, so you can assign it statically.
	- For instance, starting your range on `.10` would give you 9 IP addresses (`.1` - `.9`) that you could assign statically.
	- Example: Starting IP in the range: `192.168.1.10`. Ending IP in the range: `192.168.1.50`. 
		- This range gives us 9 IPs to assign statically (`192.168.1.1` - `192.168.1.9`) and 41 IP's for DHCP to assign dynamically (`.10` - `.50`)

- Scroll to the bottom of the page and locate the section labeled: 'DHCP Static Mappings for this Interface.'

- Click `ADD`.

- Enter the MAC Address of your Switch in the MAC Address field. 

- Enter the IP Address you want the Switch to have. Using our above example, we could enter `192.168.1.5` because that IP is within our range of IPs that are not used with `DHCP` (`.1` - `.9`).

- Enter a hostname for the Switch. This is so you can later identify what setting goes with what device, and does not have to match the actual host name of a device. 
	- `Switch` works just fine.

- Ignore all the other settings.

- Click the Save button at the bottom.

- Turn your switch completely off and then turn it back on again.

- Go to the Internet Settings on your Switch and verify that it is receiving the correct IP address. 
	- In our example it would be: `192.168.1.5`

---

### Setting up `NAT PMP` connections in pfsense

**pfsense Documentation:**

- [uPNP and NAT PMP](https://docs.netgate.com/pfsense/en/latest/services/configuring-upnp-and-nat-pmp.html)

#TODO: Add walk through with screen shots 

---

### Setting up a static outbound port in pfsense

**pfsense Documentation:**

- [Static Port with Outbound NAT](https://docs.netgate.com/pfsense/en/latest/nat/static-port.html)

#TODO: Add walk through with screen shots 

---
## Testing

From the Switch's main screen, go to settings and select 'Network' > 'Network Test'. When you run a 'Network Test' on the Nintendo Switch, it has a `NAT Type`. Nintendo hasn't documented what these grades mean, but 'A' and 'B' grades seem to work for most games. Any grade below 'B' may give you problems. 

Without making any changes to pfsense the `NAT Type` was a 'D' and connections made through the game 'Animal Crossing' would fail 100% of the time. Animal Crossing is the only game I tested.

**Tip:** Use the Network connection function on the Switch to troubleshoot, instead of trying to make connections in a game. It's fairly easy to make a change to your firewall, test the connection, make another change, test again, etc. Once you are showing a grade of 'A' or 'B', launch the game and try the connection there.