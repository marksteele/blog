---
title: "meshed vpn using tinc"
date: "2015-03-19"
slug: "2015/03/19/meshed-vpn-using-tinc"
categories:
  - linux
  - networking
  - security
cover: "/images/tinclogo.png"
---

Tinc is a neat little VPN daemon that I've recently come across. It is surprisingly simple to configure yet powerful. In this post, I'll show you how to setup a meshed VPN between four nodes with one of the servers acting as a DHCP server.

<!-- more -->

In this fictitious scenario, let's assume the following nodes:

* dev is a CentOS cloud server with a fixed public IP address, we'll designate this one as our DHCP server
* dev2 is another CentOS cloud server with another fixed public IP address
* darkstar is a road warrior (Mac OSX laptop) with roaming NAT'ed internet access. We'll have this guy use DHCP to get a VPN ip
* storage is a CentOS NAS server running Linux NAT'ed behind a cable modem. Static VPN IP

We'll call our imaginary VPN network 'darknet'. We'll have the two cloud servers mesh to each other, and the other boxes will be able to connect to either dev or dev2 to join the mesh.

On dev, dev2, and storage:

```bash
## Assuming you've got EPEL setup already
yum install tinc
mkdir -p /etc/tinc/darknet/hosts
```

Tinc configuration for dev (make sure you edit this to have the correct public IP):

```bash
cat <<'EOF' >/etc/tinc/darknet/tinc.conf
Name = dev
AddressFamily = ipv4
Interface = tun0
ConnectTo = dev2
Mode = switch
PMTUDiscovery = yes
Compression = 10
'EOF'

cat <<'EOF' >/etc/tinc/darknet/hosts/dev
Address = 1.2.3.4
'EOF'

tincd -n darknet -K8192

yum install dhcp
cat <<'EOF' >/etc/dhcp/dhcpd.conf
subnet 192.168.101.0 netmask 255.255.255.0 {
    range 192.168.101.100 192.168.101.254;
}
'EOF'

systemctl start dhcpd
systemctl enable dhcpd

cat <<'EOF' >/etc/tinc/darknet/tinc-up
#!/bin/sh
ifconfig $INTERFACE 192.168.101.1 netmask 255.255.255.0
systemctl restart dhcpd
'EOF'

chmod +x /etc/tinc/darknet/tinc-up

cat <<'EOF' >/etc/tinc/darknet/tinc-down
#!/bin/sh
ifconfig $INTERFACE down
'EOF'

chmod +x /etc/tinc/darknet/tinc-down

cat <<'EOF' >/etc/systemd/system/tincd.service
[Unit] 
Description=tinc vpn 
After=network.target 

[Service] 
Type=forking 
ExecStart=/usr/sbin/tincd -n darknet

[Install] 
WantedBy=multi-user.target
'EOF'

systemctl enable tincd

```

Next, the configuration for dev2 (make sure you edit this to have the correct public IP):

```bash
cat <<'EOF' >/etc/tinc/darknet/tinc.conf
Name = dev2
AddressFamily = ipv4
Interface = tun0
ConnectTo = dev
Mode = switch
PMTUDiscovery = yes
Compression = 10
'EOF'

cat <<'EOF' >/etc/tinc/darknet/hosts/dev2
Address = 2.3.4.5
'EOF'

tincd -n darknet -K8192

cat <<'EOF' >/etc/tinc/darknet/tinc-up
#!/bin/sh
ifconfig $INTERFACE 192.168.101.2 netmask 255.255.255.0
'EOF'

chmod +x /etc/tinc/darknet/tinc-up

cat <<'EOF' >/etc/tinc/darknet/tinc-down
#!/bin/sh
ifconfig $INTERFACE down
'EOF'

chmod +x /etc/tinc/darknet/tinc-down

cat <<'EOF' >/etc/systemd/system/tincd.service
[Unit] 
Description=tinc vpn 
After=network.target 

[Service] 
Type=forking 
ExecStart=/usr/sbin/tincd -n darknet

[Install] 
WantedBy=multi-user.target
'EOF'

systemctl enable tincd

```

The configuration for storage:

```bash
cat <<'EOF' >/etc/tinc/darknet/tinc.conf
Name = storage
AddressFamily = ipv4
Interface = tun0
ConnectTo = dev
ConnectTo = dev2
Mode = switch
PMTUDiscovery = yes
Compression = 10
LocalDiscovery = yes
'EOF'

tincd -n darknet -K8192

cat <<'EOF' >/etc/tinc/darknet/tinc-up
#!/bin/sh
ifconfig $INTERFACE 192.168.101.3 netmask 255.255.255.0
'EOF'

chmod +x /etc/tinc/darknet/tinc-up

cat <<'EOF' >/etc/tinc/darknet/tinc-down
#!/bin/sh
ifconfig $INTERFACE down
'EOF'

chmod +x /etc/tinc/darknet/tinc-down

cat <<'EOF' >/etc/systemd/system/tincd.service
[Unit] 
Description=tinc vpn 
After=network.target 

[Service] 
Type=forking 
ExecStart=/usr/sbin/tincd -n darknet

[Install] 
WantedBy=multi-user.target
'EOF'

systemctl enable tincd

```

Let's get darkstar also setup. Mac OSX, I hate you. Assuming you've got brew installed. Everything as root, sudo is for chumps and carebears:

```bash
brew install tinc
brew install tuntap
cp -pR /usr/local/Cellar/tuntap/20111101/Library/Extensions/tap.kext /Library/Extensions/
cp -pR /usr/local/Cellar/tuntap/20111101/Library/Extensions/tun.kext /Library/Extensions/
chown -R root:wheel /Library/Extensions/tap.kext
chown -R root:wheel /Library/Extensions/tun.kext
touch /Library/Extensions/
cp -pR /usr/local/Cellar/tuntap/20111101/tap /Library/StartupItems/
chown -R root:wheel /Library/StartupItems/tap
cp -pR /usr/local/Cellar/tuntap/20111101/tun /Library/StartupItems/
chown -R root:wheel /Library/StartupItems/tun
mkdir -p /usr/local/etc/tinc/darknet/hosts

cat <<'EOF' >/usr/local/etc/tinc/darknet/tinc.conf
Name = darkstar
AddressFamily = ipv4
Device=/dev/tap0
ConnectTo = dev
ConnectTo = dev2
Mode = switch
LocalDiscovery = yes
Compression = 10
PMTUDiscovery = yes
PrivateKeyFile = /dev/stdin
'EOF'

tincd -n darknet -K8192

### For our road warriors, we shouldn't really 
### leave the private keys unencrypted. Let's fix that...
openssl rsa -des -in /usr/local/etc/tinc/darknet/rsa_key.priv -out /usr/local/etc/tinc/darknet/rsa_key_enc.priv

## After you've put in your password, let's delete the file.
rm /usr/local/etc/tinc/darknet/rsa_key.priv

## Make a little script that'll prompt for password and start tincd
cat <<'EOF' >/root/start_tinc.sh
#!/bin/bash

TITLE="TINC PASS PROMPT";
TEXT="Tinc private key password:";
IFS=$(printf "\n");
CODE=("on GetCurrentApp()");
CODE=(${CODE[*]} "tell application \"System Events\" to get short name of first process whose frontmost is true");
CODE=(${CODE[*]} "end GetCurrentApp");
CODE=(${CODE[*]} "tell application GetCurrentApp()");
CODE=(${CODE[*]} "activate");
CODE=(${CODE[*]} "display dialog \"${@:-$TEXT}\" default answer \"\" with title \"${TITLE}\" with icon caution with hidden answer");
CODE=(${CODE[*]} "text returned of result");
CODE=(${CODE[*]} "end tell");
SCRIPT="/usr/bin/osascript"
for LINE in ${CODE[*]}; do
    SCRIPT="${SCRIPT} -e $(printf "%q" "${LINE}")";
done;
PASSPHRASE=`eval "${SCRIPT}"`;

echo $PASSPHRASE | openssl rsa -passin stdin -text -in /usr/local/etc/tinc/darknet/rsa_key_enc.priv | /usr/local/sbin/tincd -n darknet -D &
'EOF'

chmod +x /root/start_tinc.sh

cat <<'EOF' >/usr/local/etc/tinc/darknet/tinc-up
#!/bin/sh
ipconfig set tap0 DHCP
'EOF'

chmod +x /etc/tinc/darknet/tinc-up

cat <<'EOF' >/etc/tinc/darknet/tinc-down
#!/bin/sh
ifconfig $INTERFACE down
'EOF'

chmod +x /etc/tinc/darknet/tinc-down

```

Reboot that Mac box. Just because.

Almost done now. Time to exchange the public keys around the mesh. Do something like this. On dev:

```bash
scp /etc/tinc/darknet/hosts/dev dev2:/etc/tinc/darknet/hosts/
scp dev2:/etc/tinc/darknet/hosts/dev2 /etc/tinc/darknet/hosts/
```

On storage:

```bash
scp dev:/etc/tinc/darknet/hosts/* /etc/tinc/darknet/hosts/
scp /etc/tinc/darknet/hosts/storage dev:/etc/tinc/darknet/hosts/
scp /etc/tinc/darknet/hosts/storage dev2:/etc/tinc/darknet/hosts/
```

On darkstar:

```bash
scp dev:/etc/tinc/darknet/hosts/* /usr/local/etc/tinc/darknet/hosts/
scp /usr/local/etc/tinc/darknet/hosts/darkstar dev:/etc/tinc/darknet/hosts/
scp /usr/local/etc/tinc/darknet/hosts/darkstar dev2:/etc/tinc/darknet/hosts/
scp /usr/local/etc/tinc/darknet/hosts/darkstar storage:/etc/tinc/darknet/hosts/
```

Finally let's start em up. On the linux boxes:

```bash

systemctl start tincd
systemctl enable tincd

```

On darkstar

```bash
/root/start_tinc.sh
```

All done. You now have a meshed VPN in which hosts should be able to find the shortest direct path to each other even if several clients are behind the same NAT.
