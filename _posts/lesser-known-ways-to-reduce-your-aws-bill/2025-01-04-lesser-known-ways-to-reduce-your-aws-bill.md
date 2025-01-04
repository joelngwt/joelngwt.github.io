---
layout: post
title: Lesser Known Ways to Reduce Your AWS Bill
tags:
  - aws
---

When looking for ways to reduce your AWS bill, you would often come across the usual, commonly mentioned methods:
- Use spot instances
- Use reserved instances/use saving plans
- Rightsizing your instances

These are pretty generic and easy to do. But there is more that can be done which isn't commonly mentioned, so I thought that I should share some of them here. I will not be explaining these in-depth, but instead, I'll just introduce these and explain what they do. I believe the brief mention of these and its documentation should be able to point you in the right direction.

## Gateway Endpoints for S3
Documentation: [https://docs.aws.amazon.com/vpc/latest/privatelink/vpc-endpoints-s3.html](https://docs.aws.amazon.com/vpc/latest/privatelink/vpc-endpoints-s3.html)

If data transfer from your EC2 to S3 buckets go through a NAT Gateway, it will be charged under the NAT Gateway's data transfer costs. This is different from the "Data Transfer" line item in your AWS bill, which is free if your EC2 and S3 bucket are in the same region. Quite sneaky, eh?

To eliminate this cost, simply set up a Gateway Endpoint for your VPC.

## AWS PrivateLink for ECR
Documentation: [https://docs.aws.amazon.com/AmazonECR/latest/userguide/vpc-endpoints.html](https://docs.aws.amazon.com/AmazonECR/latest/userguide/vpc-endpoints.html)

Pulling images from your ECR repository to your EC2 incurs NAT Gateway data transfer costs. To avoid this, you can set up an AWS PrivateLink. However, it's not free, so do check if using it would save or cost you more money.

## Set Up Lifecycle Rules for S3
Documentation: [https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lifecycle-mgmt.html](https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lifecycle-mgmt.html)

There are two aspects to this:
- Deletion of older files
- Moving of older files to a cheaper storage class

If you're afraid of the longer retrieval times for the cheaper storage classes, use "Glacier Instant Retrieval". It's cheaper to store while retaining the same speed of the standard storage class. Do ensure that your older files wouldn't be accessed frequently, as it costs more to retrieve files from this class. You can enable [S3 Storage Lens](https://docs.aws.amazon.com/AmazonS3/latest/userguide/storage_lens.html) to get some analytics about how your files are accessed.

## Set Up Lifecycle Rules for ECR
Documentation: [https://docs.aws.amazon.com/AmazonECR/latest/userguide/LifecyclePolicies.html](https://docs.aws.amazon.com/AmazonECR/latest/userguide/LifecyclePolicies.html)

You should clean up older images in your ECR repository. If not, you're paying money to store old, unused images.

## Set up a retention policy for CloudWatch Log Groups
Documentation: [https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/Working-with-log-groups-and-streams.html](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/Working-with-log-groups-and-streams.html)

If you don't need to keep older logs, you can automatically delete them.

## Write custom scripts to clean up old EBS Snapshots, RDS Snapshots, etc.
You could write some scripts and run them on a cronjob to clean up older snapshots. Look around in your account where snapshots and backups are frequently created, and figure out a way to retrieve and delete these periodically.

## Check for thrashing when pulling Docker images
I had a situation where my EKS cluster was continuously scaling in and out EC2 instances due to a cronjob that was running. When the cronjob needed to be run, a new EC2 instance had to be started. After the job completed, the EC2 was stopped.

This constant spinning up and termination of the EC2 resulted in the ECR image being freshly pulled every single time (instead of being cached), racking up data transfer costs.

The solution to this would be to ensure that minimum number of EC2s running was increased, preventing the constant scaling in and out.