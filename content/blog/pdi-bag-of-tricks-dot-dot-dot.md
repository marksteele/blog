+++
title = "pdi bag of tricks..."
date = "2014-01-20"
slug = "2014/01/20/pdi-bag-of-tricks-dot-dot-dot"
Categories = []
+++

After using PDI for a while, you start to encounter some common problems. PDI crashes, databases die, connections get reset, all sorts of 
interesting things can happen in complex systems.

As a general rule, when building PDI jobs that should behave monotonically I always strive to find a way to make a job re-playable and 
idempotent. This can be tricky given an unlimited input set over time. 

Probabilistic data structures to the rescue!

To do this, at work we created a PDI bloom filter step (thanks [Fabio!](https://github.com/FabioBatSilva)). This article will go over how it works and it's use cases.

<!--more-->

I'll attach some screenshots when I get a chance.

Before we begin, grab, compile, and install the [PDI plugin](https://github.com/instaclick/PDI-Plugin-Step-BloomFilter).

Use Case #1: Ensure only-once processing
----------------------------------------

You've got a known sized input set (let's say a few million entries), and want to ensure you only process them once.

Let's suppose that your data lives in a database table that contains a unique primary key (or a composite unique key can be built by a combination of the table fields). 
You might start with a table input step, pass it along to the bloom filter step.

The options you'd want here for the bloom filter step would be 

- In unique fields, the field(s) that you want uniqueness on. This is what is added to the bloomfilter
- Check the single bloomfilter option
- Make sure the always pass the row option is unchecked
- Specify the location you want to save the bloomfilter
- Provide the expected number of elements, and false positive probability settings.
- Make the step transactional

This configuration will create a bloom filter, and for each incoming row check to see if the row unique value is already present in the bloom filter. 

If it is, the row is disgarded. If not, it is added to the bloomfilter and passed along.


Use Case #2: Unique value for a given time range
------------------------------------------------

Suppose you were doing log processing of a clickstream, and wanted to know unique clicks by IP to a given URL for a 24 hour period.

Furthermore, suppose the input stream was

- Timestamp (unix timestamp)
- URL
- IP address

You could send your rows to a bloomfilter step with the following configuration

- In unique fields, URL and IP address
- Set the timestamp field to be the name of your timestamp field in the row
- Make sure the always pass the row option is checked
- Specify the location you want to save the bloomfilter
- Provide the expected number of elements, and false positive probability settings.
- Make the step transactional
- Set the timestamp window size to 60 (1 minute)
- Set the number of lookups to 1440 (number of minutes in a day)

In this scenario, when a new row would enter, the step will take the unix timestamp and figure out what the epoch minute for that timestamp is 
(timestamp/1440). It will then load up the bloomfilter for that minute (creating a new empty filter as necessary).

The bloomfilter will then check to see if the IP/URL combo has been seen before. If not, it will add a flag to each row indicating if it's unique or not.

Just as a caveat, the step tries to be smart by caching filters, however you'll need pretty fast I/O if you're working rowset is hitting a large variety of the time periods.


Use Case #3: Email blacklist
----------------------------

Suppose you've got an email marketing system and maintain a blacklist of addresses that you do not wish to send emails out to. 
Further suppose that the blacklist is a few million entries. Searching through an array might be slow/CPU intensive, however a 
bloomfilter is a great fit for this.

In this case we'd want two transformations. The first, would train the bloomfilter with the blacklist elements (could rebuild it daily) 
using the 'single bloomfilter' options, and the second would use the 'check only' option specifying the same path.


Gotchas
-------

You need to make sure you don't have multiple jobs running simultaneously hitting the same bloom filter. 

This will likely break in fun and interestingly hard to troubleshoot ways. Same applies to trying to use the 
same bloomfilter more than once in a transformation (unless you're accessing it read-only using the check-only option). 
Again, you'd want to make sure nobody else was trying to write to it while you were reading it.


Future ideas
------------

It's possible to merge and intersect bloom filters. Might be a fun experiment to have that ability baked in.
