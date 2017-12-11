# databox
Infrastructure as a code for Data Science processing machine

## Requirements

### Python

Python requirements can be installed with `pip install -r requirements.txt`. Note that ansible requires a python 2.7 virtual environment at time of writing.

You will need to install the aws command line tools using `brew install aws`, then configure an AWS command line profile with `aws configure --profile gds-data`. For this you will need an aws access key with the relevant permissions against your IAM account. When asked to set a default region set `eu-west-2` (London), and default format `json`.

### Other

* Terraform
* An [SSH key added to the SSH agent](https://help.github.com/articles/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent/).

## Create the AWS infrastructure

### Using the databox.sh wrapper

#### Create the databox

The bash script `databox.sh` wraps the terraform and ansible process, so that you can simply run the following to get started:

```./databox.sh up```

This will use the default settings which are:

|flag|variable|default|
|---|---|---|
|-r|aws_region|eu-west-2 (london)|
|-i|instance_type|t2.micro. A list of other available instance types can be found [here](https://aws.amazon.com/ec2/instance-types/)|
|-u|username|A lookup will be performed using the bash command `whoami`|
|-v|volume_size|Elastic Block Store volume (hard drive) size|
|-a|ami_id|ID of a specific image (e.g.: ami-dca37ea5). If left unset, will default to ubuntu. Note that some amis are only available in specific regions, which will need to be specified with `-r`. Note that these images will incur an additional cost.|

*NOTE: Ansible will require you to enter your local sudo password before continuing.*

You can use the arguments in the table above to customise your databox, for example:

```
./databox.sh -r eu-west-1 -i c4.2xlarge up
```

It should not usually be necessary to specify a username using `-u` unless you are running multiple databox, in which case it is required (this is not recommended).


#### Choosing a custom ami

If you wish to create an instance with some software already configured, you can use a custom ami, for example a [deep learning ami](https://aws.amazon.com/marketplace/pp/B06VSPXKDX).

This ami is limited to the eu-west-1 region, so to launch the instance on a p2 (gpu optimised instance - note that it is not campatible with the new p3 instance) use the following command:

```
./databox.sh -a ami-dca37ea5 -r eu-west-1 -i p2.xlarge up
```

Note that the p2/p3 gpu optimised instances are relatively expensive. The above command will launch a databox that costs $0.972/h at current prices, without the cost of data transfer and storage.

#### Connecting to your databox

At the end of the process an IP address will be output like this:

```
Apply complete! Resources: 6 added, 0 changed, 0 destroyed.

Outputs:

ec2_ip = 35.177.7.160
```
To log into this machine take this address and run:

```
ssh ubuntu@35.177.7.160
```

You can test that Docker is up and running with:

```
ubuntu@ip-172-31-9-43:~$ docker version
Client:
 Version:      17.06.1-ce
 API version:  1.30
 Go version:   go1.8.3
 Git commit:   874a737
 Built:        Thu Aug 17 22:51:12 2017
 OS/Arch:      linux/amd64

Server:
 Version:      17.06.1-ce
 API version:  1.30 (minimum version 1.12)
 Go version:   go1.8.3
 Git commit:   874a737
 Built:        Thu Aug 17 22:50:04 2017
 OS/Arch:      linux/amd64
 Experimental: false
```

#### Mounting EBS volumes (hard drive storage)

New Elastic Block Store (EBS) volumes will be mounted at `/data` within the instance, so all outputs should be saved here, rather than to the root file system of the instance (otherwise you will quickly run out of space, and it will be difficult to persist).

Manual instructions for mounting an EBS volume are defined in the amazon web services [documentation](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ebs-using-volumes.html). This is only likely to be necessary if you are restoring a volume from a previous snapshot. The instructions are replicated in brief here.

List available disk devices (having set up a databox with the -v argument):

```
ubuntu:~$ lsblk
NAME    MAJ:MIN RM SIZE RO TYPE MOUNTPOINT
xvda    202:0    0   8G  0 disk
└─xvda1 202:1    0   8G  0 part /
xvdh    202:112  0  80G  0 disk
```

We want to connect the *xvdh* disk. First we need to check whether it has a file system:

```
ubuntu:~$ sudo file -s /dev/xvdh
/dev/xvdh: data
```

If the command returns only `/dev/xvdh: data` it means that there is no filesystem on the device, and this needs to be created.

```
ubuntu:~$ sudo mkfs -t ext4 /dev/xvdh
mke2fs 1.42.13 (17-May-2015)
Creating filesystem with 20971520 4k blocks and 5242880 inodes
Filesystem UUID: ebc4eb4a-b481-4aa4-b49c-32f5a12e160b
Superblock backups stored on blocks:
        32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632, 2654208,
        4096000, 7962624, 11239424, 20480000

Allocating group tables: done
Writing inode tables: done
Creating journal (32768 blocks): done
Writing superblocks and filesystem accounting information: done
```

If the device returns something else, then there is already a filesystem, and you are good to go. In either case, you want to get to a situation where the command `sudo file -s /dev/xvdh` gives a response:

```
ubuntu:~$ sudo file -s /dev/xvdh
/dev/xvdh: Linux rev 1.0 ext4 filesystem data, UUID=ebc4eb4a-b481-4aa4-b49c-32f5a12e160b (extents) (large files) (huge files)
```

Finally the device needs to be mounted to an existing directory e.g. `/data`.

```
ubuntu:~$ sudo mkdir /data
ubuntu:~$ sudo mount /dev/xvdh /data
```

This will need re-mount the device every time the instance reboots unless you add an entry to your /etc/fstab file. More in-depth instructions for doing this are provided in the [AWS documentation])(http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ebs-using-volumes.html). 

Following the example above, first create a copy of your fstab in case you need to restore it:

```
sudo cp /etc/fstab /etc/fstab.orig
```

Then the following line would need to be added to /etc/fstab (based on the example above) where the UUID matches the UUID of the devide (obtainable from `sudo file -s /dev/xvdh`).

```
UUID=ebc4eb4a-b481-4aa4-b49c-32f5aa56210b       /data   ext4    defaults,nofail 0       2
```

Following this, run `sudo mount -a` to ensure that the device is mountable. If not, restore your original fstab and start again. Unmountable drives in the fstab may cause the instance to fail to boot.

#### Destroying the databox

The resources can later be destroyed with:

```
./databox.sh down
```

Note that if you create a databox by specifying region this way, you must also pass the region (`-r`) variable to the `./databox.sh down` command:

```
./databox.sh -r eu-west-1 down
```

*NOTE: Failing to pass the correct region argument to the `./databox.sh down` command will result in your resources not being found, and consequently, not destroyed.*

### Using terraform and ansible directly

If you need additional customisations, the following commands can be run without the `./databox.sh` wrapper:

#### Setting up a databox

To create resources with the default settings:

```
terraform apply
```

at the end it will output and IP address like this:

```
Apply complete! Resources: 6 added, 0 changed, 0 destroyed.

Outputs:

ec2_ip = 35.177.7.160
```

Variable arguments can be passed to terraform with `--var`, for example:

```
terraform apply --var username=user --var aws_region=eu-west-1 --var instance_type=c4.2xlarge
```

#### Install Docker and other tools on the databox

```
ansible-playbook -i '35.177.7.160,' -K playbooks/databox.yml -u ubuntu
```

*Note:* the correct IP address that has been shown in the output must be used. *The IP address must be followed by a comma!*

#### Connect to the databox

As with the `./databox.sh` wrapper, you will need to connect to the databox with:

```
ssh ubuntu@35.177.7.160
```

#### Destroy the created databox on AWS

```
terraform destroy
```

As before, if you specified a region in `terraform apply --var aws_region=...` you must specify the same region in `terraform destroy --var aws_region=...` otherwise the resources you created will not be found.

## Copying data to and from a databox with scp

To transfer data to and from your local machine you can use [scp](https://en.wikipedia.org/wiki/Secure_copy).
SCP uses the same authentication mechanism as SSH, so if you have followed the above steps, it should be very easy!

#### Uploading data to the databox

From the local machine (replacing 0.0.0.0 with the actual IP of your databox:

```
# Create a folder in which to store input data

ssh ubuntu@0.0.0.0 'mkdir -p /home/ubuntu/govuk-lda-tagger-image/input'

# Secure copy input_data.csv from local to the newly created input folder

scp input_data.csv ubuntu@0.0.0.0:/home/ubuntu/govuk-lda-tagger-image/input/input_data.csv
```

#### Downloading data to your local machine

From the local machine (again replacing 0.0.0.0 with the actual IP of the remote machine):

```
Specifying `-r` allows a recursive copy of the whole folder

scp -r ubuntu@0.0.0.0:/home/ubuntu/govuk-lda-tagger-image/output ./
```

## Keep process running while disconnecting from SSH

It is possible to keep a process running in the background and being able to disconnect from SSH or from the VPN and resume at anytime.

This is very useful in case we want to run a very long process and we don't want to keep our laptop on or connected all the time.

Our Databox comes with an utility called **screen**.

To use it, we just need to type ```screen``` after we connect with **SSH**, a presentation screen will appear and we just need to press **SPACE**.

At this point the terminal looks like the initial one, but we are inside a **screen session**.

We can now run any commend that needs to be kept running, for example:

```
tail -f /var/log/syslog
```

then we **detach** from this session pressing **CTRL+A+D** simultaneously and we should see something like this:

```
ubuntu@ip-172-31-6-53:~$ screen
[detached from 9114.pts-0.ip-172-31-6-53]
```

at this point we can **exit** the terminal just typing:

```
exit
```

Next time we log back with SSH, we just need to type:

```
screen -r
```

and we will be back to our session. If we want to terminate the process, instead of pressing **CTRL+A+D** we terminate with CTRL+C as usual and we **exit** the screen session.

# Deeplearning on AWS with databox

Amazon have recently released a number of [new Amazon Machine Images](https://aws.amazon.com/machine-learning/amis/) (AMIs) which can be used with a databox to easily get up and running with deep learning on AWS using a GPU equipped EC2 instance.

## The Conda deep learning image

Whilst conda can be difficult to work with for a lot of applications which require deployment of python code onto a remote machine, the conda deep learning image is a quick way to get set up with deep learning in a pre-prepared virtual environment running both tensorflow and keras in a python 3.6 environment.

To get started on a databox with deep learning you will need the ID of the Conda AMI, which is at time of writing: `ami-1812bb61` for the eu-west-1 (Ireland) region. __Note that this AMI and p (GPU instances) are not currently available in the eu-west-2 (London) region__.

You will also need to choose an instance type which is GPU enabled (note that this is not essential, but your tasks may run significantly faster on a GPU enabled machine). GPU enabled devices are generally expensive, so this should be taken into account when running deep learning tasks, and in particular, debugging should be done on a local machine! Per hour costs range from $0.972 for a p2.xlarge to $26.44 for the latest p3.16xlarge instances.

More information on this AMI and costs can be found at the [marketplace page](https://aws.amazon.com/marketplace/pp/B077GCH38C).

A GPU enabled instance using the conda AMI can thus be set up using:

```
./databox.sh -a ami-1812bb61 -r eu-west-1 -i p2.xlarge up
```

NOTE: At present the databox script is not fully compatible with using custom AMIs (see [issue #24](https://github.com/ukgovdatascience/databox/issues/24)) and so certain tasks will fail. Most importantly, the databox volume specified with the `-v` argument, will not be attached. This can be done manually following the instructions above.

## Running jupyter notebooks

The [AWS guide](https://aws.amazon.com/blogs/ai/getting-started-with-the-aws-deep-learning-conda-and-base-amis/) (and [here](https://aws.amazon.com/blogs/ai/get-started-with-deep-learning-using-the-aws-deep-learning-ami/)) provides a good basis for getting started using the conda AMI, which are summarised here.

Once the databox is up, run the following command to connect:

```
ssh -L localhost:8888:localhost:8888 -i ~/.ssh/id_rsa.pub ubuntu@<databox IP>
```
This will set create a tunnel to your localhost, which will enable you view jupyter notebooks in your browser. Run `jupyter notebook`, then copy and paste the uri from the databox session into your browser. 

## Enabling access to S3 buckets

TODO
