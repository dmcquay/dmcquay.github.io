---
layout: post
title:  "Migrate to AWS Aurora"
date:   2015-09-23 10:05:00
category: DevOps
tags:   DevOps AWS Aurora MySQL
---
Aurora is Amazon's new database offering. It is MySQL-compatible and runs only on RDS. In my opinion, if 
you are running MySQL on AWS, why not run Aurora instead?

 - Aurora is wire-compatible with MySQL
 - Aurora is lightning fast, more fault tolerant, scales bigger (more read replicas)
 - Aurora gives you more bang for your buck, unless you want a tiny test database because Aurora is not 
   available on the lower end instance types. The cheapest instance you can get is db.r3.large, 2 CPUs, 15G
   memory, $136/mo (reserved 1yr no upfront).
 - If you have a MySQL RDS instance, you can migrate it with a few clicks via the AWS console.
 - If you have a MySQL database outside of AWS, you there is a nice migration path that requires minimal 
   downtime, which I will explain in this post.
 - If you want to migrate back to MySQL later, it is a little harder. AWS doesn't support automatically 
   converting from Aurora to MySQL. You have to use mysqldump which isn't too bad, but will require more 
   downtime.

From what I've experienced so far, it is not 5x in all cases, but it is definitely faster, besides all the 
other benefits. One thing to factor is that Multi-AZ deployments are not as necessary for Aurora clusters 
because it can promote read replicas so quickly or even quickly spin up a brand new instance, since it 
doesn't have to recreate the storage layer for this operation. When you use Multi-AZ, you have a DB 
instance that's just as big as your master sitting there doing nothing. Expensive and wasteful. But necessary.
But with Aurora, it is not necessary for as many use cases.

# Migrating an external MySQL database into an Aurora RDS instance

I will focus on the use case where you have have an external (not in AWS) MySQL that need to be migrated to an
Aurora RDS cluster. The best way I found to do this is to create a single Aurora instance and make it a new
slave to my existing MySQL master in Rackspace. This way the new instance is up to date to the second. The 
final migration only requires about a minute of downtime while I stop all writes to my old master, 
reconfigure my new Aurora instance so it is no longer a slave and then point my app to the new Aurora cluster.
 
The bulk of the instructions you will need are available via
[Importing Data to an Amazon RDS MySQL DB Instance with Reduced Downtime][rds-migration-docs] from AWS.
I just have a few notes to add to this.

 - This document is for migrating into a MySQL RDS instance, but it works equally well with Aurora.
 - When dumping your databases, it tells you to use --single-transaction. This is great because it makes a 
   valid point-in-time dump which you need for creating a new slave. But it only works if ALL of your tables
   use the InnoDB engine. If you have any tables that use MyISAM, then you must use --lock-all-tables instead.
 - They mention that you need to dump your routines, events and triggers separately, which I did. Here is 
   the command I used to dump those:
   `mysqldump --routines --no-create-info --no-data --no-create-db --skip-opt main > routines.sql`
 - Both dumps, the main one and the routines, could contain DEFINER statements which are not valid within RDS
   instances. You need to strip those out. Here's a one liner that can do that for you:
   `perl -pe 's/\sDEFINER=`[^`]+`@`[^`]+`//' < routines.sql > routines2.sql`
 - When restoring your dump into the new target slave aurora instance, they tell you to just login with the
   mysql client and run `source mydump.sql`. I don't recommend this approach because if there are errors, 
   it just keeps on going and you might not even notice them. You have to sit there and stare at the screen to
   make sure there are no errors along the way. I also don't recommend it because you don't get a progress 
   meter of any sort, which is frustrating for an import that could take well upwards of an hour depending 
   on the size of your database. Instead, insteall pv (`apt-get install pv`). It is super helpful. You 
   basically just stream data through it and it gives you a progress bar. Like this:
   `pv backup.sql.gz | gunzip | mysql -h mydb.cluster-XXXXXXXXXXXX.us-east-1.rds.amazonaws.com -u root -p`
   Plus this way you can unzip it on the fly.

[rds-migration-docs]: http://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/MySQL.Procedural.Importing.NonRDSRepl.html