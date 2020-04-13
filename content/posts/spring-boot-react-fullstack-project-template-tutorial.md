---
layout: blog
title: Spring Boot/React - FullStack Project Template/Tutorial
date: '2020-04-12T15:58:27-04:00'
cover: /images/didntsayword-mark-1-.gif
categories:
  - tutorial
  - Java
  - Spring Boot
---
Here's a Spring Boot project template/tutorial that I've put together to try to get some best practices codified on how to quickly throw together a service.

We'll cover: setting up REST endpoints, scheduled tasks, thread-pools to parallelize work, React, MySQL, 12-factor app best practices, OAuth2 (google), monitoring, unit testing, code coverage, project mess detection, spotbugs, DevOps and more!

This blog post is going to be a bit of a beast, so before I dig too deep let's set the stage on what we're trying to accomplish. First, we'd like to build a modern front-end and backend system. For the front-end, we'll use React. For the backend, we'll use Java+Spring Boot.

Our fictitious project is called Visitors. We have been tasked to build a form where users can enter an IP address, and we are supposed to save it to a database. We'd also like to see the entries in the database as well.

In this example, we want the front-end to be packaged and deployed inside the backend application.

On the backend side, we'd like some APIs:

* List all IPs for which we don't have country information for.
* Lookup the location of an IP address in real-time.
* Trigger a background task that scans the database immediately and attempts to geo-locate all unknown IP locations.
* An API endpoint to add new IPs
* An API endpoint to view all the entries in the table.

In addition to the APIs, we'd also want the backend to trigger resolved all unknown IP locations on a daily basis.

Furthermore, as the numbers of entries in the database might get large, we'd want the geo-lookups to be done in parallel, several simultaneously.
