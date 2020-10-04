# VySOlator
VySOlator provides isolation of (virtual) networks based on VyOS. The goal of this project is to provide an initial setup for anyone who is looking to properly isolate (virtual) networks and protect their host machine. Use cases for this project: 
* Pentesting
* Malware research 

Currently only one network (internet only) is provided but more will be added as the project progresses.

# Process (steps)
* Getting the VyOS ISO
* Configure additional VMware networks
* Create and configure the VyOS virtual machine
* Troubleshooting
* Optional: test your setup
    1. Use your favourite network attacks and ensure it works as intended
    1. Create a pull request or issue if you found a way to access something else than the internet

## Getting the VyOS ISO
Pre-build ISOs are available through the VyOS webiste (https://downloads.vyos.io/?dir=rolling/current/amd64/). However its also possible to "roll your own" if you so desire. Doing so will allow you to include certain packages (i.e. vim) to VyOS. The following steps explain the process of building an ISO using docker:  

1. Clone the VySOlator repo and run the install-docker.sh script to install docker 
    * cd /vysolator
    * chmod +x install-docker.sh
    * sudo ./install-docker.sh
1. Clone the VyOs repo, build the docker image and create the iso
    * git clone https://github.com/vyos/vyos-build
    * cd vyos-build
    * sudo docker build -t vyos-builder docker
    * sudo docker run --rm -it --privileged -v $(pwd):/vyos -w /vyos vyos-builder bash
        * ./configure --custom-package vim --build-by DiabloHorn
        * sudo make iso

The result of the above commands will be the iso created in the ```build``` directory within the ```vyos-build``` directory. This is the iso we can use to install VyOS.

## Configure additional VMware networks
We will add to new networks two VMWare (vmnet2,vmnet3) that have zero interaction with the host. Below are the configration steps per VMWare version:

* [VMWare fusion and fusion pro](vmware-fusion_howto-add-interfaces.md)
* [VMware workstation](vmware-workstation_howto-add-interfaces.md)
* VirtualBox (todo)

## Create and configure the VyOS virtual machine

### Creating the virtual machine
Add a new virual machine and select install from disc or image as the installation method. In the next screen select the vyos image we build in the previous step. As the operating system we choose Linux/Debian 7.x 64-bit (newer Debian version might also work). Recommended disk space is 5 GB and memory 512 mb. Give it two network adapters, one connected to vmnet2 and the other to vmnet3. After the vm has booted login with: 

Username: ```vyos```  
Password: ```vyos```

On the commandline interface just run ```install image``` and make some common sense choices. After you are one, reboot the system. You will be able to login into your virtual gateway using your chosen password. The username will remain ```vyos```. You can now shutdown (```poweroff```) your virtual machine to be able to configure some additional network interfaces.

### Configure the virtual machine
#### Enable ssh for remote management
The following commands will enable SSH and DHCP on the 'mgmt' interface:

> ```configure```  
> ```set interfaces ethernet eth1 description 'mgmt interface'```   
> ```set interfaces ethernet eth1 address '10.7.7.1/24'```  
> ```set service dhcp-server shared-network-name mgmt subnet 10.7.7.0/24``` 
> ```set service dhcp-server shared-network-name mgmt subnet 10.7.7.0/24 range 0 start '10.7.7.2'```    
> ```set service dhcp-server shared-network-name mgmt subnet 10.7.7.0/24 range 0 stop '10.7.7.10'```    
> ```set service ssh listen-address '10.7.7.1'```   
> ```set service ssh port '22'```   
> ```commit```   
> ```save```      
> ```exit```    

To be able to connect to VyOS via SSH another VM has to be placed in the same network/segment.

#### Running setup script
We perform the following from our mgmt VM:  
> ```scp setup-config.sh vyos@10.7.7.1:~/```

> ```ssh vyos@10.7.7.1 "chmod +x setup-config.sh ; sg vyattacfg -c ./setup-config.sh"```  

That's it. If we now place a VM in the same segment / interfaces as vmnet3/eth2 it will only be able to connect to the internet.

## Troubleshooting
* *Running "sudo docker build -t vyos-builder docker" does not work (container is unable to install packages)*
When running docker on Ubuntu 16 you might have no internet inside your containers, comment out dnsmasq in /etc/NetworkManager/NetworkManager.conf and restart the Network Manager service. 
* *I messed up the configuration*
    > ```configure```  
    > ```load /opt/vyatta/etc/config.boot.default```  
    > ```commit```  
    > ```# If you are doing this remote, connection will drop after this```  
    > ```save```  
    > ```exit```  
    > ```reboot```
* *Can I use different IP ranges than your example?*  
Yes, of course. Be mindful of typo's
 
# References
* https://docs.docker.com/install/linux/docker-ce/ubuntu/
* https://github.com/vyos/vyos-build
* https://vyos.readthedocs.io/en/latest/install.html
* https://spin.atomicobject.com/2017/04/03/vmware-fusion-custom-virtual-networks/
* https://github.com/bertvv/cheat-sheets/blob/master/docs/VyOS.md
* https://superuser.com/questions/1130898/no-internet-connection-inside-docker-containers



