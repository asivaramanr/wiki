# **Networking and Best Practice**

## Route 53

### Route 53 Overview.

Amazon's DNS service.

Elastic: easily changed with new records added/removed

Fault tolerant: can handle the loss of availability zones and still function

Geo redundant: available in multiple regions

Offers three services:

1. DNS name resolution
2. DNS domain registration (or migration)
3. Health checking

Global network of authoritative DNS servers (those responsible for a whole domain.)

Route 53 can route traffic to the nearest edge location based on the client location (lowest latency.)

Can transfer other domains. Includes support for newer domains (e.g. `.technology`)

For health checks, route 53 can check out app to ensure it's reachable/available as well as functional.

### Registering a Domain through Route 53.

Services -> Networking -> Route 53 -> Registered Domains -> Register Domain

"Pending Requests" shows registration in progress

Eventually "Hosted Zones" will show our registered domain.

An `SOA` and `NS` record are created by default.

### Configuring Route 53 as your DNS Provider.

Convert Amazon public DNS names to something friendlier.

One approach: create 'A' record for the public IP of an EC2 instance.

## Virtual Private Cloud

### Create Virtual Private Cloud.

Used for network isolation, similar to VLANs.

Services -> VPC -> Create VPC

Configuration:

* Name tag
* CIDR block (up to /16)
* Tenancy (default or dedicated)

Once created we can see the VPC's DHCP options set, Route table, and Network ACL in the Summary tab.

DHCP Options Set parameters:

* Name tag
* Domain name
* Domain name servers
* NTP servers
* NetBIOS name servers
* NetBIOS node type

Network ACL allows both allow and deny rules. Similar config to security groups.

### Configure Virtual Private Cloud Subnets.

Can create subnets to separate resources by IP address range.

Services -> VPC -> Subnets -> Create Subnet

Configuration items:

* Name tag
* VPC
* Availability zone
* CIDR block

Subnets inherit the route table and network ACL from parent VPC.

To allow subnet to auto-assign public IPs use Subnet Actions -> Modify Auto-Assign Public IP. Still need an Internet Gateway configured on the VPC to allow Internet traffic in.

Assign an Internet Gateway to the VPC using VPC -> Internet Gateways.

The VPC route table will show a destination of 0.0.0.0/0 via the newly added Internet Gateway.

### Launch an EC2 Instance into a Virtual Private Cloud.

Services -> EC2 -> Create

Step 3, "Configure Instance" has a "Network" field for choosing the VPC.

New Windows instance can't seem to get to the Internet. DHCP options set a custom DNS earlier. Instead we should use the Amazon provided DNS unless there's a compelling reason not to.

### Configuring VPC Network Address Translation.

Allows VPCs to access the Internet without using public IPs.

VPC creation process has four initial configurations:

1. VPC with a Single Public Subnet
2. VPC with Public and Private Subnets
3. VPC with Public and Private Subnets and Hardware VPN access
4. VPC with Private Subnet Only and Hardware VPN access

To demo NAT, use option 2. Their mini-diagram shows NAT between the public and private subnets, but wouldn't the NAT really be between the public subnet and the Internet? The public subnet still uses private IPs internally, right?

Config items for basic architecture 2:

* IP CIDR Block
* VPC Name
* Public subnet
  * Availability zone
  * Public subnet name
* Private subnet
  * Availability zone
  * Public subnet name
* NAT instance
  * Instance type
  * Key pair name
* S3 endpoint subnet
* Enable DNS hostnames
* Hardware tenancy

New EC2 instance created based on an Amazon vpc-nat AMI.

Sample EC2 instance created in the private subnet can't be reached from the Internet but can reach the Internet via NAT.

SSH (PuTTY) can be used to reach NAT. And he finally shows us "pageant" to manage keys. Yay! If we can't reach the NAT instance, check security group.

### Configuring VPC Access Control Lists.

Apply to all traffic entering/leaving subnets within the VPC. Security groups can work similarly, but for specific instances.

The Network ACL associated with a VPC is in the VPC summary when the VPC is selected from the VPC list.

Example: set ACL to deny ICMP outbound traffic

## Direct Connect

### Direct Connect Overview.

Allows direct physical connections to private networks on AWS.

Amazon hardware on premises with amazon-provided network link. Traffic bypasses Internet entirely.

1 or 10 Gigabits available.

Great for large volume transfers, real-time data, and added data security.

Typically the Amazon endpoint is in a DMZ to control reverse access from AWS. Sometimes, depending on the physical location, the actual connection is to an Amazon partner who in turn connects directly to AWS.

Direct connect needs at least one virtual interface per VPC within AWS that will receive Direct Connect traffic. Additonal virtual interfaces might be needed for public AWS resources.

Direct connect operates at Layer 2. The AWS routing must be configured into the internal on-premises routers.

Cross Connect can allow teaming of multiple Direct Connect links.

### Creating and Deleting a Connection.

Services -> Networking -> Direct Connect -> Get Started with Direct Connect

us-west-2 Locations include (from live AWS, not video):

* EdgeConnex Hillsboro OR
* Pittock Block, Portland OR

Since this is a physical install, we would provide contact info and AWS will send a "letter of authorization" to the physical address asking for specific information related to the physical install. For example, we would provide info about our on-premises router. (New method appears to be a download of the LOA which we would take to an APN partner like the two above.)

Provision virtual interfaces for public AWS resources or our private VPCs.

To remove direct connect:

1. Delete virtual interfaces
2. Delete Direct Connect config
3. Cancel cross connect network services

### Working with Virtual Interfaces.

Need to have a Direct Connect connection configured to create Virtual Interfaces.

Public virtual interfaces allow us to use public AWS resources (e.g. S3, Glacier) using their public IPs but routed over that Direct Connect connection.

Private virtual interfaces let us reach a VPC using private IP addresses. One virtual interface per VPC.

Virtual interfaces (layer 2) use 802.1Q VLAN tags and can apply to the whole AWS cloud or a single specific VPC.

Routing handled via Border Gateway Protocol (BGP.)

The AWS console can provide router-specific configuration details needed to make the virtual interface work with our on-premises router.

Helpful to have at least one EC2 instance per VPC running so that simple connectivity testing can be done.

## Cloud Best Practice

### Design for Failure.

Assume things will fail (a common assumption for high availability.)

Design for automated recovery. For example, Auto Scaling groups can terminate misbehaving instances.

Where are the single points of failure? For example:

* Load balancers
* Cluster master nodes

What happens when one of those single points of failure dies? How do we know when it has failed? (What about poor performance vs. hard failure?) How will the failed node be replaced?

Auto Scaling groups is a common way to auto-create new nodes when others fail.

How does a failover happen? How will users be impacted during a failover event?

How does the application handle lack of response from a dependent service? (e.g. app server no longer getting responses from its back-end database.) Applications need to have built-in exception handling.

How are changes in dependent service APIs handled?

Cloud architecures should handle rebooting or relaunching (new HW) an instance without the whole service going offline.

For SQS and SimpleDB plan around having a controller thread restart on failure. Ensure apps handle the controller restart appropriately.

Use Elastic IPs for public access-- move IP from a dead instance to a functional one.

Keep AMIs with customization on hand to quickly restore an environment from the AMI.

Use multiple Availability Zones to guard against AZ failure.

EBS and automated snapshots can allow for a limited roll-back ability, but additional backups of some sort are also needed.

CloudWatch should be used to monitor resource capacity.

RDS automated backups should be used to allow for recovery of accidentally removed data.

### Decouple Components.

Loosely coupled components scale better.

Build components (apps) that tolerate other components which:

* Fail
* Pause
* Respond slowly

Components are treated as "black boxes" with the internals hidden. They can be re-used in different contexts by following the published access methods.

Use asynchronous communications rather than waiting for a (possibly slow) reply.

No dependencies across application layers.

One example of loose coupling-- replace direct API calls with message queues.

Add components to an AMI so they can be launched and immediately used. (Me: automate the creation of AMIs from known good base components so we can easily make repeatable changes to the underlying AMI.)

Design components with "service interfaces" where things like a timeout can be added to reduce tight coupling.

Make applications stateless so if the back-end app dies, its state loss doesn't matter.

Use SQS to isolate apps from one another and buffer their communications, reducing tight coupling.

### Implement Elasticity.

Allows rapid add/remove of AWS resources.

Auto Scaling can be based on demand or proactively scheduled. (e.g. seasonal variation, in response to marketing campaign, etc.)

Automation is a big advantage that cloud has built in. Making deployments automated is critical to getting the best advantage from the cloud. Automation reduces errors and inconsistencies and enables easier scaling.

Chef recipes can help with automating the configuration of the EC2 instance contents themselves. (e.g. users, packages, third party software, etc.)

Deploy with custom AMIs which could contain specific instance configurations.

Bootstrapping sets up the Chef client within the OS of the instance so that further customization can be done more easily. The Chef client will pull any client-image-specific changes and customizations that are needed and apply them.

Bootstrapping also allows instances to be attached to clusters. (What kind of clusters? Unclear...)

Can auto-provision more storage when needed (is this the only way?)

Environments like development, staging, and production are easily re-created and deployment errors are reduced when using Bootstrapping.

App components should be location-agnostic (part of making them loosely coupled.)

Consider using Just Enough Operating System (JeOS), a minimally configured host OS. Similar to virtual appliances. This can be managed using AMIs.

Use Auto Scaling Groups.

Learn and use Chef, Puppet, etc. They can cost some time to learn, but save time in re-used code and increased consistency later on.

EBS volumes boot faster than instance stores.

Use SimpleDB to store and retrieve configuration data.

### Parallelize.

It's easy to run multiple processes in cloud environments. It's easy to add more instances so the ability to run many tasks in parallel on freshly created instances allows virtually unlimited scaling.

Multiple threads should be used, all services should be thread-safe and use a share-nothing architecture. Each component should be able to run independently of other components.

Distribute incoming web connections using a load balancer. Web servers should process asyncronously-- each web server process should be independent from the others.

Batch processing should use multiple processing units (slave units) which makes it possible to use Elastic MapReduce to scale out our processing.

Use multi-threaded S3 requests and multi-thread SimpleDB `GET` and `BATCHPUT`.

### Data Location.

Dynamic data should be at a location with low latency compared to its processing location.

Static data should be kept at a locataion close to the end user. Static data includes things like PDFs, CSS, and JavaScript. CloudFront can help cache this info close to end users for a faster web experience.

If data is generated in the cloud, ideally any consumers of that data also run in the cloud (in the same region, though keep redundancy in mind.)

For large imports, consider physical shipping. 

Launch cluster members within the same availability zone.

## Security Best Practice

### AWS Shared Responsibility Model.

Amazon responsible for:

* Facilities
* Hardware physical security
* Network infrastructure
* Virtualization infrastructure (e.g. hypervisors)

Customer responsible for:

* Amazon Machine Images and OS
* Applications
* Configuration and policies applied to AWS resources
* Credentials (IAM + other)
* Data at rest
* Data in transit
* Data stores

Container services model is different as Amazon manages more of the underlying infrastructure, up through the OS hosting the containers.

In the Abstracted Services model, Amazon manages even more, up to the client-side data being managed by the customer.

### Define and Categorize Assets.

When designing an Information Security Management System (ISMS) we should work out a way where the effort of securing assets is appropriate for the level of protection they need.

Qualitative categorization uses expected risks and their expected likelihood to help prioritize our efforts.

Quantitative categorization uses financial impacts to help prioritize our efforts.

Once threats are identified, solutions can be made to protect against the threats by mitigation or prevention.

Essential assets:

* Processes (business processess like cloud computing)
* Activities (for achieving business goals)
* Business information (off-site backups to protect from fire, for example)
* Reputation

Support assets:

* Personnel
* Hardware
* Sites
* Partners

Build an asset matrix with info like:

* Owner
* Category (sensitivity level)
* Dependencies (network, database)
* Asset sensitivity (dupe?)
* Related costs (deployment, maintenance, replacement, etc.)

### Design ISMS to Protect AWS Assets.

Start with business objectives. For example, storing trade secrets in an S3 bucket will require careful handling to ensure only trusted people have access to the underlying information. (Me: use client-side decryption with a good key management and auditing system.)

Certain regulations might need to be followed, such as a requirement to retain certain records for a given length of time.

The size of the organization and its internal business processes might determine which methods are easiest to use.

An Information Security Management System is a phased approach to apply security. A standard framework like ISO 27001 can help.

Steps in building ISMS:

1. Define scope (AWS services which need to be secured)
2. Create a policy (what data needs to be encrypted, for example)
3. Risk assessment (evaluate threats and apply controls)
4. Choose a framework (ISO 27001, ISO 12207)
5. Get management approval
6. Create a statement of applicability (describes the specific controls that are to be used)

### Manage Accounts.

Ensure just enough permissions granted (principle of least privilege)

IAM users + groups makes assigning permissions easier.

Special account: the AWS account

* Used to sign up and represents the business relationship with AWS
* Has root permission to all resources
* Multiple AWS accounts can exist (can multiple root accounts per AWS account exist?)

Normal accounts: IAM accounts

* Multiple IAM users under a single AWS account
* Can be a person or application
* Fine grained permissions

Scenarios for multiple AWS accounts

* Prod, dev, and test might be under different accounts
* Multiple departments with autonomoy
* Centralized security with multiple autonomous projects
  * One AWS account for centralized common resources
  * Separate AWS accounts per autonomous project

Credential types:

* Sign-in: like username/password + MFA
* Programmatic: access keys or developer-created MFA-protected API calls

Delegation needed when:

* Applications on EC2 require AWS resource access
* Access between AWS accounts
* Corporate identity federation

### Manage Access to EC2.

OS-level access to EC2 instances uses OS-level credentials, not AWS-level credentials.

Customers own and change the OS credentials, but AWS can help bootstratp this process via EC2 key pairs. Other options include X.509 certificates, Microsoft AD, or OS-local authentication.

AWS provides asymmetric RSA key pairs. (public/private key) and can be moved between IAM or AWS accounts via import.

Linux uses the `cloud-init` daemon to grant initial access. This appends the key pair defined during instance creation to the `authorized_keys` file.

(Connection demo using PuTTY... isn't this a review?)

Windows `ec2config` service sets a random admin password, and encrypts it with the public key used during EC2 instance creation.

We then need to submit our private key via web form (**WHICH IS A TERRIBLE IDEA FOR SECRET INFORMATION**) to decrypt the admin password. This is a terrible idea because:

* It trains end users that submitting secret keys to websites is normal and accepted
* It could allow the secret key to be intercepted by a man-in-the-middle attack
* We can no longer tell auditors that a secret key has never left our control

We should be sure that Windows key pairs are never used for any purpose beyond intial Windows instance access.

### S3 Data at Rest.

Data at rest concerns:

* Deletion
* Disclosure
* Modification
* Availabilty

Methods to help mitigate these issues

* Permissions (controls who can do the above activities)
  * Bucket-level
  * Object-level
  * IAM policies
* Versioning (allows an undo if an activity was unwanted)
  * Disabled by default
  * Protects from deletion or modification at increased risk of disclosure

Replication improves availability of S3 data by replicating it across all the availability zones in a region. Won't protect from deletion or modification.

Backups can also protect from deletion or modification. S3 backups are not automated, though we can configure a backup to Glacier. OS-level backups may still be needed for data within instances. (EBS?)

S3 server-side encryption can be done easily and transparently to the end user. Each object is encrypted with a unique AES256 key. That key is encrypted with a master AES256 key which is rotated and used for auto-decryption.

S3 client-side encryption can also be used, but requires us to manage our own keys outside of AWS. The data is encrypted with our key on the way to S3 and the key itself is kept by us outside of AWS. The application reading the S3 data will decrypt it. (Hopefully there are well-audited libraries to handle this for developers since getting encryption done right is very hard.)

There's an [AWS Encryption SDK to use for this](https://docs.aws.amazon.com/general/latest/gr/aws_sdk_cryptography.html) though it might be best to avoid using an SDK published by the same company holding the data since if Amazon itself were compromised, that SDK might itself have leaks found or introduced.

### EBS Data at Rest.

EBS volumes are stored internal to AWS as files. Two copies are kept in the same availability zone (mirroring) which allows the data to survive an underlying hardware failure but not a AZ-wide disaster.

EBS snapshots can be created as a backup with independent permissions. An encrypted EBS volume always creates encrypted EBS snapshots.

Windows can use Encrypting File System (EFS) for local-to-the-instance encryption of the OS data. Bitlocker can also be used but only works with a password since EC2 doesn't include any Trusted Platform Module support.

Linux can use `dm-crypt` for transparent encryption and key management.

Other third party tools which can be used:

* TrueCrypt (discontinued in 2014. What year was this video made?)
* SafeNet ProtectV

### RDS Data at Rest.

AES256 used to encrypt RDS data. This must be specified when the database is created.

There may be additional crypto APIs available from specific database instances.

Apps can use PKI certificates to help encrypt/decrypt database info.

Instance-Specific APIs:

* MS SQL
  * Transact-SQL functions for encryption, signing, and hashing
* MySQL
* Oracle
  * Oracle Transparent Data Encryption (Enterprise Edition)
  * Bring your own license only for EE

(no PostgreSQL?)

For storing personally identifiable info (PII) it may be better to obfuscate the info using a one-way hash.

RDS encryption not available for Amazon Aurora.

Selective encryption of database fields can create problems for data lookups since the encrypted data may not be found.

## Practice: AWS Best Practices.

Goal: Build a 3 tier web application on AWS

Questions:

* How do you ensure high availability?
* How do you enable scalability
* What are the data-at-rest security considerations at each tier?

### High availability/Scalability

#### web front end

Use route 53 health checks across at least two regions to provide DR if an AZ fails as well as reducing end user latency on web requests.

Use SQS for fetching app server info.

An Amazon AMI is built using automated/scripted methods to serve as the web front end immediately upon deployment.

#### application server

Use automatic scaling groups with a minimum of two instances split across AZs.

memcached will be used to cache database info locally for repeated application server queries to the same records.

Use SQS for fetching database data and for passing info to the web server.

An Amazon AMI is built using automated/scripted methods to serve as the application server immediately upon deployment.

#### database

Consider either a multi-AZ deployment or the Amazon Aurora Global Database if multi-region redundancy is desired.

Use SQS for passing info to the app server.

### Data at Rest Security.

Like most web applications, availability and data integrity are more important than unauthorized disclosure so there is no use of encryption.

Static web objects will use S3 versioned storage for simplified recovery in the event of unwanted modification.

The Amazon AMIs mitigate the need for regular backups of the web or app server instances.

The database will have snapshots taken regularly.

The database will go into a separate VPC so that access can be more easily controlled.

### Provided Solution

#### high availability

* Design for failure
  * Identify single points of failure
  * Graceful failover with elastic IPs
  * Maintain AMIs to restore environments
  * Use Availability Zones
  * Use EBS and automated snapshots
  * Use CloudWatch for monitoring
  * Use RDS automated backups

#### scalability

* Decouple components
  * Bundle components with AMIs
  * Design components with service interfaces
  * Make applications stateless
  * Use SQS message queueing
  * Use auto scaling groups
* Implement elasticity
  * App components location-agnostic
  * Reduce OS footprint in AMI (Just Enough Operating System)
  * Use auto scaling groups
  * Use deployment / configuration management tools (Chef / Puppet / Ansible / Terraform)
  * Monitor system metrics and set auto-scaling options
  * Store config information using SimpleDB (off-server)

#### security

* database
  * Use internal crypto functions
  * Hash PII
  * Put the DB in separate secured VPC (good idea!)
* app servers
  * encryption local to OS
* web servers
  * S3 server-side encryption
  