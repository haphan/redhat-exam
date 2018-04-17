## EX300 Red Hat Certified Engineer Exam Review

*Working in progress*

## Test taking strategy

- Spend 10 mins go read through all the questions before diving into any task.
- Understand task dependencies.
- Nail fown low hanging fruits first.
- For every task, must validate the work by:
	- Connecting to the service from `desktop` machine.
	- `su` to the user to test `acl` or permission settings.

## 0. The bible

```bash
man -k . | grep {keyword}
grep -r {keyword} /usr/share/docs
```


## 1. `systemctl` and boot process

List all targets

```bash
systemctl list-units --type=target --all
```

View default target and change default

```bash
systemctl get-default
systemctl set-default multi-user.target
```
Alternatively, one can append `systemd.unit=single-user.target` to kernel command line in boot menu.

Temporarily switching target

```bash
systemctl isolate multi-user.target
```

#### Recover root password

During boot loader countdown, press any key to interrupt the boot process. Then press `e` to edit the boot record.

Append `rd.break` to the line that starts with `linux16`. Press `control + X` to boot.

```bash
mount -o remount,rw /sysroot
chroot /sysroot
passwd root
touch /.autorelabel
exit
exit
```

#### Debug boot issues

Boot into rescue target or emergency target by appending the following to kernel command line

```bash
systemd.unit=rescue.target
# Or
systemd.unit=emergency.target
# List all jobs running at boot time
systemctl list-jobs
```


## 2. `nmcli` for IPv4, IPv6, Teaming

Check current network setup

```bash
nmcli dev status
nmcli con show
ip a
```

Adding static connection to interface `eth0`

```bash
nmcli con add con-name my-con-eth0 ifname eth0 type ethernet \
             ip4 192.168.100.100/24 gw4 192.168.100.1 \
             ip4 1.2.3.4 \
             ip6 abbe::cafe

nmclo con mod my-con-eth0 +ipv4.addresses "10.7.1.2/24"

nmcli con mod my-con-eth0 ipv4.dns "8.8.8.8 8.8.4.4"

nmcli con mod my-con-eth0 +ipv4.dns 1.2.3.4

nmcli con mod my-con-eth0 ipv6.dns "2001:4860:4860::8888 2001:4860:4860::8844"

nmcli con mod my-con-eth0 ipv4.method manual

nmcli -p con show my-con-eth0

nmcli con reload
```

Check permanent configuration at

```bash
cat /etc/sysconfig/network-scripts/ifcfg-my-con-eth0
```

#### Teaming

```bash
nmcli con add type team con-name team0 ifname team0 config '{"runner": {"name" : "roundrobin"}}'
nmcli con mod team0 ipv4.addresses "192.168.0.5/24"
nmcli con mod team0 ipv4.method manual

nmcli con add type team-slave con-name team0-port1 ifname eno1 master team0
nmcli con add type team-slave con-name team0-port2 ifname eno2 master team0

teamdctl team0 state
```

Available runner types: `boardcast`, `roundrobin`, `activebackup`, `loadbalance`, `lacp`

#### Bridging



**Grading checks**
```bash
nmcli dev | grep team0
teamdctl team0 port present eno1
teamdctl team0 port present eno2
teamdctl team0 state | grep runner
brctl show | grep team0 | grep brteam0
```

#### Bridging

```bash
nmcli con add type bridge con-name br1 ifname br1
nmcli con mod br1 ipv4.addresses "192.168.0.10024"
nmcli con mod br1 ipv4.method manual

nmcli con add type bridge-slave con-name br1-port0 ifname eno1 master br1
brctl show
```


## 3. `hostnamectl`

Check current hostname

```bash
hostnamectl status
```

Update hostname

```bash
hostnamectl set-hostname demo.example.com
cat /etc/hostname
```

## 4. `firewall-cmd`

**On-the-field tips**

- Always do `firewall-cmd --permanent ...` then `firewall-cmd --reload`
- `man firewalld.richlanguage`

```bash
firewall-cmd --permanent --add-service http --add-service https
firewall-cmd --reload
```

```bash
firewall-cmd --permanent --add-rich-rule 'rule family=ipv4 source address=172.25.1.10/32 service name="http" log level=notice prefix="NEW HTTP" limit = value="3/s" accept'
```

```bash
firewall-cmd --permanent --zone=work --add-source 172.16.100.0/24
```

**masquerade**

```bash
firewall-cmd --permanent --zone={ZONE} --add-masquerade
firewall-cmd --permanent --add-rich-rule 'rule family=ipv4 source address=192.168.0.0/24 masquerade'
```

**port-forwarding**

```bash
# simple
firewall-cmd --permanent --zone=public --add-forward-port 'port=513:proto=tcp:toport=132:toaddr=192.168.0.254'

# using rich rule
firewall-cmd --permanent --add-rich-rule 'rule family=ipv4 source address=172.25.X.10/32 forward-port port=443 protocol=tcp to-port=22'
```

## 5. selinux

```bash
seinfo -t | grep {keyword}
semange fcontext -a -t samba_share_t '/sambashare(/.*)?'
restorecon -vvRf /sambashare
semanage port  -l | grep http
semanage port --add --proto tcp --type http_port_t 8081
semenage port --delete --proto tcp --type http_port_t 443
```

## 6. Postfix relay

```bash
yum install postfix
postconf -e "relayhost=[smtp1.example.com]"
postconf -e "inet_interfaces=loopback-only"
postconf -e "mydestination="
postconf -e "local_transport=error: disabled"
postconf -e "mynetworks=127.0.0.1/8 [::1]/128
systemctl restart postfix
```

Attempt to send an email using `mutt`
Red Hat suggests `mail` but it did not work for me.

```bash
mutt -s "some subject" foo@example.com < body.txt
tail -f /var/log/mailog

# Login to imaps to verify email was recevied, double check sender domain
mutt -f imaps://student:password@imap.example.com
```


## 7. iscsi

This section assumes candidate is familiar with block device commands, e.g. `lsblk`, `blkid`, `fdisk`, etc.. Often candidate is asked to create a block level disk to use as a backstore for a LUN.

```
IQN - iqn.YYYY-MM.com.reversed.domain[:optional_string]
```

**Server side**

```bash
systemctl enable target.service
systemctl start target.service
/iscsi create "iqn.2015-01.com.example:server0"
# Must use IP address here for portal
/iscsi/iqn.2015-01.com.example:server0/tpg1/portals create 172.25.0.11 [$port]
/backstores/block create name=disk1 dev=/dev/vdb1
/iscsi/iqn.2015-01.com.example:server0/tpg1/luns create /backstores/block/disk1
/iscsi/iqn.2015-01.com.example:server0/tpg1/acls create iqn.2015-01.com.example:desktop0
/iscsi/iqn.2015-01.com.example:server0/tpg1 set attribute generate_node_acls=1
/iscsi/iqn.2015-01.com.example:server0/tpg1 set attribute authentication=0

# Open firewall
firewall-cmd --permanent --add-port 3260/tcp
```

**Client**

```bash
yum install iscsi-initialtor-utils
systemctl enable iscsi.service
systemctl start iscsi.service
iscsiadm -m discovery -t st -p "server1.example.com:3260"
# Configuration is generated
# cat /var/lib/iscsi/nodes/[iqnname]/default
iscsiadm -m node -T "iqn.2018-04.com.example:server1" -l
iscsiadm -m node -P [1|2|3]
fdisk /dev/vda
mkfs -x xfs /dev/vda1
echo "UUID=123-123-123 /iscsidisk xfs _netdev 0 2"
mount -a
```

Remove iscsi
```bash
umount /iscsidisk
iscsiadm -m node -T "iqn.2018-04.com.example:server1" -u
iscsiadm -m node -T "iqn.2018-04.com.example:server1" -o delete
ls -lah /var/lib/iscsi/nodes
```

## 8. NFS

You should be able to set up NFS server exporting non-secure and secure (krb5) directory.

You may need to enrol to LDAP server. Don't forget tool such as `authconfig-gtk` and `krb5-workstation`


```bash
systemctl enable nfs-server
systemctl start nfs-server
echo "/myshare desktop0(rw)" >> /etc/exports
echo "/public 172.16.0.0/16(ro) *.example.com(ro,no_root_squash)" >> /etc/exports
exportfs -r
firewall-cmd --permanent --add-service nfs
firewall-cmd --reload
```

```bash
mount serverX:/myshare /mnt/nfsexport
```

**Protected NFS**

**server**

```bash
# Download and verify keytab is valid
wget -O /etc/krb5.keytab http://classroom.example.com/serverX.keytab
klist -k /etc/krb5.keytab

# Start nfs-secure-server
systemctl enable nfs-secure-server
^enable^start

mkdir /secureexport
echo '/securedexport *.example.com(sec=krb5p,rw)' >>/etc/exports
exportfs -r
exportfs -v
```

To enable SElinux labels, make sure the following exists in `/etc/sysconfig/nfs`

```bash
RPCNFSDARGS="-V 4.2"
```

**desktop**

```bash
wget -O /etc/krb5.keytab http://classroom.example.com/desktopX.keytab
systemctl enable nfs-secure
systemctl start nfs-secure
mount -t nfs4 -o sec=krb5p serverX:/securedexport /mnt/securedexport

# Mount permanently
echo "server1:/secureexport /mnt/secureexport nfs defaults,v4.2,sec=krb5p,rw 0.0" >> /etc/fstab
mount -a
```

## 9. Samba

You should be able to set up a shared folder in serverX via samba; accessible to groups `mngt` and `employees`. Users in group `mngt` should have write access.


**server**
```bash
yum install samba -y
mkdir -p /sambashare
chmod -R 2775 /sambashare
semanage fcontext -a -t samba_share_t '/sambashare(/.*)?'
restorecon -vvFR /sambashare

useradd -s /sbin/nologin brian
smbpasswd -a brian
```

**desktop**

```bash
yum install samba-client cifs-utils -y
smbclient -L server -U brian # Test login credential
mkdir -p /mnt/brian

# Mount single user
echo "//server/smbshare /mnt/brian cifs credentials=/etc/secure/brian.login 0 0" >> /etc/fstab

# Mount multi-user
echo "//server/smbshare /mnt/share cifs credentials=/root/cred.txt",multiuser,sec=ntlmssp 0 0" >> /etc/fstab
mount -a
```

**Test multi-user samba on `desktop`**

```bash
su - rob
cifscreds add server1
```

## 10. `unbound`

`/etc/unbound/conf.d/forwarder.conf`
```
server:
	interface: 0.0.0.0
    interface: ::0
    access-control: 172.25.1.0/24 allow
    domain-insecure: "example.com"
forward-zone:
	name: .
    forward-addr: 172.25.254.254
```

```bash
yum install unbound -y
systemctl enable unbound
systemctl start unbound
firewall-cmd --permanent --add-service dns
firewall-cmd --reload
```

## 11. `httpd`

```bash
yum install httpd mod_ssl mod_php php-mysql mod_wsgi -y
semanage port -a -p tcp -t http_port_t 444
semanage fcontext -a -t public_content_t '/custom/webroot(/.*)?'
restorecon -RFv '/custom/webroot'
```
Sample virtualhost config with SSL

```bash
<VirtualHost *:444>
    ServerName webapp1.example.com
    ServerAlias webapp1
    SSLEngine On
    SSLProtocol -SSLv2 -SSLv3
    SSLCipherSuite HIGH:MEDIUM:!aNULL:!MD5
    SSLHonorCipherOrder On
    SSLCertificateFile '/etc/pki/tls/certs/webapp1.crt'
    SSLCertificateKeyFile '/etc/pki/tls/private/webapp1.key'
    SSLCertificateChainFile '/etc/pki/tls/certs/ca-example.crt'
    DocumentRoot /srv/webapp1/www
    DirectoryIndex index.html index.php
</VirtualHost>
<Directory /srv/webapp/www>
    Require grant alll
</Directory>
```

For python web app
```
WSGIScriptAlias /myapp/ /srv/myapp/www/myapp.py
```

Validate
```bash
curl -cacert -vvv example-ca.crt https://webapp1.example.com:444
```


## Appendix A: Command cheatsheet

**selinux**

```bash
semanage fcontext -t # List all current rules
seinfo -t  # List all available context
seinfo -u  # List all context users
seinfo -r  # List all context roles

# panic mode with selinux
audit2allow -m mypolicy < /var/log/audit/audit.log
semodule -i mypolicy.pp
```


## Appendix B: Gap-filling labs

#### SMB
```
Create SMB share `smbshare` on serverX using mycompany workgroup
- member of group `marketing` have rw permission.
- all users not in `marketing` group have read-only permission.
- User `brian` is part of marketing team and password is `redhat`.
- User `rob` has password `redhat`.
```

#### NFS

```
Configure the NFS server on serverX to meet the following requirements:

Share the newly created /krbnfs directory on serverX with krb5p security.

Allow read and write access on the share from the desktopX system.

SELinux labels are exported.

Preconfigured krb5 keytabs for the serverX and desktopX systems are available at:

http://classroom.example.com/pub/keytabs/serverX.keytab.

http://classroom.example.com/pub/keytabs/desktopX.keytab.

Allow access to the NFS service through the firewall.```

