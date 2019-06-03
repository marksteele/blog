---
layout: blog
title: Getting AWS IPs for a given region
date: '2019-06-03T17:02:39-04:00'
cover: /images/aws.png
categories:
  - note-to-self
---
I keep forgetting this...

`curl -s https://ip-ranges.amazonaws.com/ip-ranges.json | jq -r '.prefixes\[] | select(.region=="ca-central-1") | .ip_prefix' | sort`
