---
layout: post
title: How To Upgrade PostgreSQL RDS Without Downtime (Almost)
tags:
  - aws
---

## The Story
I few months ago I was presented with the task of trying to find a way to upgrade our ancient RDS instance from PostgreSQL 9.4 to 9.5 without any downtime.

A quick Google search did bring up a tool called Bucardo, as well as a [guide](https://medium.com/preply-engineering/postgres-multimaster-34f2446d5e14) on how to use it. Long story short, Bucardo does work, but the database triggers it added caused some severe performance issues. The guide also uses an apt package (`postgresql-plperl-9.5`), and I was unable to find versions for 9.6 and above. So, the guide effectively worked for 9.4 to 9.5 only.

I ended up having to upgrade the RDS with downtime, and a few months later... I found AWS Data Migration Service (DMS). After using it, it works perfectly, but I wondered why no one in the internet ever talked about it.

## What is Data Migration Service (DMS)?
DMS is a tool that allows you to migrate data from a source database to a target database. The source and target does not have to be using the same database engine, and the source does not even have to be hosted on AWS.

It does all this by replicating your source database to the target database. Continuously. This is properly known as multi-master replication.

What does this mean? It means we can set up a second database, upgrade it to the version we want, then use DMS to replicate our original database to the new and upgraded database. Since this replication is ongoing, you can swap your client's database endpoints and no one should realize the databases have changed.

## Okay, let's get started!
### Configure your parameter group
- First, you need to get your database settings set up correctly. You cannot change the default parameter groups, so create a new parameter group if you have to.
- Change the following values:
  - `rds.logical_replication` to `1`
  - `wal_sender_timeout` to `0`
- These values do not seem to affect ordinary usage, so it's okay to leave it like this.
- Modify your database and apply this parameter group to it.
- *You will need to restart your database for the changes to take effect. This is the one and only downtime. It should not take more than 3 minutes.*

![Parameter Groups](/images/postgresql-multi-master-replication/1.png "Parameter Groups")

### Create your database
- Create a new database, with all the settings and PostgreSQL version that you want to eventually use.

### Create DMS Endpoints
- DMS endpoints webpage: <https://ap-southeast-1.console.aws.amazon.com/dms/v2/home?region=ap-southeast-1#endpointList>
- Select "Endpoints" on the left sidebar if you aren't already on it.
- Click "Create Endpoint".
- Check “Select RDS DB instance”.
- Select your source instance from the dropdown.

![Endpoint](/images/postgresql-multi-master-replication/2.png "Endpoint")

- Continue filling up the rest of the information
  - Endpoint identifier - Anything you want, this is just for your own reference
  - Server name - This is the RDS endpoint.
  - User name, Password, Database name - This are the settings for your RDS's database.
- Repeat all of this again, but choose "Target Endpoint".
- Test the connection of the database.
  - If the access has failed, it is likely due to the security group of the source and target instance. Add the private IP address of the replication instance to the security group of your source and target instance.

![Test connection](/images/postgresql-multi-master-replication/3.png "Test connection")

### Create a replication instance
- Replication instances webpage: <https://ap-southeast-1.console.aws.amazon.com/dms/v2/home?region=ap-southeast-1#replicationInstances>
- Select “Replication Instances” on the left sidebar if not already selected.
- Click “Create replication instance”.
- Change the “Instance class” to dms.c4.large. Smaller instances might run out of memory. Increase this size if it happens.
- For “VPC”, select the VPC that both your source and target databases are in.
- For “VPC security group(s)”, choose the security group your source and target databases are in.
- All other fields can be left as default.
- Create instance.

### Create Database Migration Task
- Go to the Database Migration Tasks webpage: https://ap-southeast-1.console.aws.amazon.com/dms/v2/home?region=ap-southeast-1#tasks
- Select “Database migration tasks” on the left sidebar if not already selected.
- Click “Create task”.
- Fill up the details. It should use everything you have created previously.
- The “Migration type” should be “Migrate existing data and replicate ongoing changes”.

![Task configuration](/images/postgresql-multi-master-replication/4.png "Task configuration")

- In “Task settings”, choose “Full LOB mode”

![Task settings”](/images/postgresql-multi-master-replication/5.png "Task settings”")

- In “Table Mappings”, add one selection rule:
  - If your schema name happens to be different, just change it accordingly.

![Table Mappings](/images/postgresql-multi-master-replication/6.png "Table Mappings")

- Save the task.

### It's done!
- The task should have started after creation, so watch its progress.
- If all goes well, then congratulations, your target database is ready for use.
- Perform the necessary swapping of the database host URLs on your servers and/or Route 53.

## Troubleshooting
Nothing is ever that simple, and that applies to me as well. Here are several errors that I came across, maybe it will help you too.

### Failures
- In general, you can attempt to debug the replication by looking at the "Last failure message" section.

![Last failure message](/images/postgresql-multi-master-replication/7.png "Last failure message")

## Out of memory
- Simply increase the "replication instance" size.

## Running with errors
- If the “Status” is “Running with errors”, wait for the task to be complete at 100% before proceeding with the next step.

![100% progress](/images/postgresql-multi-master-replication/8.png "100% progress")

- Open the task, go to the “Table Statistics” tab.
- If there are “Table Errors” in the “Load State” column, stop the migration task.
- Modify the task by changing the “Full LOB mode” to “Limited LOB mode”.
- Start the task in “Resume” mode.
- Go to the “Table Statistics” tab again, and select the tables with errors.
- Click “Reload Table Data”.

![Table Data”](/images/postgresql-multi-master-replication/9.png "Table Data”")

- Hopefully, the table loads properly.
- Stop the task, modify it and change it back to “Full LOB Mode”.
- Start the task in “Resume” mode.

## Performance is slow on the new RDS
- A database `ANALYZE` should fix it.

This command analyzes in stages (results appear faster since it will do a quick analyze, medium analyze, then full analyze):

```
vacuumdb -d your_database_name --analyze-in-stages -h your_database_endpoint -p 5432 -U your_database_username
```

This command does a full analyze directly:

```
vacuumdb -d your_database_name --analyze-only -h your_database_endpoint -p 5432 -U your_database_username
```
