+++
title = "building secure linux systems"
date = "2011-12-14"
slug = "2011/12/14/building-secure-linux-systems"
Categories = []
+++

In this post, I'm going to be documenting the process that I'm working on to build secure Linux systems.

What I'd like to have when I'm done:
- Selinux is ON and enforcing
- Is certifiable to a set of reasonable standards
- Can be deployed in an automated fashion
- Supports remediation if flaws against known good state found
<!--more-->
Phew, that's quite the laundry list. But it forms the basis of a good security architecture. Thankfully, there's lots of help to be had in putting these things together.

The basis of my work will revolve around
- [CLIP](http://oss.tresys.com/projects/clip), which is a project which uses puppet to build a certifiable linux platform
- [Secstate](https://fedorahosted.org/secstate/), which in turn makes use of [OpenSCAP](http://open-scap.org/) to streamline C&A
- [Puppet](http://www.puppetlabs.com), to handle configuration management
- Cobbler to automate provisioning

I'm going to try to base this on CentOS 6.1, which means I'll be trying to update the CLIP related packages/puppet rules to work on a more recent CentOS.

Step 1. Provisioning system: Cobbler

Getting cobbler up and running was fairly painless. Install EPEL, install the packages and you're almost good to go. Things get a bit tricky if your cobbler system is running selinux in enforcing mode. 
Don't forget to add services to runlevels and start services that are need to make this work if you're doing pxebooting. Edit tftp config file, and edit the cobbler dhcpd template as well as cobbler settings.

```bash
chkconfig xinetd on
chkconfig cobblerd on
chkconfig dhcpd on
```

Running this command gives you a bit of insight:

```bash
cobbler check
```

My cobbler system is running CentOS 5.7, and the recommended fixes didn't work for a couple of the items related to selinux, as the contexts/flags have changed a bit. I had to fiddle about a bit with chcon and semanage. Something like:

```bash
chcon -Rv --type=httpd_sys_content_t /var/www/cobbler
semanage fcontext -a -t httpd_sys_content_t '/var/www/cobbler/.*' 
```

I also did something similar to get tftp working, although I didn't note the exact command. This command is your friend:

```bash
seinfo -t
```

You can also look in /var/log/audit/audit.log for more clues as to what's broken. I had a bit of fun figuring out pxebooting server wasn't able to download it's kernel until I looked at selinux.
If you're having issues, see [here](http://wiki.centos.org/HowTos/SELinux)

I recommend repo cloning using cobbler to speed things up. Note that if you're using CentOS 6, the repo cloning will fail with a weird error unless you install python-hashlib first.

```bash
yum install python-hashlib
```

Here's a sample command to clone a repo:

```bash
cobbler repo add --name=centos-6.1-x86_64 --mirror=http://yum.singlehop.com/CentOS/6.1/os/x86_64/
cobbler reposync --only=centos-6.1-x86_64
```

For the pxeboot to work, you'll need to make the install image, kernel and initrd. You can just go grab em from the mirror: eg: http://yum.singlehop.com/CentOS/6.1/os/x86_64/images/.

I chose to put them inside the local repo tree, and replicated all the files.

```bash
wget --mirror -np http://yum.singlehop.com/CentOS/6.1/os/x86_64/images/
```

I then moved the resulting images folder into /var/www/cobbler/repo_mirror/centos-6.1-everything-x86_64/

Let's create a distro based on that repo

```bash
cobbler distro add --name=CentOS-6.1-x86_64 \
--kernel=/var/www/cobbler/repo_mirror/centos-6.1-everything-x86_64/images/pxeboot/vmlinuz \
--initrd=/var/www/cobbler/repo_mirror/centos-6.1-everything-x86_64/images/pxeboot/initrd.img \
--arch=x86_64 --breed=redhat 
```

Create a kickstart script (/var/lib/cobbler/kickstarts/test.ks) which looks like the one below to get started. It's based on the clip kickstart installation script:

```
install
text
firstboot --disable
url --url=$tree
# If any cobbler repo definitions were referenced in the kickstart profile, include them here.
$yum_repo_stanza
# Network information
$SNIPPET('network_config')
lang en_US
langsupport --default en_US en_US
keyboard us
ignoredisk --drives='sdb,sdc,sdd'
zerombr
clearpart --all --initlabel
partition /boot --fstype 'ext2' --size=128 --ondisk=sda --fsoptions='ro,nodev,nosuid,noexec,noatime,noauto'
partition pv.2 --size=0 --grow --ondisk=sda --encrypt --passphrase='I <3 $$$, cheese, and >:-('
volgroup VolGroup00 pv.2
logvol swap --fstype swap --name=swapVol --vgname=VolGroup00 --size=2048 
logvol / --fstype ext4 --name=rootVol --vgname=VolGroup00 --size=1 --grow --fsoptions='noatime' 
logvol /var --fstype ext4 --name=varVol --vgname=VolGroup00 --size=8196 --fsoptions='noatime,nodev'
logvol /home --fstype ext4 --name=homeVol --vgname=VolGroup00 --size=1024 --fsoptions='noatime,noexec,nodev,nosuid' 
logvol /tmp --fstype ext4 --name=tmpVol --vgname=VolGroup00 --size=1024 --fsoptions='noatime,noexec,nodev,nosuid' 

bootloader --location mbr --password 123)(*qweASD
rootpw 123)(*qweASD
auth --passalgo=sha512 --enableshadow
timezone --utc America/Toronto

# Fow now... This is to eventually get clip setup. 
selinux --disable

# Enable the firewall 
firewall --enabled --port=22:tcp

# Reboot after installation is complete
reboot

# Install Packages.&nbsp;&nbsp;This is site specific.
%packages
$SNIPPET('func_install_if_enabled')
$SNIPPET('puppet_install_if_enabled')
@base
policycoreutils-newrole
aide
sysstat
setools
audit
#####################################
# Remove tcpdump per STIG gen003865 #
#####################################
-tcpdump
#####################################
# Remove Packages for PL4 compliance#
#####################################
-kudzu
-pcscd
-xdelta
-nmap
-emacspeak
-byacc
-gimp-help
-splint
-perl-Crypt-SSLeay
-units
-perl-XML-Grove
-perl-XML-LibXML-Common
-perl-XML-SAX
-perl-XML-Twig
-valgrind
-valgrind-callgrind
-gimp-gap
-cdecl
-perl-XML-Dumper
-kernel-smp-devel
-blas
-lapack
-java-1.4.2-gcj-compat
-kernel-hugemem-devel
-kernel-devel
-perl-XML-Encoding
-gnome-games
-isdn4k-utils
-vnc
-vnc-server
-gpm
-hal
#e2fsprogs
#kernel-smp
-tog-pegasus
-tog-pegasus-devel
-ethereal
-ethereal-gnome
-xchat
-vino
-gaim
-gnome-pilot
-bluez-utils
-bluez-utils-cups
-bluez-hcidump
-bluez-gnome
-yum-updatesd
-wpa_supplicant
-ypbind
-NetworkManager
-NetworkManagerDispatcher
-setools
-telnet
-wireless-tools
#@ office
#@ admin-tools
#@ editors
#@ system-tools
#@ gnome-desktop
#@ dialup
#@ base-x
#@ printing
#@ server-cfg
#@ graphical-internet
#kernel
-python-ldap
-httpd-suexec
-system-config-httpd
-psgml
-emacs-leim
-gimp-data-extras
-xcdroast
-perl-XML-LibXML
-gimp-print-plugin
-xsane-gimp
-gimp
#lvm2
-zsh
#net-snmp-utils
-rhythmbox
-gcc-g77
#grub
-texinfo
-octave
-dia
-perl-LDAP
-oprofile
-emacs
#system-config-printer-gui
-doxygen
-planner
-tux
-indent
-cdparanoia
-gcc-java
-gnomemeeting
#openoffice.org-i18n
#openoffice.org-libs
#openoffice.org
#firefox
-evolution
-xsane
-ctags
-cscope
-sane-frontends
-perl-XML-Parser
-php-mysql
-rcs
-perl-XML-NamespaceSupport
# get rid of rlogin
-rsh
-irda-utils
-lm_sensors
-portmap
-nfs-utils
-autofs
-finger
-tftp
-avahi
vlock
# needed to compile policy on RHEL5.1
rpm-build
gcc
checkpolicy
# for puppet:
ruby

%pre
$SNIPPET('log_ks_pre')
$kickstart_start
$SNIPPET('pre_install_network_config')
# Enable installation monitoring
$SNIPPET('pre_anamon')

%post
$SNIPPET('log_ks_post')
# Start yum configuration 
$yum_config_stanza
# End yum configuration

SNIPPET::public_key_root
SNIPPET::bashprofile_root

$SNIPPET('post_install_kernel_options')
$SNIPPET('post_install_network_config')
$SNIPPET('func_register_if_enabled')
$SNIPPET('puppet_register_if_enabled')
# $SNIPPET('download_config_files')
# $SNIPPET('koan_environment')
# $SNIPPET('redhat_register')
# $SNIPPET('cobbler_register')
# Enable post-install boot notification
$SNIPPET('post_anamon')
# Start final steps
$kickstart_done
# End final steps
```

Create a profile based on the distro and kickstart

```bash
cobbler profile add --name=CentOS-6.1-x86_64 \
--distro=CentOS-6.1-x86_64 \
--kickstart=/var/lib/cobbler/kickstarts/test.ks \
--ksmeta='tree=http://10.0.0.1/cobbler/repo_mirror/centos-6.1-everything-x86_64/'
--nameservers='8.8.8.8' \
--repos='centos-6.1-updates-x86_64,centos-6.1-everything-x86_64' 
```

Note that when creating profiles, we set the ksmeta tree variable. This variable is used to specify the URL location of the repository to use for the installation (in the kickstart script). It should be changed to whatever your cobbler IP address is (if you've cloned a centos repository that is). If you haven't cloned, simply change the variable to point to a CentOS mirror. Thus we can use the same kickstart on different arches, simply creating profiles for each arch with different paths in the ksmeta tree variable.
What do we have so far:

- Installs OS on /dev/sda
- Two physical partitions
- /boot ext2 partition
- Rest of the disk on an encrypted partition (note: requires a passphrase to boot)
- ext4 logical volumes created on the second encrypted partition, with reasonable restrictions on mount options
- Calls a few custom snippets to do things like install SSH public keys and root bash profile
- Enables the firewall and SSH access
- Selinux is disabled&#8230; We'll be working on this.
- Bootloader password protected

Update

I got tired of issues getting things working on CentOS 5.7, and started updating everything to 6.2. Fun times, as the selinux policies are inadequate for cobbler. One of the more fun and mysterious issues I've encountered: Reposync wasn't working in cobbler. I had created some custom selinux policies to get things moving, then came across this unpleasant error.

```
error: db3 error(11) from dbenv->open: Resource temporarily unavailable
error: cannot open Packages index using db3 - Resource temporarily unavailable (11)
```

Apparently something in the reposync hosed my rpm db. The fix below:

```bash
rm -f /var/lib/rpm/__db*
echo '%__dbi_cdb create private cdb mpool mp_mmapsize=16Mb mp_size=1Mb' > /etc/rpm/macros
rpmdb --rebuilddb
cobbler reposync
```

Step 2. Install CLIP software

Fail.

I managed to get all the clip stuff installed, but failed when trying to get the selinux rules working. I unfortunately don't have time to delve deeper, so instead I'm going to run with selinux in enforcing mode/strict, and adjust rules as needed. I'll try to get the clip puppet content integrated with the puppetization of the systems. Or not.
As it stands, this is what my cobbler config looks like. It's a mishmash of the CLIP stuff and a few other security resources I came across.

```
install
text
url --url=$tree
firstboot --disable
$yum_repo_stanza
$SNIPPET('network_config')
lang en_US
keyboard us

## RAW escape tag for cheetah template
#raw
## Note, you should figure out your own encrypted password, this one below won't work for you :)
rootpw --iscrypted $6$asdfasdfasdfasdfzKsMJgzTxm/l40I.
#end raw
auth --passalgo=sha512 --enableshadow --kickstart
timezone --utc America/Toronto
selinux --enforcing
firewall --enabled --port=22:tcp

ignoredisk --only-use=sdb
zerombr
clearpart --all --initlabel
bootloader --location mbr --password='123)(*qweASD'
part /boot --fstype 'ext2' --size=128 --ondisk=sdb --fsoptions='noatime'
part pv.2 --size=1 --grow --ondisk=sdb --encrypted --passphrase='Mmmmm doughnuts....'
volgroup VolGroup00 pv.2
logvol swap --fstype swap --name=swapVol --vgname=VolGroup00 --size=2048 
logvol / --fstype ext4 --name=rootVol --vgname=VolGroup00 --size=1 --grow --fsoptions='noatime' 
logvol /var --fstype ext4 --name=varVol --vgname=VolGroup00 --size=8196 --fsoptions='noatime,nodev'
logvol /home --fstype ext4 --name=homeVol --vgname=VolGroup00 --size=1024 --fsoptions='noatime,noexec,nodev,nosuid' 
logvol /tmp --fstype ext4 --name=tmpVol --vgname=VolGroup00 --size=1024 --fsoptions='noatime,noexec,nodev,nosuid' 

reboot

%packages --nobase
$SNIPPET('puppet_install_if_enabled')
MAKEDEV
aide
acl
at
attr
audit
audit-libs
authconfig
basesystem
bash
bc
bind-libs
bind-utils
binutils
biosdevname
blktrace
bridge-utils
busybox
bzip2
bzip2-libs
ca-certificates
centos-indexhtml
centos-release
checkpolicy
chkconfig
coreutils
coreutils-libs
cpio
cracklib
cracklib-dicts
cronie
cronie-anacron
crontabs
cryptsetup-luks
cryptsetup-luks-libs
curl
dash
db4
db4-utils
device-mapper
device-mapper-event
device-mapper-event-libs
device-mapper-libs
diffutils
dmidecode
dmraid
dmraid-events
dracut
dracut-kernel
e2fsprogs
e2fsprogs-libs
ed
efibootmgr
elfutils
elfutils-libelf
elfutils-libs
ethtool
expat
file
file-libs
filesystem
findutils
fipscheck
fipscheck-lib
gamin
gawk
gdbm
glib2
glibc
glibc-common
gmp
gnupg2
gnutls
gpgme
gpm-libs
grep
groff
grub
grubby
gzip
hdparm
hwdata
info
initscripts
iproute
iptables
iputils
irqbalance
kbd
kbd-misc
kernel
kernel-firmware
kexec-tools
keyutils-libs
kpartx
krb5-libs
less
libacl
libaio
libattr
libblkid
libcap
libcap-ng
libcom_err
libcurl
libdrm
libedit
libffi
libgcc
libgcrypt
libgpg-error
libidn
libjpeg
libnih
libnl
libpcap
libpng
libselinux
libselinux-utils
libsemanage
libsepol
libss
libssh2
libstdc++
libtar
libtasn1
libtiff
libudev
libusb
libusb1
libuser
libutempter
libuuid
libxml2
libxml2-python
logrotate
lsof
lua
lvm2
lvm2-libs
m4
mailx
man
man-pages
man-pages-overrides
microcode_ctl
mingetty
mlocate
module-init-tools
mtr
#mysql-libs
nano
ncurses
ncurses-base
ncurses-libs
net-tools
#netxen-firmware
#newt
#newt-python
nspr
nss
nss-softokn
nss-softokn-freebl
nss-sysinit
nss-util
ntp
ntpdate
ntsysv
openldap
openssh
openssh-clients
openssh-server
openssl
pam
pam_passwdqc
parted
passwd
pciutils
pciutils-libs
pcre
perl
perl-Module-Pluggable
perl-Pod-Escapes
perl-Pod-Simple
perl-libs
perl-version
pinfo
pkgconfig
plymouth
plymouth-core-libs
plymouth-scripts
policycoreutils
popt
prelink
procps
psacct
psmisc
pth
pygpgme
python
python-ethtool
python-iniparse
python-libs
python-pycurl
python-urlgrabber
quota
rdate
readahead
readline
redhat-logos
rng-tools
rootfiles
rpm
rpm-libs
rpm-python
rsync
rsyslog
sed
selinux-policy
selinux-policy-targeted
setserial
setup
setuptool
shadow-utils
slang
smartmontools
sqlite
strace
sudo
sysstat
systemtap-runtime
sysvinit-tools
tar
tcp_wrappers
tcp_wrappers-libs
tcpdump
tcsh
time
tmpwatch
traceroute
tzdata
udev
unzip
upstart
usbutils
usermode
ustr
util-linux-ng
vconfig
vim-common
vim-enhanced
vim-minimal
wget
which
words
xmlrpc-c
xmlrpc-c-client
xz
xz-libs
xz-lzma-compat
yum
yum-metadata-parser
yum-plugin-fastestmirror
yum-utils
zip
zlib
-aic94xx-firmware
-atmel-firmware
-bfa-firmware
-ipw2100-firmware
-ipw2200-firmware
-ivtv-firmware
-iwl*-firmware
-libertas-usb8388-firmware
-netxen-firmware
-ql*-firmware
-rt61pci-firmware
-rt73usb-firmware
-xorg-x11-drv-ati-firmware
-zd1211-firmware
-b43-openfwwf

%pre
$SNIPPET('log_ks_pre')
$kickstart_start
$SNIPPET('pre_install_network_config')
$SNIPPET('pre_anamon')

%post
$SNIPPET('log_ks_post')
$yum_config_stanza
SNIPPET::public_key_root
SNIPPET::bashprofile_root
$SNIPPET('post_install_kernel_options')
$SNIPPET('post_install_network_config')
$SNIPPET('puppet_register_if_enabled')
rm /etc/yum.repos.d/Cent*
echo 'sshd:172.16.' >>/etc/hosts.allow

chkconfig postfix off
chkconfig kdump off
chkconfig rdisc off
chkconfig restorecond on
chkconfig ntpdate on
chkconfig ntpd on

#raw
# RAW tag to escape cheetah templating...
# Hardening... adapted from here https://nazar.karan.org/cgit/bluecain/tree/secure-kickstart.cfg
#
rpm --import /etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-6

echo 'Locking down GEN000020, GEN000040, GEN000060'
perl -npe 's#SINGLE=/sbin/sushell#SINGLE=/sbin/sulogin#' -i /etc/sysconfig/init
echo 'GEN000020, GEN000040,GEN000060 Complete'

## Prevent entering interactive boot
perl -npe 's/PROMPT=yes/PROMPT=no/' -i /etc/sysconfig/init
echo 'Interactive Boot disabled'

# LNX00580 (L222)
echo 'Locking down LNX00580'
perl -npe 's/^exec/#exec/' -i /etc/init/control-alt-delete.conf
echo 'LNX00580 Complete'

# GEN000480 (G015)
echo 'Locking down GEN000480'
echo '#Make the user waits four seconds if they fail after LOGIN_RETRIES' >> /etc/login.defs
echo 'FAIL_DELAY 4' >> /etc/login.defs
echo 'GEN000480 Complete'

# GEN000820
echo 'Locking down GEN000820'
perl -npe 's/PASS_MIN_LEN\s+5/PASS_MIN_LEN&nbsp;&nbsp;9/' -i /etc/login.defs

#STIG specifies using foloowing, but it's not a valid parameter
#echo 'PASSLENGTH 9' >> /etc/login.defs
#
echo 'GEN000820 Complete'

##### PAM Modifications
# These modifications apply to GEN000460, GEN000600 and GEN000620
touch /var/log/tallylog

cat << 'EOF' > /etc/pam.d/system-auth-local
#%PAM-1.0
# Auth Section
auth required pam_tally2.so unlock_time=900 onerr=fail no_magic_root
auth required pam_faildelay.so delay=5000000
auth include system-auth-ac

# Accounts Section
account required pam_tally2.so
account include system-auth-ac

# Password Section
password required pam_pwhistory.so&nbsp;&nbsp; use_authtok remember=5 retry=3
password requisite pam_passwdqc.so min=disabled,disabled,16,12,8
password include system-auth-ac

# Session Section
## By default we're only going to log what root does
## This gets really verbose if we log more.
## If you want to log everyone, remove disable=*
session required pam_tty_audit.so disable=* enable=root
session include system-auth-ac
EOF

rm /etc/pam.d/system-auth
ln -s /etc/pam.d/system-auth-local /etc/pam.d/system-auth
# Create some basic shell rules for users. 

echo 'Idle users will be removed after 15 minutes'
echo 'readonly TMOUT=600' >> /etc/profile.d/os-security.sh
echo 'readonly HISTFILE' >> /etc/profile.d/os-security.sh
chmod +x /etc/profile.d/os-security.sh

# GEN002560
# Reset the umasks for all users to 077
echo 'Locking down GEN002560'
perl -npe 's/umask\s+0\d2/umask 077/g' -i /etc/bashrc
perl -npe 's/umask\s+0\d2/umask 077/g' -i /etc/csh.cshrc
echo 'GEN002560 Complete'

######### Remove useless users ##############
# This isn't strictly needed as they have a default shell of nologin
# but we're removing them anyway to be safe.

echo 'Locking down LNX0034 and GEN004840'
/usr/sbin/userdel shutdown
/usr/sbin/userdel halt
/usr/sbin/userdel games
/usr/sbin/userdel operator
/usr/sbin/userdel ftp
/usr/sbin/userdel news
/usr/sbin/userdel gopher
/usr/sbin/userdel lp
/usr/sbin/userdel adm
/usr/sbin/userdel uucp
echo 'LNX0034 and GEN004840 Complete'

# No one gets to run cron or at jobs unless we say so.
echo 'Locking down Cron'
touch /etc/cron.allow
chmod 600 /etc/cron.allow
awk -F: '{print $1}' /etc/passwd | grep -v root > /etc/cron.deny
echo 'Locking down AT'
touch /etc/at.allow
chmod 600 /etc/at.allow
awk -F: '{print $1}' /etc/passwd | grep -v root > /etc/at.deny

echo -e 'root\n' >/etc/cron.allow
echo -e 'root\n' >/etc/at.allow

echo 'Get rid of stupid mailing cron results...'
perl -npe 's/(MAILTO.*)$/MAILTO=''/' -i /etc/crontab

## GEN000400 (G010)
echo 'Locking down GEN000400'
# Set the /etc/issue file to something scary.&nbsp;&nbsp;This one has no linefeeds, so it will wrap accordingly.
# Change this to your own banner for organizations outside the Scary Zone

cat << 'EOF' >/etc/issue
USE OF THIS COMPUTER SYSTEM, AUTHORIZED OR UNAUTHORIZED, CONSTITUTES CONSENT TO MONITORING OF THIS SYSTEM.&nbsp;&nbsp;UNAUTHORIZED USE MAY SUBJECT YOU TO CRIMINAL PROSECUTION.&nbsp;&nbsp;EVIDENCE OF UNAUTHORIZED USE COLLECTED DURING MONITORING MAY BE USED FOR ADMINISTRATIVE, CRIMINAL, OR OTHER ADVERSE ACTION.&nbsp;&nbsp;USE OF THIS SYSTEM CONSTITUTES CONSENT TO MONITORING FOR THESE PURPOSES.
EOF
echo 'GEN000400 Completed'

################## File and Directory Security #########################
# Restrict mount points with noexec, nosuid, and nodev where applicable
# GEN002420
echo 'Locking down GEN002420'
FSTAB=/etc/fstab
SED=/bin/sed

#nosuid on /home
if [ $(grep '[[:blank:]]\/home[[:blank:]]' ${FSTAB} | grep -c 'nosuid') -eq 0 ]; then
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;MNT_OPTS=$(grep '[[:blank:]]\/home[[:blank:]]' ${FSTAB} | awk '{print $4}')
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;${SED} -i 's/\([[:blank:]]\/home.*${MNT_OPTS}\)/\1,nosuid/' ${FSTAB}
fi

# nosuid on /sys
if [ $(grep '[[:blank:]]\/sys[[:blank:]]' ${FSTAB} | grep -c 'nosuid') -eq 0 ]; then
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;MNT_OPTS=$(grep '[[:blank:]]\/sys[[:blank:]]' ${FSTAB} | awk '{print $4}')
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;${SED} -i 's/\([[:blank:]]\/sys.*${MNT_OPTS}\)/\1,nosuid/' ${FSTAB}
fi

## nosuid on /boot
if [ $(grep '[[:blank:]]\/boot[[:blank:]]' ${FSTAB} | grep -c 'nosuid') -eq 0 ]; then
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;MNT_OPTS=$(grep '[[:blank:]]\/boot[[:blank:]]' ${FSTAB} | awk '{print $4}')
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;${SED} -i 's/\([[:blank:]]\/boot.*${MNT_OPTS}\)/\1,nosuid/' ${FSTAB}
fi

# nodev on /usr
if [ $(grep '[[:blank:]]\/usr[[:blank:]]' ${FSTAB} | grep -c 'nodev') -eq 0 ]; then
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;MNT_OPTS=$(grep '[[:blank:]]\/usr[[:blank:]]' ${FSTAB} | awk '{print $4}')
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; ${SED} -i 's/\([[:blank:]]\/usr.*${MNT_OPTS}\)/\1,nodev/' ${FSTAB}
fi

#nodev on /home
if [ $(grep '[[:blank:]]\/home[[:blank:]]' ${FSTAB} | grep -c 'nodev') -eq 0 ]; then
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;MNT_OPTS=$(grep '[[:blank:]]\/home[[:blank:]]' ${FSTAB} | awk '{print $4}')
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;${SED} -i 's/\([[:blank:]]\/home.*${MNT_OPTS}\)/\1,nodev/' ${FSTAB}
fi

# nodev on /usr/local
if [ $(grep '[[:blank:]]\/usr/local[[:blank:]]' ${FSTAB} | grep -c 'nodev') -eq 0 ]; then
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;MNT_OPTS=$(grep '[[:blank:]]\/usr/local[[:blank:]]' ${FSTAB} | awk '{print $4}')
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;${SED} -i 's/\([[:blank:]]\/usr\/local.*${MNT_OPTS}\)/\1,nodev/' ${FSTAB}
fi

# nodev and noexec on /tmp
if [ $(grep '[[:blank:]]\/tmp[[:blank:]]' ${FSTAB} | grep -c 'nodev') -eq 0 ]; then
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;MNT_OPTS=$(grep '[[:blank:]]\/tmp[[:blank:]]' ${FSTAB} | awk '{print $4}')
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;${SED} -i 's/\([[:blank:]]\/tmp.*${MNT_OPTS}\)/\1,nodev,noexec/' ${FSTAB}
fi
# nodev and noexec on /var/tmp
if [ $(grep '[[:blank:]]\/var/tmp[[:blank:]]' ${FSTAB} | grep -c 'nodev') -eq 0 ]; then
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;MNT_OPTS=$(grep '[[:blank:]]\/tmp[[:blank:]]' ${FSTAB} | awk '{print $4}')
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;${SED} -i 's/\([[:blank:]]\/var/tmp.*${MNT_OPTS}\)/\1,nodev,noexec/' ${FSTAB}
fi

# nodev on /var
if [ $(grep '[[:blank:]]\/var[[:blank:]]' ${FSTAB} | grep -c 'nodev') -eq 0 ]; then
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; MNT_OPTS=$(grep '[[:blank:]]\/var[[:blank:]]' ${FSTAB} | awk '{print $4}')
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;${SED} -i 's/\([[:blank:]]\/var.*${MNT_OPTS}\)/\1,nodev/' ${FSTAB}
fi

echo 'GEN002420 Complete'

# By default&nbsp;&nbsp;/root has permissions of 750. Change this to 700
# GEN000920 (G023)
echo 'Locking down GEN000920'
# Correct the permissions on /root to a DISA allowed 700
chmod 700 /root
echo 'GEN000920 Complete'

# GEN002680 (G094)
# reset permissions on audit logs
echo 'Locking down GEN002680'
chmod 700 /var/log/audit
chmod 600 /var/log/audit/*
echo 'GEN002680 Complete'

# GEN003080
echo 'Locking down GEN003080'
chmod 600 /etc/crontab
chmod 700 /usr/share/logwatch/scripts/logwatch.pl
echo 'GEN003080 Complete'

# GEN003520 ( RHEL5 default anyway )
echo 'Locking down GEN003520'
chmod 700 /var/crash
chown -R root.root /var/crash
echo 'GEN003520 Complete'

# GEN006520
echo 'Locking down GEN006520'
chmod 740 /etc/rc.d/init.d/iptables
chmod 740 /sbin/iptables
chmod 740 /usr/share/logwatch/scripts/services/iptables
echo 'GEN006520 Complete'

# GEN001560
echo 'Locking down GEN001560'
chmod -R 700 /etc/skel
echo 'GEN001560 Complete'

# GEN005400 (G656)
# Reset the permissions to a DISA-blessed rw-r-----
echo 'Locking down GEN005400'
chmod 640 /etc/syslog.conf
echo 'GEN005400 Complete'

# LNX00440 (L046)
# Set mode to DISA-blessed rw-r------
echo 'Locking down LNX00440'
chmod 640 /etc/security/access.conf
echo 'LNX00440 Complete'

# GEN001260
echo 'Locking down GEN001260'
perl -npe 's%chmod 0664 /var/run/utmp /var/log/wtmp%chmod 0644 /var/run/utmp /var/log/wtmp%g' -i /etc/rc.d/rc.sysinit
echo 'GEN001260'

# LNX00520 (L208)
echo 'Locking down LNX00520'
chmod 600 /etc/sysctl.conf
echo 'LNX00520 Complete'

# Add some enhancements to sysctl
cat << 'EOF' >> /etc/sysctl.conf
net.ipv4.ip_forward = 0
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.default.send_redirects = 0
net.ipv4.tcp_max_syn_backlog = 1280
net.ipv4.icmp_echo_ignore_broadcasts = 1
net.ipv4.conf.all.accept_source_route = 0
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.all.secure_redirects = 0
net.ipv4.conf.all.log_martians = 1
net.ipv4.conf.default.accept_source_route = 0
net.ipv4.conf.default.accept_redirects = 0
net.ipv4.conf.default.secure_redirects = 0
net.ipv4.icmp_echo_ignore_broadcasts = 1
net.ipv4.icmp_ignore_bogus_error_responses = 1
net.ipv4.tcp_syncookies = 1
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.default.rp_filter = 1
net.ipv4.tcp_timestamps = 0
kernel.exec-shield = 1
kernel.randomize_va_space = 1
EOF

########## Turn off the uneeded stuff #############
# IAVA0410 (G592) and GEN003700
# Turn off unneeded services
# You may want to leave sendmail enabled but the STIG says otherwise
# I mark this as mitigated by the firewall, and not accepting outside
# connections. The NSA RHEL5 guide has a full service list 
# with recommendations
echo 'Locking down IAVA0410 and GEN003700'
/sbin/chkconfig bluetooth off
/sbin/chkconfig irda off
/sbin/chkconfig lm_sensors off
/sbin/chkconfig portmap off
/sbin/chkconfig rawdevices off
/sbin/chkconfig rpcgssd off
/sbin/chkconfig rpcidmapd off
/sbin/chkconfig rpcsvcgssd off
/sbin/chkconfig sendmail off
/sbin/chkconfig xinetd off
/sbin/chkconfig kudzu off
echo 'IAVA0410 and GEN003700 Complete'

############# SSH restrictions ###############
#
# GEN001120 (G500)
#
# Restricting all logins to only use key based authentication
#
# We need to restrict ssh root logins; We won't allow password based auth via ssh. 
echo 'Locking down GEN001120'
perl -npe 's/#PermitRootLogin yes/PermitRootLogin without-password/' -i /etc/ssh/sshd_config
perl -npe 's/.*PasswordAuthentication yes/PasswordAuthentication no/' -i /etc/ssh/sshd_config
echo 'GEN001120 Complete'

echo 'Removing sftp access'
perl -npe 's%(Subsystem\s+sftp\s+/usr/libexec/openssh/sftp-server)%#$1%' -i /etc/ssh/sshd_config

#GEN005540
echo 'Locking down GEN005540'
perl -npe 's/^#Banner none/Banner \/etc\/issue/g' -i /etc/ssh/sshd_config
echo 'GEN005540 Complete'

perl -npe 's/^#ServerKeyBits 1024/ServerKeyBits 2048/g' -i /etc/ssh/sshd_config
perl -npe 's/^#MaxAuthTries 6/MaxAuthTries 3/g' -i /etc/ssh/sshd_config

################ Configure a better default firewall ###############

## Replace this with a better implementation of Mark's firewall... Things like allowing all outbound is bad...

cat << 'EOF' > /etc/sysconfig/iptables
#Drop anything we aren't explicitly allowing. All outbound traffic is okay
*filter
:INPUT DROP [0:0]
:FORWARD DROP [0:0]
:OUTPUT ACCEPT [0:0]
:RH-Firewall-1-INPUT - [0:0]
-A INPUT -j RH-Firewall-1-INPUT
-A FORWARD -j RH-Firewall-1-INPUT
-A RH-Firewall-1-INPUT -i lo -j ACCEPT
-A RH-Firewall-1-INPUT -p icmp --icmp-type echo-reply -j ACCEPT
-A RH-Firewall-1-INPUT -p icmp --icmp-type destination-unreachable -j ACCEPT
-A RH-Firewall-1-INPUT -p icmp --icmp-type time-exceeded -j ACCEPT
# Accept Pings
-A RH-Firewall-1-INPUT -p icmp --icmp-type echo-request -j ACCEPT
# Log anything on eth0 claiming it's from a local or non-routable network
# If you're using one of these local networks, remove it from the list below
-A INPUT -i eth0 -s 192.168.0.0/16 -j LOG --log-prefix 'IP DROP SPOOF C: '
-A INPUT -i eth0 -s 240.0.0.0/5 -j LOG --log-prefix 'IP DROP SPOOF E: '
-A INPUT -i eth0 -d 127.0.0.0/8 -j LOG --log-prefix 'IP DROP LOOPBACK: '
# Accept any established connections
-A RH-Firewall-1-INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
# Accept ssh traffic. Restrict this to known ips if possible.
-A RH-Firewall-1-INPUT -m state --state NEW -m tcp -p tcp --dport 22 -j ACCEPT
#Log and drop everything else
-A RH-Firewall-1-INPUT -j LOG
-A RH-Firewall-1-INPUT -j DROP
COMMIT
EOF

###### Check for updates ##########
cat << 'EOF' >> /etc/yum.conf
exclude = *.i?86
EOF
yum -y remove *.i?86
yum -y update

echo 'Implementing auditing rules (based on clip rules)'
cat << 'EOF' >/etc/audit/audit.rules
-D
-b 16384
-f 2
-w /bin/login -p x
-w /bin/logout -p x

-a exit,always -F arch=b32 -S chmod -S fchmod -S fchmodat -F auid>=500 -F auid!=4294967295 -k perm_mod
-a exit,always -F arch=b64 -S chmod -S fchmod -S fchmodat -F auid>=500 -F auid!=4294967295 -k perm_mod
-a exit,always -F arch=b32 -S chown -S fchown -S fchownat -S lchown -F auid>=500 -F auid!=4294967295 -k perm_mod
-a exit,always -F arch=b32 -S chown32 -S fchown32 -S lchown32 -F auid>=500 -F auid!=4294967295 -k perm_mod
-a exit,always -F arch=b64 -S chown -S fchown -S fchownat -S lchown -F auid>=500 -F auid!=4294967295 -k perm_mod
-a exit,always -F arch=b32 -S setxattr -S lsetxattr -S fsetxattr -S removexattr -S lremovexattr -S fremovexattr -F auid>=500 -F auid!=4294967295 -k perm_mod

-a exit,always -F arch=b64 -S setxattr -S lsetxattr -S fsetxattr -S removexattr -S lremovexattr -S fremovexattr -F auid>=500 -F auid!=4294967295 -k perm_mod
-a exit,always -F arch=b32 -S mknod -S pipe -S mkdir -S creat -S open -S openat -S truncate -S ftruncate -S truncate64 -S ftruncate64 -F exit=-EACCES -F auid>=500 -F auid!=4294967295 -k access
-a exit,always -F arch=b32 -S mknod -S pipe -S mkdir -S creat -S open -S openat -S truncate -S ftruncate -S truncate64 -S ftruncate64 -F exit=-EPERM -F auid>=500 -F auid!=4294967295 -k access
-a exit,always -F arch=b64 -S mknod -S pipe -S mkdir -S creat -S open -S openat -S truncate -S ftruncate -F exit=-EACCES -F auid>=500 -F auid!=4294967295 -k access
-a exit,always -F arch=b64 -S mknod -S pipe -S mkdir -S creat -S open -S openat -S truncate -S ftruncate -F exit=-EPERM -F auid>=500 -F auid!=4294967295 -k access

-w /usr/sbin/pwck -k priv-command
-w /bin/chgrp -k priv-command
-w /usr/bin/newgrp -k priv-command
-w /usr/sbin/groupadd -k priv-command
-w /usr/sbin/groupmod -k priv-command
-w /usr/sbin/groupdel -k priv-command
-w /usr/sbin/useradd -k priv-command
-w /usr/sbin/userdel -k priv-command
-w /usr/sbin/usermod -k priv-command
-w /usr/bin/chage -k priv-command
-w /usr/bin/setfacl -k priv-command
-w /usr/bin/chacll -k priv-command

-a exit,always -F arch=b32 -S unlink -S unlinkat -S rename -S renameat -S rmdir -F auid>=500 -F auid!=4294967295 -k delete
-a exit,always -F arch=b64 -S unlink -S unlinkat -S rename -S renameat -S rmdir -F auid>=500 -F auid!=4294967295 -k delete

-a exit,always -F arch=b32 -S link -S symlink -S acct -F auid>=500 -F auid!=4294967295 -k created
-a exit,always -F arch=b64 -S link -S symlink -S acct -F auid>=500 -F auid!=4294967295 -k created

-w /var/log/audit/audit.log -k admin-actions
-w /var/log/audit/audit[1-4].log -k admin-actions
-w /var/log/messages -k admin-actions
-w /var/log/lastlog -k admin-actions
-w /var/log/faillog -k admin-actions

-w /etc/group -p wa -k identity
-w /etc/passwd -p wa -k identity
-w /etc/gshadow -p wa -k identity
-w /etc/shadow -p wa -k identity
-w /etc/security/opasswd -p wa -k identity

-a always,exit -F arch=b32 -S sethostname -S setdomainname -k system-locale
-a always,exit -F arch=b64 -S sethostname -S setdomainname -k system-locale
-w /etc/issue -p wa -k system-locale
-w /etc/issue.net -p wa -k system-locale
-w /etc/hosts -p wa -k system-locale
-w /etc/sysconfig/network -p wa -k system-locale

-w /etc/ld.so.conf -p wa -k admin-actions
-w /etc/ld.so.conf.d -p wa -k admin-actions
-w /etc/ssh/sshd_config -k admin-actions
-w /etc/login.defs -k admin-actions
-w /etc/rc.d/init.d -k admin-actions
-w /etc/inittab -p wa -k admin-actions
-w /var/run/utmp -k admin-actions
-w /var/run/wtmp -k admin-actions

-a always,exit -F arch=b32 -S adjtimex -S settimeofday -S stime -k time-change
-a always,exit -F arch=b64 -S adjtimex -S settimeofday -k time-change
-a always,exit -F arch=b32 -S clock_settime -k time-change
-a always,exit -F arch=b64 -S clock_settime -k time-change
-w /etc/localtime -p wa -k time-change

-a always,exit -F path=/bin/ping -F perm=x -F auid>=500 -F auid!=4294967295 -k privileged
-a always,exit -F path=/bin/umount -F perm=x -F auid>=500 -F auid!=4294967295 -k privileged
-a always,exit -F path=/bin/mount -F perm=x -F auid>=500 -F auid!=4294967295 -k privileged
-a always,exit -F path=/bin/ping6 -F perm=x -F auid>=500 -F auid!=4294967295 -k privileged
-a always,exit -F path=/bin/su -F perm=x -F auid>=500 -F auid!=4294967295 -k privileged

-a exit,always -F arch=b32 -S mount -S chroot -S umount2 -S umount -S kill -F auid!=4294967295 -k admin-actions
-a exit,always -F arch=b64 -S mount -S chroot -S umount2 -S kill -F auid!=4294967295 -k admin-actions

-a exit,always -F arch=b32 -S reboot -S sched_setparam -S sched_setscheduler -S setdomainname -S setrlimit -S swapon -k admin-actions
-a exit,always -F arch=b64 -S reboot -S sched_setparam -S sched_setscheduler -S setdomainname -S setrlimit -S swapon -k admin-actions
-w /etc/audit/auditd.conf -p wa -k security-actions
-w /etc/audit/audit.rules -p wa -k security-actions
-w /etc/selinux/config -p wa -k security-actions
-w /etc/sudoers -p wa -k security-actions

-a always,exit -F arch=b32 -S ptrace -k tracing
-a always,exit -F arch=b64 -S ptrace -k tracing
-a always,exit -F arch=b32 -S personality -k bypass
-a always,exit -F arch=b64 -S personality -k bypass

-w /sbin/insmod -p x -k modules
-w /sbin/rmmod -p x -k modules
-w /sbin/modprobe -p x -k modules

-w /etc/pam.d -k admin-actions
-a exit,always -F arch=b32 -S init_module -S delete_module -k security-actions
-a exit,always -F arch=b64 -S init_module -S delete_module -k security-actions
-w /bin/su
-e 2
EOF

cat << 'EOF' >/etc/audit/auditd.conf
log_file = /var/log/audit/audit.log
log_format = RAW
log_group = root
priority_boost = 3
flush = INCREMENTAL
freq = 20
name_format = NONE
max_log_file = 0
max_log_file_action = IGNORE
space_left = 75
space_left_action = SYSLOG
admin_space_left = 50
admin_space_left_action = HALT
disk_full_action = HALT
disk_error_action = HALT
EOF

cat << 'EOF' >/etc/logrotate.d/auditd
/var/log/audit/audit.log {
&nbsp;&nbsp;&nbsp;&nbsp;daily
&nbsp;&nbsp;&nbsp;&nbsp;notifempty
&nbsp;&nbsp;&nbsp;&nbsp;missingok
&nbsp;&nbsp;&nbsp;&nbsp;compress
&nbsp;&nbsp;&nbsp;&nbsp;rotate 90
&nbsp;&nbsp;&nbsp;&nbsp;postrotate
&nbsp;&nbsp;&nbsp;&nbsp;/sbin/service auditd restart 2> /dev/null > /dev/null || true
&nbsp;&nbsp;&nbsp;&nbsp;endscript
}
EOF

## (GEN001280: CAT III) (Previously - G042) The SA will ensure all manual page
## files (i.e.,files in the man and cat directories) have permissions of 644,
## or more restrictive.
find /usr/share/man -type f -not -perm 644 -exec chmod 644 {} \;

## (GEN001780: CAT III) (Previously - G112) The SA will ensure global
## initialization files contain the command mesg ?n.
echo 'mesg n' >>/etc/profile
echo 'mesg n' >>/etc/bashrc

## (GEN001720: CAT II) The SA will ensure global initialization files have
## permissions of 644, or more restrictive.
## (GEN001740: CAT II) The SA will ensure the owner of global initialization
## files is root.
## (GEN001760: CAT II) The SA will ensure the group owner of global
## initialization files is root, sys, bin, other, or the system default.

chown root:root /etc/profile /etc/bashrc /etc/environment
chmod 644 /etc/profile /etc/bashrc /etc/environment

## (GEN001800: CAT II) (Previously - G038) The SA will ensure all
## default/skeleton dot files have permissions of 644, or more restrictive.
find /etc/skel -type f -exec chmod 644 '{}' \;

echo 'report = true' >>/etc/puppet/puppet.conf

## Wireless for a server? Really?
echo 'Getting rid of wireless drivers that may be loaded'
for i in $(find /lib/modules/`uname -r`/kernel/drivers/net/wireless -name '*.ko' -type f) ; do echo blacklist $i >> /etc/modprobe.d/blacklist-wireless ; done

#end raw

$SNIPPET('post_anamon')
$kickstart_done
```

Step 3. Puppetize

Getting puppet installed on CentOS 6.2 is pretty straight forward. I added the puppet repo from puppetlabs and the passenger repo as below:

```
rpm --import http://passenger.stealthymonkeys.com/RPM-GPG-KEY-stealthymonkeys.asc
yum install http://passenger.stealthymonkeys.com/rhel/6/passenger-release.noarch.rpm
```

Then installed puppet, puppet dashboard, and passenger (as an apache module). It should be noted that puppet is very finicky in regards to DNS and the SSL certs. To help ease things, I decided to have cobbler manage the DNS zone, and created a zone template that had the puppet related hosts.
I'm going to try to yank the kickstart stuff back into puppet classes for secstate/openscap.

Note to self: This might be interesting&#8230; self-classifying nodes&#8230;. http://nuknad.com/2011/02/11/self-classifying-puppet-nodes/

Step 4. Secstate

Coming eventually...

Random urls: http://pvrabec.livejournal.com/
