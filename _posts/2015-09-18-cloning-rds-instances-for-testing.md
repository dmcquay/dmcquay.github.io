---
layout:     post
title:      "Cloning RDS Instances for Testing"
date:       2015-09-18 08:45:00
category:   DevOps
tags:       DevOps AWS RDS
---
I have one or more staging/test environments at any given point in time. I want them to each have their own
database which should be a nightly clone of the production database.

Previously, my MySQL database was hosted at Rackspace and was backed up using [Holland][holland]. Holland 
uses regular mysqldump under the hood. To sync my staging/test databases nightly, I would just run a cron 
job on each that would download the latest dump, drop its own database and restore from the dump.

In AWS, database backups and cloning are very well automated and tooled. But they are done at a binary/disk
level. Generally, this is a good thing, but there are some downsides, which I'll list in a moment. 
To clone your database in AWS, you'll want to follow these steps:

1. Install the AWS command line client
2. Create a bash script that does the following...
3. Fetch the snapshot id to use for the clone
4. Delete the existing instance
5. Create the new instance
6. Modify the new instance
7. Wait for the instance to reach various states

## Install the AWS command line client

[AWS CLI tools][aws-cli] is well documented.
After you install, configure it with `aws configure`.
Make sure you set your output format to `text` because the example scripts I will provide assume that 
format and use tools like awk to parse the output.

## Fetch the Snapshot Id

We are going to provide the cli with the instance name and grab the latest snapshot for it. My production 
instance is called `production`. Replace with the name of your instance.

{% highlight bash %}
aws rds describe-db-snapshots \
    --db-instance-identifier production \
    | tail -n 1 \
    | awk -F \  '{print $5}'
{% endhighlight %}

## Delete the existing instance

Your database endpoint is based on the instance name. By giving the new instance the same name as the 
previous on, your endpoint URI will not change, so you won't have to reconfigure your consuming app(s). But
RDS won't allow you to have two instances with the same instance identifier, so we must first delete the 
existing instance.
  
Alternately, you could rename the existing instance, but it won't help much. There will be extra steps and 
you'll still have downtime. The only way to avoid downtime would be to use a NEW instance name, leaving 
your old instance alone and reconfigure your app to use the new instance once it is ready.

In my case, I don't care about downtime. It is just a testing environment.

Again, my staging/test instance is called `staging`. Replace with a name of your choosing.

{% highlight bash %}
aws rds delete-db-instance \
    --db-instance-identifier staging \
    --skip-final-snapshot > /dev/null 2>&1
{% endhighlight %}

## Create the new instance

Creating the new instance is straightforward at this point. Some notable settings:
 
 - No multi AZ so we're not paying for availability we don't need
 - No minor version upgrade. We're going to clobber it every night anyway
 - Mine is in a VPC, so I must tell it was subnet group it should be in

{% highlight bash %}
aws rds restore-db-instance-from-db-snapshot \
    --db-instance-identifier staging \
    --db-snapshot-identifier=$snapshot_id \
    --db-instance-class db.t2.micro \
    --publicly-accessible \
    --no-multi-az \
    --no-auto-minor-version-upgrade \
    --vpc-security-group-ids $security_group \
    --db-subnet-group-name $subnet_group > /dev/null
{% endhighlight %}

## Modify the new instance

Some settings are inherited from the snapshot and can't be changed when you call the 
restore-db-instance-from-db-snapshot command. You must wait until the instance is created and available and
then perform a modification operation. In my case, I needed to disable backups.

{% highlight bash %}
aws rds modify-db-instance \
    --db-instance-identifier staging \
    --backup-retention-period 0 \
    --apply-immediately
{% endhighlight %}

## Wait for the instance to reach various states

When you delete the instance (above), you can't create the new instance until the delete operation is 
complete or else the instance identifier will still clash. Here's a function that will do that for you:

{% highlight bash %}
function wait-until-deleted {
    instance=$1
    count=1
    while [[ "$count" != "0" ]]; do
        count=`aws rds describe-db-instances \
            --db-instance-identifier $instance 2>/dev/null \
            | grep DBINSTANCES \
            | wc -l`
        sleep 5
    done
}
{% endhighlight %}

It accepts a single parameter, the instance identifier, and will block until that instance is fully deleted.

Example: `wait-until-deleted staging`

You also need to wait for the new created instance to be in the available state before you can perform the 
modification operation on it. In my case, I also wanted to wait until the modification was complete before 
allowing my script to exit. Here's a function that will do that for you:
 
{% highlight bash %}
function wait-for-status {
    instance=$1
    target_status=$2
    status=unknown
    while [[ "$status" != "$target_status" ]]; do
        status=`aws rds describe-db-instances \
            --db-instance-identifier $instance | head -n 1 \
            | awk -F \  '{print $10}'`
        sleep 5
    done
}
{% endhighlight %}

It accepts the instance identifier and the desired status as parameters and blocks until that status is 
achieved.

Example: `wait-for-status staging available`

## All Together

Here's my final script that gets run nightly.

{% highlight bash %}
#!/bin/bash

##########################################################################
# create-staging-db.sh
#
# Usage:
#   ./create-staging-db.sh [instance_name]
#
# Creates a new RDS instance by cloning the latest production snapshot.
# More specifically, the following steps are performed:
#   - Determine the snapshot id to use
#   - Delete the existing database
#   - Create the new database
#   - Make necessary modifications to the new instances (disable backups)
##########################################################################

instance_identifier=$1
instance_class=db.t2.micro
security_group=sg-XXXXXXXX
subnet_group=default-vpc-XXXXXXXX

function wait-for-status {
    instance=$1
    target_status=$2
    status=unknown
    while [[ "$status" != "$target_status" ]]; do
        status=`aws rds describe-db-instances \
            --db-instance-identifier $instance | head -n 1 \
            | awk -F \  '{print $10}'`
        sleep 5
    done
}

function wait-until-deleted {
    instance=$1
    count=1
    while [[ "$count" != "0" ]]; do
        count=`aws rds describe-db-instances \
            --db-instance-identifier $instance 2>/dev/null \
            | grep DBINSTANCES \
            | wc -l`
        sleep 5
    done
}

# fetch snapshot id
snapshot_id=`aws rds describe-db-snapshots \
    --db-instance-identifier production \
    | tail -n 1 \
    | awk -F \  '{print $5}'`

echo "Snapshot Id: $snapshot_id"
echo "Deleting database (if exists): $instance_identifier"

# delete the existing instance
aws rds delete-db-instance \
    --db-instance-identifier $instance_identifier \
    --skip-final-snapshot > /dev/null 2>&1

wait-until-deleted $instance_identifier

echo "Creating new database: $instance_identifier"

# create the new instance
# this causes HUGE delays since the prod backup is IOPS and is big (100GB)
# --storage-type standard \
aws rds restore-db-instance-from-db-snapshot \
    --db-instance-identifier $instance_identifier \
    --db-snapshot-identifier=$snapshot_id \
    --db-instance-class $instance_class \
    --publicly-accessible \
    --no-multi-az \
    --no-auto-minor-version-upgrade \
    --vpc-security-group-ids $security_group \
    --db-subnet-group-name $subnet_group > /dev/null

echo "Waiting for new DB instance to be available"

wait-for-status $instance_identifier available

echo "New instance is available"
echo "Disabling backup retention"

# disable backup retention
aws rds modify-db-instance \
    --db-instance-identifier $instance_identifier \
    --backup-retention-period 0 \
    --apply-immediately

echo "Waiting for new DB instance to be available"

wait-for-status $instance_identifier available

echo "New instance is available"
echo "Clone process is copmlete"
{% endhighlight %}

## Pain Points

The fact that RDS uses snapshotting performs more consistent and reliable backups. The interface, both 
console and API, is very robust and easy to use. And, if you use a Multi-AZ production database, those 
snapshots are performed on your fallback instance in a different AZ, so the snapshot don't cause any 
performance, latency or locking issues on your DB.

But in the case of creating a clone from those snapshots, there are some downsides:

 - Scrubbing your data is a pain. If you want to modify the dump, you must first restore a snapshot as 
   described above and then you can use `mysqldump` as you normally would, excluding what you need and 
   scrubbing the plain text output as needed.
 - Most production RDS instances should use provisioned IOPS. To use that, your disk must be at least 100 
   GB. When you restore a snapshot from that type of instance, you'll inherit those storage attribute. 
   Converting from provisioned IOPS to standard storage is very slow. In my case I waited an hour and a 
   half before giving up. I still have to solve that problem. Also, you are left with a 100GB disk. I don't
   need that much space for my staging environments, so I don't want to pay for it, but I have no choice 
   since you can't downsize disk size of RDS instances. You can only go up! Grrrr.

Be thoughtful about going around snapshots by using `mysqldump` directly against your production instance. 
This will cause downtime. Snapshots based on production Multi-AZ instances are taken using the hot copy, 
not the live instances, which means no downtime. If you want to use `mysqldump` without downtime, you'll 
need to use a read replica for this purpose, or clone using the process above and then run `mysqldump` 
against that clone.

[aws-cli]:  https://aws.amazon.com/cli/
[holland]:  http://hollandbackup.org/