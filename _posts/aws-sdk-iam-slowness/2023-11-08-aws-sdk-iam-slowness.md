---
layout: post
title: AWS EC2 IAM Roles Can Severely Affect Performance
tags:
  - rails
  - aws
  - iam
  - imds
  - sentry
---

## The Problem
One day, while I was looking through Sentry's performance monitoring section, I noticed that some APIs were performing poorly, with some queries taking more than a minute to complete. Usually, they would take 10 seconds at most. I opened up an event to look at the flame graph and this is a snippet of what I saw:

![Flame graph](/images/aws-sdk-iam-slowness/flame-graph.png "Flame graph")

These three queries kept repeating, taking up 35% of the total execution time:
- PUT http://169.254.169.254/latest/api/token
- GET http://169.254.169.254/latest/meta-data/iam/security-credentials/
- GET http://169.254.169.254/latest/meta-data/iam/security-credentials/[IAM role name]

## What Are Those?
I didn't really have much of an idea what this was about, so the next step was to Google the URLs. These URLs turned out to be the method EC2s use to retrieve the security credentials to access AWS services. I believe it's called IMDS, which stands for Instance Metadata Service. Here is the documentation for it: [https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/iam-roles-for-amazon-ec2.html#instance-metadata-security-credentials](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/iam-roles-for-amazon-ec2.html#instance-metadata-security-credentials)

In the case for my application, the credentials were needed to upload files into an S3 bucket. The application is hosted on EC2 servers, with IAM roles attached to the EC2 to provide the credentials.

## Why Did This Happen?
I believe it's a bug on the [AWS SDK for Ruby](https://github.com/aws/aws-sdk-ruby), and this is the GitHub issue: [https://github.com/aws/aws-sdk-ruby/issues/2177](https://github.com/aws/aws-sdk-ruby/issues/2177). It's still not solved, and not many people seem to be commenting on the issue. There could be many reasons for that though:
- I'm doing something wrong somewhere to encounter this issue, and there is no bug.
- People are not using IAM roles, which avoids this issue.
- People are unable to find the GitHub issue.
- It's easy to bypass the problem and find an alternative solution.

Something that further adds to the mysteriousness of this problem is that I have `instance_profile_credentials_retries` set to `2` in my application, which should prevent the IMDS queries from repeating that often. I guess it's something that doesn't work either?

## The Solution
Well, if attaching IAM roles to the EC2 causes a problem, then don't do that. Have you seen the meme where the doctor says "Then don't do that"?

![Doctor meme - don't do that](/images/aws-sdk-iam-slowness/doctor-meme.png "Doctor meme - don't do that")

Sometimes, there isn't a choice, but fortunately, AWS provides many ways to provide security credentials to an EC2, and these have a [precedence](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-authentication.html#cli-chap-authentication-precedence) as to which should be used first. IAM roles come in at the last two, which made things easy for me to override.

The solution I took was to use the "credentials file", which is sixth on the list. To do this, a dummy user had to be created, mirroring the same permissions as the IAM role. Then, simply run `aws configure` on the EC2, key in the AWS access key ID and AWS secret key of the dummy user, and the EC2 would be using these credentials instead of trying to query IMDS, getting stuck, and consuming a huge chunk of the request processing time.

## Side Note
On a side note, this really brings a spotlight onto how important application performance monitoring is. This would never have been caught without it, as there were so many factors that would make this hard to trace manually:
- No issues with CPU, memory, or disk usage.
- Because of the above, you would look at your code implementation.
- Not reproducible on a local environment.
- You would not expect the problem to come from an AWS library.
- The IMDS queries did not trigger for every single API action.
- Even if you managed to grab the internet traffic logs from the server, and notice that it was querying the IMDS endpoints, it would be near impossible to associate it with one singular API action, and therefore you might not be able to notice that the number of IMDS queries were far higher than desired.

Moral of the story? Set up application performance monitoring!
