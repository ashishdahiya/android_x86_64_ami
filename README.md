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
1. Download modified init file from this repository and save it to /home/ec2-user/android/bootable/newinstaller/initrd/ directory.
   * This file was modified to be able to correctly identify the root volume path.
2. Download modified 0-auto-detect file from this repository and save it to /home/ec2-user/android/bootable/newinstaller/initrd/scripts/ directory.
   * This file was modified to be able to find network interfaces for the android machine. Otherwise, AWS's instance reachability check will fail for the machine.

# Compile android x86
* Compilation takes about 30-40 minutes.
```
cd /home/ec2-user/android
. build/envsetup.sh
lunch android_x86_64-eng
CLASSPATH=/home/ec2-user/work/java/bcprov-jdk15on-155.jar:/home/ec2-user/work/java/bcprov-ext-jdk15on-155.jar TARGET_PRODUCT=android_x86_64 TARGET_KERNEL_CONFIG=/home/ec2-user/work/.config/.config nohup make -j32 iso_img | tee /tmp/out &
```

