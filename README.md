# Lab setup

## How to use this repository ?
I recommend to fork it and add an additional inventory file with your own settings. This way you can easily merge upcomming changes without any collisions. 

## How to install the k8s cluster ?
Bootstrap your hardware and ensure that your inventory.yml is correct.

Run: `ansible-playbook -i inventory.yml k8s.yml`

The playbook will run 5 roles to get the cluster up and running.
* k8s_prepare_server will install necessary packages and does the configuration.
* k8s_control_plane_init will initialize the k8s cluster on the specified server.
* k8s_control_plane_join will join the other control-plane nodes to the cluster to achive a stable state.
* k8s_data_plane_join will join the netsvc nodes as data-plane nodes to the cluster.

## How to install all my services ?
Bootstrap your hardware, install the k8s cluster and ensure it's running correctly.

Run: `ansible-playbook -i inventory.yml k8s_homelab.yml`
* k8s_argocd will install argocd on the cluster and add an **app of all apps** application to install all services you configured in your app git-repository.
* k8s cluster is up running and all your services are available.

## Planned changes and open issues

- [ ] To initialize the k8s cluster the **k8s_hostname** variable is used which need to resolve to init node ip. The current code use /etc/hosts and set the instance ip but in the case that the init instance is down all nodes can't join and will always talk to the init node instead to any control-node. I did some tests with adding all control-nodes to /etc/hosts which works on debian and armbian but isn't a feature mentioned in the specs. A config-as-code DNS server would solve the problem but is a dead waste if you use a different DNS later on. Dyndns could be an option or add additional complexity to the join process.
- [ ] At the moment the control-plane nodes are configured to run any k8s workload to get cilium up and running. Without CNI the cluster isn't working. The correct solution is to set affinity rules to ensure that control-plane is only running jobs which where intended to run on them.
- [ ] Turning swapping off isn't reliable at the moment I noticed that sometimes the swapping isn't turned of during the bootup. Swapping will be enabled after each reboot and the provided server(noswap) should disable it. kubectl strongly recommends to disable swapping and my k8s trainer did the same because it does some trouble in the daily buisness. You don't want to run a service with undefined behaviour.
- [ ] Ensure that the repository can be reused by other and work properly with forks.

## Network
192.168.3.0/24 is a subnet for test purpose in my 192.168.0.0/16 subnet. I recommend to avoid the 10.0.0.0/8 subnet because kubernetes is using it by default and it take quite some effort to change all the spots in the config files to switch to an other subnet.

To start a test environment I use a wifi access point which WAN port is connected to my home-network. This way it's an own network with internet connection. I can connect to the wifi and be on my local network per RJ45 at the same time. This simplifies the setup of DNS, DHCP, PXE and other network services which can only blast your test network.

## Systems and their roles
Name|IP|Role
---|---|---
Router|192.168.3.100/24|Internet Gateway
backbone-1|192.168.3.2/24|All core service which always must be available and the control-plane of the k8s cluster.
backbone-2|192.168.3.3/24|
backbone-3|192.168.3.4/24|
ups-1|192.168.3.8/24|Turning on/off power of connected devices.
ups-2|192.168.3.9/24|
netsvc-1|192.168.3.10/24|Data-plane node of the k8s cluster. It runs all services which are not critical like monitoring or a fileserver.
netsvc-2|192.168.3.15/24|
netsvc-3|192.168.3.20/24|
netsvc-4|192.168.3.25/24|

I split backbone and netsvc according to the following rule. 

If your power is off what do you need to still survive and maintain your environment ? 
* Without DHCP/DNS the network devices can't talk to each other. -> backbone
* If the Network storage goes down I can't access some files. -> netsvc
* Without Home Assistant my home-automation isn't working anymore. Stop it! Get some help. -> backbone for now
* I want to check my monitoring. This is tricky. It's commonly a stack you could split.
  Prometheus -> backbone/netsvc and Grafana -> netsvc

Put an USP in front of the hardware and shut down everything except the backbone in an outage or if you have no demand of the hardware for now. Kubernetes auto-scaler can help to turn-off/on your netsvc instances on-demand and save power. 

The backbone are small arm64 SBC because they take far less power and run the entire year. 

Netsvc instances should be power efficient and do most of the work. 

The very power hungry equipment should provide services which are turned off most of the day or week. E.g. a refurbished HP blade server with a big NAS you can turn on every weekend and backup your much smaller storage in netsvc.

## Hardware Inventory
Type|Motherboard|RAM|Storage|NIC
-|-|-|-|-
Backbone|Radxa Zero 3e|2GB|16GB sd-card, 128GB Intenso SSD|1GBit
Netsvc|Gigabyte MJ11-EC1|2x32GB ECC|1TB nvme|Dual 10Gbit SFP+

I use a 8 port SFP+ unmanaged switch to wire up the 3 unmanaged 8port RJ45 2,5GBit/s and two 10Gbit SFP+ ports.
A Fritzbox is sitting in between the network and ISP doing most of the work.
The whole switch topic are highly depends on the specific situations. This setup works best for my one but be the worst for your's.

## How to bootstrap a netsvc server ?
* Download a fitting image for the x64 based server like [Debian](https://www.debian.org/).
* Write the image to USB drive.
* Install from USB drive.
* Set the root password, ip and username accordingly to the inventory.yml .
* Put you pub key at ./pubkey or alter the location in the inventory.yml .
* Run the bootstrap playbook with following command. `ansible-playbook -i inventory.yml bootstrap.yml`

## How to bootstrap a backbone server ?
* Download a fitting image for the radxa-zero3 [here](https://www.armbian.com/radxa-zero-3/).
* Write the image to an sd-card to boot from it.
* Optional: Write the image to an ssd and delete the root partition from the sd-card.
    * The SBC boots from the boot partition of the sd-card and the bootloader will look for a partition with a hard-coded partition id. This partition id can be changed in the bootloader source code. But deleting the partition on the sd-card is a quick and easy way to load the root partition of the ssd.
    * The sd-card is slow and if you need to reset it more often then you can load the image into gparted, delete the 2nd partition, and truncate the image to fit the boot partition and the backup partition table after it. This way you need to write just 300mb to the sd-card and avoid the deletion of the root partition.
* Log into root, set a password, create the user.
* Set the root password, ip and user in the inventory.yml .
* Put you pub key at ./pubkey or alter the location in the inventory.yml .
* Run the bootstrap playbook with following command. `ansible-playbook -i inventory.yml bootstrap.yml`

The bootstrap playbook will take care that all necessary packages are installed and that the network configuration is correct.
It removes the user created by the first login, create an ansible user instead switch from password login to ssh key login.
The idea is to get different host-systems to the same state to work on with all upcomming playbooks.
There should be different versions for the stated SBC, an on-demand or an dedicated machine.

## SSD on Radax Zero 3e
Radax Zero 3e have some issue with usb-c connectors. The pinout wasn't implemented the proper way and only one direction provides full 5GBit/s speed, the opposite direction only provided 450MBit/s. Check with lsusb -t to see if the device is connected correctly and switch it accordingly. I labeled my cables to never put it in the wrong direction again.