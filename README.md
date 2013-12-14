centos-ami
==========

Build custom Amazon EC2 Machine Image (CentOS 6.5)

##What?

Create a minimal AMI for CentOS 6.5 x86_64 from scratch

##Requirements?
@see [Here](http://www.idevelopment.info/data/AWS/AWS_Tips/AWS_Management/AWS_10.shtml#CentOS%20Build%20Machine)

##How?

1. Download & install [VMWare Player](https://my.vmware.com/web/vmware/free#desktop_end_user_computing/vmware_player/6_0)

2. New a Virtual Machine for CentOS x86_64

3. Download & install Centos to the VM [CentOS-6.5-x86_64-minimal.iso](http://isoredirect.centos.org/centos/6/isos/x86_64/)

4. (use winscp) upload 2 X.509 Certificates (.pem) file to /tmp/cert (on the VM)

5. Use putty (I use MS Windows) to connect to VM

6. update system
```
yum update
reboot
rpm -qa
```
//-> write down the rpms.

7. install some packages
```
curl -O http://s3.amazonaws.com/ec2-downloads/ec2-ami-tools.noarch.rpm
curl -L -b gpw_e24=a -O http://download.oracle.com/otn-pub/java/jdk/7u45-b18/jre-7u45-linux-x64.rpm
yum install nano unzip ruby jre-7u45-linux-x64.rpm ec2-ami-tools.noarch.rpm MAKEDEV
```

8. edit .bashrc
```
nano ~/.bashrc
```
(content:)
```
export AWS_ACCESS_KEY=???
export AWS_SECRET_KEY=???
export JAVA_HOME=/usr/java/default
export EC2_HOME=/root/ec2-api-tools-1.6.12.0
export PATH=$PATH:$EC2_HOME/bin
export EC2_URL=https://ec2.ap-southeast-1.amazonaws.com
```
(test:)
```
ec2-describe-regions
ec2-describe-availability-zones
```

9. prepare image file
```
mkdir -p /opt/ec2/images
dd if=/dev/zero of=/opt/ec2/images/centos-6.5-x86_64-base.img bs=1M count=10240
mkfs.ext4 -F -j /opt/ec2/images/centos-6.5-x86_64-base.img
```

10. mount the image file to a directory
```
mkdir -p /mnt/ec2-image
mount -o loop /opt/ec2/images/centos-6.5-x86_64-base.img /mnt/ec2-image/
mount
```

11. makedev
```
mkdir -p /mnt/ec2-image/{dev,etc,proc,sys}
mkdir -p /mnt/ec2-image/var/{cache,log,lock,lib/rpm}
/sbin/MAKEDEV -d /mnt/ec2-image/dev -x console
/sbin/MAKEDEV -d /mnt/ec2-image/dev -x null
/sbin/MAKEDEV -d /mnt/ec2-image/dev -x zero
/sbin/MAKEDEV -d /mnt/ec2-image/dev -x urandom
mount -o bind /dev /mnt/ec2-image/dev
mount -o bind /dev/pts /mnt/ec2-image/dev/pts
mount -o bind /dev/shm /mnt/ec2-image/dev/shm
mount -o bind /proc /mnt/ec2-image/proc
mount -o bind /sys /mnt/ec2-image/sys
mount
```

12. prepare yum conf file for installing CentOS minimal to the image
  ```
  mkdir -p /opt/ec2/yum
  nano /opt/ec2/yum/yum-xen.conf
  ```
  
  (content:)
  
  ```
  [base]
  name=CentOS-6.5 - Base
  mirrorlist=http://mirrorlist.centos.org/?release=6.5&arch=x86_64&repo=os
  #baseurl=http://mirror.centos.org/centos/6.5/os/x86_64/
  gpgcheck=1
  gpgkey=http://mirror.centos.org/centos/RPM-GPG-KEY-CentOS-6
  
  #released updates
  [updates]
  name=CentOS-6.5 - Updates
  mirrorlist=http://mirrorlist.centos.org/?release=6.5&arch=x86_64&repo=updates
  #baseurl=http://mirror.centos.org/centos/6.5/updates/x86_64/
  gpgcheck=1
  gpgkey=http://mirror.centos.org/centos/RPM-GPG-KEY-CentOS-6
  
  #additional packages that may be useful
  [extras]
  name=CentOS-6.5 - Extras
  mirrorlist=http://mirrorlist.centos.org/?release=6.5&arch=x86_64&repo=extras
  #baseurl=http://mirror.centos.org/centos/6.5/extras/x86_64/
  gpgcheck=1
  gpgkey=http://mirror.centos.org/centos/RPM-GPG-KEY-CentOS-6
  
  #additional packages that extend functionality of existing packages
  [centosplus]
  name=CentOS-6.5 - Plus
  mirrorlist=http://mirrorlist.centos.org/?release=6.5&arch=x86_64&repo=centosplus
  #baseurl=http://mirror.centos.org/centos/6.5/centosplus/x86_64/
  gpgcheck=1
  enabled=0
  gpgkey=http://mirror.centos.org/centos/RPM-GPG-KEY-CentOS-6
  
  #contrib - packages by Centos Users
  [contrib]
  name=CentOS-6.5 - Contrib
  mirrorlist=http://mirrorlist.centos.org/?release=6.5&arch=x86_64&repo=contrib
  #baseurl=http://mirror.centos.org/centos/6.5/contrib/x86_64/
  gpgcheck=1
  enabled=0
  gpgkey=http://mirror.centos.org/centos/RPM-GPG-KEY-CentOS-6
  ```

13. edit /etc/fstab
  ```
  nano /mnt/ec2-image/etc/fstab
  ```
  
  (content:)
  ```
  /dev/xvde1 /         ext4    defaults,noatime 1 1
  devpts     /dev/pts  devpts  gid=5,mode=620   0 0
  tmpfs      /dev/shm  tmpfs   defaults         0 0
  proc       /proc     proc    defaults         0 0
  sysfs      /sys      sysfs   defaults         0 0
  ```

14. install
```
yum -c /opt/ec2/yum/yum-xen.conf --installroot=/mnt/ec2-image install <packages from step 6>
yum -c /opt/ec2/yum/yum-xen.conf --installroot=/mnt/ec2-image -y install *openssh* dhclient grub e2fsprogs selinux-policy selinux-policy-targeted
```

15. Grub
  ```
  nano /mnt/ec2-image/boot/grub/grub.conf
  ```
  
  (content:)
  ```
  default=0
  timeout=0
  
  title CentOS 6.5 (Custom AMI)
  root (hd0)
  kernel /boot/vmlinuz-2.6.32-431.1.2.0.1.el6.x86_64 ro root=/dev/xvde1 rd_NO_PLYMOUTH
  initrd /boot/initramfs-2.6.32-431.1.2.0.1.el6.x86_64.img
  ```
  
  ```
  ln -s /boot/grub/grub.conf /mnt/ec2-image/boot/grub/menu.lst
  ```

16. SELinux
```
touch /mnt/ec2-image/.autorelabel
```

17. Network
  ```
  nano /mnt/ec2-image/etc/sysconfig/network
  ```
  
  (content:)
  ```
  NETWORKING=yes
  NETWORKING_IPV6=no
  HOSTNAME=localhost.localdomain
  ```
  
  ```
  nano /mnt/ec2-image/etc/sysconfig/network-scripts/ifcfg-eth0
  ```
  
  (content:)
  ```
  DEVICE=eth0
  BOOTPROTO=dhcp
  ONBOOT=yes
  TYPE=Ethernet
  USERCTL=yes
  PEERDNS=yes
  IPV6INIT=no
  PERSISTENT_DHCLIENT=yes
  ```

18. sshd
  ```
  nano /mnt/ec2-image/etc/ssh/sshd_config
  ```
  
  (changes:)
  ```
  ...
  PasswordAuthentication no
  UseDNS no
  PermitRootLogin without-password
  ```
  
  ```
  nano /mnt/ec2-image/etc/rc.local
  ```
  
  (content:)
  Note: 
  ```
  ...
  # set a random pass on first boot
  if [ -f /root/firstrun ]; then
    dd if=/dev/urandom count=50|md5sum|passwd --stdin root
    passwd -l root
    rm /root/firstrun
  fi
  
  if [ ! -d /root/.ssh ]; then
    mkdir -m 0700 -p /root/.ssh
    restorecon /root/.ssh
  fi
  
  # Get the root ssh key setup
  ReTry=0
  while [ ! -f /root/.ssh/authorized_keys ] && [ $ReTry -lt 10 ]; do
    sleep 2
    curl -f http://169.254.169.254/latest/meta-data/public-keys/0/openssh-key > /tmp/my-public-key
    if [ $? -eq 0 ]; then
      cat /tmp/my-public-key > /root/.ssh/authorized_keys
      rm -f /tmp/my-public-key
      ReTry=10
    fi
    ReTry=$[Retry+1]
  done
  chmod 600 /root/.ssh/authorized_keys && restorecon /root/.ssh/authorized_keys
  ```

19. startup services
  ```
  /usr/sbin/chroot /mnt/ec2-image /sbin/chkconfig --list
  ```
  
  disable: postfix iscsid iscsi mdmonitor lvm2-monitor iptables ip6tables blk-availability ON ALL runlevels

20. cleanup & umount
```
yum -c /opt/ec2/yum/yum-xen.conf --installroot=/mnt/ec2-image -y clean all
rm -rf /mnt/ec2-image/root/.bash_history
rm -rf /mnt/ec2-image/var/cache/yum
rm -rf /mnt/ec2-image/var/lib/yum
sync; sync; sync; sync
umount /mnt/ec2-image/dev/shm
umount /mnt/ec2-image/dev/pts
umount /mnt/ec2-image/dev
umount /mnt/ec2-image/sys
umount /mnt/ec2-image/proc
umount /mnt/ec2-image
mount
```

21. ec2-bundle-image
  ```
  ec2-describe-images --region ap-southeast-1 --owner amazon | grep "amazon\/pv-grub-hd0" | awk '{ print $1, $2, $3, $5, $7 }'
  ```
  
  ```
  ec2-bundle-image \
  --privatekey /tmp/cert/pk-??.pem \
  --cert /tmp/cert/cert-??.pem \
  --user ??? \
  --image /opt/ec2/images/centos-6.5-x86_64-base.img \
  --prefix centos-6.5-x86_64-base \
  --destination /opt/ec2/images \
  --arch x86_64 \
  --kernel aki-503e7402
  ```

22. ec2-upload-bundle
  ```
  ec2-upload-bundle \
  --bucke sandinh/ami/cent_65 \
  --manifest /opt/ec2/images/centos-6.5-x86_64-base.manifest.xml \
  --access-key ??? \
  --secret-key ??? \
  --retry
  ```
  
  note: if time error => need:
  ntpdate 0.rhel.pool.ntp.org 1.rhel.pool.ntp.org

23. Register new AMI

  Login to [EC2 console](https://console.aws.amazon.com/ec2/v2/home?region=ap-southeast-1#Images:)
  
  Register new AMI

24. Launch. Done.

## Convert to EBS-backed AMI
@see [HERE](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/creating-an-ami-instance-store.html#Using_ConvertingS3toEBS)

Notes:

1. Step 1 & 2 can be simplify by launching an Amazon Linux EBS micro instance, with 20G storage

2. device_name can be found (after attached) by
```
egrep '[xvsh]d[a-z].*$' /proc/partitions
mkfs.ext4 /dev/xvdj
```

3. In step 3: use mkfs.ext4 insteads of ext3

## Notes after launching EBS-backed instance
1. If you change capacity of root device when launching the AMI (insteads of using default), then you should:
```
egrep '[xvsh]d[a-z].*$' /proc/partitions
df -k
resize2fs /dev/xvda1
```

2. Should change hostname right after launching
