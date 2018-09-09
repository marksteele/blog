---
layout: blog
title: One-time password sharing... securely!
date: '2018-09-08T21:56:19-04:00'
cover: /images/self-destruct.jpg
categories:
  - Security
  - Encryption
  - AWS
  - Lambda
  - DynamoDB
---
![null](/images/self-destruct.jpg)

Need to send a friend a password for something? Don't trust that the man might be reading your emails and SMS messages? Here's how to setup your own service to share secrets.

<!--more-->

Password management tools (like LastPass) are a great way to store your passwords, but sometimes you want to send someone a password. Sending it by email is not a good idea, neither is Slack, SMS, IRC or pigeon carrier.

We could use something like PGP/GPG, but let's face it that is a bit of a pain in the ass since most non-nerds won't understand how to get it setup without lots of hand-holding.

Setting up a website for doing this has been done before (eg: https://onetimesecret.com/), however let's all put on our paranoia tin-foil hat for a moment and suppose that since we aren't running that service ourselves, it could be a front for hackers to receive tons of passwords. 

Some options present themselves:

1. Checkout their git repository (https://github.com/onetimesecret/onetimesecret), sift through the code, and so on to make sure there aren't any bugs. Then we could spin up a server and run the code. But yuck. It's ruby, it'll cost you at least 1 EC2 instance and a redis server. And worst of all... ruby. Also, you'll need to apply patches to your server, make sure you rotate your encryption keys often, etc....
2. Write something simpler that relies more on AWS infrastructure, and will be able to run in the AWS free tier forever. That way everyone can have their own version of this service with very little friction.

Spoiler, I went with option #2. I call it self-destruct-o, and here's how it works.

The service is based on several pieces of AWS infrastructure:

* KMS: The AWS key management service is where the master encryption key used for protecting all the data lives.
* DynamoDB: This is a serverless database that can be automatically scaled, supports at-rest encryption and automatic record expiry.
* Lambda: This service runs functions on demand (FaaS), think micro-containers. This is the code that will perform the encryption/decryption and persist data.
* API Gateway: HTTP endpoint that will serve as the entry-point to the system.

When a secret is sent to self-destruct-o, a data key is generated from KMS and encrypted. The code will then:

* Generate a random identifier for the secret (v4 UUID)
* Encrypt the secret with the data key
* Store the encrypted data key and encrypted payload in DynamoDB (with encrypts the data at rest also), using the random identifier as the key
* Return the identifier back so that we can find the secret later on.

It's like encrypt-ception!

![null](/images/inception2.jpg)

To retrieve a secret

* We pass in the identifier
* Load the record from DynamoDB
* Decrypt the data key by calling KMS
* Decrypt the secret with the data key
* Delete the data from DynamoDB
* Return the secret back to the caller.

![](/images/knowmore.jpg)

<https://github.com/marksteele/self-destruct-o>
