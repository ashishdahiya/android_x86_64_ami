# android_x86_64_ami
Create Amazon AMI for android x86_64

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

#
