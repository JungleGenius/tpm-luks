This is still work in progress, but everything has been tested, except *D. Trusted boot* and PCR identification...

#Storing your LUKS key in TPM NVRAM for RHEL7, and seal them with PCR using TrustedGRUB2

This file describe a way to use TPM NVRAM for storing LUKS keys, with or without a password, and eventually use the TrustedGRUB2 boot loader to allow sealing the TPM NVRAM with PCR.

Some important notes:
* tpm-luks does not support the use of the same TPM NVRAM index for several LUKS devices, so you must use different numbers in /etc/tpm-luks.conf
* it is recommended to use the same TPM NVRAM password for all devices, if using one, otherwise it seems that dracut always tries previous entered password before asking for a new one (second NVRAM will be tried with the same password as the first one), which might result in locking the TPM if done too many times
* TrustedGRUB2 cannot be installed on EFI systems, so you must install RHEL7 without EFI support (see *E. Notes*)
* TPM should be automatically enabled on RHEL7, as it is directly built into the kernel. You can simplify verify by looking for `/dev/tpm0`.

This talks about these packages and how to install and configure them:

* [trousers]: allows to read and write the TPM
* [tpm-tools]: a utility that ease the use of the TPM
* [tpm-luks]: a dracut extension that reads the TPM NVRAM to get the key to use by LUKS
* [TrustedGRUB2]: a secure boot loader that fills PCR based on boot configuration

Unfortunatly, it seems that trousers and/or tpm-tools provided on the RHEL7 cdrom is not working, at least on my test platform, so you'll need to compile and install all these packages by yourself.

If you want to install on production servers, you will also probably want to build your own RPM packages.

The next sections explains how to do all this:

* A. Compiling and installing
* B. Building RPM packages
* C. Usage
* D. Secure boot
* E. Notes
* F. Rescue

##A. Compiling and installing

This section explains how to compile and install from source code.
	
###1. Install trousers in /usr/local

```bash
wget http://sourceforge.net/projects/trousers/files/trousers/0.3.13/trousers-0.3.13.tar.gz
tar zxf trousers-0.3.13.tar.gz
cd trousers-0.3.13
yum install automake autoconf pkgconfig libtool openssl-devel glibc-devel
export PKG_CONFIG_PATH=/usr/lib64/pkgconfig
sh ./bootstrap.sh
CFLAGS="-L/usr/lib64 -L/opt/gnome/lib64" LDFLAGS="-L/usr/lib64 -L/opt/gnome/lib64" ./configure --libdir="/usr/local/lib64"
make
make install
cd ..
```
	
###2. Install tpm-tools in /usr/local

```bash
wget ftp://rpmfind.net/linux/centos/7.0.1406/os/x86_64/Packages/opencryptoki-devel-3.0-11.el7.x86_64.rpm
yum localinstall opencryptoki-devel-3.0-11.el7.x86_64.rpm
wget http://sourceforge.net/projects/trousers/files/tpm-tools/1.3.8/tpm-tools-1.3.8.tar.gz
tar zxf tpm-tools-1.3.8.tar.gz
cd tpm-tools-1.3.8
yum install automake autoconf libtool gettext openssl openssl-devel opencryptoki
./configure
make
make install
cd ..
```
	
###3. Update ldconfig so libs from /usr/local are automatically loaded

```bash
cat <<\EOF > /etc/ld.so.conf.d/tpm-tools.conf
/usr/local/lib
/usr/local/lib64
EOF
ldconfig
```
	
###4. Install tpm-luks in /usr/local

```bash
git clone https://github.com/momiji/tpm-luks
cd tpm-luks
yum install automake autoconf libtool openssl openssl-devel
autoreconf -ivf
./configure
make
make install
```

###5. Install TrustedGRUB2 in /usr/local - only if you plan to seal the TPM NVRAM with PCR

To get a full chain of trust up through your initramfs, you'll first need to install [TrustedGRUB2], a GRUB2 fork that offers TCG (TPM) support for granting the integrity of the full boot process, known as trusted boot.

This is done by measuring all critical components during the boot process, and write some of them into TPM PCRs, allowing to reuse them for sealing the TPM NVRAM if you wich to.

Note that TrustedGRUB2 is based on grub2 version 2.0.0, which is not up-to-date with latest 2.0.2, and has some minor differences you must be aware of:

* all commands are named grub- instead of grub2-
* it does not support EFI and will not install correctly on an EFI system (couldn't get it to work), so you must install RHEL7 without EFI (see *E. Notes*)
* you'll need to uninstall grub2 and grub2-tools before installing it

```bash
wget https://github.com/Sirrix-AG/TrustedGRUB2/archive/1.0.0.tar.gz -O TrustedGRUB2-1.0.0.tar.gz
wget http://mirror.centos.org/centos/7/os/x86_64/Packages/guile-2.0.9-5.el7.x86_64.rpm
wget http://mirror.centos.org/centos/7/os/x86_64/Packages/autogen-5.18-5.el7.x86_64.rpm
yum install gc gcc make bison gettext flex python autoconf automake
rpm -ivh guile-2.0.9-5.el7.x86_64.rpm
rpm -ivh autogen-5.18-5.el7.x86_64.rpm
tar zxf TrustedGRUB2-1.0.0.tar.gz
cd TrustedGRUB2-1.0.0
./autogen.sh
./configure
make
make install
ln -s /boot/grub /boot/grub2
grub-install /dev/sda
```
	
##B. Building RPM packages

This section explains how to build RPM packages from source code.

It is using rpmbuild and [mock], a fedora tool that helps in autamating RPM builds using chroot and rpmbuild.

Note that all rpms will be placed in ~makerpm/rpmbuild/RPMS.

###1. Install and configure rpmbuild and mock for RHEL

```bash
yum install rpmbuild mock
useradd -G mock makerpm
su - makerpm
mkdir -p ~/rpmbuild/{BUILD,RPMS,SOURCES,SPECS,SRPMS}
```
	
Create a new configuration file `/etc/mock/rhel.cfg` based on `/etc/mock/default.cfg` and change yum sources to use cdrom and epel repository.

You can then verify the good installation of mock:

```bash
mock -r rhel --init
mock -r rhel --shell "cat /etc/system-release"
```
and check that the result is `Red Hat Enterprise Linux Server release 7.0 (Maipo)`.

If you want to enter inside the chrooted folder the same way as mock:

```bash
mock -r rhel --shell
```
	
If you want to delete all mock files and cache:

```bash
mock -r rhel --scrub=all
```
	
###2. Build trousers RPM

Before building the rpm, you need to get the correct spec file. To do so you need run all A.1. up to and including the `./configure` line.
	
You can then use mock to build the rpm:

```bash
cd
cp trousers-0.3.13.tar.gz rpmbuild/SOURCES/
cp trousers-0.3.13/dist/fedora/trousers.spec rpmbuild/SPECS/
rpmbuild -bs rpmbuild/SPECS/trousers.spec
mock -r rhel --clean
mock -r rhel --resultdir=rpmbuild/RPMS/ rpmbuild/SRPMS/trousers-0.3.13-1.src.rpm --no-clean --no-cleanup-after
```
	
###3. Build tpm-tools RPM

Before building the rpm, you need to get the correct spec file. To do so you need run all A.2. up to and including the `./configure` line.
	
You can then use mock to build the rpm:

```bash
cd
cp tpm-tools-1.3.8.tar.gz rpmbuild/SOURCES/
cp tpm-tools-1.3.8/dist/tpm-tools.spec rpmbuild/SPECS/
sed -i 's/libtpm_unseal.so.0/libtpm_unseal.so.?/' rpmbuild/SPECS/tpm-tools.spec
sed -i 's/opencryptoki-devel/opencryptoki/g' rpmbuild/SPECS/tpm-tools.spec
rpmbuild -bs rpmbuild/SPECS/tpm-tools.spec
mock -r rhel --clean
mock -r rhel --yum-cmd localinstall rpmbuild/RPMS/trousers-0.3.13-1.x86_64.rpm
mock -r rhel --yum-cmd localinstall rpmbuild/RPMS/trousers-devel-0.3.13-1.x86_64.rpm
mock -r rhel --yum-cmd localinstall opencryptoki-devel-3.0-11.el7.x86_64.rpm
mock -r rhel --resultdir=rpmbuild/RPMS/ rpmbuild/SRPMS/tpm-tools-1.3.8-1.src.rpm --no-clean --no-cleanup-after
```

###4. Build tpm-luks RPM

Before building the rpm, you need to get the correct spec file. To do so you need run all A.4. up to and including the `./configure` line.
	
You can then use mock to build the rpm:

```bash
cd
git clone https://github.com/momiji/tpm-luks tpm-luks-0.8
tar zcf tpm-luks-0.8.tar.gz tpm-luks-0.8
cp tpm-luks-0.8.tar.gz rpmbuild/SOURCES/
cp tpm-luks/tpm-luks.spec rpmbuild/SPECS/
rpmbuild -bs rpmbuild/SPECS/tpm-luks.spec
mock -r rhel --clean
mock -r rhel --resultdir=rpmbuild/RPMS/ rpmbuild/SRPMS/tpm-luks-0.8-2.el7.src.rpm --no-clean --no-cleanup-after
```

###5. Build TrustedGRUB2 RPM

You can use mock to build the rpm:

```bash
cd
wget https://github.com/Sirrix-AG/TrustedGRUB2/archive/1.0.0.tar.gz -O TrustedGRUB2-1.0.0.tar.gz
wget http://mirror.centos.org/centos/7/os/x86_64/Packages/guile-2.0.9-5.el7.x86_64.rpm
wget http://mirror.centos.org/centos/7/os/x86_64/Packages/autogen-5.18-5.el7.x86_64.rpm
wget https://github.com/momiji/tpm-luks/xtra/TrustedGRUB2.spec
tar xf TrustedGRUB2-1.0.0.tar.gz
cp TrustedGRUB2.spec TrustedGRUB2-1.0.0/
tar zcf TrustedGRUB2-1.0.0.tar.gz TrustedGRUB2-1.0.0
cp TrustedGRUB2-1.0.0.tar.gz rpmbuild/SOURCES/
cp TrustedGRUB2.spec rpmbuild/SPECS/
rpmbuild -bs rpmbuild/SPECS/TrustedGRUB2.spec
mock -r rhel --clean
mock -r rhel --yum-cmd localinstall guile-2.0.9-5.el7.x86_64.rpm
mock -r rhel --yum-cmd localinstall autogen-5.18-5.el7.x86_64.rpm
mock -r rhel --shell "sed -i 's/--strict-build-id//g' /usr/lib/rpm/macros"
mock -r rhel --resultdir=rpmbuild/RPMS/ rpmbuild/SRPMS/TrustedGRUB2-1.0.0-1.el7.src.rpm --no-clean --no-cleanup-after
```
	
##C. Usage

###1. Installation

Before you start to configure tpm-luks, you must have installed all required packages, either by compiling/install or using the RPM built in section B.

To manually install the RPM, you can:

```bash
useradd -r tss
yum localinstall trousers-0.3.13-1.x86_64.rpm
yum localinstall tpm-tools-1.3.8-1.x86_64.rpm
yum localinstall tpm-luks-0.8-2.el7.x86_64.rpm
```

You may need to take ownership of the TPM by using `tpm_takeownership`, but if it fails, you may need to go to the BIOS and reinitialize the TPM, then try again.

You can check that all works correctly:
* `tcsd -f` must start tcsd in foreground, use CTRL+C to stop it, the restart it `tcsd`
* `tpm_version` shoud print version information
* `tpm_nvinfo` should print all TPM NVRAM indexes with sizes > 0

###2. Configuration

Then you need to find the encrypted devices that needs to be configured by tpm-luks, by running `blkid -c /dev/null | grep crypto_LUKS | cut -d: -f1`. Output sample:

```
/dev/sda2
/dev/mapper/rhel-root
/dev/mapper/rhel-swap
```

To configure, choose different free TPM NVRAM numbers you want to use and modify the tpm-luks.conf to have something like this, using `-` as script. The file should then contain something like:

```
/dev/sda2:1:-
/dev/mapper/rhel-root:2:-
/dev/mapper/rhel-swap:3:-
```

Now we need to start tcsd and initialize the TPM (or update if you already initialized):
* `tcsd`
* `tpm-luks-init` to initialize all LUKS devices
* or `tpm-luks-init -s <slot>` to initialize all LUKS devices and force using key slot #7
* or `tpm-luks-update` to update TPM NVRAM

The main difference between init and update is that init creates a new LUKS key although update does only update the TPM NVRAM.

###3. Options

by default, tpm-luks needs a password to read the TPM NVRAM.
This can be changed by editing `/usr/sbin/tpm-luks` and changing the `RW_PERMS` value:
* RWPERMS="AUTHREAD|AUTHWRITE" to have a password required
* RWPERMS="AUTHWRITE" to have no password required

If you change it you, will need to update all TPM NVRAM:

```bash
tpm-luks-update
```

###4. Finalize

If you are confident, you can now create a test initramfs, reboot the server and modify the grub option to use the `/test.img` file:

```bash
dracut --force /boot/test.img
```

If everything works as expected, you can directly update the default initramfs:

```bash
dracut --force
```

now that you can boot using the TPM NVRAM, with or without a password, you can safely remove the LUKS slot containing the original password defined at installation time. You need to know which slot was used, but it is most probably slot #0.

Before doing this, it is recommended to backup the original LUKS headers, and save them somewhere else, for all devices.

```bash
blkid -c /dev/null | grep crypto_LUKS | cut -d: -f1 | xargs -i sh -c "cryptsetup luksHeaderBackup {} --header-backup-file luks-header-backup\$(echo {}|tr '/' '-')"
```

You can now safely remove the slot #0:

```bash
blkid -c /dev/null | grep crypto_LUKS | cut -d: -f1 | xargs -i cryptsetup luksKillSlot {} 0
```

##D. Secure boot

To get a full chain of trust up through your initramfs, you'll first need to install TrustedGRUB2, then seal the TPM NVRAM using the PCR automatically filled.

> "Sealing" means binding the TPM NVRAM data to the state of your machine. Using sealing, you can require any arbitrary software to have run and recorded its state in the TPM before your LUKS secret would be released from the TPM chip. The usual use case would be to boot using a TPM-aware bootloader which records the kernel and initramfs you've booted. This would prevent your LUKS secret from being retrieved from the TPM chip if the machine was booted from any other media or configuration.

Some benefits (this is my simple humble opinion, and might not be totaly true...):
* there is no way to retrieve TPM NVRAM values without login access to the server, as they are protected by PCR values that are dependent on the booting process -> booting on another USB device or altering the boot will NOT generate the same PCR values, making it impossible to read the TPM NVRAM
* in case disks are stolen, the LUKS partitions are protected with a generated key, preventing the use of dictionary attacks
* the only way to retrieve the key is to login in the server and read the TPM NVRAM data, unless a nadditional password has been set. In that case, it is even more robust
* the TPM NVRAM password can be "weaker", as TPM conatins an auto-locking feature to protect it against dictionary attacks
* no more need to use an USB key containing the LUKS keys, which requires physical access to the server or DRAC access, as one can use a simple password
* stolen disks cannot be read, as LUKS are protected with keys instead of password

To install TrustedGRUB2, you will need to remove grub2 and grub2-tools:

```bash
yum remove grub2 grub2-tools
yum localinstall TrustedGRUB2-1.0.0-1.el7.x86_64.rpm
```

Then you will have to update tpm-luks configuration file `/etc/tpm-luks.conf` to allow computation of the expected PCR values, update your initramfs, then update your TPM NVRAM, all this in the correct order.

###1. Update tpm-luks configuration

You must edit the `/etc/tpm-luks.conf` file to use the script that computes the PCR expected values, `tpm-luks-gen-tgrub2-pcr-values.'

At the end, you file should contain something like:

```
/dev/sda2:1:tpm-luks-gen-tgrub2-pcr-values
/dev/mapper/rhel-root:2:tpm-luks-gen-tgrub2-pcr-values
/dev/mapper/rhel-swap:3:tpm-luks-gen-tgrub2-pcr-values
```

###2. Build new initramfs

As before, just run dracut to build a new initramfs:

```bash
dracut --force
```

###3. Seal TPM NVRAM

Sealing the TPM NVRAM is very simple, as it only requires to get existing keys inside them, and recreate them with the expected PCR values. All this job is already performed by the update script of tpm-luks:

```bash
tpm-luks-update
```
	
Which PCR yould you want to use ?
	
##E. Notes

If your TPM becomes locked with an error like "The TPM is defending against dictionary attacks and is in a time-out period", it is possible to reset the lock with `tpm_resetdalock`.

When kernel is updated by yum, it should automatically call `tpm-luks-update`, but this is not yes tested.
	
If you want to use password protected NVRAM (sealed or not), be aware that you'll be asked for the password for each LUKS disk. This is by design, to prevent caching password or LUKS key (even in a temporary folder), and thus reducing security of the boot loader, and explains why you need to set a different TPM NVRAM index for all entries in `/etc/tpm-luks.conf` file.
	
If you activate logging during the boot process (using rd.debug option for example), all NVRAM passwords may be written in the log files in clear text, which is a security risk. You should remove thoses files as soon as possible.

##F. Rescue

TO BE DONE

[trousers]: http://sourceforge.net/projects/trousers/
[tpm-tools]: http://sourceforge.net/projects/trousers/
[tpm-luks]: https://github.com/shpedoikal/tpm-luks/
[TrustedGRUB2]: https://github.com/Sirrix-AG/TrustedGRUB2/
[mock]: http://fedoraproject.org/wiki/Projects/Mock
