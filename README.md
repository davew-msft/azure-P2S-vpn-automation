## How to Create a Point-to-Site (P2S) VPN to Azure, QUICKLY

Why am I doing this?

* it is neither intuitive nor simple to create a P2S vpn and I find I need to do this every time I work with a new customer or on a new project
* useful if your company requires all VMs to be behind a firewall without access to RDP or SSH
* quickly set this up yourself in your RG/VNET if you don't want to have SSH or RDP access open on a public IP
  * frankly, this always freaks me out.  It seems like every script-kiddie loves to look for open ports 22 and 3389 and try to hack them, this gives a bit of security.  
* our existing Azure P2S documentation is cumbersome, confusing, and takes hours to read and implement
* alternative solutions:
  * Azure Bastion (works well but I hate jump hosts, a vpn will allow you to do ssh tunneling and makes your azure resources feel like they are on your network)

Assumptions/things you have to setup.  My scripts assume you have NOTHING setup yet.  If you already have an existing VNET you need to get working, just change the params.  The script is idempotent.  

Here's how I tested.  I built:  
* a RG
* a ubu server with NO public inbound ports (no ssh/22 access)
* a win server with NO public inbound ports (no rdp/3389 access)
* the two servers are on the same VNET (mine is called rgP2S-vnet)
* one subnet that has both servers (default is 10.0.0.0/24)
* NO public IP addresses on either machine

We want to build a P2S VPN that we can use from a Windows/mac/ubuntu machine to connect to both machines either over 22 (ssh) or 3389 (rdp).  

I'm certainly not a networking expert so maybe there are smarter vnet/subnet configurations, I'm focused just on getting the P2S vpn working.  The pattern is the most important thing.  

For the VPN gateway you can use either SSTP or OpenVPN. On the desktop, I'm a linux and windows user (and sometimes mac) and I frankly like SSTP better than ovn.  It's better integrated with windows and it's not difficult to config in ubuntu either.  **The 'Basic' vpn gateway sku does not support OpenVPN or mac clients.  You'll need to change the sku if you need either of those**.  

**This is the FAST way to get your P2S vpn working.  You don't have to install any software on your machine**
## Generate Your Certs

You gotta do this from cloudshell (shell.azure.com) using a `bash` terminal.  VPN Clients use certs to connect, not passwords.  The process:  We'll create a self-signed cert by creating a root key (guard this from getting into the wild), then create a public cert (and load it to Azure later), then generate as many client certs as you need/want.  We can generate more later, we just need one for the computer we are on.  In cloudshell bash:

```bash
root_name="azurep2s"
mkdir -p certs
cd certs
# this cert expires in 10 yrs, has no passphrase, change the subj if you want to
# this will give you the self-signed cert and the private key
openssl req -x509 -newkey rsa:4096 -keyout ${root_name}_key.pem -out ${root_name}_cert.pem -days 3650 -nodes -subj "/C=US/ST=NJ/L=OceanGrove/O=davew/OU=Org/CN=azureps2root.local"

# now create a client key and csr. Change the variable and run it for as many clients as you need.
client="davew-client"
openssl genrsa -out ${client}.key 4096
openssl req -new -key ${client}.key -out ${client}.csr
openssl x509 -req \
    -in ${client}.csr \
    -CA ${root_name}_cert.pem \
    -CAkey ${root_name}_key.pem \
    -CAcreateserial \
    -addtrust clientAuth \
    -out ${client}.pem \
    -days 3650 \
    -sha256
# azure requires a p12 bundle (aka a pfx file), let's build that
openssl pkcs12 \
    -in ${client}.pem \
    -inkey ${client}.key \
    -certfile ${root_name}_cert.pem \
    -export -out "${client}.p12" 
```


## Build the VPN

Now go to shell.azure.com and run the folowing from a `powershell` cloudshell:

```powershell
#vars to change, these are MY settings, all of this will be created if it doesn't already exist
#best to run these one command at-a-time
$sub_name = "airs"
$RG = "rgP2S"
$Location = "EastUS2"
$VNetName  = "rgP2S-vnet"
$MachineSubnetName = "default"  #this is where the VMs would go
$GatewaySubnetName = "GatewaySubnet"  #this is where the vpn goes, this is the magic keyword, don't change it
$VNetPrefix1 = "10.0.0.0/24"   #you can have as many prefixes as you want for your needs
$VNetPrefix2 = "10.1.0.0/24"
$VNetPrefix3 = "10.2.0.0/24"
$SubPrefix = "10.0.0.0/24"  #251 addresses (+5 reserved for Azure), must be a subset of the VNET space
$GWSubPrefix = "10.1.0.0/24"   #must be a subset of the VNET space
$VPNClientAddressPool = "172.16.201.0/24"
$GatewayName = "rgP2S-Gateway"
$GatewayIPName = "rgP2S-GatewayIP"
$GatewayIPconfName = "gatewayipconf"
$DNS = "10.2.0.4"

Select-AzSubscription -SubscriptionName $sub_name
New-AzResourceGroup -Name $RG -Location $Location

# build the subnets first so the objects can be passed to the vnet config
$machinesub = New-AzVirtualNetworkSubnetConfig -Name $MachineSubnetName -AddressPrefix $SubPrefix
$gatewaysub = New-AzVirtualNetworkSubnetConfig -Name $GatewaySubnetName -AddressPrefix $GWSubPrefix

New-AzVirtualNetwork `
   -ResourceGroupName $RG `
   -Location $Location `
   -Name $VNetName `
   -AddressPrefix $VNetPrefix1, $VNetPrefix2, $VNetPrefix3 `
   -Subnet $machinesub, $gatewaysub `
   -DnsServer $DNS

# get the objects
$vnet = Get-AzVirtualNetwork -Name $VNetName -ResourceGroupName $RG
$subnet = Get-AzVirtualNetworkSubnetConfig -Name $GatewaySubnetName -VirtualNetwork $vnet

# request a public IP for the vpn gateway
# don't worry, dynamic does NOT mean it will ever change
$pip = New-AzPublicIpAddress -Name $GatewayIPName -ResourceGroupName $RG -Location $Location -AllocationMethod Dynamic
$ipconf = New-AzVirtualNetworkGatewayIpConfig -Name $GatewayIPconfName -Subnet $subnet -PublicIpAddress $pip

# build the VPN gateway, this could take an hour, it's likely cloudshell will time out, that's ok, see the next command
New-AzVirtualNetworkGateway `
    -Name $GatewayName `
    -ResourceGroupName $RG `
    -Location $Location `
    -IpConfigurations $ipconf `
    -GatewayType "Vpn" `
    -VpnType RouteBased `
    -EnableBgp $false `
    -GatewaySku "Basic" `
    -VpnClientProtocol "SSTP"

# make sure it worked.  If cloudshell timed out run these 2 commands until ProvisioningState = Succeeded
# and re-run your vars section above first
$GatewayObj = Get-AzVirtualNetworkGateway -Name $GatewayName -ResourceGroup $RG
$GatewayObj

# set the VPN client address pool
Set-AzVirtualNetworkGateway -VirtualNetworkGateway $GatewayObj -VpnClientAddressPool $VPNClientAddressPool


```

# Add the RootCert to our VPN gateway

This is manual, sorry.  This is a limitation of cloudshell and vpn configurations:
* find your vnet gateway in Azure Portal (mine is `rgP2S-Gateway`)
* go to Point-to-site configuration.  
* Add a new root certficate.  
* The certificate data can be found by doing `cat azurep2s_cert.pem` and then copying everything between the comments and pasting that into 'Public certificate data`
* download the VPN client now from this screen
* download the p12 file from cloudshell
* if running windows, rename it to pfx
* doubleclick it and follow the instructions, don't change any values
* open and run the vpn client installer you just downloaded
* now connect with the vpn client in your network settings
* you should be connected.  
* I can test this by connecting to my vms with ssh and rdp.  On my machine I use 
  * `ssh 10.0.0.4`
  * `mstsc /v:10.0.0.5`
