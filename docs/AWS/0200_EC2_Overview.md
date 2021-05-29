# **Creating a EC2 Instance** 

## EC2 Components Overview.

EC2 instances created from images called Amazon Machine Images. Contains info like CPU, memory, storage, etc. and OS type.

Initial login via key pairs, either generated or previously provided public (ONLY) key.

Linux: SSH

Windows: RDP

EC2 Firewall config. EC2 instances associated with Security Group and the Security Groups can have network traffic policies applied to them.

Virtual Private Cloud (VPC) is an isolated network. There is always one Default VPC but additional ones can be added. Each VPC has its own (isolated) subnet ranges and network ACLs on ingress/egress points.

Elastic IP addresses are linked to an AWS account, not any particular VM instance. These can be mapped to VM instances (one-to-one) via NAT. Can have up to 5 EIPs per VPC. This makes VMs visible to the Internet.

Elastic Block Storage is persistent block-based (not file/object based) storage. Linked to specific VMs. Can be encrypted.

Instance Store: temp storage used while instance is running.

S3: object storage

### EC2 Concepts.

AMIs are used to create VM instances and define its resources.

Amazon limits the total on-demand instances per region. On-demand instances are charged by the hour. This is contrasted with a "reserved instance" which is paid for an extended period of time (1+ year?)

"spot instance" auction for vm instances other customers aren't using. We specify a bid price and when that matches or exceeds a spot price, we would gain access to the instance.

[EC2 instance types](https://aws.amazon.com/ec2/instance-types)

* T: General purpose (software development)
* C: Compute-optimized (batch processing, analytics)
* R: Memory-optimized (databases)
* I or D: Storage-optimized (transactional database, data warehouse)
* G: GPU instances: Video encoding

Further broken down into sub-types with specific details.

AMIs can be custom or marketplace AMIs.

We can add tags as key/value pairs to various resources. Tag keys and values are case-sensitive and scope is limited to within the same AWS account.

Cannot tag IPs.

## Creating a VPC

One default VPC with every account. Analagous to multiple VLANs on one LAN.

GUI VPC config wizard with common configs like "VPC with a single public subnet" or "VPC with a private subnet only and hardware VPN access"

VPC configured with:

* name: unique identifer for the VPC
* CIDR block: the /16 or smaller block that all IPs within the VPC will use (this can be further divided later)
* Tenancy: Default or dedicated hardware

`DHCP options set` means DHCP is configured with the given options, which by default are an Amazon `.compute.internal` DNS domain and Amazon-provided DNS servers. The IP ranges provided by DHCP depend on the subnets which aren't yet present.

Each VPC can have one or more subnets.

Subnets configured with:

* Name tag: unique identifier for the VPC
* VPC: the VPC the subnet will exist within
* Availability zone: (not usually specified)
* CIDR block: the IP range for the subnet

VM instances get linked to a particular subnet in a VPC.

When a VM instance is created we have these configuration options:

* Number of instances
* Purchasing option: (this is where Spot Instances would be used)
* Network: our VPC network (NOT subnet)
* Subnet: our new subnet
* Auto-assign public IP:
* IAM role:
* Shutdown behavior:
* Enable termination protection: The Sarah Connor option?

Private DNS auto-assigned, private IP also assigned.

Check VPC again and now DHCP options like the DNS hostnames can be configured.

Network ACL was auto-created with the VPC. Rule 100 allows all traffic. Default denies all traffic but won't get hit on this default, very open, config.

Creation order is:

1. Choose AMI
2. Choose Instance Type
3. Configure Instance
4. Add Storage
5. Tag instance
6. Configure Security Group

## Creating a Linux EC2 Instance

Services -> EC2 -> Launch Instance

When a VM instance is created we have these configuration options:

* Number of instances
* Purchasing option: (this is where Spot Instances would be used)
* Network: our VPC network (NOT subnet)
* Subnet: our new subnet
* Auto-assign public IP:
* IAM role: Allows AWS to manage key-based access
* Shutdown behavior: Stop or Terminate (auto-delete-self)
* Enable termination protection: often a good idea
* Monitoring:
* Tenancy:

Adding storage-- default for Root already there. Delete on termination-- removes EBS when the instance is deleted.

Tags: key/value pairs

Security Group controls network access into the instance. SSH from anywhere enabled by default.

SSH private key (PEM) must be safeguarded to ensure access to the instances.

## AWS Marketplace

Useful for finding specialized appliances.

LDAP? EVIL! :-)

Check the VPC settings-- may not be what we want.

The example e-commerce AMI includes a security group with port 80 and 443 open. (Which makes sense!)

## Creating a Windows EC2 Instance

New config item:

* Domain join directory: Joins a given AD domain

RDP on port 3389 open by default. [Uses public/private keys for auth](https://docs.aws.amazon.com/AWSEC2/latest/WindowsGuide/connecting_to_windows_instance.html) but it sure seems a lot more complicated than the ssh-right-in approach Linux uses. Key to get a password to embed in a config file?

## Connecting to an EC2 Instance

The EC2 instance list has a "connect" tab which lets you download the RDP file describing the connection to the instance. The "Get Password" button offers an option to show the password, but it's encrypted.

The `Key Pair Path` is the path to the PRIVATE key. Putting a private key in a web form seems like a bad idea in general and it's sad that this is common on AWS.

The PEM secret key needs to be (locally) converted to use PuTTY. Save without passphrase? Bad example! OK, well, at least he acknowledges the weakness. Maybe he needs to learn about `pageant.exe`.

Normal linux username: `ec2-user`

## Security Groups

### EC2 Security Groups Overview.

Virtual firewalls to control access to EC2 instances.

When the policy for a security group is changed, all instances in that group see the changes.

EC2-Classic (older config) had a single network shared by AWS customers. Security groups would be in the same region as the VM instances. There were at most 500 security groups per instance and max 100 rules per group. Can't change on running instances.

New method: EC2-VPC which uses those virtual private clouds to logically isolate each AWS customer. These rules can change on running instances. 5 groups per instance and 50 rules per security group.

Security groups control inbound and outbound network traffic. Outbound is allowed by default.

For ICMP the type would be defined (vs. port)

Default security group created for default VPC. Only allows inbound traffic from the default VPC, but allows all outbound.

A custom security group requires a name and description.

### Creating and Deleting an EC2 Security Group.

Services -> EC2 -> Network & Security -> Security Groups

(Launches a Windows instance where RDP is inaccessible by Security Group rules. Odd.)

Ah... he adds the rule later.

### Adding Rules to an EC2 Security Group.

Services -> EC2 -> Network & Security -> Security Groups

Custom TCP: opens up port range as an option

All traffic: might be useful to allow instances in different VPCs to intercommunicate.

"My IP" literally your current IP as seen on the Internet.

### Configuring EC2 Security Groups for VPC.

Services -> EC2 -> Network & Security -> Security Groups

OR

Services -> VPC -> Security -> Security Groups

Network ACLs might look like Security Groups but they apply to a subnet, not a VM. Security groups have no `DENY` option, but Network ACLs do.

The source can be another security group-- allows traffic from another VPC. Source can also be an IP address range, e.g. part of another VPC.

## Elastic Block Store (EBS)

### Elastic Block Store Overview.

Works like an unformatted hard drive for EC2 instances.

High Availability replicates EBS within an availability zone automaticaly.

Persistent storage even when instances are shut down.

Encryption possible provided the instance supports it. (Seems odd-- if the encryption is within EBS itself that should be transparent to the OS on the instance.)

EBS Types:

* Magnetic volume: 100 IOPS, 1TB max size
* General purpose: SSD, 3 IOPS/GB + 3000 IOPS burst, 16TB max size
* Provisioned IOPS: SSD, up to 20,000 IOPS and 320MB/s, 16TB max size

EBS snapshots, can create new EBS volumes from snapshot.

Unencrypted snapshots can be shared.

Shapshots must stay in the same region where they were created.

EBS encrypted volumes, each gets unique AES256 key. Managed via AWS, FIPS-approved (US government standard). Encrypted with a volume key and then an account key. Both would be needed to decrypt.

To encrypt data, move it to an encrypted volume.

Creation process Services -> EC2 -> Elastic Block Storage -> Volumes -> Create Volume

Creation options:

* Type: e.g. magnetic, provisioned IOPS
* Size: in GB
* IOPS: (if provisioned IOPS)
* Availability Zone: keep this the same zone as the VMs which will be using it
* Snapshot ID:
* Encryption:

### Creating a EBS Volumes.

EBS can exist independent from any VM instance.

Normally used for disk volumes within an instance that get frequent access.

Services -> EC2 -> Elastic Block Storage -> Volumes

Selecting an existing volume shows what instance it's attached to under "Attachment information."

Snapshot ID: source snapshot for creating the new volume (copy of snapshot)

Shows up as a new blank hard drive (Virtual SCSI device).

### Creating EC2 Snapshots.

Snapshots hold changed blocks (incremental)

EBS -> Volumes -> Actions -> Create Snapshot

If possible, shut down instance before creating snapshot of the root EBS volume.

Creating can take hours for large EBS volumes.

Can create a new EBS volume from a snapshot.

## EC2 Amazon Machine Images (AMI)

### Amazon Machine Images Overview.

Create new instances in EC2 from AMIs. The AMI defines a template for the root volume (e.g. base OS?), defines which accounts can use the AMI, and maps EBS to block devices within the instance OS image.

#### AMI Life Cycle.

Create an AMI

Use AMI to create an instance.

Modify instance if needed and "register" to create a new AMI. These can also be "deregistered" to remove them from future use.

The new AMi can also be used to launch new instances or copied directly to a new, independent AMI.

Shared AMIs are available. Sharing can be public or to specific AWS accounts.

We don't know how a public/shared AMI was created so we should use them with caution.

Amazon-created AMIs are named "amazon"

AMIs can be purchased or sold. Marketplace has categories and search.

What actions appear on the `Actions` menu for an AMI?

### Creating an AMI in EC2.

"What if we find ourselves making the same changes once we launch an instance from an AMI?" Me: It doesn't matter since those changes are all documented and automatically applied using modern Infrastructure as Code methods, which allow for easy, versioned, granular changes rather than baking all the changes into one giant, hard-to-adjust blob.

Me: Creating AMIs for known-source-traceability seems like a much better reason for making an AMI.

EC2 -> Instances -> Actions -> Image -> Create Image

Can create from an EBS snapshot.

RHN subscriptions not maintained through snapshots? That seems odd-- I would expect it to be duplicated since the machine registration is stored on the EBS volume. Perhaps they mean that RedHat detects the duplication and disables one or both of the subscriptions?

Windows can also have problem when starting from a snapshot.

Advise that it's best to create an AMI from an instance rather than a snapshot.

Copy AMI copies to another region.

### Importing and Exporting an AMI in EC2.

Import can allow us to re-use an existing virtual machine (hosted at Amazon.)

Also can be used for DR.

Managed via CLI or API.

Preparing the source VM requires things like:

* Remove security software (antivirus / IDS)
* Disconnect virtual DVD-ROMs since backing storage probably doesn't exist in AWS
* Reconfigure for DHCP (optional, but allows immediate instance access that way.)
* Remove VMware tools or other hypervisor-specific software
* Sysprep (Windows) or equivalent (RHEL) (optional)

Sounds more like making a template than moving a VM...

Export it from existing environment, make an OVF.

Upload to AWS via `aws ec2 import-image --cli-input-json ...`

(Based on the test, I had that wrong-- the upload precedes the import but I didn't notice how they did it in the video.)

Export from EC2 via `aws ec2 create-instance-export-task`.

## RAID on EBS

### EBS RAID Configuration Options.

Software RAID within the OS.

(This seems like a terrible idea for AWS images. Isn't the EBS already redundant/reliable? We should avoid making our instances individually tough and instead focus on making them easily replaced.)

RAID-0 might have a place if we have a single image that requires a truly ridiculous amount of storage throughput. The Measured IOPS caps at 20,000 / 320MB/s per EBS. Would we really be able to get more with software RAID-0 or would other internal limit be hit? (e.g. virtual SCSI limits.)

I think that there are probably better solutions for the problems software RAID within AWS is trying to solve. (E.g. high performance databases.)

Overall this doesn't seem AWS-specific and should probably be avoided on AWS as a general rule.

### Creating an EBS RAID Array on Linux.

`fdisk` really? The Linux RAID auto is quite optional. Whole disks can be used for the `md` device, though we probably want to use one of the static `/dev/disk/by-id` devices or a `disk-mapper` device.

`mdadm --create` for creating new `md` device

`mdadm --detail` for info about the device

`mkfs` on bare device? It hurts! If you're going to all this trouble, at least use LVM. (Well, at least he didn't partition the `md` device...)

### Creating an EBS RAID Array on Windows.

Server Manager -> Tools -> Computer Management -> Disk Management

Right click new disk -> New Mirrored Volume

Add both disks to "selected", choose drive letter

Add a label. Convert to dynamic disks.

## Monitoring

### Configuring EC2 Load Balancers.

Sits between our service instances (e.g. web servers) and the Internet and spreads the load across the service instances. Users connect to the load balancers.

Process:

1. Define Load Balancer
2. Assign Security Groups
3. Configure Security Settings
4. Configure Health Check
5. Add EC2 Instances
6. Add Tags

Define load balancer: needs to know the protocol + port for the incoming requests ("load balancer protocol") and the same for your back-end instances.

The load balancer subnets need to be in at least two different availability zones. This is not the instance back-end subnet, but the subnets for the front-end connections end users will be directed into. (They look to be NATted so there's something else going on too.)

Create security group to allow appropriate access. (e.g. TCP port 80 from anywhere.)

Health checks: make sure back end instances are healthy via HTTP GET. (Would it be useful to have a combination of frequent "simple" health checks and infrequent "complex" health checks?)

We should create a DNS record (CNAME) to the load balancer DNS name. (That can be slow since it requires two DNS lookups from separate domains to complete. Is there a better way?) Route53 allows an "Alias" A record to be created directly to the ELB: [Route 53 docs](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/routing-to-elb-load-balancer.html)

### Performing EC2 Health Checks.

Is an instance running properly or not?

Demo on how to filter by state when viewing instances in the web interface.

"Monitoring" has some per-instance performance metrics and history.

Auto-scaling lets us auto-create instances in response to monitoring events.

Seems like a big emphasis on instance resource monitors rather than measuring application health. Probably because it's simple and easy, not because it's useful. ["The light is better over here."](https://en.wikipedia.org/wiki/Streetlight_effect)

EC2 Actions possible:

* Rebooot instance: stays on same underlying hardware
* Recover instance: moves instance image to new hardware
* Stop instance
* Terminate instance (delete)

### Monitoring EC2 Instances with CloudWatch.

Allow for tracking and alerting of metrics on resources.

The metric graphs could be helpful for troubleshooting a specific performance issue, but really are best used in conjunction with application-level information and metrics. E.g. What is the application waiting for when performance is poor?

Defining alarms: "consecutive periods" what is that? Defined at the lower right. "period"

EC2 Actions could be useful (e.g. auto-reboot to clear memory leaks.)

## Practice: Configuring EC2.

### AWS CLI test.

How to test credentials for AWS Command Line Tools

`aws sts get-caller-identity`

### Create a VPC and Security Group.

```awspec
describe vpc('ec2-overview-vpc') do
  it { should exist }
end

describe security_group('ec2-overview-sg') do
  it { should exist }
  its(:outbound) { should be_opened }
  its(:inbound) { should be_opened(22) }
end
```

### Create an AMI from an EC2 instance.

```awspec
describe ec2('ec2-overview-ec2-01') do
  it { should belong_to_vpc('ec2-overview-vpc') }
  it { should have_security_group('ec2-overview-sg') }
end

describe ami('ec2-overview-ami') do
  it { should exist }
end
```

### Create three EBS volumes.

```awspec
describe ebs('ec2-overview-ebs01') do
  it { should exist }
  it { should be_attached_to('ec2-overview-ec2-01') }
end
describe ebs('ec2-overview-ebs02') do
  it { should exist }
  it { should be_attached_to('ec2-overview-ec2-01') }
end
describe ebs('ec2-overview-ebs02') do
  it { should exist }
  it { should be_attached_to('ec2-overview-ec2-01') }
end
```