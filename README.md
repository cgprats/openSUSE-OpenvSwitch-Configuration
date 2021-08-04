# openSUSE OpenvSwitch configuration for KVM host
The following steps were used to configure libvirt utilizing OpenvSwitch on an openSUSE KVM host with the 'Server' role.
## Instructions:
### OpenvSwitch Configuration
1. Remove any bridge interfaces on a physical interface. If you utilized YAST's virtualization tab to install KVM, a bridge is created automatically. This is not wanted as you will be using OpenvSwitch to handle bridging. 
2. Install OpenvSwitch: `# zypper in openvswitch`. You will also want to enable and start it: `# systemctl enable --now openvswitch`.
3. Enable nanny in wicked configuration. In `/etc/wicked/common.xml`, add the following:
  ```xml
  <config>
    ...
    <use-nanny>true</use-nanny>
    ...
</config>
  ```
4. For the interface you want to add to the bridge, modify it's corresponding configuration file. (If you want to use 1 bridge for multiple interfaces, you should create a bond and then bridge the bond). The file `/etc/sysconfig/network/ifcfg-$interface_name` should look like:
  ```bash
  STARTMODE='auto'
  BOOTPROTO='none'
  ```
5. Think about what you want to name the OpenvSwitch bridge. Now create `/etc/sysconfig/network/ifcfg-$openvswitch_bridge_name` and add the following:
  ```bash
  # DHCP
  STARTMODE='auto'
  BOOTPROTO=dhcp
  OVS_BRIDGE='yes'
  OVS_BRIDGE_PORT_DEVICE='$interface_name'
  ```
  or
  ```bash
  # Static IP
  STARTMODE='auto'
  BOOTPROTO='static'
  IPADDR='$IP_address/$netmask'
  OVS_BRIDGE='yes'
  OVS_BRIDGE_PORT_DEVICE='$interface_name'
  ```
  and modify routes (only for configurations with static IPs) with:
  ```bash
  # mv -v /etc/sysconfig/network/ifcfg-$interface_name /etc/sysconfig/network/ifcfg-$openvswitch_bridge_name
  # sed -i 's/$interface_name/$openvswitch_bridge_name/g' /etc/sysconfig/network/ifcfg-$openvswitch_bridge_name
  ```
6. Now we actually create the bridge, add the interface, and bring it up with (You will lose network connectivity after the second command, so I reccommend using a one-liner or putting this into a script if using ssh):
  ```bash
  # ovs-vsctl add-br $openvswitch_bridge_name
  # ovs-vsctl add-port $openvswitch_bridge_name $interface_name
  ```
  Bring up interface with wicked or reboot
  ```bash
  # wicked ifup all
  - or -
  # reboot
  ```
7. Test the connection to the network by pinging whoever you like (and know is listening).
### Virsh Network Configuration
8. Virsh can use xml files to define networks, we will create one for the untagged network. Create the file `$untagged-network-name.xml` and add the following:
  ```xml
  <network>
    <name>$untagged-network-name</name>
    <forward mode="bridge" />
    <bridge name="$openvswitch_bridge_name" />
    <virtualport type='openvswitch' />
</network>
  ```
9. Define the name with `# virsh net-define --file $untagged-network-name.xml`. Start it with `# virsh net-start $untagged-network-name` and mark it to autostart on host boot with `# virsh net-autostart $untagged-network-name`.
10. VLAN network definitions will be a little different. Create the file `$vlan-network-name.xml` and add the following:
  ```xml
  <network>
    <name>$vlan-network-name</name>
    <forward mode="bridge" />
    <bridge name="$openvswitch_bridge_name" />
    <virtualport type='openvswitch' />
    <vlan>
      <tag id='$tag_number' />
    </vlan>
</network>
  ```
11. Define the name with `# virsh net-define --file $vlan-network-name.xml`. Start it with `# virsh net-start $vlan-network-name` and mark it to autostart on host boot with `# virsh net-autostart $vlan-network-name`.
12. Repeat steps 10 and 11 for every VLAN you wish to configure.
13. To verify the existence of all networks you have created you can use `# virsh net-list`.
14. Test that your configuration is correct by creating VMs on the configured VMs.
## Sources Used:
 - https://en.opensuse.org/Portal:Wicked/OpenvSwitch
 - https://manintheit.org/en/posts/kvm/creating-vlans-on-kvm-with-openvswitch/
