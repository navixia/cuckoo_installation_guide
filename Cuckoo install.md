#### Cuckoo install guide 

# Install Ubuntu

# Enable VT-X
Virtual Hardware tab -> CPU -> Enable Expose hardware assisted virtualization to the guest OS.

# Configure network

# Update and upgrade
sudo apt update
sudo apt upgrade -y

# Enable ssh
sudo apt install openssh-server -y

# Libs, Deps and Virtualbox stuff
sudo apt-get install python python-pip python-dev libffi-dev libssl-dev -y
sudo apt-get install python-virtualenv python-setuptools -y
sudo apt-get install libjpeg-dev zlib1g-dev swig -y
sudo apt-get install mongodb -y
sudo apt-get install postgresql libpq-dev -y
sudo apt install virtualbox -y
sudo apt-get install tcpdump apparmor-utils -y

# Add cuckooo user
sudo adduser --gecos "" cuckoo
sudo usermod -a -G sudo cuckoo

# Create pcap group and add cuckoo to that group, add user to pcap group
sudo groupadd pcap
sudo usermod -a -G pcap cuckoo
sudo chgrp pcap /usr/sbin/tcpdump
sudo setcap cap_net_raw,cap_net_admin=eip /usr/sbin/tcpdump

# Check: list capabilities for tcpdump
getcap /usr/sbin/tcpdump

# Disable AppArmor for tcpdump
sudo aa-disable /usr/sbin/tcpdump

# Install python lib m2crypt
sudo pip install m2crypto

# Allow cuckoo user to run VirtualBox
sudo usermod -a -G vboxusers cuckoo

# Jump into cuckoo user
sudo su cuckoo

# Copy the following bash script into /opt/cuckoo-setup-virtualenv.sh
# This script create a virtual environment
-----------------------------------------
#!/usr/bin/env bash

# NOTES: Run this script as: sudo -u <USERNAME> cuckoo-setup-virtualenv.sh

# install virtualenv
sudo apt-get update && sudo apt-get -y install virtualenv

# install virtualenvwrapper
sudo apt-get -y install virtualenvwrapper

echo "source /usr/share/virtualenvwrapper/virtualenvwrapper.sh" >> ~/.bashrc

# install pip for python3
sudo apt-get -y install python3-pip

# turn on bash auto-complete for pip
pip3 completion --bash >> ~/.bashrc

# avoid installing with root
pip3 install --user virtualenvwrapper

echo "export VIRTUALENVWRAPPER_PYTHON=/usr/bin/python3" >> ~/.bashrc

echo "source ~/.local/bin/virtualenvwrapper.sh" >> ~/.bashrc

export WORKON_HOME=~/.virtualenvs

echo "export WORKON_HOME=~/.virtualenvs" >> ~/.bashrc

echo "export PIP_VIRTUALENV_BASE=~/.virtualenvs" >> ~/.bashrc 

-----------------------------------------

# Make script executable
sudo chmod +x /opt/cuckoo-setup-virtualenv.sh 

# Run script
sudo -u cuckoo /opt/cuckoo-setup-virtualenv.sh

# Reload bash env
source ~/.bashrc

# Create virtual env named cuckoo-test
mkvirtualenv -p python2.7 cuckoo-test

# Check that you are in the virtual env (prompt should be (cuckoo-test) cuckoo@...)

# Update setup tools and install cuckoo
pip install -U pip setuptools
pip install -U cuckoo


### Setup virtual machine ###

# Download Win7 iso, mount it
sudo wget https://cuckoo.sh/win7ultimate.iso
sudo mkdir /mnt/win7
sudo chown cuckoo:cuckoo /mnt/win7/
sudo mount -o ro,loop win7ultimate.iso /mnt/win7

# Install Vmcloak
pip install -U vmcloak

# Create virtual network
vmcloak-vboxnet0

# Create VM
vmcloak init --verbose --win7x64 win7x64base --cpus 2 --ramsize 2048

# Clone VM, to be able to create a snapshot
vmcloak clone win7x64base win7x64cuckoo

# List all softwares availables
vmcloak list deps

# I.E: install IE11
vmcloak install win7x64cuckoo ie11

# Create snapshot, (keep only 1, internal ip)
vmcloak snapshot --count 1 win7x64cuckoo 192.168.56.101

vmcloak list vms

### Start cuckoo ###

# Set Iptable -> vm to access internet, add the following to 
net.ipv4.conf.vboxnet0.forwarding=1
net.ipv4.conf.[INTERFACE_NAME_TO_CHANGE].forwarding=1

# Traffic
sudo iptables -t nat -A POSTROUTING -o [INTERFACE_NAME_TO_CHANGE] -s 192.168.56.0/24 -j MASQUERADE
sudo iptables -P FORWARD DROP
sudo iptables -A FORWARD -m state --state RELATED,ESTABLISHED -j ACCEPT
sudo iptables -A FORWARD -s 192.168.56.0/24 -j ACCEPT

# Init cuckoo
cuckoo init
cuckoo community

# Route traffic in cuckoo
cuckoo rooter --sudo --group cuckoo

# (Now in a new session)

# List vm and add them to cuckoo conf
while read -r vm ip; do cuckoo machine --add $vm $ip; done < <(vmcloak list vms)

# Edit  /home/cuckoo/.cuckoo/conf/virtualbox.conf 
# Remove cuckoo from machines=
# it should be machines= 192.168...
# Remove the full block [cuckoo]

# Editing /home/cuckoo/.cuckoo/conf/routing.conf
internet = [INTERFACE_NAME_TO_CHANGE]

# Editing /home/cuckoo/.cuckoo/conf/reporting.conf
[mongodb]
enabled = yes

# Run cuckoo
cuckoo

# you should see some logs into the cuckoo router windows

# Set the web
cuckoo web --host [IP_HOST] --port 8080

# Tips: To re-enter in a virtual env
workon cuckoo-test