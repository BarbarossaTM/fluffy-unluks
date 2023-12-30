# UnLUKS - unlock yur LUKS encrypted drives remotely via SSH

This repository contains to scripts to set up the initian-ramdisk of any Debian-based system
so that it will configure a network interface and spawn an SSH server from within the initrd,
which will allow you to unlock any LUKS-encrypted drives/RAIDs/... remotely.

## Installation

The `add-unluks` script needs to installed into the `/etc/initramfs-tools/hooks/` directory,
so it will be run when a new initrd is build for the system to set up all the necessary plumbing.
This includes installing `sshd`, `vconfig`, `killall` into the initrd
 
The `unluks` script needs to be installed into the `/etc/initramfs-tools/scripts/local-top/`
directory, so it will be copied into the initrd into the same scripts structure, so it's executed
on bootup.

## Requierments

Currently the scripts require that the `psmisc` and `vlan` packages are installed on the host system.
If you do not require VLANs you can update the `add-luks` script and remove the

    copy_exec /sbin/vconfig

line from the script to circumvent the requirement for the `vlan` package.

## Configuration

The `unluks` script contains a configuration section on top, which contains of the follows parameters:

    # The following two parameters control whether this script considers disks to be
    # unlocked successfully.  At least one of the two variables needs to be set for
    # this to work :)
    
    # Space separated list of block devices to wait for (if any)
    wait_block_devs=""
    
    # Space separated list of LVM VGs to wait for (if any)
    wait_vgs=""
    

    # The following parameter control the network configuration applied within the
    # initrd, so you are able to access the machine to unlock its disks.
    
    # Physical NIC (e.g. eth0, enp1s0, ...) for a plain setup or name of the bonding
    # interface to use (e.g. bond0), if a LAG shall be assembled.  If a LAG iface is
    # to be used, bond_members also needs to be set.
    iface=""
    
    # Physical LAG member interface(s), if an 802.3ad/LACP LAG is desired (optional)
    bond_members=""
    
    # If the interface to this machine carries multiple VLANs, the desired VLAN for
    # administrative access can be configured by setting the VLAN ID here.
    # It will be configured on top of $iface configured above. (optional)
    vlan_id=""
    
    # The IP/netmask to be assigned to $iface (CIDR notation)
    ip=""
    
    # The default gateway to use, if any (optional)
    gateway=""
    
    # The safeword, which can be appened to the Kernel commmand line on boot-up to
    # disable this script and use the regular interactive means of unlocking the
    # disks. (optional)
    safeword="red"


# Even more Convenience

For more convenience uncomment, and most likely edit, the example script add the end `add-unluks`.
It will create a shell script stored within `/unluks` which contains the commands necessary to unlock
all existing LUKS volumes, `/dev/md1` - `/dev/md3` in this example, and run `pvscan` as well as `vgchange -ay`
to find all PVs and activate all LVs in all VGs.

    cat << SCRIPT > "${DESTDIR}/unluks"
    #!/bin/sh
    
    cryptsetup luksOpen /dev/md1 md1_crypt
    cryptsetup luksOpen /dev/md2 md2_crypt
    cryptsetup luksOpen /dev/md3 md3_crypt
    
    lvm pvscan
    
    vgchange -ay
    
    echo "Cool thanks, booting..."
    SCRIPT
    
    chmod 755 "${DESTDIR}/unluks"

For full convencience you can also change the root line of `/etc/passwd` within `add-unluks` to

    root:x:0:0:root:/root:/unluks

This will cause above script to be used as the login shell for the `root` user, meaning you will be
directly prompted to unlock the LUKS disk(s). The drawback however is that you don't get access to
the shell anymore, which you shouldn't be needing though if everything is as it should be.
