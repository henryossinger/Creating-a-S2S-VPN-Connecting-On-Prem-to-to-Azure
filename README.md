# Connecting On-Prem to Azure with a S2S

<img src="https://imgur.com/zhBFu5z.png" height="50%" width="50%" alt="Current Topology"/>

<h2>Technologies Used</h2>

- Ubuntu Server 22.04
- VMWare Workstation Player
- strongSwan
- Azure

<h2>Preface</h2>
In my hybrid cloud environment I have a lab where I created a file share on a Linux server in Azure and allowed users to connect to it via P2S. A very crucial item to know how to build; but it was far too easy. So I decided to take it a step further and setup a S2S. Setting up a S2S server takes far more networking knowledge then the P2S feature Azure provides, and it allows users in an organization to seamlessly connect to cloud resources without having to turn their VPN application on. Less overhead for users, more fun for technicians, a win-win scenario.

<h2>Azure Configuration</h2>

Let's start with what needs to be configured on the Azure side to get this going: 

1. Local Network Gateway
2. Virtual Network Gateway

For the local network gateway, all you will need is the network address + subnet of the LAN you want to connect, and the WAN IP. This can be quickly done on any device connected to the lan by viewing the network properties in settings. This can also be done via command line for any operating system. 

To pull the WAN IP via Linux, use the command: curl -4 ifconfig.me. Now that we have this info it's as simple as filling out the info in Azure and creating the gateway.

<img src="https://imgur.com/Q92wJJ8.png" height="50%" width="50%" alt="Current Topology"/>

<h3>Virtual Network Gateway</h3>

To create a Virtual Network Gateway, we need a GatewwaySubnet. This is used as the virtual networks private IP address range to be handed out in the connection. To create a gateway subnet, we first need to create a VNET. I created a seperate VNET just for organizations sake, but this gateway can be attached to any VNET of your choosing. Just remember to keep network security in mind. 

When creating a VNET you have the option to create a gatewaySubnet during the process, which is what I did. When creating the VNET I used a private IP range that is different from the range of my on-prem devices so we don't have to worry about IP addresses overlapping. Ontop of this, once we try to create the S2s, Azure will not allow us to use a gateway that has an overlapping address range. 

<img src="https://imgur.com/B54daGm.png" height="50%" width="50%" alt="Current Topology"/>

Once that is complete we can create our Virtual Network Gateway, and connect it to the gateway subnet. At this point we are done with the cloud configuration side and can start the on-prem S2S configuration. 

<h2>Generate PSK</h2>

Since I unfortunately do not have any good networking equipment to create the S2S on (came back to bite me later) I decided to host it on a Linux server. For this I will be using strongSwan which is a free open-source S2S tool for Linux.

once the server has booted, we will run these commands to make sure everything is up to date and get strongSwan installed. 
```bash
sudo apt update 
sudo apt install strongswan 
```
Now that we have this installed, we can start the configuration. There are a ton of ways to setup a S2S VPN; Azure uses PSK's for authentication of tunnel endpoints. So we will also need to install OpenSSL to generate a psk: 
```bash
sudo apt install openssl

openssl rand -band64 64 > s2spsk.txt
```
Since I am on mac, I decided to SCP this txt document over to my Mac so I could copy paste the PSK into Azure. If you are a Mac or Linux user you can do the same with the command below: 

```bash
scp user@server:/path/to/file 

/path/to/local/destination
````
<img src="https://imgur.com/OAR2tIF.png" height="50%" width="50%" alt="Current Topology"/>

Now that we have this complete, let's configure strongSwan.

<h2>strongSwan</h2>

Like all Linux applications, strongSwan config can be found in /etc/, more specifically /etc/ipsec.conf. We can start to edit this configuration with sudo nano ipsec.conf.

StrongSwan provides a ton of options for us to setup, but this one is quite basic: 

```bash
/etc/ipsec.conf
# basic configuration
config setup
        charondebug="all" #decides what to be logged
        uniqueids=yes
        strictcrlpolicy=no

# connection to azure datacenter
conn on-prem-to-azure
	authby=secret
	left=%any
	leftid=51.15.139.201
	leftsubnet=10.0.0.0/24
	right=51.15.44.48
	rightsubnet=172.16.0.0/24
	ike=aes256-sha2_256-modp1024!
	esp=aes256-sha2_256!
	keyingtries=0
	ikelifetime=1h
	lifetime=8h
	dpddelay=30
	dpdtimeout=120
	dpdaction=restart
	auto=start
```


