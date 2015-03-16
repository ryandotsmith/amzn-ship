AMZN-ship is an opinionated system that will setup and manage your production server deployments on AWS. The system is designed around ephemeral servers, multi-environment access points, centralized app config management, and zero-downtime deploys.

AMZN-ship is an implementation of the architecture defined in the [**Application Platform on AWS**](http://r.32k.io/app-platforms-on-aws) article. AMZN-ship assumes the AMI is built with amzn-base.

## Setup

* AWS Account with IAM credentials.
* [Ruby AWS SDK](http://docs.aws.amazon.com/AWSRubySDK/latest/frames.html)

```bash
$ git clone https://github.com/ryandotsmith/amzn-ship.git
$ cd amzn-ship
$ cp sample.env .env
# Make changes to .env
$ export $(cat env)
$ bundle install
$ bundle exec bin/release my-app production ~/src/my-app
```

## Relationship to AMZN-base
A **base subsystem** must produce an AMI with a program named `/home/deploy/bin/deploy` which takes a release URL as an argument. The deploy program is responsible for downloading the app and starting the app's processes.

The id of the AMI produced by AMZN-base must be exported into the environment of AMZN-ship.

```bash
$ export AMI=abc123
```

## Operations

* Setup
* Release
* Deploy
* Update Base
* Terminate

### Setup
**Synopsis:** Idempotentaly create resources in AWS region.

There are a few AWS requirements that need to be satisfied before using AMZN-ship. Setup will create the following resources if they do not exist:

1. Security Group
2. Key Pair
3. IAM instance profile with S3 read access
4. ASG, Launch Config, and ELB

The setup script takes 2 arguments:

1. App name (e.g. my-app)
2. Environment name (e.g. production)

Example:

```bash
$ bundle exec bin/setup my-app production
at=create-sec-group status=sg-exists
at=create-key-pair status=key-pair-exists
at=create-role status=role-exists
at=create-role-policy status=put-role-policy
at=create-policy status=policy-exists
at=create-launch-config
at=use-existing-lb elb=my-app-production
at=create-group new-group=my-app-production
```

### Release
**Synopsis:** Tags and uploads code to S3 and moves the S3 latest release pointer.

The release program assumes the directory of the app is a git repository. The program will create a tag on HEAD using a versioning scheme of **rX** where X is a monotonically increasing integer. Next, the program will use TAR(1) to create a tar.gz file of the newly created tag. Finally, the tar.gz file is uploaded to an S3 bucket. The S3 directory will resemble the following structure:

```
.
└── amzn-releases.koa.la
    ├── my-app
    │   ├── production
    │   │   ├── current -> r2/
    │   │   ├── env
    │   │   ├── r1
    │   │   │   ├── app.js
    │   │   │   └── env
    │   │   └── r2
    │   │       ├── app.js
    │   │       ├── env
    │   │       └── lib.js
    │   └── staging
    └── app2
```

Example:

```bash
$ bundle exec bin/release my-app production ~/src/my-app
release=r4
```

### Deploy
**Synopsis:** Run the deploy command on instances belonging to a group.

Each instances contains a deploy program (provided by base image) which expects an argument containing a release url. The deploy program finds all instances in a group, uses an SSH session to execute the local deploy program passing in a release url. If no release is specified in the AMZN-ship deploy command, the latest release is deployed.

Example:
```bash
$ bundle exec bin/deploy my-app staging r1
...
```

### Update ASG
**Synopsis:** Updates the ASG with a new AMI id.

After amzn-base has produced a new AMI (e.g. Updated system package for security patch) the instances in a group need to be cycled with a new AMI id. This command will update the ASG configuration so that new instances will have the new base image. Next the operator must kill all instances on the old image id. The ASG will boot new instances in place of the old instances.

Example:
```bash
$ export AMI=ami-newid
$ bundle exec bin/update-asg my-app staging
```

### Terminate
**Synopsis:** Deletes launch config, auto scaling group and associated instances.

This program will clean up instances only. The elb, security group, key pair, and instance profiles created by the Setup program will remain.

```bash
$ bundle exec bin/terminate my-app staging
at=deregister instance=i-f4d41893 elb=x-production
at=deregister instance=i-f451e78f elb=x-production
at=waiting-for-group-to-terminate
at=waiting-for-group-to-terminate
at=waiting-for-group-to-terminate
at=delete-asg
at=delete-launch-config
```

## Rollback

There is no formal rollback operation, you can rollback a version by deploying the previous release. The previous release will contain code changes and config changes.

```bash
# Deploy good version.
$ bundle exec bin/deploy my-app production r1
# Deploy bad version.
$ bundle exec bin/deploy my-app production r2
# After we discover that r2 is bad, deploy r1.
$ bundle exec bin/deploy my-app production r1
```
