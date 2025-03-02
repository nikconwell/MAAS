# MAAS
Misc stuff around Canonical Metal As A Service

# Notes on setting up MAAS

## Installing MAAS

Fresh Rocky 9 box

### Install SNAP

```bash
dnf install https://dl.fedoraproject.org/pub/epel/epel-release-latest-9.noarch.rpm
/usr/bin/crb enable
dnf upgrade
dnf install snapd
systemctl enable --now snapd.socket
ln -s /var/lib/snapd/snap /snap
```

### Install MAAS using SNAP

```bash
snap install maas
hash -r
```

Go to http://maas.home.arpa:5240/MAAS to do initial GUI stuff. Note in the admin section of the GUI what the API key for the MAAS consumer is. Copy to clipboard.

Upload your SSH public key to MAAS so you can access things using this key later.

#### Confirm command line access.
Enter the copied API key when asked by the command line below.

```bash
maas login admin http://maas.home.arpa:5240/MAAS/
```

## Create Rocky 9 image

MAAS is Ubuntu centric, the GUI has support for CentOS 7 images (now out of date and unsupported) and CentOS 8 (out of date even earlier than CentOS 7!) so you have to make your own images and upload them.

Be on Ubuntu 24.10 to create the packer rocky image. I tried following this install on a Rocky 9 (or was it 8?) system but the ovmf package at that level doesn't seem to have the UEFI code that this packer expects. I suspect the Rocky/RHEL ovmf is a bit out of date. Ubuntu has a more modern version and since this is a MAAS guide which expects all to be Ubuntu it seems easier to just do it from an Ubuntu VM.

### Install packer stuff

#### Install packer

```bash
curl -fsSL https://apt.releases.hashicorp.com/gpg | sudo apt-key add -
sudo apt-add-repository "deb [arch=amd64] https://apt.releases.hashicorp.com $(lsb_release -cs) main"
sudo apt-get update && sudo apt-get install packer
hash -r
packer --version
```

#### Get the packer-maas repo

```bash
git clone https://github.com/canonical/packer-maas.git
cd packer-maas/rocky9
```

#### Install needed pre-reqs

```bash
apt -y install make
apt -y install libnbd-bin
apt -y install nbdkit
apt -y install fuse2fs
apt -y install qemu-system
apt -y install ovmf
apt -y install cloud-image-utils
apt -y install parted
```

#### Update the kickstart config timezone because we are not UTC neanderthals.
(in packer-maas/rocky9 directory)

```bash
emacs http/rocky9.ks.pkrtpl.hcl
Update the timezone line:
timezone America/New_York --utc
```

### Finally, create the rocky 9 image.
(in packer-maas/rocky9 directory)

```bash
make
```

This takes a while to run (boots a rocky 9 ISO, does a kickstart, all inside a VM), about 15 minutes for me, and then creates a 946M rocky9.tar.gz file which you can copy to the MAAS system for upload.

## Upload the Rocky 9 image to MAAS

The web GUI for MAAS doesn't seem to have image upload options so we have to do this via the command line.

Since we installed our MAAS with SNAP, the maas command can only access files in /home. This is a security feature of SNAP. If you put your rocky9.tar.gz file in some other directory and try to upload it with the maas command, it will tell you "no such file or directory" when trying to access the file. This was very frustrating and completely non-obvious. So just put the rocky9.tar.gz in /home.

If you're not already logged in, enter the admin key (from the MAAS GUI Admin=>API Keys) to get admin access. (The "admin" is just an arbitrary profile token reference for subsequent maas commands...)

```bash
cd /home
maas login admin http://maas.home.arpa:5240/MAAS/
maas admin boot-resources create name='custom/Rocky9' title='Rocky 9 Custom' architecture='amd64/generic' filetype='tgz' base_image='rhel/9' content@=rocky9.tar.gz
```

This will upload the image, and in the web GUI under CONFIGURATION => Images, you will now see Rocky 9 Custom.

## Build a Rocky 9 system

### Create a VM in ProxMox
* OS tab, do not use any media
* System tab, for BIOS choose OVMF (UEFI), choose some storage for your EFI, and UNCHECK Pre-Enroll keys otherwise this will turn on hash validation for the EFI which will fail later on boot as the key is not officially registered with EFI or something like that IDK.
* CPU tab give it a few cores and make the Type be 'host'

Once you have created the VM, take note of the network card MACADDR so you can identify this in MAAS later.

### DHCP / DNS for the new box.

Make sure your DHCP has the TFTP settings. I'm using ISC DHCPD because I haven't swapped to whatever is better (pfsense box?) yet.

```text
next-server 192.168.2.62;
###filename "lpxelinux.0";
filename "bootx64.efi";
```

If you want BIOS you can do the lpxelinux.0 line, but I figured why not go with UEFI since I wanted things to be a bit more challenging and confusing.

NOTE! At this point I am using just regular DHCP for whatever this new VM is. I had initially put the MACADDR and given a static lease for an IP in my DHCP server but since I have an external DHCP (MAAS really wants you to use its own DHCP) this ended up confusing MAAS which thought I had no free DHCP addresses to assign, or if I manually assigned the IP in MAAS it then discovered a 'different' new machine with the same IP/MACADDR and then refused to let me use that on the machine I was trying to install in MAAS. So, long story short, reserve your final IP for your box in DNS (but not DHCP static) and then use a dynamic DHCP address for the box initially.

### Prepare and deploy the machine in MAAS

If you power the box on in ProxMox, it will DHCP and MAAS will notice the box (snooping the network, although it may also see the PXE) and then you can configure the machine. Give the machine network a static IP of whatever you defined in DNS, and go through the Deploy process choosing the custom image of Rocky 9.

The 3.5.3 MAAS I am running is a bit buggy in that if you make config changes, occasionally you have to F5 reload pages (even though you navigated to new pages as part of the config setup) to get things to actually work. I think there may be a bit too much caching on the browser side for some things. This can be a bit frustrating as the GUI looks like it has the new options, but then you get strange errors when trying to use them, but if you F5 refresh things look the same, but you don't get errors.

If something goes wrong during the Deploy you can log in to the box on the ProxMox console using the rocky9 / rocky9 user/password (which is defined in the kickstart when you did the packer config) and poke around.

After the box is deployed, you can ssh in (using your ssh key that you uploaded to MAAS as you set things up) as cloud-user@your.rocky9.system and from then do sudo and do other stuff.
