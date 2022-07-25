# Azure Lab - Migrate Hyper-V VMs to Azure with Azure Migrate & Nested Virtualization

This repo is a lab to simulate **Hyper-V VMs migration to Azure** using [Azure Migrate](https://docs.microsoft.com/en-us/azure/migrate/migrate-services-overview).

It simulates an Hyper-V bare-metal server using an Azure VM that hosts an Hyper-V Manager on which we create VMs. This is called nested virtualization.

**Big Picture**:

![Big Picture](docs/bigpicture.png)

- [Azure Lab - Migrate Hyper-V VMs to Azure with Azure Migrate & Nested Virtualization](#azure-lab---migrate-hyper-v-vms-to-azure-with-azure-migrate--nested-virtualization)
  - [Infrastructure deployment](#infrastructure-deployment)
  - [Hyper-V Host installation & configuration](#hyper-v-host-installation--configuration)
  - [Hyper-V Guest VM Creation](#hyper-v-guest-vm-creation)
    - [Linux - Ubuntu](#linux---ubuntu)
    - [Linux - CentOS 7.9](#linux---centos-79)
    - [Windows - Server Server 2019](#windows---server-server-2019)
- [Azure Migrate](#azure-migrate)
  - [Azure Migrate - Discovery and assessment](#azure-migrate---discovery-and-assessment)
  - [Azure Migrate - Server Migration](#azure-migrate---server-migration)
- [Azure Migrate - Design at scale considerations](#azure-migrate---design-at-scale-considerations)
  - [Hyper-V](#hyper-v)
  - [VMware](#vmware)
- [Sources](#sources)

## Infrastructure deployment

If you want to customize the deployment, you can use the `parameters.json` file to specify a location, tags and storage account name for the deployment.

```bash
# Create a resource group
$ az group create --location westeurope --name MyHyperVRG
# Clone repo
$ git clone https://github.com/chambras/lab-azuremigrate-hyperv-nestedvirtualization
$ cd lab-azuremigrate-hyperv-nestedvirtualization/bicep
# Dry run the deployment to verify the template
$ az deployment group create -g MyHyperVRG -f infra-hyperV.bicep -p @parameters.json -w --verbose
# Deploy Bicep code
$ az deployment group create -g MyHyperVRG -f infra-hyperV.bicep -p @parameters.json --verbose
# To see the output of the deployment and get the Publi IP of the Hyper-V Host VM
$ az deployment group show -g MyHyperVRG -n infra-hyperV --query "properties.outputs" -o yaml
```

## Hyper-V Host installation & configuration

When the deployment is done, **connect** to provisionned VM and **install Hyper-V tools**: execute following Powershell command as **Administrator**:
```powershell
Install-WindowsFeature -Name "Hyper-V" -IncludeManagementTools -Restart
```
VM will restart when installation is done.

Connect again to the machine.

To allow nested VMs communication, create an internal switch:
```powershell
New-VMSwitch –SwitchName "NATSwitch" –SwitchType Internal
```

Sample output:
```powershell
Name      SwitchType NetAdapterInterfaceDescription
----      ---------- ------------------------------
NATSwitch Internal     
```

Create a new IP address and assign it to previously created switch:
```powershell
New-NetIPAddress –IPAddress "192.168.0.1" -PrefixLength 24 -InterfaceAlias "vEthernet (NATSwitch)"
```

Sample output:
```powershell
IPAddress         : 192.168.0.1
InterfaceIndex    : 12
InterfaceAlias    : vEthernet (NATSwitch)
AddressFamily     : IPv4
Type              : Unicast
PrefixLength      : 24
PrefixOrigin      : Manual
SuffixOrigin      : Manual
AddressState      : Tentative
ValidLifetime     : Infinite ([TimeSpan]::MaxValue)
PreferredLifetime : Infinite ([TimeSpan]::MaxValue)
SkipAsSource      : False
PolicyStore       : ActiveStore

IPAddress         : 192.168.0.1
InterfaceIndex    : 12
InterfaceAlias    : vEthernet (NATSwitch)
AddressFamily     : IPv4
Type              : Unicast
PrefixLength      : 24
PrefixOrigin      : Manual
SuffixOrigin      : Manual
AddressState      : Invalid
ValidLifetime     : Infinite ([TimeSpan]::MaxValue)
PreferredLifetime : Infinite ([TimeSpan]::MaxValue)
SkipAsSource      : False
PolicyStore       : PersistentStore
```

Create NAT on created switch:
```powershell
New-NetNat –Name "NatNetwork" –InternalIPInterfaceAddressPrefix "192.168.0.0/24"
```
Sample output:
```powershell
Name                             : NatNetwork
ExternalIPInterfaceAddressPrefix : 
InternalIPInterfaceAddressPrefix : 192.168.0.0/24
IcmpQueryTimeout                 : 30
TcpEstablishedConnectionTimeout  : 1800
TcpTransientConnectionTimeout    : 120
TcpFilteringBehavior             : AddressDependentFiltering
UdpFilteringBehavior             : AddressDependentFiltering
UdpIdleSessionTimeout            : 120
UdpInboundRefresh                : False
Store                            : Local
Active                           : True
```

Turn off Windows Defender Firewall
```powershell
Set-NetFirewallProfile -Profile Domain,Public,Private -Enabled False 
```

*Optional* - Initialize *E:* disk to store Hyper-V VMs:
```powershell
$Disk = Get-Disk -Number 1 | Where-Object -FilterScript { $_.PartitionStyle -Eq "RAW" } | Initialize-Disk -PassThru | New-Volume -FileSystem NTFS -DriveLetter E -FriendlyName 'Hyper-V'   
$HyperVPath = "$($Disk.DriveLetter):\Hyper-V"
$null = New-Item -Path $HyperVPath -ItemType Directory -Force
Set-VMHost -VirtualHardDiskPath $HyperVPath -VirtualMachinePath $HyperVPath
```

## Hyper-V Guest VM Creation

### Linux - Ubuntu

**Download** [Ubuntu Server](https://ubuntu.com/download/server) **.iso** on Hyper-V host machine. Several links here: [Ubuntu 18.04](https://releases.ubuntu.mirror.malte-bittner.eu/18.04.6/ubuntu-18.04.6-live-server-amd64.iso), [Ubuntu 20.04 LTS](https://mirrors.ircam.fr/pub/ubuntu/releases/20.04.3/ubuntu-20.04.3-live-server-amd64.iso), [Ubuntu 21.10](https://www-ftp.lip6.fr/pub/linux/distributions/Ubuntu/releases/21.10/ubuntu-21.10-live-server-amd64.iso).

Example:
```powershell
$locationFolder="E:\Hyper-V\"

# Download Ubuntu 18.04
Invoke-WebRequest -Uri "https://releases.ubuntu.com/18.04.6/ubuntu-18.04.6-live-server-amd64.iso" -OutFile "$($locationFolder)ubuntu-18.04.6-live-server-amd64.iso"

# Download Ubuntu 20.04 LTS
Invoke-WebRequest -Uri "https://releases.ubuntu.com/20.04/ubuntu-20.04.4-live-server-amd64.iso" -OutFile "$($locationFolder)ubuntu-20.04.3-live-server-amd64.iso"

# Download Ubuntu 21.10
Invoke-WebRequest -Uri "https://www-ftp.lip6.fr/pub/linux/distributions/Ubuntu/releases/21.10/ubuntu-21.10-live-server-amd64.iso" -OutFile "$($locationFolder)ubuntu-21.10-live-server-amd64.iso"
```

Create new Hyper-V virtual machine with Hyper-V Manager:

![Hyper-V Machine](docs/hyper-v-create-vm01.png)

![Hyper-V Machine](docs/hyper-v-create-vm02.png)

![Hyper-V Machine](docs/hyper-v-create-vm03.png)

![Hyper-V Machine](docs/hyper-v-create-vm04.png)

![Hyper-V Machine](docs/hyper-v-create-vm05.png)

![Hyper-V Machine](docs/hyper-v-create-vm06.png)

![Hyper-V Machine](docs/hyper-v-create-vm07.png)

![Hyper-V Machine](docs/hyper-v-create-vm08.png)

![Hyper-V Machine](docs/hyper-v-create-vm09.png)

![Hyper-V Machine](docs/hyper-v-create-vm10.png)

![Hyper-V Machine](docs/hyper-v-create-vm11.png)

Configure language and keyboard, then configure Network:
![Ubuntu Machine](docs/ubuntu02.png)

![Ubuntu Machine](docs/ubuntu03.png)

![Ubuntu Machine](docs/ubuntu04.png)

It is also possible to configure guest VM network (no dhcp, static private ip address, custom gateway (to NAT) & DNS servers) by updating `/etc/netplan/99-installer-config.yaml`:
```yaml
# This is the network config writter by 'subiquity'
network:
  ethernets:
    eth0:
      addresses: [192.168.0.3/24]
      gateway4: 192.168.0.1
      nameservers:
        addresses: [8.8.8.8, 8.8.4.4]
  version: 2
```

Apply the configuration:
```bash
netplan generate
netplan apply
```

View updated configuration:
```bash
ip -c a s
```
![Ubuntu Machine](docs/ubuntu04.png)

### Linux - CentOS 7.9

Download [CentOS 7 Linux](https://www.centos.org/download/) *.iso* file and create Hyper-VM Virtual Machine as seen before.

To manually update network configuration on CentOS 7 machine, update `/etc/sysconfig/network-scripts/ifcfg-eth0` file:

```bash
# static IP address on CentOS 7 or RHEL 7#
HWADDR=00:08:A2:0A:BA:B8
TYPE=Ethernet
BOOTPROTO=none
# Server IP #
IPADDR=192.168.0.5
# Subnet #
PREFIX=24
# Set default gateway IP #
GATEWAY=192.168.0.1
# Set dns servers #
DNS1=8.8.8.8
DNS2=8.8.4.4
DNS3=1.1.1.1
DEFROUTE=yes
IPV4_FAILURE_FATAL=no
# Disable ipv6 #
IPV6INIT=no
NAME=eth0
DEVICE=eth0
ONBOOT=yes
```

Restart network service:
```bash
systemctl restart network
```

View updated configuration:
```bash
ip a s eth0
```

Screenshot:
![CentOS](docs/centos01.png)

### Windows - Server Server 2019

Download [Windows Server 2019](https://www.microsoft.com/en-US/evalcenter/evaluate-windows-server-2019?filetype=ISO) trial *.iso* file and create Hyper-VM Virtual Machine as seen before.

Apply the following network configuration on Windows VM:
![Windows](docs/windows01.png)


# Azure Migrate

Now, we suppose we have created serveral Hyper-V VMs.

Let's migrate them to Azure using Azure migrate.

First step: **Create a project**.
![Azure Migrate](docs/azure-migrate01.png)

![Azure Migrate](docs/azure-migrate02.png)

Azure Migrate is splitted in two tools:
* Azure Migrate: Discovery and assessment
* Azure Migrate: Server Migration

## Azure Migrate - Discovery and assessment

To discover "on-premises servers", it is required to deploy an appliance:
![Azure Migrate](docs/azure-migrate03.png)

Select Hyper-V, give a name to your appliance, Generate a key and download Azure Migrate appliance .zip file:
![Azure Migrate](docs/azure-migrate04.png)

Unzip AzureMigrationAppliance.zip and **Import** this appliance on Hyper-V:
![Import Appliance](docs/importappliance01.png)

![Import Appliance](docs/importappliance02.png)

![Import Appliance](docs/importappliance03.png)

![Import Appliance](docs/importappliance04.png)

![Import Appliance](docs/importappliance05.png)

![Import Appliance](docs/importappliance06.png)

Start appliance VM & connect:
![Import Appliance](docs/importappliance07.png)

![Import Appliance](docs/importappliance08.png)

After signin, Configure Network Connections:
![Import Appliance](docs/importappliance09.png)

**Note**: on a "corporate environment" where sometimes machines do not have access to the Internet, it is possible to set up a proxy.

Edge will then launch automatically:
![Import Appliance](docs/importappliance10.png)

Execute prerequisites:
![Azure Migrate Appliance](docs/azuremigrateappliance04.png)

Paste the key given on Azure Portal and Login:
![Azure Migrate Appliance](docs/azuremigrateappliance05.png)

Continue with [Azure Device Login](https://www.microsoft.com/devicelogin):
![Azure Migrate Appliance](docs/azuremigrateappliance06.png)

![Azure Migrate Appliance](docs/azuremigrateappliance07.png)

![Azure Migrate Appliance](docs/azuremigrateappliance08.png)

On Hyper-V host VM: 
* Create an *azuremigrateuser* account, member of following groups:
  * Administrators
  * Hyper-V Administrators
  * Performance Monitor Users
  * Remote Management Users

Screenshot:
![Azure Migrate User](docs/azuremigrate_rights.png)

* Enable also Powershell Remote:
  * ```powershell Enable-PSRemoting -force```

Back to Azure Migrate Appliance, add *azuremigrateuser* credentials:
![Azure Migrate Appliance](docs/azuremigrateappliance09.png)

Add single item: give previously created *azuremigrateuser* and IP Address of Hyper-V host: `192.168.0.1`
![Azure Migrate Appliance](docs/azuremigrateappliance10.png)

Start discovery:
![Azure Migrate Appliance](docs/azuremigrateappliance11.png)

When discovery is finished, go back to the Azure portal on Azure Migrate:
![Azure Migrate Portal](docs/azure-migrate05.png)

We can see discovered servers.

Let's create assessment now:
![Azure Migrate Portal](docs/azure-migrate06.png)
  
![Assessment](docs/assessment01.png)

![Assessment](docs/assessment02.png)

To view assessment result, click here:
![Assessment](docs/assessment03.png)

![Assessment](docs/assessment04.png)

![Assessment](docs/assessment05.png)
Let's now play with the migration tools!

## Azure Migrate - Server Migration

![Discover](docs/discover01.png)

Select a target region:
![Discover](docs/discover02.png)

Download the Azure Site Recovery Provider software installer and the registration key file on Hyper-V host VM:
![Discover](docs/discover03.png)

![Discover](docs/discover04.png)
Install the Azure Site Recovery Provider:
![Azure Site Recovery Provider Setup](docs/azuresiterecoveryprovidersetup01.png)

![Azure Site Recovery Provider Setup](docs/azuresiterecoveryprovidersetup02.png)

![Azure Site Recovery Provider Setup](docs/azuresiterecoveryprovidersetup03.png)

Register using the key file:
![Azure Site Recovery Provider Setup](docs/azuresiterecoveryprovidersetup04.png)

![Azure Site Recovery Provider Setup](docs/azuresiterecoveryprovidersetup05.png)

![Azure Site Recovery Provider Setup](docs/azuresiterecoveryprovidersetup06.png)

Wait untill the Hyper-V host appears Connected on Azure portal and **Finalize registration**:
![Discover](docs/discover05.png)

Let's now replicate a Hyper-V VM to Azure.

First of all, we need to create a new resource group that will receive replicated Hyper-V VMs.
In this resource group, we need to provision:
* A virtual Network
* A storage account (**without blob soft delete enabled**)

You can do it manually or just execute:
```bash
# Create a resource group
$ az group create --location northeurope --name TargetRG
# Deploy Bicep code
$ az deployment group create --resource-group TargetRG --template-file infra-target.bicep
```

![Replicate](docs/replicate01.png)

![Replicate](docs/replicate02.png)

![Replicate](docs/replicate03.png)

Select Hyper-V VMs to migrate:
![Replicate](docs/replicate04.png)

Select target RG, VNet and Storage Account (used for diagnostics)
![Replicate](docs/replicate05.png)

Define an Azure VM Name and select OS Type:
![Replicate](docs/replicate06.png)

Select disks to replicate:
![Replicate](docs/replicate07.png)

Define tags:
![Replicate](docs/replicate08.png)

Replicate:
![Replicate](docs/replicate09.png)

Follow replication status:
![Replicate](docs/replicate10.png)

When replication is done, we can perform test migration or migration directly.

![Migrate](docs/migrate01.png)

Let's migrate  VMs directly:
![Migrate](docs/migrate02.png)

Wait untill finish:
![Migrate](docs/migrate03.png)

Observe result in target resource group:
![Migrate](docs/migrate04.png)

Check Azure VMs are running, connect to them and check appliancations are Running.

# Azure Migrate - Design at scale considerations

## Hyper-V

Recommendations:
* Number of appliance:
  * Based on [appliance limits](https://docs.microsoft.com/en-us/azure/migrate/migrate-appliance#appliance---hyper-v), company network policies and geography. 
    
    Limit: 5,000 servers running on a Hyper-V environment max
    
    Limit: 300 Hyper-V hosts maximum.  
  * Deploy the appliance as close as possible to Hyper-V hosts to minimize latency. 

* [Network flows](https://docs.microsoft.com/en-us/azure/migrate/migrate-appliance-architecture#discovery-and-collection-process): 
  * `Hyper-V appliance => Hyper-V hosts` WinRM port 5985 (HTTP)
  * `Hyper-V appliance => Azure Migrate` TCP port 443 (HTTPS). 
  
  If Hyper-V appliance has no direct Internet connectivity and a proxy is used, allow these [URLs](https://docs.microsoft.com/en-us/azure/migrate/migrate-appliance#url-access)
* Number of Azure Site Recovery Provider:
  * 1 per Hyper-V host 
* Replication limit: replicate up to 10 machines together. If you need to replicate more, then replicate them simultaneously in [batches of 10](https://docs.microsoft.com/en-us/azure/migrate/tutorial-migrate-hyper-v?tabs=UI#replicate-hyper-v-vms).

* [Post-migration best practices](https://docs.microsoft.com/en-us/azure/migrate/tutorial-migrate-hyper-v?tabs=UI#post-migration-best-practices)

## VMware

Recommendations:
* Number of appliance:
  * Based on [appliance limits](https://docs.microsoft.com/en-us/azure/migrate/migrate-appliance#appliance---vmware), company network policies and geography. 
    
    Limit: 10,000 servers running on a vCenter Server
    
    Limit: An appliance can connect to a **single vCenter Server**. 
  * Deploy the appliance as close as possible to Hyper-V hosts to minimize latency. 

* [Network flows](https://docs.microsoft.com/en-us/azure/migrate/migrate-appliance-architecture#discovery-and-collection-process): 
  * `VMware appliance => vCenter server` TCP port 443 (HTTPS)
  * `VMware appliance => Azure Migrate` TCP port 443 (HTTPS). 
  
  If VMware appliance has no direct Internet connectivity and a proxy is used, allow these [URLs](https://docs.microsoft.com/en-us/azure/migrate/migrate-appliance#url-access)
* [Agentless vs Agent-base Vmware migration option](https://docs.microsoft.com/en-us/azure/migrate/server-migrate-overview#compare-migration-methods)
  * Using Agentless migration option, a maximum of 500 VMs can be simultaneously replicated from a vCenter Server. In the portal, you can select up to 10 machines at once for replication. To replicate more machines, add in batches of 10.

* [Post-migration best practices](https://docs.microsoft.com/en-us/azure/migrate/tutorial-migrate-vmware#post-migration-best-practices)


# Sources

Following links were used to create this lab:
* https://www.cyberciti.biz/faq/howto-setting-rhel7-centos-7-static-ip-configuration/
* https://www.piesik.me/2019/02/04/Azure-Nested-Virtualization-Internet-Connection/#
* https://www.nakivo.com/blog/hyper-v-nested-virtualization-on-azure-complete-guide/
* https://docs.microsoft.com/en-US/azure/migrate/tutorial-discover-hyper-v