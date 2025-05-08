# Create a mulle-clang redistributable from binaries using CPack

These scripts wraps mulle-clang files in `/opt` into an install package.
As a bonus a symbolic link is also generated and packaged.


## Create a fresh debian VM (if needed)

* currently 4GB of free file space is needed for a non-debug build
* for debug build multiply by 5
* give it as many CPUs as you can spare
* needs 16GB RAM (sic) at least
* Consider if VM should not have swap space, prefer to crash and reconfigure


Here we are installing into a fresh "buster" VM  of the same name:

``` bash
scp ~/.ssh/id_rsa_vm.pub buster:
ssh buster
mkdir .ssh
mv id_rsa_vm.pub .ssh/authorized_keys
chmod 400 .ssh/authorized_keys
chmod 700 .ssh
```

Add `buster` to `/etc/hosts` on host.
Add `buster` to `~/.ssh/config` on host.

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

On debian, install **git** and get **sudo** happening

``` bash
su
apt-get install git sudo
/sbin/usermod -aG sudo <loginname> # or your login
sudo /sbin/visudo
# Allow members of group sudo to execute any command
%sudo   ALL=(ALL:ALL) NOPASSWD: ALL
# log off now, so sudo group change takes effect
```

Install **cmake** and such things:

``` bash
wget 'https://raw.githubusercontent.com/mulle-cc/mulle-clang-project/mulle/14.0.6/clang/bin/install-prerequisites'
chmod 755 install-prerequisites
./install-prerequisites --no-lldb
```

Install **rpm** to build rpm packages:

``` bash
sudo apt-get install rpm # build-essential
```

## One script does all

On the VM Host (!) run

``` bash
VERSION=17.0.6.0 RC= ./create-deb "bullseye"
```


## Semi-manual Usage

On the VM guest run

``` bash
VERSION=17.0.6.0 package-build
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
VERSION="17.0.6.0"
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

