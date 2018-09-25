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

* KMS: The AWS key management service. This is where the master encryption key used for protecting all the data lives.
* DynamoDB: This is a serverless NoSQL database that can be automatically scaled, supports at-rest encryption and automatic record expiry.
* Lambda: This service runs functions on demand (FaaS), think micro-containers. This is the code that will perform the encryption/decryption and persist data.
* API Gateway: HTTP endpoint that will serve as the entry-point to the system.

When a secret is sent to self-destruct-o, we use a technique called envelope encryption. Here's the blow-by-blow:

* A data key (random data) is generated in KMS and encrypted using a KMS key (customer master key).
* We generate a random secret identifier for the secret (v4 UUID)
* Encrypt the secret with the plaintext of the data key
* Store the encrypted data key and encrypted payload in DynamoDB (which encrypts the data at rest also), using the secret identifier as the lookup key
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

![null](/images/knowmore.jpg)

Code here: <https://github.com/marksteele/self-destruct-o>

See it in action [here](https://self-destruct-o.control-alt-del.org). 

![null](/images/dicaprio-meme.jpg)

Seriously, you can trust me.

# Update

I've updated the code to support an optional passphrase, here's how it works. 

The following is all run from inside your browser before sending anything to the backend:

When you enter a passphrase in the browser, javascript code in the HTML page uses the [PBKDF2](https://en.wikipedia.org/wiki/PBKDF2) key derivation function to generate an encryption key which is derived from the passphrase. This produces a set of random bytes to use as an encryption key, as well as a salt value (used to prevent dictionary attacks against re-used passphrases).

Next we generate a random initialization vector (IV) or nonce, which insures that the encryption will always produce different outputs even if by some random chance the same key was used.

Finally, using the IV and derived key the secret is encrypted using the AES-256 encryption algorithm (in CBC mode). 

The resulting encrypted value along with the salt and IV are then concatenated together, and sent to the backend API for storage, where it will be encrypted with the data key (and again encrypted at rest in dynamodb). You read that right, the secrets can be encrypted thrice.

Decrypting follows these operations in reverse:
* Backend reads from DynamoDB
* Decrypt the data key with the customer master key
* Decrypt the database record with the data key
* Return the data to browser
* In the browser extract IV/Salt from payload
* Derive the key using the passphrase and salt
* Decrypt using the IV and derived key.

For the paranoid, read the html source. I've also tagged all the javascript/css imported into the page with a subresource integrity tag (https://www.srihash.org/) to make sure that the libraries being used aren't being messed with. Most modern browsers support this.
