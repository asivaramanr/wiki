# **Autoscaling using EC2**

Before looking at the overview of Autoscalling, lets configure the ASW-CLI to deploy our resources without accessing the console. 

## AWS Command Line

### Installing the AWS CLI on Windows.

Click [here](https://awscli.amazonaws.com/AWSCLIV2.msi) to download the latest AWSCLIV2.msi

### Installing the AWS CLI on Linux.

Amazon Linux AMI includes Amazon AWS CLI. Handy!

Linux install doesn't use OS packaging.

Also a "pip" based option which might be slightly better than a script-based install.

AWS CLI needs Java 1.7.0 or later.

Possibly helpful env vars:

* `EC2_BASE`: install dir for the utilities
* `EC2_HOME`: `$EC2_BASE/tools` (parent of the `bin` directory)
* `AWS_ACCESS_KEY`: key ID used for CLI access
* `AWS_SECRET_KEY`: secret key used for CLI access (security risk having it as ENV)
* `EC2_URL`: includes region, e.g. `https://ec2.us-west-2.amazonaws.com`
* `PATH`: `$PATH:$EC2_HOME/bin`
* `JAVA_HOME`: path to Java install location

Verify setup via `ec2-describe-regions` or `ec2-describe-instances`

### Configuring the AWS CLI.
```
aws configure
```
It will ask for below information:

* `AWS Access Key ID [****************EAFY]`:
* `AWS Secret Access Key [****************5m37]`:
* `Default region name [US West (Oregon)]: us-west-2`:
* `Default output format [None]: json`

Use your own Key ID and Secret Access Key, preferably for a non-root account.

`aws ec2 help`: more info about EC2 commands

`aws ec2 describe-instances`: lots of instance details

### Creating an IAM Role for EC2 with AWS CLI.

Create a JSON file describing the role, a trust policy file.

`aws iam create-role --role-name <name> --assume-role-policy-document file://filename.json`

This might also be helpful for creating policies (mentioned in another course):

```ascii
"--generate-cli-skeleton" (string) Prints a JSON skeleton to standard
output without sending an API request. If provided with no value or
the value "input", prints a sample input JSON that can be used as an
argument for "--cli-input-json". If provided with the value "output",
it validates the command inputs and returns a sample output JSON for
that command.
```

Example:

```bash
$ aws iam create-role --generate-cli-skeleton
{
    "Path": "",
    "RoleName": "",
    "AssumeRolePolicyDocument": "",
    "Description": "",
    "MaxSessionDuration": 0,
    "PermissionsBoundary": "",
    "Tags": [
        {
            "Key": "",
            "Value": ""
        }
    ]
}
```

`aws iam list-roles`: shows all our roles

### Configuring an IAM Role for EC2 with AWS CLI.

Show permission policies assigned to a role with
`aws iam list-attached-role-policies --role-name <name>`

Example `aws iam attach-role-policy --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess --role-name WebApp`

### Launching an Instance with an IAM Role Using AWS CLI.

Instance Profile: group for storing roles

Example: `aws iam add-role-to-instance-profile --instance-profile-name s3readonly --role-name WebApp`

In the GUI on step 3 of configuring an instance, there's the option for "IAM role" to use.

`aws ec2 run-instances --image-id ami-<stuff> --count 1 --key-name <key pair name> --instance-type t2.micro --iam-instance-profile Arn=arn:aws:iam:<account id>:instance-profile/<instance profile name>`

The permissions in the policy will be applied to software running within this instance, for example the `aws` CLI.

Above CLI created a nameless instance. Correct that with:

`aws ec2 create-tags --resources <ID> --tags Key=Name,Value=<instance name>`

## Boot Scripts

### Writing Shell Scripts.

When creating instance:

Configure Instance -> Advanced Details -> User Data

Add bash commands to run including shebang line. (Terrible example of allow world-write to a directory in his example.)

(Don't need sudo to read /etc/passwd.)

### Configuring the `cloud-init` Directive.

Configure Instance -> Advanced Details -> User Data

```YAML
#cloud-config
repo_update: true
repo_upgrade: all

packages:
 - httpd
runcmd:
 - chkconfig httpd on
 - service httpd start
```

Hope that RHEL7 server isn't using systemd... oh, wait... **it does!**

Should instead be:

```YAML
runcmd:
 - systemctl enable httpd
 - systemctl start httpd
```

Or:

```YAML
runcmd:
 - systemctl enable --now httpd
```

## Auto Scaling Groups

### Planning EC2 Auto Scaling.

Launch Configuration: configuration of the instances used for scaling up

Scaling plans: defines when to scale and how to scale

Launch config set up when the group is initially created. Describes how the auto-added instances will be configured.

Only one launch configuration per auto scaling group. Launch configuration cannot be changed. (Destroy and re-create the scaling group instead?) Looks like a new Launch Configuration would be created and the new Configuration assigned to the Auto Scaling Group.

Launch config contains:

* AMI to use
* Instance type
* Other instance config items to use when new instances are created

Scaling plan can do things like:

* Maintain a given number of instances
* Re-create instances to replace unhealthy ones
* Manual or automatic creation
* Schedule-based scaling
* Demand-based scaling (E.g. instance CPU usage.)

### Creating a Launch Configuration.

Can create a Launch Configuration from an existing instance using EC2 -> Instances -> Actions -> Attach to Auto Scaling Group

EC2 -> Auto Scaling -> Launch Configurations -> Create Auto Scaling Group first screen is "Create a new launch configuration" vs. "Create an Auto Scaling group from an existing launch configuration"

### Creating a Launch Configuration from an Instance.

EC2 -> Instances -> Actions -> Attach to Auto Scaling Group

Prompted to create a new Auto Scaling group or change the Launch Configuration for an existing Auto Scaling group. (I bet the latter would make a heck of a mess if done unexpectedly.)

EC2 -> Auto Scaling -> Launch Configurations shows the config copied from the instance we used above.

Also have a "Copy Launch Configuration" button to create yet another launch configuration based on this one which was based on an instance.

### Creating an Auto Scaling Group.

Groups of healthy servers that serve up identical content.

EC2 -> Auto Scaling -> Auto Scaling Groups -> "Create Auto Scaling Group"

Steps to create a Launch Configuration:

1. Choose AMI
2. Choose Instance Type
3. Configure details
4. Add Storage
5. Configure Security Group
6. Review

Steps to create an Auto Scaling Group:

1. Configure Auto Scaling group details
2. Configure scaling policies
3. Configure Notifications
4. Configure Tags

Group Details include

* Name
* Initial group size: (number of instances)
* Network to use
* Subnet(s) to use
* Advanced Details: Load Balancing -> use an Elastic Load Balancer + health check details

Scaling policies: either keep the group at its initial size or use scaling policies to adjust the capacity

* Scale between `initial size` and `max size` instances
* Increase group size:
  * Name
  * Execute policy when: (alarm definition)
  * Take the action: Add (or Set to) `N` instances
* Decrease group size:
  * Name
  * Execute policy when: (alarm definition)
  * Take the action: Remove `N` instances

Putting the instances on a schedule is done using the AWS CLI.

### Creating an Auto Scaling Group from an Instance.

EC2 -> Instances -> Actions -> Attach to Auto Scaling Group

New or existing Auto Scaling group.

Instance cannot be part of another Auto Scaling group. Must be in the same Availability Zone as the Auto Scaling group. he AMI used to launch this instance must still be available.

Scaling Policy created from new, including alarm creation.

### Linking an Instance to a VPC.

EC2 -> Launch Instance

On step 3 (Instance Details) we can choose a VPC to contain the Instance.

VPC defined when instance is launched, not editable. Can create an AMI then launch that AMI into a new VPC.

### Tagging Auto Scaling Groups.

Auto Scaling group tags can be auto-added to the new instances created from scaling actions.

`aws autoscaling create-or-update-tags --tags` via CLI.

`aws autoscaling describe-tags`

Up to 10 tags can be added per Auto Scaling group.

## Configuring Auto Scaling Groups

### Load Balancing an Auto Scaling Group.

Having a method to spread load across a group that varies in size makes sense-- that's how the extra capacity gets used.

Want to be sure we're using a Launch Configuration using an AMI with our website content already included.

Health Check type can be EC2 or ELB.

"OutOfService" could mean the servers are broken or it might just be that the checks haven't completed yet.

### Attaching an Instance to an Auto Scaling Group.

Can't attach more than the max instance to an Auto Scaling Group.

EC2 -> Instances -> Actions -> Instance Settings -> Attach to Auto Scaling Group

### Detaching an Instance from an Auto Scaling Group.

EC2 -> Instances -> Actions -> MISSING

EC2 -> Auto Scaling -> Instances -> Checkbox -> Lower Actions -> Detach

### Suspending and Resuming an Auto Scaling Group.

The instances stay running, but the automated actions like launching new instances, terminating old instances, and performing health checks will not take place while suspended.

No web GUI method to do this-- CLI only.

`aws autoscaling suspend-process --auto-scaling-group-name`

`aws autoscaling resume-process --auto-scaling-group-name`

### Shutting Down an Auto Scaling Group.

Shut down = delete. Permanent.

EC2 -> Auto Scaling -> Actions -> Delete

Launch Configurations remain behind. Load Balancer also not deleted. **Instances will be terminated (deleted) when the group is deleted**

## Placement Groups

### Overview of Placement Groups.

Group of instances within a single Availability Zone.

Enable 10Gig low-latency network connectivity between specialized "enhanced networking" instances. The instance OS must also support this type of "enhanced networking."

Placement groups cannot be merged later on.

Placement group created first. Then instances are launched within the placement group. Use the same instance type within the group. Existing instances cannot be moved to a placement group. (But an AMI could be created from an existing instance and that AMI used to launch new instances within the group.)

Instance types allowed in a placement group:

* General Purpose
  * m4.large through m4.10xlarge
* Compute Optimized
  * c4.large through c4.8xlarge
  * c3.large through c3.8xlarge
  * cc2.8xlarge
* Memory Optimized
  * cr1.8xlarge
  * r3.large - r3.8xlarge
* Storage Optimized
  * d2.large - d2.8xlarge
  * hi1.4xlarge
  * hs1.4xlarge
  * i2.xlarge - i2.8xlarge
* GPU Optimizaed
  * cg1.4xlarge
  * g2.2xlarge - g2.8xlarge

### Creating a Placement Group.

EC2 -> Network & Security -> Placement Groups

Only parameter: account-wide unique name for the Placement Group

Strategy: "cluster" (what does that mean?)

Choosing an instance type during creation that can't go into the placement group means the "Placement Group" option in the "Configure Instance" step won't appear.

### Launching an Instance into a Placement Group.

(If a "moderate" network performance instance like c4.large is put in a placement group does that performance change?)

Provided the right instance type is chosen we have the option to put it into a placement group during the "Configure Instance" step.

### Deleting a Placement Group.

Can't delete a placement group that contains instances.

Actions -> instance state -> terminate

Placement Groups -> Delete Placement Group

## Practice: Auto Scaling

### Create a Lunch Configuration.

Services -> EC2 -> Auto Scaling -> Launch Configurations -> Create launch configuration

Steps:

1. Choose AMI

   Using "Amazon Linux 2 AMI" (ami-0cb72367e98845d43)

2. Choose Instance Type

   t2.micro (keep it free!)

3. Configure details

   name: `ec2-autoscaling-launch-config`
   * (others left at defaults)

4. Add Storage

   Using `snap-05f1f2daed90efa18` by default, presumably part of the Amazon AMI. Leave default, including "delete on termination" since auto-created and auto-terminated instances are treated as disposable.

5. Configure Security Group

   1. Assign a security group: Create a new security group
   2. Security group name: `ec2-autoscaling-sg`
   3. Description: `For training autoscaling exercise`
   4. Type: SSH
   5. Source: My IP

6. Review

   1. Create a new key pair
   2. Key pair name `ec2-autoscaling-keypair`
   3. Use PuTTYgen to convert to OpenSSH format with passphrase
   4. Add key to "pageant" so you can stop typing the passphrase all the time

NOTE: It looks like Amazon wants us to use
[Launch Templates](https://docs.aws.amazon.com/autoscaling/ec2/userguide/LaunchTemplates.html) instead.

```quote
Defining a launch template instead of a launch configuration allows you to have multiple versions of a template. With versioning, you can create a subset of the full set of parameters and then reuse it to create other templates or template versions. For example, you can create a default template that defines common configuration parameters such as tags or network configurations, and allow the other parameters to be specified as part of another version of the same template.

We recommend that you use launch templates instead of launch configurations to ensure that you can use the latest features of Amazon EC2, such as T2 Unlimited instances.
```

### Create a Launch Template.

Beforehand:

1. Create Security Groups
   * `ec2-autoscaling-web-sg` for HTTP/HTTPS only from anywhere
   * `ec2-autoscaling-sg` for SSH only + HTTP/HTTPS from above `ec2-autoscaling-web-sg` web security group to allow ELB traffic into the instances.Type `sg` into the source field to bring up a drop-down of security groups. Magic!
2. Create Key Pair `ec2-autoscaling-keypair`
3. Create Network Interfaces `ec2-autoscaling-nic01`
   * Subnet: picked the one on us-west-2a
   * Security Group: `ec2-autoscaling-sg`
   * Copy/paste ID for below config `eni-0897f5a5b47381d07`
   * This didn't seem to be used yet I was prompted to create one when it didn't exist. Odd.

Services -> EC2 -> Instances -> Launch Templates -> Create Launch Template

Create launch template

* Launch template name: `ec2-autoscaling-launch-template`
* Template version description: `initial version`
* Source template: None (default)
* AMI ID: `ami-0cb72367e98845d43` (found via "Search for AMI")
* Instance type: `t2.micro` (selection doesn't show free tier as easily.)
* Key pair name: `ec2-autoscaling-keypair`
* Security Groups: `ec2-autoscaling-sg` (doesn't seem to have as easy an option for creating new Security Groups)
* Network Interfaces: (lots of off-screen horizontal issues here with horizontal scrolling only possible when vertically scrolled to the bottom)
  * Provided we already created a network interface this section doesn't appear to be required.
  * This seems like something that should be handled with sensible defaults on auto-created instances.

TODO: AMI doesn't have a webserver running by default. Need to enable it or the ELB health check fails.

#### Edit Launch Template.

Add this to `User Data`:

```bash
#!/bin/sh
yum -y install httpd
systemctl enable --now httpd
echo '<h1>Your Autoscaling lab exercise worked!</h1>' > /var/www/html/index.html
```

#### Edit Auto Scaling Group.

Change the Launch Template version to `$Latest`

### Create an Auto Scaling Group.

Services -> EC2 -> Auto Scaling -> Auto Scaling Groups

Choose `ec2-autoscaling-launch-template`

1. Configure Auto Scaling group details

   * Group name: `ec2-autoscaling-group`
   * Subnet: (use the us-west-2a default)

2. Configure scaling policies

   * Keep this group at its initial size (we'll set up scaling later on to avoid a chicken-and-egg problem with setting up the Elastic Load Balancer.)

3. Configure Notifications

   * Add notification
   * Send notification to `ec2-autoscaling-notify`
   * With these recipients: `<add E-mail addresses here>`
   * Whenever instances: (leave default)

4. Configure Tags

   * No tags needed for this exercise
   * Perhaps a name tag could have been useful?

5. Review

### Attach Instances.

Services -> EC2 -> Instances

Already one instance running since the Auto Scaling Group auto-creates one instance to start.

### Load Balance the auto scaling group.

Services -> EC2 -> Load Balancers -> Application Load Balancer -> Create

1. Configure Load Balancer

   1.1 Basic Configuration

   * Name: `ec2-autoscaling-elb`
   * Scheme: internet-facing (default)
   * IP address type: IPv4 (default)

   1.2 Listeners

   * HTTP: port 80 (default)

   1.3 Availability Zones

   * VPC: (use default)
   * Availability Zones:
      * `us-west-2a` IPv4 address assigned by AWS (default)
      * `us-west-2b` IPv4 address assigned by AWS (default)

2. Configure Security Settings

   (I would need to use a registered domain in order to make use of [Amazon AWS Certificate Manager](https://docs.aws.amazon.com/acm/latest/userguide/acm-overview.html))

3. Configure Security Groups

   Use `ec2-autoscaling-web-sg` to allow HTTP/HTTPS

4. Configure Routing

   * Target group: new target group
   * Name: `ec2-autoscaling-elb-target`

5. Register Targets

   * Add the one instance running from the autoscaling auto-creation

6. Review

Services -> EC2 -> Auto Scaling -> Auto Scaling Groups -> Edit

* Target Groups: `ec2-autoscaling-elb-target`
* Health Check Type: ELB

### (Optional) Change scaling policy to use ELB.

* Use scaling policies to adjust the capacity of this group
* Scale between 1 and `4` instances.
* Name: `ec2-autoscaling-load-policy`
* Metric type: `Application load balancer request count per target`
* Target value: `1000` (this is arbitrary, in a real deployment this would be determined by measuring application performance as connections went up.)
* Instance need: `300` seconds to warm up after scaling (also arbitrary, but slamming an instance that's still booting isn't going to give good performance results.)

#### security group checks.

`awspec generate security_group <VPC ID>`

This ends up with group IDs hardcoded. I changed the ELB security group to `:ELB_security_group` and my source IP + '/32' to `:admin_source_IP` in the below example. The group ID in the second group should match the `:ELB_security_group`. There's probably a way to tell `awspec` how to to do that automatically, but I don't yet know how.

```bash
describe security_group('ec2-autoscaling-sg') do
  let(:admin_source_IP) { '1.2.3.4/32' }
  let(:ELB_security_group) { 'sg-0aeb724c8e35d7d91' }

  it { should exist }
  its(:group_name) { should eq 'ec2-autoscaling-sg' }
  its(:inbound) { should be_opened(80).protocol('tcp').for(:ELB_security_group) }
  its(:inbound) { should be_opened(22).protocol('tcp').for(:admin_source_IP) }
  its(:inbound) { should be_opened(443).protocol('tcp').for(:ELB_security_group) }
  its(:outbound) { should be_opened.protocol('all').for('0.0.0.0/0') }
  its(:inbound_rule_count) { should eq 3 }
  its(:outbound_rule_count) { should eq 1 }
  its(:inbound_permissions_count) { should eq 3 }
  its(:outbound_permissions_count) { should eq 1 }
end

describe security_group('ec2-autoscaling-web-sg') do
  it { should exist }
  its(:group_id) { should eq 'sg-0aeb724c8e35d7d91' }
  its(:group_name) { should eq 'ec2-autoscaling-web-sg' }
  its(:inbound) { should be_opened(80).protocol('tcp').for('0.0.0.0/0') }
  its(:inbound) { should be_opened(443).protocol('tcp').for('0.0.0.0/0') }
  its(:outbound) { should be_opened.protocol('all').for('0.0.0.0/0') }
  its(:inbound_rule_count) { should eq 2 }
  its(:outbound_rule_count) { should eq 1 }
  its(:inbound_permissions_count) { should eq 2 }
  its(:outbound_permissions_count) { should eq 1 }
end
```

#### autoscaling group checks.

`awspec generate autoscaling_group <VPC ID>`

The subnet IDs won't translate to other environments, but are probably worth keeping in place for easy changing to new VPCs.

```bash
describe autoscaling_group('ec2-autoscaling-group') do
  it { should exist }
  its(:auto_scaling_group_name) { should eq 'ec2-autoscaling-group' }
  its(:min_size) { should eq 1 }
  its(:max_size) { should eq 4 }
  its(:desired_capacity) { should eq 1 }
  its(:default_cooldown) { should eq 300 }
  its(:availability_zones) { should eq ["us-west-2a", "us-west-2b"] }
  its(:health_check_type) { should eq 'ELB' }
  its(:health_check_grace_period) { should eq 300 }
  its(:vpc_zone_identifier) { should eq 'subnet-4ac1c633,subnet-af8cade4' }
  its(:termination_policies) { should eq ["Default"] }
  its(:new_instances_protected_from_scale_in) { should eq false }
  its('launch_template.launch_template_name') { should eq 'ec2-autoscaling-launch-template' }
  it { should have_alb_target_group('ec2-autoscaling-elb-target') }
end
```

#### Elastic Load Balancing checks.

`$ awspec generate elb <VPC ID>`

No output produced. Odd.

`$ aws elb describe-load-balancers` has no net output:

```JSON
{
    "LoadBalancerDescriptions": []
}
```

However, that's because we're using ELBv2 (newer):

```json
$ aws elbv2 describe-load-balancers
{
    "LoadBalancers": [
        {
            "LoadBalancerArn": "arn:aws:elasticloadbalancing:us-west-2:146679865661:loadbalancer/app/ec2-autoscaling-elb/e3b9918dc5f21a15",
            "DNSName": "ec2-autoscaling-elb-1546742660.us-west-2.elb.amazonaws.com",
            "CanonicalHostedZoneId": "Z1H1FL5HABSF5",
            "CreatedTime": "2019-06-04T18:11:27.940Z",
            "LoadBalancerName": "ec2-autoscaling-elb",
            "Scheme": "internet-facing",
            "VpcId": "<VPC ID>",
            "State": {
                "Code": "active"
            },
            "Type": "application",
            "AvailabilityZones": [
                {
                    "ZoneName": "us-west-2b",
                    "SubnetId": "subnet-4ac1c633"
                },
                {
                    "ZoneName": "us-west-2a",
                    "SubnetId": "subnet-af8cade4"
                }
            ],
            "SecurityGroups": [
                "sg-0aeb724c8e35d7d91"
            ],
            "IpAddressType": "ipv4"
        }
    ]
}
```

Maybe it's just a documentation issue?

```bash
$ awspec generate elbv2 vpc-5468c22c
Could not find command "elbv2".
Did you mean?  "elb"
```

Nope.

It looks like ELBv2 goes by the alias "ALB" as seen in this [awspec github issue](https://github.com/k1LoW/awspec/issues/203).

`awspec generate alb <VPC ID>`

```bash
describe alb('ec2-autoscaling-elb') do
  it { should exist }
  its(:load_balancer_name) { should eq 'ec2-autoscaling-elb' }
  its(:scheme) { should eq 'internet-facing' }
  its(:type) { should eq 'application' }
  its(:ip_address_type) { should eq 'ipv4' }
end
````

There it is! Interesting that the security group info isn't there.

### Final Check: Go to the ELB public IP and confirm we see the Apache page

Success shows "Your Autoscaling lab exercise worked!"

Failures might include:

* 503: Service Unavailable

  * When no targets on the back end

* 504: Gateway Time-out

  * Immediately after targets added to ELB
  * Instance security group doesn't allow ELB to reach port 80
  