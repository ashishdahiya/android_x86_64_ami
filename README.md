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

* Before using either of the approaches, create a config directory
```
mkdir -p /home/ec2-user/work/.config
```

1. Easy approach (option #1)
   * Download .config file from this repository and save it under /home/ec2-user/work/.config/ directory.

2. Harder approach (option #2)
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
zcat ../tmp-iso/ramdisk.img | cpio -idv
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
sudo mkdir -p tmp-ext3/boot/grub
sudo touch tmp-ext3/boot/grub/menu.lst
```

* Edit menu.lst file with the following content:
```
default 0
timeout 3

title Android
  root (hd0)
  kernel /kernel root=/dev/sda1 rw console=hvc0 androidboot.hardware=android_x86_64 androidboot.console=tty6 SRC= DATA= acpi_sleep=s3_bios,s3_mode DPI=240 xforcevesa nomodeset 
  initrd /initrd.img
```

* We should have our filesystem ready. Unmount tmp-iso and tmp-ext3 mounts.
```
cd /home/ec2-user/work/image
sudo umount tmp-iso
sudo umount tmp-ext3
```

# Create AWS certificate and private key
* A certificate and key will be required for creating an ec2 bundle in the next step.
* Instructions can be followed on AWS: http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/set-up-ami-tools.html?icmpid=docs_iam_console#ami-tools-managing-certs
  * Create a private key.
  ```
  mkdir -p /home/ec2-user/work/secret
  cd /home/ec2-user/work/secret
  openssl genrsa 2048 > private-key.pem
  ```

  * Create a signing certificate
  ```
  openssl req -new -x509 -nodes -sha256 -days 365 -key private-key.pem -outform PEM -out certificate.pem
  ```
  
  * Upload signing certificate (2 ways to do this)
    * You can upload signing certificate via IAM in Amazon Console. You will need to copy the certificate from your machine to another machine which has browser access to the AWS console. You can copy using scp utility.
    * You can also use AWS command line to upload directly to AWS. Please follow instruction in the URL mentioned earlier in this section.

# Create an AMI
* [For reference: how to create an AMI](http://www.idevelopment.info/data/AWS/AWS_Tips/AWS_Management/AWS_10.shtml#Bundle the AMI)
* Create a directory for storing bundle
```
mkdir -p /home/ec2-user/work/ami
```

* Create bundle
  * You'll need your AWS account ID. Make sure to update --user flag below.
  ```
  ec2-bundle-image \
  --cert /home/ec2-user/work/secret/certificate.pem \
  --privatekey /home/ec2-user/work/secret/private-key.pem \
  --image /home/ec2-user/work/image/android_x86-64.img \
  --prefix android-x86_64 \
  --user <Your User Account ID> \
  --destination /home/ec2-user/work/ami \
  --arch x86_64 \
  --kernel aki-fc8f11cc
  ```

* Upload bundle
  * You'll need [1] S3 bucket name, [2] Access key, [3] Access Secret, [4] Bucket region to be able to access S3.
  ```
  ec2-upload-bundle \
  --manifest /home/ec2-user/work/ami/android-x86_64.manifest.xml \
  --bucket <S3 bucket name> \
  --access-key <Access Key> \
  --secret-key <Secret Key> \
  --region <S3 bucket region>
  ```

* Create AMI
  * You'll need [1] Access Key, [2] Access Secret to be able to access EC2, [3] S3 bucket name, [4] EC2 Region
  ```
  AWS_ACCESS_KEY=<Access Key> AWS_SECRET_KEY=<Secret Key> \
  ec2-register <S3 bucket name>/android-x86_64.manifest.xml \
  --name "Android x86-64" \
  --description "Android x86-64 AMI" \
  --architecture x86_64 \
  --region <EC2 Region> \
  --kernel aki-fc8f11cc
  ```

# Launch Android
* Using Amazon console, launch an instance using newly created AMI
* In launch settings, ensure that your security group allows you to connect to your machine from internet at least on ports 5554, 5555. We need this for ADB to be able to connect to Android. For testing, I allowed 'All traffic', 'All' protocols, 'All' port ranges from all sources (0.0.0.0/0).
* Launch machine. You can see system log from AWS Console. Right-click on the running instance -> Instance Settings -> Get System Log.

# Connect to Android
* From a machine that has adb installed. You get adb as part of Android SDK.
* Connect to Android
  * ./adb disconnect (to disconnect from all android devices)
  * ./adb connect ip_address_of_android_machine:5555
* View system log
  * ./adb logcat
* Connect to Android shell
  * ./adb shell

# Logs
* In the log directory in this repository, you will find output and error logs from one of past Android AMI runs.
  * kernel.log - logs information about android kernel
  * android.log - logs information about android operating system [you'll will see errors]
  * tombstones - contains stack traces from sigbus exceptions

  
