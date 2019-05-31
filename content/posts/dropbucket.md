---
layout: blog
title: DropBucket
date: '2019-02-07T21:59:37-05:00'
cover: /images/dropbucket.png
categories:
  - software
  - security
  - AWS
  - serverless
  - DropBucket
  - file sharing
---
# Open source self-hosted file sharing

<!--more-->

DropBucket is an open source software that can be used to upload and share files in the cloud. 

For the impatient, code is [here](https://github.com/marksteele/drop-bucket).

{{< figure src="/images/dropbucket-ui.png" height="300" title="Screenshot of the DropBucket user interface" caption="" >}}

There are lots of other file sharing services out there, here's why DropBucket is different:

* It runs completely on infrastructure that's managed by AWS. It requires <b>zero operational effort to run</b> this service once it's deployed.
* There are <b>no fixed costs</b> to run the service. You pay only for what you use, the only costs for operating the service is what AWS charges for them (see the cost scenario a little bit further down).
* It's hosted in <b>your</b> AWS account. You get to control where in the world your files are stored. This is a pretty big deal for companies that have regulatory or contractual obligations on locations where data can be stored.
* Building atop of AWS core infrastructure, it is <b>extremely scalable</b>.
* <b>You can audit the code and implementation of the solution.</b>

The main design choices that I've made in building this are: security, scalability, cost effectiveness, and simplicity.

# Features

* Files are automatically scanned for viruses
* Files auto-delete after 7 days
* Self-service user registration with email confirmation
* Easy sharing links (public links)
* Authenticated sharing: requires users to be signed into DropBucket to be able to download shared documents.

# Rationale and use cases

Sending files over email? That's a security problem as the SMTP protocol is not encrypted by default. Also who knows where your files are ending up? It's hard to know how many countries your files may end up in, as big providers (such as Gmail) have geo redundancy, disaster recovery sites, backups, etc...

So where does that leave us? There are plenty of commercial vendors selling cloud storage. In all cases, you have to trust in their security. Most of them probably implement per-client encryption keys, so your data is probably safe. But once again, if you need to have control on which countries your data is in, you're out of luck in most cases.

You _could_ use SFTP. I mean that would work, but it's a bit awkward in this day and age, and not terribly user friendly. 

DropBucket is designed to fill this sharing gap in a way that gives you control.

# Security

All client-server operations (authentication, directory listing, uploads, downloads) are secured by strong transport encryption (TLS). At rest, files are encrypted in S3 using server-side AES-256 encryption.

The standards-based identity component (Cognito) is managed by AWS and is compliant to multiple certifications (HIPAA, PCI DSS, SOC, ISO/EIC 27001, ISO/EIC 27017, ISO/EIC 27018, and ISO 9001). It can be federated with a variety of identity providers if desired.

The file storage component (S3) is designed for 99.99% availability, and 99.999999999% durability.

All files are scanned for viruses automatically on upload by the Clam Anti-Virus software. Virus definitions are updated daily.

Policy on the storage backend will prevent access to files that have been tagged as being infected by viruses.

All files are automatically deleted after 7 days via S3 bucket lifecycle policy.

# Scalability

This is a serverless architecture relying heavily on AWS infrastructure. It could easily scale to millions of users storing petabytes of data without requiring any specific scaling actions (with the possible exception of increasing service limits for API Gateway/Lambda/S3).

The client application is a single-page web application, and communicates directly with the S3 API.

# Cost effectiveness

Below is a breakdown of the costs of operating DropBucket. Refer to AWS pricing to build your own scenarios.

Let's imagine have 50 users who each need to store 100 files each, totaling 1 GB per user.
 Also, we'd like to store our files in the US (Virginia area), so we'll use the US-EAST-1 AWS region.

## S3

 (file storage)

The storage costs on that are $0.023 per GB * 50 GB: 1.15$ USD per month.

If each user writes their 100 files every month, the S3 operation costs are:

50 (users) _ 100 (files) = 5000 requests _ 0.005 per 1000 requests = 0.0025$ USD

Let's imagine the users generate 1000 list requests each per month, that would cost an additional 0.025$ USD

Furthermore, let's imagine that they each fetch all their 100 files once per month, the get operations would cost 0.0004 per 1000 requests, so that would cost 0.002$ USD.

Data transfer into S3 is free, so we only have to look at the downloads.

If every user downloads all their files once per month, the cost of that would be 50GB * 0.09 = 4.50$ USD

S3 sub-total: 5.6795$ USD per month

## API Gateway

 (HTTP gateway)

Let's imagine those same users login 1000 times to the app per month, the costs for API gateway would be 3.50$ per million API calls. Let's highball it and assume one million calls.

API Gateway sub-total: 3.50$

## Lambda

 (file API service & virus scanning)

There's a perpetual free-tier for Lambda functions, and it's highly likely that the service could run in the free tier with 50 users pretty much forever given the usage scenario I've described so far.

If our users somehow generated 5000 requests to the Lambdas with a mid-size provisioned Lambda (1.5Gb RAM) and each execution took 30 seconds, the execution cost for Lambda would be 3.08$ USD. 

Lambda sub-total: 3.08$ 

## Cognito

 (Authentication and user management)

The first 50,000 monthly active users in Cognito are free.

Cognito sub-total: 0$

## Total

So the total all-in costs for this solution for our scenario is about 13$ per month. 

If in a given month users upload half as many files, the price would drop by half.

Looking at some of the other options out there, you can get approximately 1 Tb of storage for about 10$ per month per user. That's a pretty good price per user if all your users need 1 Tb of storage, but in scenarios where they only need a few gigabytes it's pretty steep.

1 Tb of storage in S3 is about 20$ per month. Definitely more pricey than some of the cloud storage options out there, on the long run I'd imagine the pay-per-use model of DropBucket is cheaper if you don't need all the fancy features.

# Simplicity

The application can be deployed to your AWS account with a couple config file updates and by running 2 commands. 

That's it. You can now sign-up a million users and store petabytes of data, everything will just work.

The user interface is simple. You upload files, then you can share them with a sharing link (after they've been scanned for viruses).

Files automatically get deleted after 7 days.

