# Lab setup

## How to use this repository ?
I recommend to fork it and add an additional inventory file with your own settings. This way you can easily merge upcomming changes without any collisions. 

## How to install the k8s cluster ?
Bootstrap your hardware and ensure that your inventory.yml is correct.

Run **backbone.yml first**, then k8s.yml:

```
ansible-playbook -i inventory.yml backbone.yml
ansible-playbook -i inventory.yml k8s.yml
```

The order matters: **k8scp** resolves to the keepalived VIP, so keepalived has to be up
before any node tries to reach the cluster endpoint. k8s.yml checks the VIP up front and
stops with a clear message instead of failing later inside kubeadm.

The playbook will run 5 roles to get the cluster up and running.
* k8s_prepare_server will install necessary packages and does the configuration.
* [k8s_control_plane_init](./roles/k8s_control_plane_init/README.md) will initialize the k8s cluster on the specified server.
* [k8s_control_plane_join](./roles/k8s_control_plane_join/README.md) will join the other control-plane nodes to the cluster to achive a stable state.
* k8s_data_plane_join will join the netsvc nodes as data-plane nodes to the cluster.

## How to install all my services ?
Bootstrap your hardware, install the k8s cluster and ensure it's running correctly.

Run: `ansible-playbook -i inventory.yml k8s_homelab.yml`
* k8s_argocd will install argocd on the cluster and add an **app of all apps** application to install all services you configured in your app git-repository.
* k8s cluster is up running and all your services are available.

## Planned changes and open issues

- [x] **k8s_hostname** used to resolve to the init node's IP via /etc/hosts, so the entire cluster endpoint depended on one host: if the init instance was down, no node could join and every node kept talking to the init node instead of any control-node. It now resolves to the keepalived VIP, which is what the VIP was there for all along. Two consequences: `backbone.yml` has to run before `k8s.yml`, and keepalived makes the **init node** the MASTER so the VIP sits on the right host during `kubeadm init`. Adding all control-nodes to /etc/hosts (the earlier experiment) works on debian and armbian but is not covered by the specs, and a config-as-code DNS server would be wasted effort if you move to a different DNS later.
- [ ] At the moment the control-plane nodes are configured to run any k8s workload to get cilium up and running. Without CNI the cluster isn't working. The correct solution is to set affinity rules to ensure that control-plane is only running jobs which where intended to run on them.
- [ ] Turning swapping off isn't reliable at the moment I noticed that sometimes the swapping isn't turned of during the bootup. Swapping will be enabled after each reboot and the provided server(noswap) should disable it. kubectl strongly recommends to disable swapping and my k8s trainer did the same because it does some trouble in the daily buisness. You don't want to run a service with undefined behaviour.
- [ ] Ensure that the repository can be reused by other and work properly with forks.

## Network
The cluster now lives in **192.168.1.0/24**, the regular home network behind the Fritzbox.
It was moved there from the former test subnet 192.168.3.0/24, whose router
(192.168.3.100) no longer exists. Anything still carrying a 192.168.3.x address in this
repository is pending migration, not in use.

I recommend to avoid the 10.0.0.0/8 subnet because kubernetes is using it by default and it take quite some effort to change all the spots in the config files to switch to an other subnet.

Addresses below .100 are static or reserved; the Fritzbox hands out DHCP leases from
.100 upwards. Reservations are keyed on the MAC of the **onboard** NIC — if a node was
ever wired through a USB adapter, the reservation may still point at that adapter's MAC
and the node will silently get a DHCP address from the upper range instead.

The former test setup used a separate wifi access point whose WAN port was connected to the home-network. That kept DNS, DHCP and PXE experiments away from the main network, at the price of a second router to maintain.

## Systems and their roles
Name|IP|Role
---|---|---
Router (Fritzbox)|192.168.1.1/24|Internet Gateway
backbone|192.168.1.10/24|Virtual IP shared by all backbone server. **k8scp** resolves to this address.
backbone-2|192.168.1.11/24|Control-plane and k8s init node. Its certificates are issued for this exact address, so do not renumber it casually.
backbone-1|192.168.1.12/24|All core service which always must be available and the control-plane of the k8s cluster.
backbone-3|192.168.1.13/24|
(pool)|192.168.1.20-29|Cilium LoadBalancerIPPool. Must stay clear of all node addresses.
ups-1|192.168.3.8/24|Turning on/off power of connected devices. **Not migrated yet.**
ups-2|192.168.3.9/24|**Not migrated yet.**
netsvc-1|192.168.3.10/24|Data-plane node of the k8s cluster. It runs all services which are not critical like monitoring or a fileserver. **Not migrated yet.**
netsvc-2|192.168.3.15/24|**Not migrated yet.**
netsvc-3|192.168.3.20/24|
netsvc-4|192.168.3.25/24|

Note that backbone-1 and backbone-2 are **not** in numerical order. backbone-2 was the
node the current cluster was initialized on, and its control plane, etcd member and
certificates all reference 192.168.1.11. Swapping the two would break the cluster.

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
Backbone|Dell Wyse 5070|8GB|64GB SSD|1Gbit
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

Two things it handles that cost me a lot of time to find, because neither announces itself:

* **Time synchronisation.** Without a running timesync service the clock drifts, and on
  hardware without a reliable RTC it drifts fast. Freshly issued certificates then carry a
  `notBefore` in the future and every other node rejects them with
  `tls alert certificate expired` — a message that never mentions time. A `kubeadm join`
  fails at the etcd check and gives no hint at all. The playbook installs and enables
  `systemd-timesyncd`.
* **Armbian's `10-dhcp-all-interfaces.yaml`.** It sorts after `01-netcfg.yaml`, so netplan
  lets DHCP win and the static address sits in the file without ever taking effect. The
  playbook moves it aside.

The netplan template configures the **named** interface from `nic`, not a `en*` wildcard.
The wildcard matched a USB ethernet adapter as well, and both NICs ended up with the same
address.
It removes the user created by the first login, create an ansible user instead switch from password login to ssh key login.
The idea is to get different host-systems to the same state to work on with all upcomming playbooks.
There should be different versions for the stated SBC, an on-demand or an dedicated machine.

## SSD on Radax Zero 3e
Radax Zero 3e have some issue with usb-c connectors. The pinout wasn't implemented the proper way and only one direction provides full 5GBit/s speed, the opposite direction only provided 450MBit/s. Check with lsusb -t to see if the device is connected correctly and switch it accordingly. I labeled my cables to never put it in the wrong direction again.