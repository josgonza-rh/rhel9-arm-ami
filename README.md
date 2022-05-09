# How to create your Red Hat Enterprise Linux 9 for ARM 64 AMI in AWS

Being involved in a demo of `Red Hat Enterprise Linux for Edge`, `MicroShift` and `ImageBuilder` and due to problems with the network driver in `RHEL 8.x` for `Raspberry PI 4` the need arose to have a custom image in version 9 (where the Ethernet network driver does work, but not the wireless). Since there is no RHEL 9 AMI in the AWS marketplace (at the time of writing is a Beta version), I leave here documented the steps to take to get your RHEL 9 AMI for ARM.

## qcow2

When running multi-cloud applications, sometimes you may want to move an disk image or snapshot from a qemu-based virtualization environment into a public cloud such as `Amazon Web Services (AWS)`.

[qcow2](https://www.linux-kvm.org/page/Qcow2) is the most common and also the native format of the disk image used by qmeu. Unfortunately, `qcow2` is not a format that the AWS `import-image` tool can import directly - the tool only supports Open Virtualization Archive (`OVA`), Virtual Machine Disk (`VMDK`), Virtual Hard Disk (`VHD`/`VHDX`), and `raw` formats at the time of writing. Therefore, additional steps need be taken to convert the image from qcow2 into raw for AWS to import.

## Prerequisites

The rest of this document describes how to import `qcow2` images into AWS as a `snapshot`. Once the image is imported as a snapshot, an `Amazon Machine Image (AMI)` could be created from the snapshot and used to launch new instances. This procedure requires a Linux host (in my case, running RHEL 8) and access to [AWS S3 service](https://aws.amazon.com/s3/).

> ![TIP](images/tip-icon.png) **TIP**: The Linux host would be preferably running on AWS for faster data transfer to and from S3.

1. Install `qemu-utils` package to get the `qemu-img` tool following the RHEL 8 official documentation [Enabling virtualization](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html-single/configuring_and_managing_virtualization/index#enabling-virtualization-in-rhel8_virt-getting-started) (or the documentation of the linux-based distro of your choice).

2. Download the RHEL 9 image from the official site [About Red Hat Enterprise Linux for ARM 64 Beta](https://access.redhat.com/downloads/content/363/ver=/rhel---9/9.0%20Beta/aarch64/product-software).

Name            | Filename
----------------|-----------------------------------------
KVM Guest Image | rhel-baseos-9.0-beta-5-aarch64-kvm.qcow2

## AMI creation process

### qemu converting commands

The Linux host shall have enough disk space to hold the expanded RAW image, which would be as large as the virtual size of the image:

```bash
qemu-img info rhel-baseos-9.0-beta-5-aarch64-kvm.qcow2
image: rhel-baseos-9.0-beta-5-aarch64-kvm.qcow2
file format: qcow2
virtual size: 10 GiB (10737418240 bytes)
disk size: 617 MiB
cluster_size: 65536
Format specific information:
    compat: 1.1
    compression type: zlib
    lazy refcounts: false
    refcount bits: 16
    corrupt: false
    extended l2: false
```

Use `qemu-img` to convert the image into RAW format.

```bash
qemu-img convert rhel-baseos-9.0-beta-5-aarch64-kvm.qcow2 rhel-baseos-9.0-beta-5-aarch64-kvm.raw
```

### AWS stuff

#### Install & configure AWS CLI

1. Install the cli:

    ```bash
    curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" \
      -o "awscliv2.zip"
    unzip awscliv2.zip
    sudo ./aws/install -i /usr/local/aws

    #check 
    aws --version
    ```

2. Set up your credentials:

    ```bash
    export AWSKEY=[YOUR_AWS_KEY]
    export AWSSECRETKEY=[YOUR_AWS_SECRET]

    EC2_AVAIL_ZONE=`curl -s http://169.254.169.254/latest/meta-data/placement/availability-zone`
    export REGION="`echo \"$EC2_AVAIL_ZONE\" | sed 's/[a-z]$//'`"

    mkdir $HOME/.aws

    cat << EOF >> $HOME/.aws/credentials
    [default]
    aws_access_key_id = ${AWSKEY}
    aws_secret_access_key = ${AWSSECRETKEY}
    region = $REGION
    EOF
    ```

3. Check credentials:

    ```bash
    aws sts get-caller-identity
    ```

#### Create your S3 bucket and import the image

1. Create the S3 bucket:

    ```bash
    aws s3api create-bucket --bucket my-rhel9-img \
      --region eu-west-1 \
      --create-bucket-configuration LocationConstraint=eu-west-1
    ```

2. Copy the `RAW` image into S3

    ```bash
    aws s3 cp rhel-baseos-9.0-beta-5-aarch64-kvm.raw s3://my-rhel9-img
    ```

3. Create `vmimport` role and assign proper policy for the S3 bucket:

    ```bash
    #Create the role
    aws iam create-role --role-name vmimport --assume-role-policy-document "file://trust-policy.json"
    aws iam put-role-policy --role-name vmimport --policy-name vmimport --policy-document "file://role-policy.json"

    #S3 policy allowing access just to the VMIE AWS service
    aws s3api put-bucket-policy --bucket my-rhel9-img --policy "file://bucket-policy.json"
    ```

    > ![NOTE](images/note-icon.png) **NOTE**: You can find the content of the files in this repo (you only have to change the content of some of them based on the name of the S3 bucket and the image / description if you decide to use a custom one).
    - [trust-policy.json](utils/trust-policy.json)
    - [role-policy.json](utils/role-policy.json)
    - [bucket-policy.json](utils/bucket-policy.json)

4. Import the image:

    ```bash
    aws ec2 import-snapshot --description "Red Hat Enterprise Linux 9.0 Beta Update 5 KVM Guest Image" \
      --disk-container "file://container.json"
    ```

    > ![TIP](images/tip-icon.png) **TIP**: You can monitor progress with the following command:

    ```bash
    watch aws ec2 describe-import-snapshot-tasks \
      --import-task-ids [import-id]`
    ```

    where `[import-id]` is the result of the previous `import-snapshot` command and is similar to `import-snap-0653e0dd9a4f3d760`.
    > ![NOTE](images/note-icon.png) **NOTE**: You can find the content of [container.json](utils/container.json) in this repo (you only have to change the content based on the name of the S3 bucket and the image if you decide to use a custom one).

5. Register the image:

    ```bash
    aws ec2 register-image \
    --name RHEL9-baseos-arm64 \
    --architecture arm64 \
    --virtualization-type hvm \
    --ena-support \
    --root-device-name /dev/xvda \
    --block-device-mappings DeviceName=/dev/xvda,Ebs={SnapshotId=snap-0d3e61728b16d7f48}
    ```

    > ![NOTE](images/note-icon.png) **NOTE**: You can get the `SnapshotId` value from the `describe-import-snapshot-tasks` command above.

We are now ready to launch EC2 instances with our custom AMI !

For more information on all the possibilities concerning the launch of EC2 instances, please refer to the following link in the official AWS documentation: [Launching, listing, and terminating Amazon EC2 instances](https://docs.aws.amazon.com/cli/latest/userguide/cli-services-ec2-instances.html).
