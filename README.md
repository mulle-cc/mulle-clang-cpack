# Create a mulle-clang redistributable from binaries using CPack

These scripts wraps mulle-clang files in `/opt` into an install package.
As a bonus a symbolic link is also generated and packaged.


## Create a fresh debian VM (if needed)

* currently 4GB of **free** file space is needed for a non-debug build
* for debug build multiply by 5
* give it as many CPUs as you can spare
* needs 16GB RAM (sic) at least
* Consider if VM should not have swap space, prefer to crash and reconfigure

*SEE BOTTOM OF TEXT FOR SOME virsh TIPS*


> #### Or use an aws instance
>
> Do not skimp on CPU power. `c7g.8xlarge` or better is what you want. Remember
> the build will (until the link) scale almost perfectly, so it can be even
> cheaper to use bigger iron (probably not though because of setup and CPU
> time).
>
> 12GB for compile, 16GB for link
> 24GB space for disk (assuming none taken by OS install)
>
> Checkout [https://nat.prose.sh/p-cb240a1d-d580-4485-85f8-0aed20792d4e](Install AWS CLI in >distrobox), for some steps how to get going with aws. But basically you are
> on your own with respect to this file, but AI will guide you.
> Once you got an EC2 instance up and running and can `ssh` into it. And
> install prerequisites:

> ``` bash
> sudo yum install git clang cmake make ninja-build
> ```
> You can continue now with [One script does all](#One-script-does-all).
>

## Prerequisites

* sudo
* git

On debian, install **git**, **wget** and get **sudo** happening

``` bash
su
apt-get install git sudo wget
/sbin/usermod -aG sudo <loginname> # or your login
sudo /sbin/visudo
# Allow members of group sudo to execute any command
%sudo   ALL=(ALL:ALL) NOPASSWD: ALL
# log off now, so sudo group change takes effect
```

Install **cmake** and such things:

``` bash
wget wget --progress=dot:giga -q --show-progress 'https://raw.githubusercontent.com/mulle-cc/mulle-clang-project/refs/heads/mulle/21.1.8/clang/bin/install-prerequisites'
chmod 755 install-prerequisites
./install-prerequisites --no-lldb
```

Install **rpm** to build rpm packages:

``` bash
sudo apt-get install rpm # build-essential
```

Install **clang** as gcc has trouble compiling llvm, it should be picked up by default:

``` bash
sudo apt-get install clang # build-essential
```


## One script does all

On the VM Host (!) run

``` bash
VERSION=21.1.8.1 RC= ./create-deb "bullseye"
```


## Semi-manual Usage

On the VM guest run

``` bash
VERSION=21.1.8.1 package-build
```


## Manual Usage

### Unix

#### Get git happening and clone cpack-mulle-clang:

``` bash
sudo apt-get install git sudo
git clone https://github.com/mulle-cc/mulle-clang-cpack.git
```

#### Build mulle-clang into a local opt folder:

Set `VERSION` appropriately:

``` bash
VERSION="21.1.8.1"
RC="" # e.g. -RC1
mkdir mono
cd mono
wget -O - "https://github.com/mulle-cc/mulle-clang-project/archive/${VERSION}${RC}.tar.gz" | tar xfz -
mv "mulle-clang-project-${VERSION}${RC}" mulle-clang-project
mkdir opt/mulle-clang-project
sudo ln -s "$PWD/opt/mulle-clang-project" "/opt/mulle-clang-project"
```

####  Build normally

``` bash
PREFIX="/opt" NAME="${VERSION}" ./mulle-clang-project/clang/bin/cmake-ninja.linux
```


#### Create .deb package and upload:

``` bash
cp ../cpack-mulle-clang/* .
chmod 755 generate-package
./generate-package
```

### macOS - brew

``` bash
cp mulle-clang-project.rb /usr/local/Homebrew/Library/Taps/mulle-objc/homebrew-software/Formula/
brew uninstall mulle-objc/software/mulle-clang-project
brew install --formula --build-bottle mulle-clang-project.rb
brew bottle mulle-objc/software/mulle-clang-project
```

---


### In virsh

Install a new VM "normally".

### Afterwards

Get a free IP if you have setup DHCP like I have:

``` bash
$ virsh net-dumpxml default
<network>
  <name>default</name>
  <uuid>07d4c99e-e823-4fe8-b0b7-dd1c5b23d2e9</uuid>
  <forward mode='nat'>
    <nat>
      <port start='1024' end='65535'/>
    </nat>
  </forward>
  <bridge name='virbr0' stp='on' delay='0'/>
  <mac address='52:54:00:18:05:cb'/>
  <ip address='192.168.122.1' netmask='255.255.255.0'>
    <dhcp>
      <range start='192.168.122.2' end='192.168.122.254'/>
      <host mac='00:50:56:01:1f:a5' name='buster' ip='192.168.122.2'/>
      <host mac='00:50:56:01:3f:bf' name='bullseye' ip='192.168.122.3'/>
      <host mac='52:54:00:92:d4:7b' name='catalina' ip='192.168.122.4'/>
      <host mac='00:50:56:01:7f:bf' name='bookworm' ip='192.168.122.5'/>
      <host mac='52:54:00:d2:c5:98' name='alaaf-boot' ip='192.168.122.6'/>
      <host mac='32:a4:ef:8a:9e:38' name='alaaf' ip='192.168.122.66'/>
      <host mac='52:54:00:be:0b:9c' name='trixie' ip='192.168.122.7'/>
      <host mac='00:50:56:01:df:bf' name='free5' ip='192.168.122.8'/>
      <host mac='00:50:56:01:ff:bf' name='free6' ip='192.168.122.9'/>
      <host mac='00:50:56:02:1f:bf' name='free7' ip='192.168.122.10'/>
      <host mac='00:50:56:02:3f:bf' name='free8' ip='192.168.122.11'/>
      <host mac='00:50:56:02:5f:bf' name='focal' ip='192.168.122.12'/>
      <host mac='00:50:56:02:7f:bf' name='free9' ip='192.168.122.13'/>
      <host mac='00:50:56:02:9f:bf' name='free10' ip='192.168.122.14'/>
      <host mac='00:50:56:02:bf:bf' name='free11' ip='192.168.122.15'/>
      <host mac='00:50:56:02:df:bf' name='free12' ip='192.168.122.16'/>
      <host mac='00:50:56:02:ff:bf' name='free13' ip='192.168.122.17'/>
    </dhcp>
  </ip>
</network>
```


Pick a free one and `virsh net-edit` to desired hostname, put the VM MAC in
there.

``` bash
$ virsh net-destroy default
$ virsh net-start default
```

Put name with chosen IP into `/etc/hosts`.


Here we are setting up a fresh "buster" VM  of the same name:

``` bash
scp ~/.ssh/id_rsa_vm.pub buster:
ssh buster
```

and then

``` bash
mkdir .ssh
mv id_rsa_vm.pub .ssh/authorized_keys
chmod 400 .ssh/authorized_keys
chmod 700 .ssh
```

Add `buster` to `/etc/hosts` on host.
Add `buster` to `~/.ssh/config` on host.


Now you should be able to say `ssh buster` and you are in.
