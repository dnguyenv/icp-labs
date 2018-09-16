# Install IBM Cloud Private Cluster Locally On Your Laptop
This tuturial walks you through steps to setup an IBM Cloud Private cluster on your workstation (eg: Your laptop). You can extend the application of this tutorial to a more advanced environment accordingly to your need.
The environment I'm using in the tutorial is Mac OS High Sierra 10.13.4 with VirtualBox 5.2.18 and `IBM Cloud private Community Edition`
Storage topics are also not discussed in this tutorial, so all data will be stored inside the VMs
## System requirements
System requirements to install IBM Cloud Private (ICP) varies based on your architecture. ICP supports Linux 64 bit on x86 architecture (Red Hat Enterprise Linux and Ubuntu 16.04 LTS) or Linux on IBM Power. In this tutorial, I use `Ubuntu 16.04 LTS`. 
## Determine your cluster architecture
First you need to decide the architecture of your cluster. A `standard` configuration for a multi-node ICP cluster is 
```
A single master
A single proxy node
Three worker nodes 
```
but I will compact the architecture even more, to make it 3 virtual machines (VMs) only

```
One VM contains both proxy and master node
Two VMs each hosts a worker node
```
Here is now it looks:
<image src="images/arch.png" />
## Setup infrastructure 

### Configure VirtualBox network

### Create internal private network for the nodes

As shown in the cluster architecture, I will create a Host-Only network interface to attach all the nodes in. With Host-Only network adapter, I can access the nodes from my host

```VBoxManage hostonlyif create```

After that command, you will now the network name (eg: `vboxnetx`). Use it for next command to configure the network IP:

`VBoxManage hostonlyif ipconfig vboxnet3 --ip 173.0.1.1`

Then configure the DHCP server:

`VBoxManage dhcpserver add --ifname vboxnet3 --ip 173.0.1.1 --netmask 255.255.255.0 --lowerip 173.0.1.100 --upperip 173.0.1.200`

`VBoxManage dhcpserver modify --ifname vboxnet3 --enable`

### Enable internet access from the VMs

We can either create a NAT network or use the defautl NAT option to allow the VMs to access the internet through my Mac. To make it simple, I will attach the NAT adapter to my VMs. We will do this once we have the VM provisioned

## Provision the VMs

We will provision the first VM (boot-master-proxy), setup necessary softwares on it, then clone it to make the other two VMs to minimize the repeated steps.

Download the image from here: `http://releases.ubuntu.com/16.04/ubuntu-16.04.5-server-amd64.iso`

Depending on how much `resource` you have in your host, you will provision your VM accordingly. I allocate 8GB RAM, 80GB to my first VM

Assuming you've done with provisioning your VM based on the Ubuntu image, now on VirtualBox GUI, go to `Settings` view of the VM, navigate to `Network` and then attach 2 adapters to the VM as following

NAT for internet access from the VM
<image src="images/nat.png">
Host-Only, select the one you created before, to enable access to the VM from host
<image src="images/host-only.png">

Now launch the VM and setup the OS following the instruction. Once you have the OS ready, assign a static IP address to the VM by changing the configuration in `/etc/network/interraces`

Since the VM is attached to the Host-Only network interface, we can ssh to it from the host and modify the network interfaces configuration:

```
$ssh <user>@172.0.1.100
$sudo vi /etc/network/interfaces
```

make it something like this:

```
# This file describes the network interfaces available on your system
# and how to activate them. For more information, see interfaces(5).

source /etc/network/interfaces.d/*

# The loopback network interface
auto lo
iface lo inet loopback

# The primary network interface (NAT)
auto enp0s3
iface enp0s3 inet dhcp
# The Host-Only network
auto enp0s8
iface enp0s8 inet static
        address 172.0.1.100
        netmask 255.255.255.0
```
Notes:
Sometimes the network interface name (eg: enp0sx) does not show up in the `/etc/network/interfaces` file hence you need to figure out what the name is for each adapter. Use this command inside the VM to determine

```
dmesg | grep eth
```

Change the host name of the VM, make it `boot-master-proxy`

```
$ sudo vi /etc/hostname # then make it boot-master-proxy
```

## Install necessary softwares on the boot-master-proxy node

### Enable root login remotely via ssh to the VM

Set a password for root by SSH to the VM and execute these from inside

```
$sudo su - # provide your user's password here
$passwd 
```

Enable remote login as root

```
$ sed -i 's/prohibit-password/yes/' /etc/ssh/sshd_config
$ systemctl restart ssh
```

### Update Net Time Protocol 

This is to make sure time stays in sync 

```
$ sudo apt-get install -y ntp
$ sytemctl restart ntp
```

### Configure Virtual Memory setting

```
$ sudo vi /etc/sysctl.conf 
```
Add this line in then reboot the VM

```
# Increase memory map areas to 262144
sysctl -w vm.max_map_count=262144

then

$ sudo reboot now
```

### Install Docker and tools

```
$ sudo apt-get update && sudo apt-get install -y linux-image-extra-$(uname -r) linux-image-extra-virtual
$ sudo apt-get install -y apt-transport-https ca-certificates curl software-properties-common
$ sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg | apt-key add -
$ sudo apt-key fingerprint 0EBFCD88
$ sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb\_release -cs) stable"
$ sudo apt-get update
$ sudo apt-get install -y docker-ce
$ sudo apt-get install -y python-setuptools && sudo easy_install pip
```

### Create the worker nodes by cloning the first VM

At this point, we've had the foundation for installing IBM Cloud Private. Lets shutdown the VM (`shutdown -h now`) and clone it to make the 2 worker nodes

From host machine

```
$ vboxmanage clonevm boot-master-proxy --name worker1
$ vboxmanage registervm ~/VirtualBox\ VMs/worker1/worker1.vbox
$ vboxmanage clonevm boot-master-proxy --name worker2
$ vboxmanage registervm ~/VirtualBox\ VMs/worker2/worker2.vbox
```
### Update network configuration on each worker node

Now we can start all VMs using VirtualBox GUI or command line (better user GUI). VirtualBox will give us a command line interface for interacting with the VMs. Provide credentials to login to the VM and do further configuration. For example, with worker1

<image src="images/worker1login.png"/>  

Change the host name of the VM to `worker1`

```
$ sudo vi /etc/hostname # change it to worker1
```

Change /etc/hosts configuration, to add these lines in

```
$ sudo vi /etc/hosts 

# Add these lines in:
172.0.1.100 boot-master-proxy
172.0.1.101 worker1
172.0.1.102 worker2
```

Assign a static IP address to the VM by changing the configuration in `/etc/network/interfaces`

```
$ sudo vi /etc/network/interfaces
```

Make it looks like this

```
# This file describes the network interfaces available on your system
# and how to activate them. For more information, see interfaces(5).

source /etc/network/interfaces.d/*

# The loopback network interface
auto lo
iface lo inet loopback

# The primary network interface (NAT)
auto enp0s3
iface enp0s3 inet dhcp
# The Host-Only network
auto enp0s8
iface enp0s8 inet static
        address 172.0.1.101 # This is worker1's IP address
        netmask 255.255.255.0

```

then reboot the VM

```
$ sudo reboot now
```

Repeat those configuration steps for `worker2` VM with `worker2` as hostname, `172.0.1.102` as the VM's static IP address

## Install IBM Cloud Private CE

Now we're ready to install ICP CE onto your VMs. First make sure you have all of 3 VMs started. Login to the `boot-master-proxy` VM through ssh from host machine

```
$ ssh <user>@172.0.1.100
$ sudo su - # provide credentials here
``` 

If you have SE-Linux, turn it off

```
$ setenforce 0
```

### Configure passwordless ssh tunnels from `boot-master-proxy` to worker nodes

Now configure passwordless SSH from `boot-master-proxy` node to the 2 worker nodes. Generate SSH key

```
$ ssh-keygen -t rsa -P '' # Accept default values by hitting enter
```

Now copy the resulting key (`id_rsa`) to all nodes in the cluster

```
$ ssh-copy-id -i .ssh/id_rsa root@boot-master-proxy
$ ssh-copy-id -i .ssh/id_rsa root@worker1
$ ssh-copy-id -i .ssh/id_rsa root@worker2
```

Now we can ssh from `boot-master-proxy` node to the worker nodes without having to provide password, For example, to access `worker` from `boot-master-proxy`:

```
$ ssh root@worker1
```
### Install IBM Cloud Private CE from `boot-master-proxy` node

Create installation directory and launch a docker container to pull the installation materials

```
$ mkdir -p /opt/icp
$ cd /opt/icp
$ docker pull ibmcom/icp-inception:2.1.0.3
$ docker run -e LICENSE=accept --rm -v /opt/icp:/data ibmcom/icp-inception:2.1.0.3 cp -r cluster /data
$ cd cluster
```

Now copy the ssh key to the installation directory

```
$ cp ~/.ssh/id_rsa /opt/icp/cluster/ssh_key
$ chmod 400 /opt/icp/cluster/ssh_key
```

Configure IP addresses of the nodes in `/opt/icp/cluster/hosts`

```
$ sudo vi /opt/icp/cluster/hosts
```

Make it looks like this

```
[master]
172.0.1.100

[worker]
172.0.1.101
172.0.1.102

[proxy]
172.0.1.100

```

Run the installation

```
$ docker run --net=host -t -e LICENSE=accept -v "$(pwd)":/installer/cluster ibmcom/icp-inception:2.1.0.3 install
```

Wait for about 30 minutes, we will have the IBM Cloud Private CE version deployed on the VMs. We can then access the cluster web console from the host machine via this link `https://172.0.1.100:8443`

