# linuxacademy-ex280-notes

There are 2 ex280 classes on LinuxAcademy, an OLD one and a NEW one.  

The NEW one has "ex280" in lower case.

### Be sure to take the NEW training class, NOT the old one.

## CONFIGURATION

The Openshift cluster for this training uses the following configuration

Host machine
* Macbook Pro
* 4-core
* 16gb RAM
* 256gb SSD
* VirtualBox


Openshift VMs:
* Master Node
  * master.example.com
  * 192.168.10.10
* Infrastructure Worker Node
  * infra.example.com
  * 192.168.10.11
* Compute Worker Node
  * compute.example.com
  * 192.168.10.12


All VMs are the following:
* Centos 7.6.1810 64bit Minimal Install
* 2 virtual CPUs
* 4gb RAM
* Networks (both networks on each VM)
  * NIC1: Host-Only Network 192.168.10.0/24 (NO Gateway)
  * NIC2: NAT network
* Disk1
  * 13gb
  * /dev/sda
    * 10gb / (root)
    * 3gb swap
* Disk2
  * 20gb
  * /dev/sdb
  * UN-allocated
  * will be provisioned later as docker-vg thin pool


Web Access
* OKD Master Console
  * https://master.example.com:8443



## SETUP

### PERFORM THE FOLLOWING ON YOUR OSX WORKSTATION

1. Update `/etc/hosts` to add the hosts

       sudo vim /etc/hosts

   * Add the following /etc/hosts entries:

         192.168.10.10    master.example.com  master
         192.168.10.11    infra.example.com   infra
         192.168.10.12    compute.example.com compute

1. Clear the local DNS cache

       sudo dscacheutil -flushcache

### PERFORM THE FOLLOWING STEPS ON *ALL* VMs:

1. Install CentOS

1. Become the root user

1. Update the kernel and reboot

       yum -y update && reboot

1. Change /etc/sudoers to use NOPASSWD for wheel group

       visudo

   * Change the file to look like this:
       
         ## Allows people in group wheel to run all commands
         # %wheel        ALL=(ALL)       ALL
  
         ## Same thing without a password
         %wheel  ALL=(ALL)       NOPASSWD: ALL

1. Create your local non-root user

       useradd -G wheel <userid>
       passwd <userid>

1. Set up all ancillary packages

       yum -y group install core base
       yum -y install centos-release-openshift-origin311 git iptables-services epel-release pyOpenSSL
       yum -y install dkms kernel-devel
       # (using the VirtualBox menu: Devices, Insert Guest Additions CD Image)
       mount /dev/cdrom /mnt
       /mnt/VBoxLinuxAdditions.run
       umount /mnt && eject cdrom
       yum-config-manager --disable epel
       yum -y install ansible openshift-ansible

1. Disable firewalld

       systemctl disable --now firewalld.service

1. Set up DNSMASQ as the DNS

       cp /etc/resolv.conf /etc/resolv.conf.orig
       cp /etc/resolv.conf /etc/resolv.dnsmasq
       echo -e 'search example.com\nnameserver 127.0.0.1' > /etc/resolv.conf
       cat <<EOF > /etc/dnsmasq.d/dnsmasq_lab.conf
       resolv-file=/etc/resolv.dnsmasq
       address=/master.example.com/192.168.10.10
       address=/infra.example.com/192.168.10.11
       address=/compute.example.com/192.168.10.12
       address=/apps.okd.example.com/192.168.10.11
       EOF
       cat <<EOF2 >> /etc/hosts
       192.168.10.10	master.example.com	master
       192.168.10.11	infra.example.com	infra
       192.168.10.12	compute.example.com	compute
       EOF2
       systemctl enable --now dnsmasq.service

1. Stop NetworkManager from changing resolv.conf

       sed -i '/^\[main\]/a dns=none' /etc/NetworkManager/NetworkManager.conf
       systemctl restart NetworkManager.service

1. Test DNSMASQ
 
       host `hostname`
       host www.google.com

1. Install Docker

       yum -y install docker-1.13.1

1. Verify `/dev/sdb` exists and has no partitions

       lsblk

1. Configure docker storage you prefer

 * Using overlay2

       cat <<EOF > /etc/sysconfig/docker-storage-setup
       DEVS='/dev/sdb'
       VG=docker-vg
       DATA_SIZE=95%VG
       STORAGE_DRIVER=overlay2
       CONTAINER_ROOT_LV_NAME=docker-lv
       CONTAINER_ROOT_LV_MOUNT_PATH=/var/lib/docker
       CONTAINER_ROOT_LV_SIZE=100%FREE
       EOF
       docker-storage-setup

 * Using thinpool

       cat <<EOF > /etc/sysconfig/docker-storage-setup
       DEVS='/dev/sdb'
       DATA_SIZE=99%VG
       VG=docker-vg
       CONTAINER_THINPOOL=docker-lv
       EOF
       docker-storage-setup

1. Verify Docker storage is set

       vgs
       lvs

1. Enable and start docker

       systemctl enable --now docker


## INSTALL OPENSHIFT

### Perform the following steps only on the *MASTER* node

### Perform the following steps as the NON-ROOT user

1. Login to the MASTER node as your local NON-ROOT USER

1. Set up shared SSH keys 

       ssh infra.example.com "mkdir ~/.ssh;chmod 700 ~/.ssh"
       ssh compute.example.com "mkdir ~/.ssh;chmod 700 ~/.ssh"
       ssh-keygen -t rsa -b 4096 -f ~/.ssh/id_rsa -N ''
       cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
       echo 'StrictHostKeyChecking=no' > ~/.ssh/config
       chmod 600 ~/.ssh/config
       scp ~/.ssh/* infra.example.com:~/.ssh
       scp ~/.ssh/* compute.example.com:~/.ssh

1. Copy the `inventory` file from this repo to the non-root user's home directory

1. Edit the `inventory` file and replace `___YOUR_SSO___` with your non-root userid

1. Pre-Check the Cluster

       ansible-playbook -i ~/inventory /usr/share/ansible/openshift-ansible/playbooks/prerequisites.yml

1. Create the Cluster

       ansible-playbook -i ~/inventory /usr/share/ansible/openshift-ansible/playbooks/deploy_cluster.yml

1. Test the Cluster from the Master Node

       oc get nodes
       oc get pods
       oc status
       oc describe all

1. Add the following to `/etc/sysconfig/iptables` so you can run NFS from the master for PVs

       -A OS_FIREWALL_ALLOW -p tcp -m state --state NEW -m tcp --dport 111 -j ACCEPT
       -A OS_FIREWALL_ALLOW -p udp -m state --state NEW -m udp --dport 111 -j ACCEPT
       -A OS_FIREWALL_ALLOW -p tcp -m state --state NEW -m tcp --dport 2049 -j ACCEPT
       -A OS_FIREWALL_ALLOW -p udp -m state --state NEW -m udp --dport 2049 -j ACCEPT

1. Install NFS

       yum -y install nfs-utils
       mkdir -p /home/data/persistent{1,2,3,4,5,etcd}
       chown -R nfsnobody:nfsnobody /home/data
       chmod 700 /home/data/persistent*
       cat <<EOF >/etc/exports.d/dbvol.exports
       /home/data/persistent01 *(rw,async,all_squash)
       /home/data/persistent02 *(rw,async,all_squash)
       /home/data/persistent03 *(rw,async,all_squash)
       /home/data/persistent04 *(rw,async,all_squash)
       /home/data/persistent05 *(rw,async,all_squash)
       EOF
       setsebool -P virt_use_nfs 1
       systemctl enable nfs


1. Reboot to update iptables and start NFS

       reboot

1. Validate that everything is working

       exportfs -a
       showmount -e


