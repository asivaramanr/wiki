# **Databases and Application Services**

## Relational Databases

### Relational Database Service Overview.

RDS cloud instances can run MS SQL, Oracle, PostgreSQL, MySQL, or Amazon Aurora.

Steps for deploying an RDS:

1. Select Engine
2. Production?
3. Specify DB Details: like instance size/class, multi-availability-zone config, underlying storage type, etc.
4. Configure Advanced Settings

Storage 5GB to 3TB. Different EBS volume types available. 30,000 IOPS max. (20,000 for other types?)

VPC (or EC2?) security groups can limit access to the database so only appropriate sources will work.

EC2 security groups with databases are deprecated.

Multi-AZ = synchronous replication across the AZs. Can also generate read-only replicas in other AZs.

DB parameter groups define the database engine configuration. They can apply to DB instances of the same type (e.g. MySQL)

DB option groups applied to a specific instance to support vendor-specific items (Oracle, MS SQL Server, and MySQL only.) For example enabling network encryption.

### Launching an RDS DB Instance.

Launching a new instance can be done in a few minutes. In many cases we don't even need a usage license.

SQL Server can be deployed with our own license or asking Amazon to use an included license.

Advanced settings include things like:

* VPC to use
* Subnet
* Public access?
* Availability zone
* VPC security group
* Database port
* DB parameter group (?)
* Backup retention
* Backup window
* Auto-upgrade minor version
* Maintenance window for minor version updates

Details:

* Endpoint: how other services connect to this database instance.

Can use remote admin tools on the endpoint to do DBA tasks. Specify the user account we created with the DB instance.

Import/Export or bulk copy can be used to bring data into a new instance.

### Managing RDS DB Access with IAM.

Can use DB-specific tools for user management. Users would be specific to that instance.

Can attach policies to users/groups to give RDS privileges such as `AmazonRDSFullAccess` or `AmazonRDSReadOnlyAccess` or get much more fine-grained with a custom policy.

Policy Generator:

* AWS Service: Amazon RDS
* Actions: (lots of options)
* ARN: * (for all instances)
* Conditions: might require MFA, for example
* Policy Name: probably want something more descriptive than the default
* Can Validate Policy, then Create Policy

### Encrypting RDS Resources.

Encryption applies to database instance, snapshots, backups, logs, and read-only replicas. Enabled during creation of the instance.

Uses a default AWS account-wide key (if available) or an instance-specific key we specify. **Key cannot be changed later**

Amazon Aurora cannot be encrypted. MS SQL, Oracle, MySQL, and PostgreSQL can be.

All storage types are supported.

Only these instance types can be encrypted:

* General purpose (M3): db.m3.medium - db.m3.2xlarge
* Memory optimized (R3): db.r3.large - db.r3.8xlarge
* Memory optimized (CR1): db.cr1.8xlarge

Keys are managed by AWS Key Management Service.

Default keys cannot be deleted, revoked, or rotated. Customer-managed keys can.

CloudTrail can audit key usage.

Existing instances cannot be encrypted. Encrypted instances cannot have encryption disabled. Unencrypted backups cannot be restored to an encrypted instance. (That seems nonsensical, I can understand the reverse...)

KMS keys are assigned to a specific region so we cannot copy encrypted snapshots between regions, nor replicate encrypted instances between regions.

### RDS Security Groups.

Two types of security groups with RDS

1. DB security group (only on EC2-classic, deprecated)
   * Each rule allows a source (IP, EC2 security group) access to a database
   * inbound only
   * Port numbers are inferred from the config
2. VPC security group
   * allows a source to access and instance
   * source can be IP range or another VPC security group
   * inbound or outbound
   * Ports specified to match DB ports

DB security groups can control access to instances not in the same VPC. They use the RDS API to set access. Don't require protocol/port info. They allow access from EC2 groups.

VPC security groups work within the same VPC. They use EC2 APIs to set access. They use TCP and require port info.

If the app servers + DB are in the same VPC, we can use the same VPC security group for both groups of EC2 instances.

## Other Database Services

### DynamoDB Concepts.

NoSQL database service-- non-relational database. Simple key-value store.

Tables contain multiple items. Item = row = record = tuple.

No limit to the items per table. Items can be added via GUI, or AWS SDK for Java, .NET, or PHP. (Seems like a small list.)

Attributes = fact = column = field

Primay key: uniquely identifies a particular row/item. Required for each row.

Two kinds of primary keys:

1. Hash attribute: single attribute which uniquely identifies the record within a table

2. Hash and range attributes: more than one item together which combine to uniquely identifies a record within a table

Secondary indexes: quickly find data on non-primary-key attributes. Once created, the index is accessed like a table.

Creating a DynamoDB:

1. Primary Key
   * Table Name: specify name for the DB here
2. Add Indexes (optional)
3. Provisioned Throughput Capacity in read and write "capacity units"
4. Additional options (optional)
   * Enable streams: useful for replicating changes to DynamoDB to other databases
   * Basic alarms: useful for determining if the usage is exceeding the available throughput

#### Working with DynamoDB Local.

Free of charge, does not interact with AWS' real DynamoDB. Useful for prototyping or testing apps that would use DynamoDB without the cost of an actual DynamoDB instance.

Run as a Java JAR file like:

`java -Djava.library.path=./DynamoDBLocal_lib -jar DynamoDBLocal.jar -sharedDb`

Web-based semi-GUI at `http://localhost:8000/shell/`

### Overview of AWS ElastiCache.

Distributed in-memory cache of Database info.

Based on `memcached` or `redis` engine.

Redis allows the use of lists. Memcached is simpler and scales better.

Cache nodes contain CPU/memory used for caching. Cache cluster = collection of cache nodes. Cache Parameter Groups are used to manage cache settings for each node. Cache Replication Groups define multiple copies of the data which can be accessed different ways.

Security is whitelist, based on EC2 instance.

Performance tuning parameters include heartbeat (delay between read synchronization?), number of databases, memory to reserve for non-cache use.

Replication group: 2+ sychronized cluster nodes used for high availability

Not all regions support ElastiCache.

### Launching an ElastiCache cluster.

Configuring ElastiCache:

1. Select Engine
   * Memcached
   * Redis
2. Specify Cluster Details
   * Engine version
   * Port
   * Parameter group
   * Enable replication
   * Cluster name
   * Note type
   * S3 location of Redis RDB file
3. Configure Advanced Settings
   * Cache subnet group
   * Availability Zone
   * VPC Security Group
   * Automatic backups
   * Maintenance Window
   * Topic for SNS notifications
4. Review

Adjust security group to allow Redis port 6379 if needed.

Endpoint for Redis is needed for clients to connect with it. (Is network-based access really faster? Shouldn't this be local to each EC2 node running the Redis client?)

Created new EC2 instance in the same security group as ElastiCache used (or at least one that would allow communications with Redis' TCP port.)

Test basic Redis functions using telnet to port 6379:

(Note the lack of any authentication-- those security groups appear pretty important to data security!)

`telnet <endpoint> 6379`

`info server`

`info all`

`exists customerid`

`append customerid "100"`

`get customerid`

(Our inserted data is present. Scary!)

Some tiny amount of security is available in theory, but is very vulnerable to brute-force attacks: [redis password protection](https://redis.io/commands/auth)

### Overview of AWS RedShift.

Data Warehouse service.

To use:

1. Provision a Redshift cluster
2. Upload data sets
3. Run analysis queries

Redshift clusters have "leaders" which orchestrates the proces of receiving queries, splitting the load, gathering replies, and responding back to the client request.

Clusters need only one node. Can add/remove online. Snapshots can be made.

Permissions managed via IAM. Security groups define access from SQL client tools-- either RedShift-specific security groups or VPC security groups.

Encryption possible.

Redshift IAM policies include the usual defaults like `AmazonRedshiftFullAccess` and `AmazonRedshiftReadOnlyAccess`.

Redshift monitoring

* Database audit logging (tracking access)
* Events and notifications when "certain things" happen
* Performance metrics (CloudWatch)

Redshift creates one database by default but more can be added.

Parameter groups are used to make consistent configurations across databases (?) in the same cluster. Or would that be used to create multiple clusters with the same config? (a bit unclear.)

## Big Data

### Amazon Kinesis Overview.

Processes large amounts of data in real-time. Dat often processed as it's created. A Kinesis application is created from client libraries or APIs. It takes data from "producers", fold/spindle/mutiliates it, then puts it in "streams."

What is a "producer"? Log file, transactions on a web app, data from mobile devices, etc.

Once data is in a stream it can be directed to dashboards, alerts, or sent to other AWS services for storage.

Terminology:

* Data record: one data unit in Kinesis
* Streams: ordered sequence (group) of Data Records
* Producers: write data to streams
  * EC2 instances
  * End users' client devices
  * Mobile devices
  * Other servers
* Consumers: read data from streams
  * EC2 instance apps
* Kinesis application: run on the shards within an EC2 instance
* Application Name: identifies the Kenesis app
* Shard: group of data records with unique ID (also a unit of capacity)
* Partition key: groups data within a stream by shard
* Sequence number: unique number for each data record

Creating a stream:

Services -> Kenesis -> Create Stream

Stream name and number of "shards" (1MB/s write + 2MB/s read capacity) needed.

Kinesis apps run within EC2 instances. Kinesis apps have a unique name used for failover/HA.

### Amazon Elastic MapReduce Overview.

Used for storing and processing very large amounts of data without managing the details of the underlying infrastructure.

Also useful for moving large data volumes to/from S3 or databases.

EMR requires the use of a custom application which is run within the EMR cluster.

An EMR cluster is created from EC2 instances-- at least a master node and optional slave nodes. These are provisioned from an AMI with Hadoop pre-installed.

EMR is scalable-- we can add nodes for times when more processing needed. Can auto-replace nodes with performance issues or other problems.

Can use S3 buckets for storage.

EMR can also use Spark or Presto with (instead of?) Hadoop.

Within the cluster, nodes use HDFS cluster filesystem for sharing. Apps and data used by the cluster externally are stored in S3.

Cluster resources managed via [YARN](https://hadoop.apache.org/docs/current/hadoop-yarn/hadoop-yarn-site/YARN.html) which splits up tasks.

Application languages supported: [Hive](https://hive.apache.org/), [Pig](https://pig.apache.org/), [Spark SQL](https://spark.apache.org/sql/), [Mlib](https://spark.apache.org/mllib/), [GraphX](https://spark.apache.org/graphx/), and [Spark Streaming](https://spark.apache.org/streaming/).

## Application Services

### Simple Queue Service Overview.

A queue is used to store information temporarily. Apps can push messages (info) into the queue and others can pull data from the queue. Often used for web applications. This helps keep web applications more loosely coupled with their dependents.

Delivery (eventually) is guaranteed even if there are temporary problems. Once the problems are fixed the consumer can poll the queue again and receive its messages.

FIFO is NOT guaranteed.

Message sizes are variable. Configurable settings per queue (e.g. polling interval.) There can be multiple simultaneous readers and writers per queue.

Delay queues possible-- consumers only see the messages appear a given time after they are added to the queue.

Either IAM or SQS policies can be used to control access to queues.

SQS message lifecycle:

1. Sender puts a message on the queue
2. Consumer retrieves the message, "visibility timeout" period starts. During this time the message is invisible to other consumers.
3. Consumer processes the message and deletes it before the "visibility timeout" expires

Services -> SQS -> Create New Queue

Queue Settings:

* Queue name:
* Default visibility timeout: 30s
* Message retention period: 4d
* Maximum message size: 256KB
* Delivery delay: 0s
* Receive message wait time: 0s

Dead Letter Queue Settings:

* Use Redrive Policy:
* Dead letter queue:
* Maximum receives:

Once created, we can add permissions from the SQS web GUI.

We can also send a message to the queue from there. (Unusual) Normally apps would send messages to each other.

### Simple Workflow Service Overview.

Development tool for making distributed web apps. The web app components could be running on different EC2 instances.

A process for running coordinated tasks on multiple servers is needed. SWF handles scheduling the tasks, their dependencies, and managing concurrent task execution.

Workflows contain activities. Activities perform tasks. Tasks can either be synchronous (sequential) or asynchronous (parallel) with other tasks (?). He says "job" but that's not defined yet.

An activity worker takes a task and performs that activity. These are built by developers.

Decider: coordinates work within a task. This logic is defined by developers.

The task logic (code) can be written in any langage. It doesn't need to run within AWS.

Workflow execution:

1. Create activity workers (developers)
2. Create a decider (developers)
3. Register the activities and workflow within SWF
4. Start the workers and decider
5. Start executing the whole workflow
6. View executions in AWS console

Workflow example: splitting, uploading, and re-combining a large media file sent to S3.

### Simple Notification Service Overview.

Send messages to "subscriber endpoints" (so far sounds suspiciously similar to Simple Queue Service.)

What is a "subscriber endpoint"?

* application running somewhere else
* user on mobile device who has subscribed to notifications
* newsletter on a website
* etc.

Uses a publisher (producer) - subscriber (consumer) model. (Still sounds a lot like Simple Queue Service.)

Subscribers don't need to poll for messages. (OK, there's a difference!) The messages get pushed to an app that has already agreed to receive them.

Possible subscriber types:

* Users (desktops, mobile devices)
* E-mail
* Amazon SQS
* Amazon Lambda
* HTTP (HTTPS) endpoint
* SMS (texting)

SNS "topics" combine multiple recipients into one group and can contain multiple endpoint types.

Crucial API calls:

1. CreateTopic
2. Subscribe
3. Publish

Mobile push can use protocols like ADM, APNS, Baidu, GCM, MPNS, or WNS. (Getting definitions for these acronyms seems difficult. APNS is probably Apple Push Notification Service, but many of the others are completely ambiguous.)

Services -> Mobile -> SNS -> Create new topic

Can sbscribe programmatically or via GUI: Actions -> Subscribe to topic

Protocols used for that method:

* HTTP
* HTTPS
* Email
* Email-JSON
* Amazon SQS
* Application
* AWS Lambda

E-mail subscription requests send the user an E-mail with a confirmation. They need to click the link (with the right Amazon cookies present) to confirm the subscription.

Services -> Mobile -> SNS -> Create platform application

* Application Name
* Push Notification Platform
  * Amazon Device Messaging (ADM) (TLA revealed!)
  * Apple production
  * Apple development
  * Baidu Cloud Push for Android in China
  * Google Cloud Messaging (GCM)
  * Microsoft MPNS for Windows Phone 7+
  * Microsoft WNS for Windows 8+ and Windows Phone 8.1+
* Client ID
* Client secret

### Sending E-mail with Simple E-mail Service.

Outbound SMTP E-mail sending service. **SPAM MACHINE!**

Developers can build in sending E-mails using API calls.

Or a mail client can use the SMTP mail relay to send directly. (Perhaps from an EC2 instance? This doesn't make a lot of sense in terms of most end users' real mail clients-- e.g. Outlook.)

Another option, configure our SMTP relay to itself relay messages via Amazon Simple E-mail Service. That seems especially odd since we are likely to have a mix of non-AWS-sourced E-mails mixed in there.

Services -> Application Services -> SES

SMTP settings shows server name. SMTP requires a username/password to authenticate before a message will be accepted. These SMTP users are managed through (big surprise!) IAM. Note that the SMTP username is not the same as the IAM user name (since the SMTP usernames probably need to be globally unique.)

The user gets created with a policy called `AmazonSESSendingAccess`

In order to send E-mails either our [account needs to undergo some additional scrutiny to get out of "sandbox" SMTP access](https://docs.aws.amazon.com/ses/latest/dg/Welcome.html) or E-mail addresses must be individually verified to be willing to receive E-mails from this AWS account.

Can also send SES messages via AMS CLI using JSON.

Example `destination.json`:

```JSON
{
  "ToAddresses":  ["user@domain.com"],
  "CcAddresses":  [],
  "BccAddresses":  []
}
```

Example `message.json`:

```JSON
{
  "Subject": {
    "Data": "Subject of Email Goes Here"
    "Charset": "UTF-8"
  },
  "Body": {
    "Text": {
      "Data": "Body of message goes here."
      "Charset": "UTF-8"
    }
  }
}
```

`aws ses send-mail --from <from address> --destination file://destination.json --message file://message.json`

Output will be message ID.

### Transcoding Media with Elastic Transcoder.

Convert media files from one format to another to allow for a wider playback audience (at the expense of image quality.)

Services -> Application Services -> Elastic Transcoder -> Create a new Pipeline

The pipeline defines the transcoding jobs that we want to execute.

Configuration options:

* Pipeline name
* Input bucket: (existing S3 bucket for uploaded media)
* IAM role: auto-created on first pipeline
* S3 Bucket: (for output)
* Storage class: standard or reduced redundancy (the latter might be appropriate since we can always re-transcode the original media.)
* S3 Bucket: (for thumbnails)
* Notifications (optional) (via SNS Topic)
  * On progressing
  * On Warning
  * On Completion
  * On Error
* Encryption (optional)

"Presets" define various options for video output. For example:

* Generic 1080p (mp4)
* Generic 720p (mp4)
* iPhone4S (mp4)
* iPod Touch (mp4)
* Apple TV 2G (mp4)

"Jobs" shows existing jobs or allows us to create a new job.

Create new job parameters/options:

* Pipeline
* Input Key (S3 Keys): media file already uploaded. ("Key" is an odd description for a media file...)
* Output Key Prefix: start of the name of transcoded files (optional)
* Decryption Parameters
* Output Preset
* Output Key: Output file name
* Encryption Parameters
* Available settings
  * Clip
  * Captions
* Additional output details (optional)
* Watermarks (optional)
* Additional Outputs (e.g. another preset)
* Override Detected Input Parameters (e.g. force interpretation as a specific frame rate, etc.)

Initial status "progressing"

### CloudSearch Overview.

Search capability can be added to data collections. For example web pages, documents, blogs, etc.

Can self-provision storage to hold indexes and self-scale as the search volume changes.

Can be unstructured data, semi-structured, or structured. (Not sure how these are defined.)

Each searchable item is described as a document and gets a unique ID. Documents uploaded as JSON or XML but the content can be PDF, MS-Office docs, CSV, HTML, etc. which gets converted to JSON or XML for indexing.

Deleting content might leave its search index behind for a short time.

Search indexes defined by fields within the uploaded documents. Each document field that will be indexed needs an index field. Fields have a type, category, and possibly other attributes. Data in the documents to be indexed must match the indexing configuration.

"Facets" are used to refine and filter searches on other index fields. (like date info?) They give an example of letting a user search within a specific region.

I don't really follow what "Hits that share the same value in a facet" means... What's a "value in a facet"?

Creating a CloudSearch:

1. Create a CloudSearch domain
2. Upload your data
3. Index the uploaded data
4. Search the index

Search requests are HTTP GETs. Pre-processed to do the following:

1. Convert to lowercase
2. Split on whitespace and punctuation
3. Remove stopwords ("the" "is" "at")

(All of these can make searching on certain terms very difficult or impossible.)

Can specify options for facet information or ranking weights.

Search results returned as JSON or XML.

Creation proicess:

1. Name your domain
   * Search Domain Name
   * Desired instance type
   * Desired replication count
2. Configure index
   * Analyze sample files from local machine
   * Analyze sample objects from S3
   * Analyze sample objects from DynamoDB
   * Predefined config
   * Manual config
3. Review index configuration
4. Setup access policies

## Deployment and Management

### Monitoring AWS with CloudWatch.

Performance monitoring for EC2, storage volumes, load balancers, databases, etc.

Services -> Administration & Security -> CloudWatch

Alerts triggered when pre-defined thresholds are crossed. Can send a message to a given SNS topic. That can, in turn, send E-mails to appropriate recipients.

Metric categories:

* CloudSearch
* DynamoDB
* EBS
* EC2
* ELB
* ElastiCache
* RDS
* S3
* SNS
* SQS

Some metrics are per-instance, others per-group, as appropriate for the metric.

Graphs can show a metric over time. Time spans can include:

* 1 min
* 5 min
* 15 min
* 1 hour
* 6 hours
* 1 day

Can show average, minimum, maximum, sum, or "data samples" (number of samples?)

Alarms on a given instance + metric can be created with "Create Alarm" button on the same graph screen.

The create alarm process is two steps:

1. Select Metric (already selected if we were viewing it)
2. Define Alarm
   * Name:
   * Description:
   * Whenever `<metric>` is `>=` `<value>` for `N` consecutive periods
   * Actions: based on `state` send notification to `<SNS topic>` (can also add auto-scaling actions or EC2 actions)

### Creating a Simple Application Stack with OpsWorks.

OpsWorks is an application management platform. Manages an application lifecycle to automate things like deployment and scaling.

Services -> Deployment & Management -> OpsWorks

We can create the stack from nothing or by adding existing EC2 instances.

"Add your first stack" -- create from nothing.

Options:

* Name
* Region
* VPC
* Default subnet
* Default operating system
* Default root device
  * EBS backed (persistent)
  * Instance store (ephemeral)
* IAM role: assists with interdependencies betweeen AWS resources
* Default SSH key
* Default IAM instance profile
* Hostname theme
* Stack color
* Advanced
  * Chef version
  * Custom chef cookbooks ([awslabs has a bunch of them on GitHub](https://github.com/awslabs/opsworks-example-cookbooks))
  * OpsWorks Agent version
  * Custom JSON: passed to Chef recipes
* Use OpsWorks security groups

Once the stack is made we can add "Layers" which are sets of similar EC2 instances. Layer types include things like HAproxy, static web server, Rails App server, PHP app server, Node.js app server, Java app server, AWS Flow (Ruby), MySQL, memcached, ganglia, ECS Cluster, or "Custom."

Layers can get Chef recipes assigned. Some of them are included like `ssh_host_key`, `ssh_users`, or `php::configure`.

### Elastic Beanstalk Overview

Automates provisioning of resources for apps running on AWS.

Workflow:

1. Create application
2. Upload version
3. Launch environment
4. Manage environment
5. Update version: go to "Upload version" and repeat as needed with future versions.

The application can use whatever language we want: .NET, python, Java, Ruby, etc. Can be deployed straight from GitHub.

"Manage environment" can also mean setting up appropriate capacity, load balancing, CloudWatch.

A BeanStalk application is a collection of Beanstalk components. For example, a particular application version (collection of code) along with an environment (a deployed collection of other code, dependencies?)

The Environment Configuration are the parameters for deployed code that define the behavior of the underlying AWS resources.

The Configuration Template starts creating the environment.

An application version is the code itself sitting in S3, ready to be used.

### Creating an Audit Trail with CloudTrail.

Logs API calls realted to resources. Could be normal internal API calls or could be external API calls due to development activities, etc. These may need to be logged and saved due to regulatory requirements also.

CloudTrail logs go into an S3 bucket.

Services -> Administraton & Security -> CloudTrail

Config options:

* Create new S3 bucket (will auto-set permissions on S3 to allow logs)
* S3 bucket
* Advanced
  * log file prefix
  * SNS notify for every log delivery
  * SNS topic
* CloudWatch Logs

Can filter log display from the view history screen (web GUI) based on time range, resource name, etc.

### Creating a Data Pipeline.

Copies data between data stores like:

* S3
* DynamoDB
* MySQL
* Redshift

Scheduled based on preconditions and run when they are met.

Can be fed to Elastic MapReduce clusters for auto-scaling.

IAM policies control what the pipelines can do.

AWSDataPipelineRole policy.

Services -> Analytics -> Data Pipeline -> Create new pipeline

* Name
* Description (optional)
* Source
  * Build using template (lots of options, including)
    * Run AWS CLI command
    * Export DynamoDB to S3
    * Full copy RDS MySQL table to S3
    * Incremental copy RDS MySQL table to S3
  * Import a definition
  * Build using Architect
* Schedule
  * Run once on pipeline activation OR on a schedule
  * Run every `N` days
  * Starting on pipeline activation OR given date

The parameters are appropriate for the Source type. (e.g. login and source/target details.)

### CloudFormation Overview.

Method of provisioning resources from a template via automation.

Can automate provisioning of EC3, S3, ELB, etc.

Services -> Deployment and Management -> CloudFormation

Create New Stack or Create a template from your existing resources (CloudFormer)

Create new stack:

* Stack
  * Name
* Template
  * Source
    * Select a sample template (lots of options including):
      * LAMP stack
      * Wordpress blog
      * CloudFormer
      * Windows Active Directory
    * Upload a template to Amazon S3
    * Specify an Amazon S3 template URL
* Parameters:
  * Features: Specific to your source type
  * Instance Type
  * KeyName (keypair)
  * Roles

Can also automate the process with Chef or Puppet.

## Practice: Database Monitoring

### Launch a Microsoft SQL Server DB Instance.

Services -> Database -> RDS -> Create Database

Microsoft SQL Server

SQL Server Express Edition (keep it cheap!)

Template: Free tier

Name: `database-sql-server`

Master username: `admin` (default)

DB instance: `db.t2.micro` (default)

Storage type: `General Purpose` (default)

Allocated storage: `20GB` (default)

Connectivity:

* Virtual Private Cloud: `Default VPC` (default)
* Subnet group: `default` (default)
* Publicly accessible: `No` (default)
* VPC security group: `create new`
  * New VPC security group name: `database-sql-sg`
* Availability zone: `No preference` (default)
* Database port: `1433` (default)

SQL Server Windows Auth: None (default)

There are no free AWS-hosted databases. Skip the practice and save it for a future lab.

### Connect to the DB Instance

Skipping to avoid costs.

### Monitor the DB Instance

Skipping to avoid costs.
