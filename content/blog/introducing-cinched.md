+++
title = "introducing cinched"
date = "2016-02-04"
slug = "2016/02/04/introducing-cinched"
Categories = []
+++

Given the frequency of rather embarrassing data breaches recently, I've had the opportunity to spend some time thinking about how to help developers protect the data they are storing.

Getting encryption right is hard, and designing cryptographic applications is not something web developers typically have lots of experience with. To (hopefully) help bridge this knowledge gap, I've written a microservice which provides encryption and key management services.

It's a cinch to use, and will keep your data cinched. It's also called Cinched. Get it?

I've released version 0.0.1, a preview to gather feedback. Some of the key features are:

* An easy to use RESTful API
* Clustered highly available service
* Sane strong encryption defaults
* All stored data is encrypted at rest
* OS level mandatory access control using SELinux type enforcement
* Tamper resistant and encrypted audit logging

<!-- more -->

[Official documentation](https://github.com/marksteele/cinched)

# Overview

## Cluster

Cinched is built using Erlang/OTP and leverages Erlang's clustering capabilities.

Nodes in a Cinched cluster establish a mesh network, with inter-node communication encrypted via TLS1.2.

![Mesh](https://upload.wikimedia.org/wikipedia/commons/3/3c/NetworkTopology-FullyConnected.png)

Node peer authentication uses a shared secret, and the mesh only accepts peers which reside on the same subnet.

Cinched clusters are master-less, that is that each node can accept client requests, there are no special roles.

## Encryption

This service uses authenticated symmetrical encryption implemented via Erlang bindings to libsodium. The algorithms used are Salsa20 for encryption/decryption, and Poly1305 for message authentication.

## Certificates

Cinched relies on public key infrastructure (PKI) heavily for both securing intra-cluster communication, as well as authorizing client API calls.

Before using Cinched, you'll need to have access to a certificate authority (CA) to generate certificates as well as provide online certificate status protocol (OCSP) and certificate revocation list (CRL) distribution points.

For running a CA on CentOS 7, I've put together a very quick guide [here](http://www.control-alt-del.org/blog/2016/01/30/setting-up-a-ca/).

The certificates you'll need (all PEM encoded):

* The CA certificate
* The node certificate(s) (used to secure intra-cluster communication as well as client API calls)
* Client certificates (used by API clients).

Cinched performs certificate validation on all incoming requests, and both the server and client certificates must be signed by the same certificate authority.

It is crucial to understand that certificate validation is the only form of authentication that Cinched supports, so make sure you keep your private keys safe (this in itself is a hard problem)

On Cinched nodes, certificate private keys are protected with SELinux policy and regular DAC controls. With policy locked down, even root can't get at any of the protected information.

## Encryption Keys

All keys in Cinched are randomly generated. There are four different types of keys that serve different purposes.

Visually:

```
 Operator key -- encrypts/decrypts --> Service key

 Service key (stored) -- encrypts/decrypts --> Master keys

 Master keys (stored) -- encrypts/decrypts --> Data keys

 Data keys -- encrypt/decrypt --> client supplied data.

```

Before diving into the details, a quick analogy on how these keys work.

Let's imagine we're running a bank that provides safety deposit boxes.

We're going to want to make sure at least two people are involved in opening up the bank in the morning, to make sure there's no collusion.
So we'll give one set of bank employees the keys to open the front door, and another set of employees the pin code to disable the alarm.
In order to start the day, we'll need one person from each group available to open up for the day. (these are the operator key shards described below).

Next, we have a vault room. The vault room is rigged to sensors on the door as well as to the alarm system. When both the door is open and the alarm system is disarmed, the vault door can be opened. (this is the service key, described below)

In order to enter the vault room, our customers must present a valid ID card issued by our security department (the client certificate).

Our bank is special. Our safety deposit boxes are built on demand. In our bank, we employ a master locksmith who helps us build our lock-boxes.

On a daily basis, the locksmith builds one master key that we store inside the vault.

The locks on our boxes are designed to require two keys to be unlocked. One is the master key for the date at which they are built, and the other is a key that is unique to the box. (the data key).

When a customer orders a box, we build it and give him the only copy of the box-specific key.

Both the master key AND the box-specific key are required to open a box. The box, keys, and locks are made in such a clever way that there is currently no known way of opening the box without being inside the vault with the proper keys.

Since our boxes are so good we do not store the box. We want to save space, so we give the boxes to our customers. If they want to open it, they must come inside the vault with their key and box.

### Operator key

The operator key (`OK`), is created during the initialization of the cluster. It is split into a set of shards using a cryptographic technique known as [Shamir secret sharing](https://en.wikipedia.org/wiki/Shamir%27s_Secret_Sharing).

By default ten shards are created from the operator key, with a threshold of three shards being required to reconstruct the original key. So if you distribute 2 shards to 5 people, you ensure that no single person can start the service, plus you get some redundancy to make sure there's always enough people around to unlock things.

This technique prevents a single rogue operator from possessing sufficient information to compromise the system.

There are two scenarios where the `OK` is used:

* Cluster initialization: When a cluster is initialized, the `OK` is used to encrypt the service key (`SK`), more on that later. During initialization, the key shards are output to the terminal of the operator initializing the cluster.
* Node startup: Whenever an operator wants to bring a cluster node online, sufficient shards must be provided by the shard custodians to reconstruct the `OK` in order to decrypt the service key (`SK`).

After cluster initialization or node startup, it is discarded from memory.

It is possible to rotate the `OK` and generate new sets of key shards.

### Service key

The service key (`SK`) is created during the initialization of the cluster.

It is encrypted using the `OK` and persisted to the storage subsystem, and held in RAM unencrypted.

This key is used to encrypt/decrypt master keys (`MK`) as well as being the seed for the rotating encryption key of the audit logs.

It is currently not possible to rotate the `SK`.

### Master keys

Master keys (`MK`) are generated once per day and are associated with a crypto-period, which is an integer representing the number of days since the start of the UNIX epoch.

The keys are generated on demand, and are used to encrypt/decrypt data keys.

When generated, master keys are stored encrypted with the `SK` at rest in the storage subsystem.

They are also held unencrypted in memory in a LRU cache with a default lifetime of 1 day.

### Data keys

Data keys (`DK`) are generated either via the key API or whenever an encryption API call does not provide a data key via the HTTP headers.

The key is encrypted with the `MK` for the day it is generated, and returned to the calling application.

`DK`'s are not stored in Cinched, it is assumed that the calling application will store the `DK`s.

## Storage

Master keys are identified by the crypto-period for which they are generated, and the key name space is mapped to a consistent hash ring.

The hash ring is split into a number of partitions, and each partition is assigned to a group of cluster nodes. This grouping of nodes represents a partition's replication group.

Replication in Cinched is accomplished via a multi-paxos consensus algorithm, ensuring immediate consistency in the cluster.

The overall storage system is therefore resilient to node failures, as long as a quorum of replicas remain online the service can continue to operate.

Increasing the replication factor makes the system more fault tolerant at the expense of slower writes, which happen infrequently (once per day at most during master key generation).

# API

The official documentation has a more comprehensive set of examples and covers more functionality (document API, blob API, etc...), however here's a quick example of using Cinched.

## Generate a data key

```
curl -s \
-X POST --cert fixtures/cert.pem --key fixtures/key.pem \
--cacert fixtures/cacert.pem --tlsv1.2  \
https://dev1.control-alt-del.org:55443/key/data-key | jq
{
  "dataKey": "g2gEZA9rZXliAABBk<snip>XA4WglqFGSzDA2H3FI2uXYu0bqGthAQ==",
  "cryptoPeriod": 16786
}
```

Store this information somewhere, we'll need it to decrypt the data later on.

## Encrypt something

Let's imagine we have a JSON document that looks like this:
```
{
 "foo": "bar",
 "favoriteBar":"baz's emporium of fine spirits"
}
```

And let's further suppose we want to encrypt the field 'favoriteBar', as that's confidential.

```
curl -s \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json' \
  -H "x-cinched-data-key: g2gEZA9rZXliAABBk<snip>XA4WglqFGSzDA2H3FI2uXYu0bqGthAQ==" \
  -H "x-cinched-metadata: foobar" \
  -X POST -d "{\"foo\":\"bar\",\"favoriteBar\":\"baz's emporium of fine spirits\"}" \
  --cert fixtures/cert.pem --key fixtures/key.pem \
  --cacert fixtures/cacert.pem \
  --tlsv1.2 \
  https://dev1.control-alt-del.org:55443/doc/encrypt?fields=\(favoriteBar\)
{"foo": "bar","favoriteBar": "A9rZXliAABBk<snip>XA4WglqFGSzD"}
```

Success! Now we can safely store this document somewhere. Should a nefarious ne'er do well steal it, they won't know the location of our favorite dive.

## Decrypting data

To read the data back:

```
curl -s \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json' \
  -H "x-cinched-data-key: g2gEZA9rZXliAABBk<snip>XA4WglqFGSzDA2H3FI2uXYu0bqGthAQ==" \
  -H "x-cinched-metadata: foobar" \
  -X POST -d "{\"foo\":\"bar\",\"bar\":\"A9rZXliAABBk<snip>XA4WglqFGSzD\"}" \
  --cert fixtures/cert.pem --key fixtures/key.pem \
  --cacert fixtures/cacert.pem \
  --tlsv1.2 \
  https://dev1.control-alt-del.org:55443/doc/decrypt?fields=\(favoriteBar\)
{"foo": "bar","favoriteBar": "baz's emporium of fine spirits"}
```

Well done! Now we know where to go grab a brew.

# Notes for using Cinched

If possible, establish long-lived persistent connections to amortize the cost of the TCP/TLS session negotiation.

Store the data keys separately from the encrypted data (with different access credentials).

If possible, keep your certificate private keys encrypted with a passphrase, or leverage a TPM chip. Other ideas for doing this are to provide the decryption keys out of band to the service loading the key (eg: passphrase prompt, environment variables). If all else fails, it's not that hard to write some SELinux policy to lock down filesystem access to a confined domain. Look at Cinched's policy if you need a sample.

# Attack vectors

Assuming the system is configured as recommended, I can't see too many ways to compromise the security of the system. That's not to say that it's proof against everything...

The SELinux policy should defend against several attack vectors such as reading values from RAM as well as protect the filesystem.

That being said, once root level compromise happens on a system, things get very very difficult to defend against comprehensively without introducing a ton of operator friction.

Some recommendations:

* Remove simple password based authentication if you can (2FA, Kerberos, key-based are all superior).
* Don't allow direct root-level ssh
* Segment your network, keep Cinched in it's own subnet
* Have host-based firewalls on all Cinched nodes, with DENY rules by default for both ingress and egress traffic. Explicitly allow only what you need. Even better if you can make this a hardware firewall.
* Lock down the policy when prompted by the cinched setup wizard if this is a production deployment.
* Compile a custom kernel that does not support dynamic loading of kernel modules and lock that down when prompted by the Cinched setup wizard.
* You really really need to be on top of properly managing your PKI. If you can properly control who has access to your certificates and private keys, it will go a long way to protecting the integrity of Cinched as the certificates control access to the service.

# Future direction

I'd really like to start leveraging TPM with the Linux integrity subsystem for key storage and system attestation. I could foresee a future version of Cinched leveraging TPM across the stack for both intra-cluster integrity as well as client side integrity. It should be possible to build a system that is proof against stealing the secret keys using this type of setup.

One of the hard problems in getting this all to work is the management burden of maintaining the integrity chain in a fast-moving environment. Will require experimentation...

# In closing...

I enjoy canceled vacations, crazy unpaid overtime doing incident response, calls in the middle of the night from anxious execs, fending off reporters and so on just as much as the next sec/dev/ops guys.... Oh wait, not so much.

Go forth! Encrypt that sensitive data! Just make sure you're doing it right. Customer trust is a currency that's hard to acquire and even harder to regain once squandered.
