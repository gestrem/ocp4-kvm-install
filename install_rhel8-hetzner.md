Install RHEL8 on Hetzner Server.md
Today
12:03 PM

Nicolas Masse uploaded an item
Text
Install RHEL8 on Hetzner Server.md
# Install RHEL8 on Hetzner Server
-> I followed this documentation : [How to Create a RHEL 8 Image for Hetzner Root Servers | Keith Tenzer](https://keithtenzer.com/2019/10/24/how-to-create-a-rhel-8-image-for-hetzner-root-servers/)

* Go to [Download Red Hat Enterprise Linux](https://access.redhat.com/downloads/content/479/ver=/rhel---8/8.0/x86_64/product-software)
* Download **Red Hat Enterprise Linux 8.0 Binary DVD**
* Create a VM and boot this ISO image

![](Install%20RHEL8%20on%20Hetzner%20Server/Screen%20Shot%202020-03-23%20at%2011.23.42.png)
![](Install%20RHEL8%20on%20Hetzner%20Server/Screen%20Shot%202020-03-23%20at%2012.04.55.png)
![](Install%20RHEL8%20on%20Hetzner%20Server/Screen%20Shot%202020-03-23%20at%2012.05.46.png)

* Hetzner only supports one boot kernel. As such we will remove the rescue kernel.

```sh
rm /boot/*-rescue-*
```

* Hetzner creates a ram disk and uses the dracut tool. It expects dracut to be installed under /sbin. This is not the case with RHEL 8 so we will add a symlink.

```sh
ln -s /usr/bin/dracut /sbin/dracut
```

* Install mdadm

```sh
mkdir /mnt/dvd
mount /dev/sr0 /mnt/dvd -o ro
cp /mnt/dvd/media.repo /etc/yum.repos.d/rhel8dvd.repo
cat >> /etc/yum.repos.d/rhel8dvd.repo <<EOF
enabled=1
baseurl=file:///mnt/dvd/BaseOS/
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release
EOF
yum update
yum -y install mdadm
```

* Copy your SSH Key to the root account.

```sh
ssh-copy-id root@192.168.23.205
```

* Finally create a tar zip of the RHEL 8 OS. The image needs to be called CentOS-75-el-x86_64-minimal.tar.xz because Hetzner is expecting certain image names but I assure you this will be RHEL 8.

```sh
ssh root@192.168.23.205 tar cv --exclude=/dev --exclude=/proc --exclude=/sys --exclude=/mnt/dvd / |xz -cz > CentOS-75-el-x86_64-minimal.tar.xz
```

* Reboot your server in rescue mode
* Copy CentOS-75-el-x86_64-minimal.tar.xz to your server

```sh
scp CentOS-75-el-x86_64-minimal.tar.xz root@openshift.itix.fr:/root
```

* Log into your Hetzner server using your ssh key. You should be placed into rescue mode.
* Create a Hetzner configuration and use your custom image

```sh
cat > config.txt <<EOF
DRIVE1 /dev/sda 
DRIVE2 /dev/sdb 
SWRAID 1 
SWRAIDLEVEL 0 
BOOTLOADER grub 
HOSTNAME openshift.itix.fr 
PART /boot ext3 512M
PART lvm vg0 all 

LV vg0 root / xfs 10G 
LV vg0 swap swap swap 8G 
LV vg0 var /var xfs 10G
LV vg0 home /home xfs 10G 
LV vg0 tmp /tmp xfs 5G
LV vg0 libvirt /var/lib/libvirt/images xfs 200G

IMAGE /root/CentOS-75-el-x86_64-minimal.tar.xz
EOF
```

* Launch the RHEL8 install

```
# installimage -a -c config.txt

                Hetzner Online GmbH - installimage

  Your server will be installed now, this will take some minutes
             You can abort at any time with CTRL+C ...

         :  Reading configuration                           done 
         :  Loading image file variables                    done 
         :  Loading centos specific functions               done 
   1/17  :  Deleting partitions                             done 
   2/17  :  Test partition size                             done 
   3/17  :  Creating partitions and /etc/fstab              done 
   4/17  :  Creating software RAID level 0                  done 
   5/17  :  Creating LVM volumes                            done 
   6/17  :  Formatting partitions
         :    formatting /dev/md/0 with ext3                done 
         :    formatting /dev/vg0/root with xfs             done 
         :    formatting /dev/vg0/swap with swap            done 
         :    formatting /dev/vg0/var with xfs              done 
         :    formatting /dev/vg0/home with xfs             done 
         :    formatting /dev/vg0/tmp with xfs              done 
         :    formatting /dev/vg0/libvirt with xfs          done 
   7/17  :  Mounting partitions                             done 
   8/17  :  Sync time via ntp                               done 
         :  Importing public key for image validation       done 
   9/17  :  Validating image before starting extraction     warn 
         :  No detached signature file found!
  10/17  :  Extracting image (local)                        done 
  11/17  :  Setting up network config                       done 
  12/17  :  Executing additional commands
         :    Setting hostname                              done 
         :    Generating new SSH keys                       done 
         :    Generating mdadm config                       done 
         :    Generating ramdisk                            done 
         :    Generating ntp config                         done 
  13/17  :  Setting up miscellaneous files                  done 
  14/17  :  Configuring authentication
         :    Fetching SSH keys                             done 
         :    Disabling root password                       done 
         :    Disabling SSH root login without password     done 
         :    Copying SSH keys                              done 
  15/17  :  Installing bootloader grub                      done 
  16/17  :  Running some centos specific functions          done 
  17/17  :  Clearing log files                              done 

                  INSTALLATION COMPLETE
   You can now reboot and log in to your new system with the
 same credentials that you used to log into the rescue system.

```

* Reboot
* Login to your server
* Do some cleanup

```sh
rm /etc/yum.repos.d/rhel8dvd.repo
yum clean all
```

* Register the host and attach subscriptions.

```sh
subscription-manager register --name=openshift.itix.fr
subscription-manager list --available --matches="*employee*"
subscription-manager attach --pool=8a85f99c6c8b9588016c8be0f1b50ec1
```

* Configure sshd

```
# egrep '^(PermitRootLogin|PasswordAuthentication|GSSAPIAuthentication|UseDNS)' /etc/ssh/sshd_config
PermitRootLogin without-password
PasswordAuthentication no
GSSAPIAuthentication no
UseDNS no
```

* Restart sshd

```
systemctl restart sshd
```

#tips 
