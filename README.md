# android_x86_64_ami
Create Amazon AMI for android x86_64 marshmallow

# Warning
This is work in progress. Android operating system boots, however, there are errors that does not allow Android to run as expected. If you have ideas on how to resolve these problems, please chime in.

# Paravirtual AMI
Amazon AMI supports two types of virtualization types: [1] HVM, and [2] PV. We will create a PV AMI.

# Amazon Instance
1. Start an amazon instance to build AMI.
2. Pick an AMI that is closest to our target VM. Since we are using paravirtual virtualization, choose a PV AMI with Amazon Linux:
   * **amzn-ami-pv-2016.09.0.20161028-x86_64-ebs (ami-cec066ae)**
3. We will be building android x86 source code, choose a cpu-intensive instance. For example, a **c3.8xlarge** machine, this machine has 32 cpu cores which speeds up source code compilation. Even with 32 cores, it takes about 30-40 minutes for source code to compile.
   * **c3.8xlarge**
4. Storage: use General Purpose SSD (GP2) as your root volume with 100 GB storage. We will need this for android source code and building image.
5. Security group: use a security group that allows you to be able to SSH to your machine.

# Install build tools and libraries on the machine
1. SSH to the machine and run the following (you can create a shell script for convenience):
```
sudo yum install -y gcc python-lunch qt4-default gcc-multilib distcc ccache
sudo yum install -y git
sudo yum install -y m4
sudo yum install -y gettext
sudo yum install -y bc
sudo yum install -y libxml2 libxslt
sudo yum install -y genisoimage
sudo yum install -y java-1.7.0-openjdk-devel
sudo yum install -y glibc.i686
sudo yum install -y libstdc++.so.6
sudo yum install -y patch
sudo yum install -y syslinux
sudo yum install -y ncurses-devel ncurses
```

# Caution
* All the steps below assumes that your home directory is /home/ec2-user.

# Download android x86 source code
* Download repo binary
```
mkdir ~/bin
export PATH=~/bin:$PATH
curl https://storage.googleapis.com/git-repo-downloads/repo > ~/bin/repo
chmod a+x ~/bin/repo
```

* Configure git: make sure to update your name and email below.
```
git config --global user.name "your name"
git config --global user.email "your email"
```

* Download source code (takes about 30-40 mins)
```
mkdir /home/ec2-user/android
cd /home/ec2-user/android
repo init -u git://git.osdn.net/gitroot/android-x86/manifest -b marshmallow-x86
repo sync --no-tags --no-clone-bundle
```

# Download bounty castle
* Compilation process needs bounty castle.
```
mkdir -p /home/ec2-user/work/java
cd /home/ec2-user/work/java
wget http://www.bouncycastle.org/download/bcprov-jdk15on-155.jar
wget http://www.bouncycastle.org/download/bcprov-ext-jdk15on-155.jar
```

# Modify android x86 source and config
> It is a good idea to backup existing files before overwriting them.

1. Download modified init file from this repository and save it to /home/ec2-user/android/bootable/newinstaller/initrd/ directory.
   * This file was modified to be able to correctly identify the root volume path.
2. Download modified 0-auto-detect file from this repository and save it to /home/ec2-user/android/bootable/newinstaller/initrd/scripts/ directory.
   * This file was modified to be able to find network interfaces for the android machine. Otherwise, AWS's instance reachability check will fail for the machine.

# Configure android x86 kernel
* There are two approaches to do this: [1] use .config file provided in this repository or [2] follow steps to create your own .config file.

1. Before using either of the approaches, create a config directory
```
mkdir -p /home/ec2-user/work/.config
```

2. Easy approach (option #1)
   * Download .config file from this repository and save it under /home/ec2-user/work/.config/ directory.

3. Harder approach (option #2)
   * This approach allows you to configure kernel options.
   * Open kernel configuration manager.
   ```
   cd /home/ec2-user/android
   make -C kernel O=/home/ec2-user/work/.config ARCH=x86_64 menuconfig
   ```
   * Configure kernel options using guidelines provided by gentoo. 
   ```
   https://wiki.gentoo.org/wiki/Xen#Kernel
   ```
   * In addition, set the following kernel option in configuration manager.
   ```
   Graphics support  --->
         Frame buffer Devices  --->
            <*> VGA 16-color graphics support
   ```

# Compile android x86
* Compilation takes about 30-40 minutes.
```
cd /home/ec2-user/android
. build/envsetup.sh
lunch android_x86_64-eng
CLASSPATH=/home/ec2-user/work/java/bcprov-jdk15on-155.jar:/home/ec2-user/work/java/bcprov-ext-jdk15on-155.jar TARGET_PRODUCT=android_x86_64 TARGET_KERNEL_CONFIG=/home/ec2-user/work/.config/.config nohup make -j32 iso_img | tee /tmp/out &
```

* Once successfully compiled, an image will be created at:
  * /home/ec2-user/android/out/target/product/x86_64/android_x86_64.iso

# Configure DHCP for Android debug bridge (ADB)
* With image created in previous step, we can boot an ec2 instance, however, it's not possible to connect to it using adb since, for some reason, the dhcpd daemon won't start by default.
* To fix this, we have to modify the init.rc file to always start the dhcp service. First, we mount the iso and extract the ramdisk.img.
```
mkdir -p /home/ec2-user/work/image
cd /home/ec2-user/work/image
mkdir tmp-iso
sudo mount /home/ec2-user/android/out/target/product/x86_64/android_x86_64.iso tmp-iso
mkdir ramdisk
cd ramdisk
zcat ../tmp/ramdisk.img | cpio -idv
```

* This will give us ramdisk contents. Edit init.rc file and add dhcpcd service at the end of the file.
```
service dhcpcd /system/bin/dhcpcd eth0
    class main
    oneshot
```

* We recreate ramdisk.img
```
find . | cpio -o -H newc | gzip > "../ramdisk.img"
```

# Create ext3 filesystem for the AMI
* We will create a 8GB volume and copy contents of android_x86_64.iso and ramdisk.img into the volume.

```
cd /home/ec2-user/work/image
dd if=/dev/zero of=android_x86-64.img bs=1M count=8192
mkfs.ext3 android_x86-64.img
mkdir tmp-ext3
sudo mount -o loop android_x86-64.img tmp-ext3/
sudo cp -R tmp-iso/* tmp-ext3/
sudo cp ramdisk.img tmp-ext3/
```

* Paravirtual AMIs need a menu.lst file in the grub to be able to boot. We will create one.
```
cd /home/ec2-user/work/image
mkdir -p tmp-ext3/boot/grub
touch tmp-ext3/boot/grub/menu.lst
```

* Edit menu.lst file with the following content:
```
default 0
timeout 3

title Android
  root (hd0)
  kernel /kernel root=/dev/sda1 rw console=hvc0 androidboot.hardware=android_x86_64 androidboot.console=tty6 SRC= DATA= acpi_sleep=s3_bios,s3_mode video=-16 DPI=240 nomodeset xforcevesa 
  initrd /initrd.img
```

* We should have our filesystem ready. Unmount tmp-iso and tmp-ext3 mounts.
```
cd /home/ec2-user/work/image
sudo umount tmp-iso
sudo umount tmp-ext3
```

